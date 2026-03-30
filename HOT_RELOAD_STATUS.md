# Hot Reload & Live Coding — Status Tracker

> Last updated: 2026-03-27

## Legend

### Hot Reload (file watcher detects change → re-apply)
| Icon | Meaning |
|------|---------|
| ✅ | Working — tested and confirmed |
| 🔧 | Implemented but untested |
| ❌ | Not implemented — needs re-registration handler |
| 🚫 | Impossible — requires engine restart |

### Live Coding (add new declarations at runtime)
| Icon | Meaning |
|------|---------|
| ✅ | Supported — append/replace trivial |
| ⚠️ | Possible but needs work (shader compile, ID allocation) |
| ❌ | Impossible without restart (GPU layout, mod identity) |

---

## Module & Structure (#1–5)

| # | Annotation | Hot Reload | Live Code | Notes |
|---|-----------|:---:|:---:|-------|
| 1 | `#![evo_mod]` | 🚫 | ❌ | Mod identity = global namespace |
| 2 | `#[evo_include]` | ✅ | ✅ | Watcher detects new includes — tested 2026-03-26 |
| 3 | `#[evo_settings]` | 🚫 | ❌ | Compile-time flags |
| 4 | `#[evo_batch]` | 🔧 | ✅ | Expands to individual declarations |
| 5 | `#[evo_data]` | 🔧 | ✅ | Key-value, read from manifest |

## Blocks & Crafting (#6–15)

| # | Annotation | Hot Reload | Live Code | Notes |
|---|-----------|:---:|:---:|-------|
| 6 | `#[evo_block]` | ❌ | ⚠️ | Needs BlockRegistry re-registration + GPU palette |
| 7 | `#[evo_recipe]` / `#[evo_craft]` | ❌ | ⚠️ | Needs GPU-side re-registration via execute_recipes() |
| 8 | `#[evo_smelt]` / `#[evo_smelting_recipe]` | ❌ | ⚠️ | Same — GPU smelting table |
| 9 | `#[evo_fuel_burn_time]` | 🔧 | ✅ | Read from manifest via cache |
| 10 | `#[evo_block_interaction]` | 🔧 | ✅ | Cache reset by reset_for_manifest_reload() |
| 11 | `#[evo_machine_recipe]` | 🔧 | ✅ | Read from manifest |
| 12 | `#[evo_block_hardness]` | 🔧 | ✅ | Cache rebuild via mod_aggregation_dirty |
| 13 | `#[evo_mining_config]` | 🔧 | ✅ | Cache rebuild via mod_aggregation_dirty |
| 14 | `#[evo_block_ticker]` | ❌ | ⚠️ | Needs shader compile + service creation |
| 15 | `#[evo_ore]` | 🔧 | ✅ | Future chunks use new ore table |

## Entities & Creatures (#16–24)

| # | Annotation | Hot Reload | Live Code | Notes |
|---|-----------|:---:|:---:|-------|
| 16 | `#[evo_entity]` | ❌ | ⚠️ | Needs execute_entities() for spawn-on-load |
| 17 | `#[evo_entity_schema]` | 🚫 | ❌ | GPU buffer layout |
| 18 | `#[evo_creature_definition]` | ✅ | ✅ | CreatureRegistry + geometry + spawn — tested 2026-03-26 |
| 19 | `#[evo_creature]` | 🔧 | ✅ | Same path as creature_definition |
| 20 | `#[evo_creature_from_primitive]` | 🔧 | ✅ | Expands to creature_definition |
| 21 | `#[evo_spawn_defaults]` | 🔧 | ✅ | Cache rebuild via mod_aggregation_dirty |
| 22 | `#[evo_breed_rule]` | 🔧 | ✅ | Read from manifest |
| 23 | `#[evo_boss]` | 🔧 | ✅ | Read from manifest |
| 24 | `#[evo_projectile]` | 🔧 | ✅ | Read from manifest |

## Flags & State (#25–28)

| # | Annotation | Hot Reload | Live Code | Notes |
|---|-----------|:---:|:---:|-------|
| 25 | `#[evo_flag]` | 🚫 | ❌ | GPU bitfield layout fixed at init |
| 26 | `#[evo_session_flag]` | 🔧 | ✅ | Read from manifest |
| 27 | `#[evo_game_state]` | 🚫 | ❌ | Enum compiled into dispatch conditions |
| 28 | `#[evo_faction]` | 🔧 | ✅ | Read from manifest |

## Geometry & Animation (#29–30)

| # | Annotation | Hot Reload | Live Code | Notes |
|---|-----------|:---:|:---:|-------|
| 29 | `#[evo_geometry]` | ❌ | ⚠️ | Needs execute_geometries() — GPU mesh upload |
| 30 | `#[evo_anim]` | 🔧 | ✅ | Read from manifest |

