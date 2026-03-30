# Evo-Shader-Compiler: MANIFEST Extraction Pipeline

End-to-end documentation of how `.evo_mod` files are parsed into `RonModManifest`.

**Source directory**: `libs/evo-shader-compiler/src/manifest_extract/`

---

## Table of Contents

1. [Pipeline Overview](#1-pipeline-overview)
2. [Entry Points](#2-entry-points)
3. [Preprocessing: Implicit Sentinels](#3-preprocessing-implicit-sentinels)
4. [Two-Pass Pre-Scan](#4-two-pass-pre-scan)
5. [Main Extraction: `extract_manifest_from_ast`](#5-main-extraction-extract_manifest_from_ast)
6. [Attribute Parser Registry](#6-attribute-parser-registry)
7. [Complete Attribute Reference](#7-complete-attribute-reference)
8. [Include System (`#[evo_include]`)](#8-include-system-evo_include)
9. [Identifier Resolution](#9-identifier-resolution)
10. [Post-Parse Transforms](#10-post-parse-transforms)
11. [Validation](#11-validation)
12. [Patch System (Phase 5)](#12-patch-system-phase-5)
13. [Data Structures](#13-data-structures)
14. [Thread-Local State](#14-thread-local-state)

---

## 1. Pipeline Overview

```
.evo_mod file on disk
    |
    v
[discover_mods] walks mods_dir, finds mod_def.evo_mod
    |
    v
[Phase 1.5] prescan_mod_symbols() -- lightweight pre-scan for mod names + symbols
    |
    v
[prepopulate_known_namespaces] -- seed thread-local KNOWN_MOD_NAMESPACES
    |
    v
[extract_manifest_from_path_and_source]
    |
    +-- preprocess_implicit_sentinels() -- insert `const _: () = ();` after bare attrs
    +-- syn::parse_file() -- parse into syn::File AST (cached)
    +-- scan `use` statements -- populate USE_IMPORT_MAP, CROSS_MOD_IMPORT_MAP
    +-- Phase 0a: extract mod name + settings pre-scan
    +-- Phase 0b: build IDENTIFIER_MAP (Zero-String Typing)
    +-- Phase 1: parse #![evo_mod(...)] inner attributes
    +-- Phase 2: dispatch all #[evo_*] outer attributes through AttributeParserRegistry
    +-- Phase 3: process #[evo_include] -- recursive sub-file parsing + merge_from
    +-- Phase 4: post-parse transforms (extends, boss inheritance, inline conversion)
    +-- Phase 5: extract patches (evo_hook, evo_overridable, evo_override)
    +-- Phase 6: validation (name uniqueness, dimension worldgen, cross-references)
    |
    v
ExtractedManifest { ron: RonModManifest, settings, parse_state, warnings, patches }
    |
    v
[engine-runner] convert_extracted_to_ron() -> used by ModLoader
```

---

## 2. Entry Points

### Mod Discovery (engine-runner side)

**File**: `apps/engine-runner/src/mod_loader/discover.rs`

`ModLoader::discover_mods(mods_dir)` uses `walkdir` to find directories containing
`mod_def.evo_mod` (or legacy `mod.toml`). For each mod:

1. **Phase 1**: Collect `(mod_name, path)` pairs. Directory name = mod_name.
2. **Phase 1.5**: `prescan_mod_symbols()` on each `mod_def.evo_mod` to collect all mod
   names for cross-mod validation.
3. **Phase 2** (parallel via rayon): For each mod, call
   `extract_manifest_from_path_and_source()`, with a `ManifestCache` layer that skips
   parsing on cache hit (keyed by file content hash).

### Compiler-Side Entry Points

**File**: `manifest_extract/mod.rs`

| Function | Purpose |
|----------|---------|
| `extract_manifest(source: &str)` | Parse from in-memory source string |
| `extract_manifest_from_path(path, include_dirs)` | Parse from file path, reads file |
| `extract_manifest_from_path_and_source(path, source, include_dirs)` | Parse from path + pre-read source (avoids double read) |
| `prescan_mod_symbols(path)` | Lightweight pre-scan: returns `PreScanResult { mod_name, symbols }` |
| `prepopulate_known_namespaces(names)` | Seed thread-local with all mod names for cross-mod validation |

All three `extract_manifest*` functions converge to the private
`extract_manifest_from_ast(path, source)`.

---

## 3. Preprocessing: Implicit Sentinels

**Function**: `preprocess_implicit_sentinels(source) -> String`

`.evo_mod` files use bare `#[evo_*(...)]` attributes without trailing Rust items.
`syn::parse_file` requires every outer attribute to be attached to an item. The
preprocessor inserts `const _: () = ();` after any `#[evo_*]` attribute that is not
followed by a Rust item keyword (`const`, `fn`, `pub`, `struct`, `enum`, `use`, etc.)
or another attribute.

Correctly skips: line comments (`//`), block comments (`/* */`), string literals (`"..."`).

---

## 4. Two-Pass Pre-Scan

**Function**: `prescan_mod_symbols(path) -> PreScanResult`

A lightweight pass that parses the AST but does NOT deserialize attributes via serde.
Returns:

- `mod_name`: extracted from `#![evo_mod(name = "...")]`
- `symbols`: `Vec<(attr_type, declaration_name)>` pairs, e.g. `("evo_block", "STONE")`

Recursively scans `#[evo_include]` paths in the file.

Used by `discover_mods` to build a global symbol table BEFORE full manifest extraction,
so that cross-mod namespace validation works without requiring explicit `#[evo_dependencies]`.

---

## 5. Main Extraction: `extract_manifest_from_ast`

### Phase 0a: Pre-scan mod name + settings

Scans inner attributes for `#![evo_mod(name = "...")]` to populate `KNOWN_MOD_NAMESPACES`
early. Scans outer attributes for `#[evo_settings(...)]` to populate dynamic overrides.

### Phase 0b: Build IDENTIFIER_MAP (Zero-String Typing)

Scans all items and their `#[evo_*]` attributes to build a mapping from Rust identifiers
to registered names. For example, if a struct `Stone` has `#[evo_block(name = "STONE")]`,
then `Stone` -> `"STONE"` is registered. This enables later attributes to reference
`Stone` instead of `"STONE"`.

Supported types for pre-scan: `evo_block`, `evo_entity`, `evo_flag`, `evo_game_state`,
`evo_geometry`, `evo_loot_table`, `evo_batch`.

### Phase 1: Inner attributes

Parses `#![evo_mod(...)]` via `parse_evo_mod_attr`. Registers the mod's own namespace.
If `enabled = false`, returns early.

### Phase 2: Outer attributes (registry-dispatched)

Builds the full registry:
```rust
let attr_registry = AttributeParserRegistry::with_core_parsers().with_game_parsers();
```

For each item in the file, for each attribute:
1. Extract the attribute identifier name.
2. Call `attr_registry.dispatch(name, attr, manifest)`.
3. If dispatch returns `Ok(false)` and name starts with `evo_`, error: unknown attribute.
4. If dispatch returns `Err(e)`, propagate with file + span location.

### Phase 3: Include processing

See [Section 8](#8-include-system-evo_include).

### Phase 4: Post-parse transforms

See [Section 10](#10-post-parse-transforms).

### Phase 5: Patch extraction

Calls `parse_patches::extract_patches(&file, source_path)`.

### Phase 6: Validation + finalization

- Merge auto-inferred dependencies from `INFERRED_DEPENDENCIES` thread-local.
- Collect parse warnings.
- Run `validate_cross_references`.

---

## 6. Attribute Parser Registry

**File**: `manifest_extract/attribute_registry.rs`

### Structure

```rust
pub struct AttributeParserRegistry {
    parsers: HashMap<String, AttributeParserFn>,
}

pub type AttributeParserFn =
    fn(attr: &syn::Attribute, manifest: &mut ExtractedManifest) -> Result<()>;
```

### Construction

```rust
AttributeParserRegistry::with_core_parsers()  // 40 core parsers
    .with_game_parsers()                        // +48 game parsers = 88 total
```

### Dispatch

`dispatch(attr_name, attr, manifest) -> Result<bool>`:
- `Ok(true)`: attribute handled
- `Ok(false)`: no parser registered (caller checks if `evo_*` prefix -> error)
- `Err(...)`: parse error

### Deserialization: `serde_attr<T>`

**File**: `manifest_extract/mod.rs` (line ~2181)

Most parsers use the generic `serde_attr<T>(attr_name, attr, manifest) -> Result<T>`
helper, which:

1. Extracts the token stream from the attribute's parenthesized content.
2. Pre-processes tokens:
   - `resolve_cross_mod_string_arrays` -- resolves `mod_blocks::STONE` paths in arrays
   - `normalize_creature_geometry` -- normalizes `Creature(...)` syntax
   - `normalize_path_values` -- converts bare cross-mod paths to string literals
   - `quote_bare_ident_string_values` -- quotes bare identifiers for String fields
   - `insert_missing_equals` -- adds `= true` for bare boolean flags
   - `resolve_flag_array` -- resolves `flags = [FLAG_GPU_ACTIVE, ...]` to bitmasks
   - `parens_to_braces` -- converts `(...)` to `{...}` for serde_tokenstream
3. Calls `serde_tokenstream::from_tokenstream::<T>()`.

The target struct IS the parser: `#[serde(deny_unknown_fields)]` rejects typos,
`#[serde(default)]` handles optional fields.

---

## 7. Complete Attribute Reference

### Core Attributes (40 parsers) -- `parse_core.rs`

| # | Attribute | Parser | Deserializes Into | Stored In |
|---|-----------|--------|-------------------|-----------|
| 1 | `evo_mod` | `parse_evo_mod_attr` (custom) | Sets `manifest.ron.core.{name,version,description,...}` | `ron.core.*` (multiple fields) |
| 2 | `evo_settings` | `parse_evo_settings_attr` (custom) | `EvoSettings` (`shared::manifest::common`) | `manifest.settings` |
| 3 | `evo_service` | `parse_evo_service_attr` (custom -> `ServiceDefFlat`) | `ServiceDef` (`shared::manifest::services`) | `ron.core.services` |
| 4 | `evo_gpu_pipeline` | `parse_evo_gpu_pipeline_attr` (custom) | `GpuPipelineDef` | `ron.gpu_pipelines` |
| 5 | `evo_compute` | `parse_evo_compute_attr` (custom) | `GpuPipelineDef` | `ron.gpu_pipelines` |
| 6 | `evo_pipeline` | `parse_evo_pipeline_attr` (custom) | `GpuPipelineDef` | `ron.gpu_pipelines` |
| 7 | `evo_dispatch` | `parse_evo_dispatch_attr` (custom) | `DispatchDef` | `ron.dispatches` |
| 8 | `evo_buffer` | `serde_attr` | `BufferDef` | `ron.buffers` |
| 9 | `evo_binding` | `serde_attr` | `BindingDef` | `ron.bindings` |
| 10 | `evo_instance_services` | `parse_evo_instance_services_attr` (custom) | Populates `instance_services` | `ron.core.instance_services` |
| 11 | `evo_game_state` | `serde_attr` | `GameStateDef` | `ron.game_states` |
| 12 | `evo_flag` | `parse_evo_flag_attr` (custom) | `FlagDef` | `ron.flags` |
| 13 | `evo_faction` | `parse_evo_faction_attr` (custom) | `FactionDef` | `ron.factions` |
| 14 | `evo_core_field` | `serde_attr` | `CoreFieldDef` | `ron.core_fields` |
| 15 | `evo_ext_field` | `serde_attr` | `ExtFieldDef` | `ron.ext_fields` |
| 16 | `evo_entity` | `parse_evo_entity_attr` (custom) | `EntityDef` | `ron.entities` |
| 17 | `evo_entity_schema` | `serde_attr` | `EntitySchemaDef` | `ron.entity_schemas` |
| 18 | `evo_geometry` | `parse_evo_geometry_attr` (custom) | `GeometryDef` | `ron.geometries` |
| 19 | `evo_event` | `serde_attr` | `EventDef` | `ron.events` |
| 20 | `evo_keybind` | `serde_attr` | `KeybindDef` | `ron.keybinds` |
| 21 | `evo_action_keybind` | `serde_attr` | `ActionKeybindDef` | `ron.action_keybinds` |
| 22 | `evo_input_mapping` | `serde_attr` | `InputMappingDef` | `ron.input_mappings` |
| 23 | `evo_screen` | `serde_attr` | `ScreenDef` | `ron.screens` |
| 24 | `evo_scenario` | `serde_attr` | `QaScenarioDef` | `ron.scenarios` |
| 25 | `evo_persistence` | `serde_attr` | `PersistenceDef` | `ron.persistence` (Option) |
| 26 | `evo_camera_follow` | `serde_attr` | `CameraFollowDef` | `ron.camera_follow` (Option) |
| 27 | `evo_fps_input_capture` | `serde_attr` | `FpsInputCaptureDef` | `ron.fps_input` (Option) |
| 28 | `evo_protocol` | `serde_attr` | `ProtocolDef` | `ron.protocols` |
| 29 | `evo_provide` | `serde_attr` | `ProvideDef` | `ron.provides` |
| 30 | `evo_emitter` | `serde_attr` | `EmitterDef` | `ron.emitters` |
| 31 | `evo_batch` | `parse_evo_batch_attr` (custom) | Expands inline array declarations into individual defs | Various `ron.*` fields |
| 32 | `evo_data` | `parse_evo_data_attr` (custom) | Generic extensible data bag | Varies by data type |
| 33 | `evo_compute_system` | `parse_evo_compute_system_attr` (custom) | `ComputeSystemDef` | `ron.core.compute_systems` |
| 34 | `evo_temp_buffer_pool` | `serde_attr` | `TempBufferPoolDef` | `ron.temp_buffer_pools` |
| 35 | `evo_dispatch_sequence` | `serde_attr` | `DispatchSequenceDef` | `ron.dispatch_sequences` |
| 36 | `evo_entity_service` | `serde_attr` | `EntityServiceDef` (`shared::manifest::entity_service`) | `ron.core.entity_services` |
| 37 | `evo_shader_inject` | custom (`parse_shader_transforms.rs`) | `ShaderTransformDef` (kind=Inject) | `ron.shader_transforms` |
| 38 | `evo_shader_replace` | custom (`parse_shader_transforms.rs`) | `ShaderTransformDef` (kind=Replace) | `ron.shader_transforms` |
| 39 | `evo_shader_add_function` | custom (`parse_shader_transforms.rs`) | `ShaderTransformDef` (kind=AddFunction) | `ron.shader_transforms` |
| 40 | `evo_include` | no-op (handled manually in extraction loop) | N/A | N/A |

### Game Attributes (48 parsers) -- `parse_game.rs`

| # | Attribute | Parser | Deserializes Into | Stored In |
|---|-----------|--------|-------------------|-----------|
| 1 | `evo_block` | `parse_evo_block_attr` (custom) | `BlockDef` | `ron.game.blocks` |
| 2 | `evo_recipe` | `serde_attr` | `RecipeDef` | `ron.game.recipes` |
| 3 | `evo_block_interaction` | `serde_attr` | `BlockInteractionDef` | `ron.game.block_interactions` |
| 4 | `evo_machine_recipe` | `parse_evo_machine_recipe_attr` (custom) | `MachineRecipeDef` | `ron.game.machine_recipes` |
| 5 | `evo_block_ticker` | `serde_attr` | `BlockTickerDef` (`shared::manifest::world`) | `ron.game.block_tickers` |
| 6 | `evo_breed_rule` | `serde_attr` | `BreedRuleDef` | `ron.game.breed_rules` |
| 7 | `evo_sun_cycle` | `serde_attr` | `SunCycleDef` | `ron.game.sun_cycle` (Option) |
| 8 | `evo_weather` | `serde_attr` | `WeatherDef` | `ron.game.weather` (Option) |
| 9 | `evo_dimension` | `serde_attr` | `DimensionDef` | `ron.game.dimensions` |
| 10 | `evo_worldgen_service` | `serde_attr` | `WorldGenServiceDef` (`shared::manifest::world`) | `ron.game.worldgen_services` |
| 11 | `evo_prefab` | `serde_attr` | `PrefabDef` | `ron.game.prefabs` |
| 12 | `evo_biome` | `serde_attr` | `BiomeDef` | `ron.game.biomes` |
| 13 | `evo_damage_formula` | `serde_attr` | `DamageFormulaDef` (`shared::manifest::behavior`) | `ron.game.damage_formulas` + auto-registers a `ServiceDef` in `ron.core.services` |
| 14 | `evo_ore` | `serde_attr` | `OreDef` | `ron.game.ores` |
| 15 | `evo_portal` | `serde_attr` | `PortalDef` (auto-namespaces `target_dimension`) | `ron.game.portals` |
| 16 | `evo_creature` | `serde_attr` | `CreatureDef` | `ron.game.creatures` |
| 17 | `evo_creature_definition` | `serde_attr` (custom post-processing) | `CreatureDefinition` | `ron.game.creature_definitions` |
| 18 | `evo_spawn_defaults` | `serde_attr` | `SpawnRule` | `manifest.parse_state.spawn_defaults` (not in final manifest directly) |
| 19 | `evo_entity_interaction` | `serde_attr` + flag mask resolution | `EntityInteractionDef` | `ron.game.entity_interaction` (Option) |
| 20 | `evo_mob_attack` | `serde_attr` + flag mask resolution | `MobAttackDef` | `ron.game.mob_attack` (Option) |
| 21 | `evo_loot_table` | `parse_evo_loot_table_attr` (custom) | `LootTableDef` | `ron.game.loot_tables` |
| 22 | `evo_audio` | `serde_attr` | `AudioDef` | `ron.game.audio` (Option) |
| 23 | `evo_raycast_chain` | `serde_attr` | `RaycastChainDef` | `ron.game.raycast_chain` (Option) |
| 24 | `evo_chunk_pipeline` | `serde_attr` | `ChunkPipelineDef` | `ron.game.chunk_pipeline` (Option) |
| 25 | `evo_inventory_interaction` | `serde_attr` | `InventoryInteractionDef` | `ron.game.inventory_interaction` (Option) |
| 26 | `evo_give_item` | `serde_attr` + `resolve_identifier` | `GiveItemDef` | `ron.game.give_items` |
| 27 | `evo_worldspace_ui` | `serde_attr` | `WorldSpaceUiDef` | `ron.game.worldspace_ui` (Option) |
| 28 | `evo_tick` | `serde_attr` | `EvoTickDef` | `ron.core.evo_ticks` |
| 29 | `evo_selector` | `parse_evo_selector_attr` (custom) | `EvoSelectorDef` | `ron.game.evo_selectors` |
| 30 | `evo_anim` | `serde_attr` | `EvoAnimDef` | `ron.game.evo_anims` |
| 31 | `evo_session_flag` | `serde_attr` | `SessionFlagDef` | `ron.game.session_flags` |
| 32 | `evo_weapon` | `serde_attr` + `resolve_identifier` | `WeaponDef` | `ron.game.weapons` |
| 33 | `evo_weapon_material` | `serde_attr` | `WeaponMaterialDef` | `ron.game.weapon_materials` |
| 34 | `evo_loot_table_patch` | `serde_attr` | `LootTablePatchDef` | `ron.game.loot_table_patches` |
| 35 | `evo_armor` | `serde_attr` + `resolve_identifier` | `ArmorDef` | `ron.game.armor` |
| 36 | `evo_armor_tier` | `serde_attr` + expansion | `ArmorTierDef` -> 4x `ArmorDef` | `ron.game.armor` (4 entries) |
| 37 | `evo_tool` | `serde_attr` + `resolve_identifier` | `ToolDef` | `ron.game.tools` |
| 38 | `evo_block_hardness` | `serde_attr` | `BlockHardnessDef` | `ron.game.block_hardness` |
| 39 | `evo_mining_config` | `serde_attr` | `MiningConfigDef` | `ron.game.mining_config` (Option) |
| 40 | `evo_boss` | `serde_attr` | `BossDef` | `ron.game.bosses` |
| 41 | `evo_projectile` | `serde_attr` | `ProjectileDef` | `ron.game.projectiles` |
| 42 | `evo_chat` | `serde_attr` | `ChatDef` | `ron.game.chat` (Option) |
| 43 | `evo_achievements` | `serde_attr` | `AchievementsDef` | `ron.game.achievements` (Option) |
| 44 | `evo_music_playlist` | `serde_attr` | `MusicPlaylistDef` | `ron.game.music_playlists` |
| 45 | `evo_screen_theme` | `serde_attr` | `ScreenThemeDef` (compiler-local) | `manifest.parse_state.screen_theme` |
| 46 | `evo_creature_from_primitive` | `serde_attr` + `expand_primitive_spec` | `CreaturePrimitiveSpec` -> `CreatureDefinition` | `ron.game.creature_definitions` |
| 47 | `evo_synth` | `serde_attr` | `SynthDef` (`shared::manifest::audio`) | `ron.game.synth_definitions` |
| 48 | `evo_sequence` | `serde_attr` | `SequenceDef` (`shared::manifest::audio`) | `ron.game.sequences` |

### Shader Transform Attributes (registered via `parse_shader_transforms.rs`)

These are counted as part of the 40 core parsers (entries 37-39 above).

| Attribute | Intermediate Struct | Final Struct | Stored In |
|-----------|-------------------|-------------|-----------|
| `evo_shader_inject` | `InjectAttr` | `ShaderTransformDef { kind: Inject }` | `ron.shader_transforms` |
| `evo_shader_replace` | `ReplaceAttr` | `ShaderTransformDef { kind: Replace }` | `ron.shader_transforms` |
| `evo_shader_add_function` | `AddFunctionAttr` | `ShaderTransformDef { kind: AddFunction }` | `ron.shader_transforms` |

### Patch Attributes (handled outside the registry)

These are extracted by `parse_patches::extract_patches()` in a separate pass over the AST,
NOT through the `AttributeParserRegistry`.

| Attribute | Struct | Stored In |
|-----------|--------|-----------|
| `evo_hook("name")` | `HookPoint` | `manifest.patches.hook_points` |
| `evo_overridable` | `OverridableConst` | `manifest.patches.overridable_consts` |
| `evo_override("mod::TARGET")` | `ValueOverride` | `manifest.patches.value_overrides` |

---

## 8. Include System (`#[evo_include]`)

### Syntax

```rust
#[evo_include("relative/path/creatures.evo_mod")]
#[evo_include("creatures.evo_mod", "blocks.evo_mod")]  // multiple files
```

### Processing Flow

1. After Phase 2 (all outer attributes dispatched), collect all `#[evo_include]` paths.
2. Resolve each path relative to the parent file's directory.
3. For each include, call `extract_include_file(path, parent_settings)`:
   - Reads the file, saves/restores `CURRENT_FILE_PATH`.
   - Calls `extract_include_ast()` which:
     - Preprocesses bare attributes (same sentinel insertion).
     - Parses AST (uses shared cache).
     - Processes `use` statements (adds to parent's thread-local import maps).
     - Inherits parent mod name via `CURRENT_MOD_NAME` thread-local.
     - Builds identifier mappings for items in the include file.
     - Dispatches all attributes through the same registry.
     - Skips `evo_mod` and `evo_include` in dispatch (handled specially).
     - Recursively processes nested `#[evo_include]` directives.
4. Merge sub-manifest into parent via `manifest.ron.merge_from(sub.ron)`.

### `merge_from` (in `shared/src/manifest/mod.rs`)

Appends all `Vec<T>` fields from the included manifest into the parent.
Uses a `merge_vec!` macro that either swaps (if parent is empty) or appends.
Merges both `CoreManifest` and `GameManifest` fields.

### Key Properties

- Include files do NOT need `#![evo_mod(...)]` -- mod identity comes from the parent.
- Thread-local `USE_IMPORT_MAP`, `CROSS_MOD_IMPORT_MAP`, and `IDENTIFIER_MAP` are shared
  between parent and includes (they live in the same thread).
- Recursive includes are supported (an included file can itself use `#[evo_include]`).
- `evo_include` is registered as a no-op in the attribute registry to avoid
  "unknown attribute" errors during dispatch.

---

## 9. Identifier Resolution

### IDENTIFIER_MAP (Zero-String Typing)

**Thread-local**: `IDENTIFIER_MAP: HashMap<String, String>`

Maps Rust identifiers to their registered names. Built during Phase 0b.

Example: struct `Stone` with `#[evo_block(name = "STONE")]` registers `"Stone" -> "STONE"`.

**Usage**: `resolve_identifier(ident) -> String` looks up the map, returns the ident
unchanged if not found. Called by `evo_weapon`, `evo_armor`, `evo_tool`, `evo_give_item`,
`evo_armor_tier`, and inside `parse_ident_or_string`.

### USE_IMPORT_MAP

**Thread-local**: `USE_IMPORT_MAP: HashMap<String, Vec<String>>`

Maps imported type names to full module path segments. Built from `use` statements.
Example: `use shared::manifest::blocks::BlockProperties` -> `"BlockProperties" -> ["shared", "manifest", "blocks", "BlockProperties"]`.

Used by:
- `resolve_batch_field()` to determine which `RonModManifest` field a batch type maps to.
- `resolve_use_import_flags()` to resolve flag identifiers to bitmask values.
- `resolve_flag_array()` to resolve compile-time flag constants.

### CROSS_MOD_IMPORT_MAP

**Thread-local**: `CROSS_MOD_IMPORT_MAP: HashMap<String, String>`

Maps bare cross-mod symbol names to qualified references.
Example: `use mod_blocks::STONE` -> `"STONE" -> "mod-blocks::STONE"`.

Detection: first path segment starts with `mod_`. Underscores converted to hyphens
(`mod_blocks` -> `mod-blocks`).

Used by `resolve_cross_mod_string_arrays()` to rewrite bare identifiers in array
fields like `block_slots = [STONE, ...]` to string literals.

### KNOWN_MOD_NAMESPACES + PREPOPULATED_MOD_NAMESPACES

**Thread-locals**: `HashSet<String>` each.

`KNOWN_MOD_NAMESPACES` = current mod name + declared dependencies.
Cleared per-file, then re-seeded from `PREPOPULATED_MOD_NAMESPACES`.

`PREPOPULATED_MOD_NAMESPACES` = all mod names from the two-pass pre-scan.
Survives per-file clears.

Used to validate cross-mod namespace references (`mod_blocks::STONE` -- is `mod-blocks`
a known mod?).

### INFERRED_DEPENDENCIES

**Thread-local**: `HashSet<String>`

Whenever a cross-mod path is encountered (`mod_blocks::STONE`), the mod namespace is
recorded via `record_inferred_dependency("mod-blocks")`. Self-references are skipped.
After parsing, drained into `manifest.ron.dependencies`.

---

## 10. Post-Parse Transforms

Executed in `extract_manifest_from_ast` after include processing, in this order:

### 10.1 `resolve_extends_inplace_creatures`

**File**: `parse_game.rs`

For each `CreatureDefinition` with an `extends` field:
1. If the reference is cross-mod (`ns::name`) and same-mod, strip the namespace.
2. If truly cross-mod, defer to engine post-discovery.
3. For intra-mod: find the base creature (must be declared before the child).
4. Enforce single-level inheritance (base cannot itself extend).
5. Apply `CreatureDefinition::apply_overrides(&base, &child)`.

### 10.2 `resolve_boss_creature_inheritance`

**File**: `parse_game.rs`

For each `evo_boss` referencing a `creature_definition`:
1. Find the referenced creature by name.
2. Clone it into a new creature named `boss_<boss_name>`.
3. Skip if the mod already provides a creature with that name (manual override).
4. Apply boss-specific overrides (HP, damage, etc.).

### 10.3 `convert_inline_creature_definitions`

**File**: `parse_game.rs`

Processes all `CreatureDefinition` entries to expand inline declarations:

1. **Validates** that `behavior` and `follow` are mutually exclusive.
2. **Auto-generates geometry**: For each creature with `parts` but no existing
   `GeometryDef`, creates `GeometryDef(name, Creature(mod::name))`.
3. **Inline spawn -> SpawnRule**: For each creature with `spawn = (...)`, converts
   to a standalone `SpawnRule` entry, applying `spawn_defaults` fallbacks.
4. **Inline behavior -> CreatureBehaviorDef + flag + service + entity_service**: For
   each creature with `behavior = (...)`, generates:
   - A `CreatureBehaviorDef`
   - A `FlagDef` (if the behavior needs a flag)
   - A `ServiceDef` (compute service for the AI shader)
   - An `EntityServiceDef` (for the auto-assembly pipeline)

### 10.4 Block Spread Services

For each block with a `spread` field, auto-registers a compute service named
`{block_name}_spread` using the `block_spread_template` shader with spec constants
from `spread.build_spec_constants()`.

---

## 11. Validation

### Name Uniqueness (`validation.rs`)

**Function**: `validate_name_uniqueness(manifest, source_map, mod_name)`

Checks that no two declarations of the same type share the same name within a mod.
Validated types: `CreatureDefinition`, `Block`, `SpawnRule`, `CreatureBehavior`, `DamageFormula`.

Uses the `source_map` (populated during parsing) to produce error messages that
reference the declaring file.

### Dimension Worldgen Coverage (`validation.rs`)

**Function**: `validate_dimension_worldgen(manifest, mod_name)`

For each declared dimension:
1. Checks that at least one `evo_worldgen_service` targets it.
2. Checks that at least one service covers `phase = Terrain`.
3. Warns if a service's shader name differs from its service name (cross-mod shader reference).

### Import Path Validation (`validation.rs`)

**Function**: `validate_import_path(path_segments, include_dirs)`

For `use` statements starting with `"shared"`:
1. Resolves the longest prefix to a module file on disk.
2. If the full path is a module, valid.
3. Otherwise, checks that the first non-module segment exists as an item in the file.

### Cross-Reference Validation

**Function**: `validate_cross_references(manifest)` (in `mod.rs`)

Builds a set of all known registered names from `IDENTIFIER_MAP`. Then validates:
1. No duplicate block names.
2. Various field references (geometry references, loot table references, etc.)
   resolve to known identifiers.
3. Cross-mod references (containing `::`) are allowed to pass through (resolved at runtime).
4. Unknown identifiers produce an error with fuzzy-matched suggestions.

---

## 12. Patch System (Phase 5)

**File**: `manifest_extract/parse_patches.rs`

Extracted separately from the main attribute registry. Scans `syn::Item::Fn`,
`syn::Item::Const`, and `syn::Item::Static` for three attributes:

| Attribute | Purpose | Security Guarantee |
|-----------|---------|--------------------|
| `#[evo_hook("name")]` | Declares an injection site | P1: only explicit sites can be injected |
| `#[evo_overridable]` | Marks a const/fn as overridable | P2: overrides must target declared overridables |
| `#[evo_override("mod::TARGET")]` | Overrides a const/fn from another mod | P4: only from declared dependencies |

### Cross-Mod Validation

**Function**: `validate_patches(manifests, dep_graph) -> Vec<String>`

- **P2**: Every `evo_override` target must be declared `evo_overridable` in some mod.
- **P3**: Detects ambiguous overrides (two mods overriding the same target).
- **P4**: Override only allowed if the overrider declares the target mod as a dependency.

---

## 13. Data Structures

### ExtractedManifest

```rust
pub struct ExtractedManifest {
    pub ron: RonModManifest,         // The canonical manifest (shared with executor)
    pub settings: EvoSettings,       // Dynamic overrides from #[evo_settings]
    pub parse_state: ParseState,     // Ephemeral compiler state (source_map, spawn_defaults, screen_theme)
    pub warnings: Vec<String>,       // Parse warnings
    pub patches: PatchManifest,      // Phase 5 patch points
}
```

### RonModManifest

```rust
pub struct RonModManifest {
    pub core: CoreManifest,     // Engine-generic (services, buffers, entities, ...)
    pub game: GameManifest,     // Voxel-game-specific (blocks, creatures, weather, ...)
    pub includes: Vec<String>,  // Paths to external .ron fragment files
}
```

`Deref<Target = CoreManifest>` allows `manifest.name` shorthand.

### ParseState

```rust
pub struct ParseState {
    pub spawn_defaults: Option<SpawnRule>,    // Set by #[evo_spawn_defaults]
    pub screen_theme: Option<ScreenThemeDef>, // Set by #[evo_screen_theme]
    pub source_map: HashMap<(DeclType, String), String>, // (type, name) -> source file
}
```

### DeclType

Enum with 30 variants used for name uniqueness validation and source mapping:
`CreatureDefinition`, `Block`, `SpawnRule`, `Recipe`, `SmeltingRecipe`, `Weapon`,
`Armor`, `Tool`, `Projectile`, `EntityInteraction`, `Boss`, `Dimension`, `Service`,
`BlockTicker`, `Dispatch`, `Ore`, `Portal`, `Biome`, `Prefab`, `BreedRule`,
`MachineRecipe`, `LootTable`, `LootTablePatch`, `SessionFlag`, `FuelBurnTime`,
`CreatureBehavior`, `DamageFormula`, `Entity`, `Screen`, `Event`, `Sound`, `MusicTrack`.

---

## 14. Thread-Local State

All parse-time state is stored in thread-locals for rayon safety. Each is cleared or
re-seeded at the start of `extract_manifest_from_ast`.

| Thread-Local | Type | Purpose |
|-------------|------|---------|
| `CURRENT_MOD_NAME` | `Option<String>` | Current mod name; inherited by include files |
| `PARSE_WARNINGS` | `Vec<String>` | Accumulated warnings, drained after each file |
| `IDENTIFIER_MAP` | `HashMap<String, String>` | Zero-String Typing: Rust ident -> registered name |
| `USE_IMPORT_MAP` | `HashMap<String, Vec<String>>` | `use` import type name -> full path segments |
| `CROSS_MOD_IMPORT_MAP` | `HashMap<String, String>` | Cross-mod symbol -> qualified ref (`"STONE" -> "mod-blocks::STONE"`) |
| `INCLUDE_DIRS` | `Vec<PathBuf>` | Directories for resolving `use` paths to source files |
| `CURRENT_FILE_PATH` | `Option<PathBuf>` | Path of file being parsed (for error messages) |
| `KNOWN_MOD_NAMESPACES` | `HashSet<String>` | Current mod + dependencies (cleared per file, re-seeded from prepopulated) |
| `PREPOPULATED_MOD_NAMESPACES` | `HashSet<String>` | All mod names from pre-scan (survives per-file clear) |
| `INFERRED_DEPENDENCIES` | `HashSet<String>` | Auto-detected cross-mod dependencies, drained into manifest |

---

## File Map

| File | Purpose |
|------|---------|
| `mod.rs` | Main extraction logic, `extract_manifest_from_ast`, include system, identifier resolution, `serde_attr`, preprocessing |
| `attribute_registry.rs` | `AttributeParserRegistry` struct, `dispatch()`, `with_core_parsers()`, `with_game_parsers()` |
| `parse_core.rs` | 40 core attribute parsers (engine-generic) |
| `parse_game.rs` | 48 game attribute parsers (voxel-specific) + post-parse transforms |
| `parse_shader_transforms.rs` | 3 shader transform parsers (`evo_shader_inject/replace/add_function`) |
| `parse_pipeline.rs` | Service, pipeline, dispatch, compute attribute parser implementations |
| `parse_patches.rs` | Phase 5 patch extraction (`evo_hook`, `evo_overridable`, `evo_override`) + cross-mod validation |
| `validation.rs` | Name uniqueness, dimension worldgen coverage, import path validation |
| `lint.rs` | Post-parse lint diagnostics (quality warnings) |
| `mod_analysis.rs` | `.evo_mod` analysis for the LSP (EL020/EL021/EL022 checks, structural validation) |