## Equipment & Combat (#31–40)

| # | Annotation | Hot Reload | Live Code | Notes |
|---|-----------|:---:|:---:|-------|
| 31 | `#[evo_weapon]` | 🔧 | ✅ | Read from manifest |
| 32 | `#[evo_weapon_material]` | 🔧 | ✅ | Read from manifest |
| 33 | `#[evo_armor]` | 🔧 | ✅ | Read from manifest |
| 34 | `#[evo_armor_tier]` | 🔧 | ✅ | Expands to armor defs |
| 35 | `#[evo_tool]` | 🔧 | ✅ | Cache rebuild via mod_aggregation_dirty |
| 36 | `#[evo_loot_table]` | 🔧 | ✅ | Cache rebuild via mod_aggregation_dirty |
| 37 | `#[evo_loot_table_patch]` | 🔧 | ✅ | Read from manifest |
| 38 | `#[evo_damage_formula]` | ❌ | ⚠️ | Needs shader template recompile |
| 39 | `#[evo_mob_attack]` | 🔧 | ✅ | Read from manifest |
| 40 | `#[evo_entity_interaction]` | 🔧 | ✅ | Read from manifest |

## GPU Services & Pipelines (#41–53)

| # | Annotation | Hot Reload | Live Code | Notes |
|---|-----------|:---:|:---:|-------|
| 41 | `#[evo_service]` | ❌ | ⚠️ | Needs execute_services() + pipeline creation |
| 42 | `#[evo_compute]` | ❌ | ⚠️ | Same |
| 43 | `#[evo_gpu_pipeline]` | ❌ | ⚠️ | Same |
| 44 | `#[evo_chunk_pipeline]` | ❌ | ⚠️ | Same |
| 45 | `#[evo_buffer]` | 🚫 | ❌ | VkBuffer reallocation |
| 46 | `#[evo_binding]` | 🚫 | ❌ | Descriptor set reallocation |
| 47 | `#[evo_dispatch]` | 🔧 | ✅ | Graph rebuild via rebuild_baked_*_graph_inline |
| 48 | `#[evo_dispatch_sequence]` | 🔧 | ✅ | Graph rebuild |
| 49 | `#[evo_instance_services]` | ❌ | ⚠️ | Metadata OK, service init dynamic |
| 50 | `#[evo_entity_service]` | ❌ | ⚠️ | Same |
| 51 | `#[evo_compute_system]` | ❌ | ⚠️ | Needs shader compile |
| 52 | `#[evo_temp_buffer_pool]` | 🚫 | ❌ | Vulkan pool sizing |
| 53 | `#[evo_block_ticker]` | ❌ | ⚠️ | Shader compile + service creation |

## World & Dimensions (#54–58)

| # | Annotation | Hot Reload | Live Code | Notes |
|---|-----------|:---:|:---:|-------|
| 54 | `#[evo_dimension]` | 🚫 | ❌ | Render target + buffer set setup |
| 55 | `#[evo_worldgen_service]` | ❌ | ⚠️ | Needs shader compile |
| 56 | `#[evo_prefab]` | 🔧 | ✅ | Cache rebuild via mod_aggregation_dirty |
| 57 | `#[evo_biome]` | 🔧 | ✅ | Read from manifest |
| 58 | `#[evo_portal]` | 🔧 | ✅ | Read from manifest |

## UI & Input (#59–65)

| # | Annotation | Hot Reload | Live Code | Notes |
|---|-----------|:---:|:---:|-------|
| 59 | `#[evo_screen]` | ✅ | ✅ | Read from manifest each frame — tested 2026-03-26 |
| 60 | `#[evo_screen_theme]` | 🔧 | ✅ | Read from manifest |
| 61 | `#[evo_keybind]` | ❌ | ✅ | Needs KeybindRegistry re-registration |
| 62 | `#[evo_action_keybind]` | 🔧 | ✅ | Read from manifest |
| 63 | `#[evo_input_mapping]` | 🔧 | ✅ | Read from manifest |
| 64 | `#[evo_worldspace_ui]` | 🔧 | ✅ | Read from manifest |
| 65 | `#[evo_chat]` | 🔧 | ✅ | Read from manifest |

## Audio (#66–69)

| # | Annotation | Hot Reload | Live Code | Notes |
|---|-----------|:---:|:---:|-------|
| 66 | `#[evo_audio]` | 🔧 | ✅ | Read from manifest |
| 67 | `#[evo_synth]` | ❌ | ⚠️ | GPU synth shader |
| 68 | `#[evo_sequence]` | 🔧 | ✅ | Read from manifest |
| 69 | `#[evo_music_playlist]` | 🔧 | ✅ | Read from manifest |

## Environment (#70–71)

| # | Annotation | Hot Reload | Live Code | Notes |
|---|-----------|:---:|:---:|-------|
| 70 | `#[evo_sun_cycle]` | ✅ | ✅ | Read from manifest — tested 2026-03-26 |
| 71 | `#[evo_weather]` | ✅ | ✅ | Read from manifest — tested 2026-03-26 |

## Shader Transforms (#72–74)

| # | Annotation | Hot Reload | Live Code | Notes |
|---|-----------|:---:|:---:|-------|
| 72 | `#[evo_shader_inject]` | ❌ | ⚠️ | Needs target shader recompilation |
| 73 | `#[evo_shader_replace]` | ❌ | ⚠️ | Same |
| 74 | `#[evo_shader_add_function]` | ❌ | ⚠️ | Same |

## Misc (#75–89)

| # | Annotation | Hot Reload | Live Code | Notes |
|---|-----------|:---:|:---:|-------|
| 75 | `#[evo_event]` | 🔧 | ✅ | Read from manifest |
| 76 | `#[evo_persistence]` | 🔧 | ✅ | Config replace |
| 77 | `#[evo_camera_follow]` | 🔧 | ✅ | Read from manifest |
| 78 | `#[evo_fps_input_capture]` | 🔧 | ✅ | Read from manifest |
| 79 | `#[evo_protocol]` | 🚫 | ❌ | Inter-mod API contract |
| 80 | `#[evo_provide]` | 🚫 | ❌ | Coupled to protocol |
| 81 | `#[evo_emitter]` | 🔧 | ✅ | Read from manifest |
| 82 | `#[evo_scenario]` | 🔧 | ✅ | Read from manifest |
| 83 | `#[evo_achievements]` | 🔧 | ✅ | Read from manifest |
| 84 | `#[evo_give_item]` | 🔧 | ✅ | Read from manifest |
| 85 | `#[evo_inventory_interaction]` | 🔧 | ✅ | Read from manifest |
| 86 | `#[evo_raycast_chain]` | ❌ | ⚠️ | GPU pipeline |
| 87 | `#[evo_tick]` | 🔧 | ✅ | Read from manifest |
| 88 | `#[evo_selector]` | 🔧 | ✅ | Read from manifest |
| 89 | `#[evo_core_field]` / `#[evo_ext_field]` | 🚫 | ❌ | GPU buffer layout |

---

## Summary

| Status | Hot Reload | Count |
|--------|-----------|-------|
| ✅ Working (tested) | Confirmed in mod-hot-reload-test | 5 |
| 🔧 Implemented (untested) | Manifest swap + cache rebuild | 44 |
| ❌ Needs handler | Re-registration in engine registries | 21 |
| 🚫 Restart required | GPU layout / identity / contracts | 11 |

| Status | Live Code | Count |
|--------|----------|-------|
| ✅ Supported | Data-only append/replace | 53 |
| ⚠️ Possible (work needed) | Shader compile, ID alloc, pipeline | 22 |
| ❌ Impossible | GPU layout, mod identity | 11 |

Tested ✅ hot-reload items: `#[evo_include]` (#2), `#[evo_creature_definition]` (#18), `#[evo_screen]` (#59), `#[evo_sun_cycle]` (#70), `#[evo_weather]` (#71).

---

## Known Gaps

### #233 — Hot reload only updates new creatures
Creature definition hot reload updates the CreatureRegistry and spawn rules, but already-spawned GPU entities keep their old stats (HP, speed, damage, geometry_id). Patching in-place on GPU requires a dedicated "entity patch" compute dispatch — see brainstorm below.

### #4 — Template hot reload propagation (VERIFIED OK)
When a template shader (e.g., `passive_ai_template.evo_shader`) is hot-reloaded, `reinitialize_services_using_shader()` matches ALL services whose `shader_name` equals the template name. Since template-instantiated services store the template as their `shader_name`, all instantiated services ARE rebuilt with the new SPIR-V. However, spec constants are NOT re-read from the manifest — the old spec constant values are reused. If a modder changes both the template AND the creature definition spec constants, only the shader change takes effect.

### Auto-history / evo diff (not implemented)
No `.evo_history/` folder or diff recording exists. Hot reload overwrites are irreversible.

---

## Next Steps (priority order)

1. **Test the 🔧 annotations** — change values in mod-hot-reload-test, promote to ✅
2. **#233 GPU entity patching** — compute shader to patch entity_core/entity_ext for live entities
3. **Blocks + recipes** — add re-registration handlers (most visible for modders)
4. **Keybinds** — add KeybindRegistry re-registration
5. **Geometries + entities** — add re-registration for spawn-on-load
6. **Services** — dynamic pipeline creation for new compute services
7. **Auto-history** — `.evo_history/` diff recording per hot reload session
8. **Template spec constant re-read** — on manifest hot reload, update service spec_constants before reinit
