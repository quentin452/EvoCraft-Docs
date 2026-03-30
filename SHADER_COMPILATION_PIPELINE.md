# EvoShader Compilation Pipeline

End-to-end documentation of how `.evo_shader` source files are compiled to SPIR-V binaries by `evo-shader-compiler`.

**Source**: `libs/evo-shader-compiler/src/`

---

## Table of Contents

1. [Pipeline Overview](#1-pipeline-overview)
2. [Entry Point](#2-entry-point)
3. [Source Language](#3-source-language)
4. [Preprocessing](#4-preprocessing)
5. [Parsing (Frontend)](#5-parsing-frontend)
6. [Static Analysis (Analyzer)](#6-static-analysis-analyzer)
7. [Custom Attributes](#7-custom-attributes)
8. [IR Transforms](#8-ir-transforms)
9. [SPIR-V Code Generation (Backend)](#9-spir-v-code-generation-backend)
10. [Optimization Passes](#10-optimization-passes)
11. [Validation](#11-validation)
12. [External Optimization (spirv-opt)](#12-external-optimization-spirv-opt)
13. [Manifest Generation](#13-manifest-generation)
14. [GBT Integration](#14-gbt-integration)
15. [Spec Constants and Template Shaders](#15-spec-constants-and-template-shaders)
16. [Safety Guards (EvoLint)](#16-safety-guards-evolint)
17. [Caching](#17-caching)
18. [Parallel Compilation](#18-parallel-compilation)

---

## 1. Pipeline Overview

```
.evo_shader source
       |
       v
[Preprocessor]        #ifdef / #ifndef / #else / #endif stripping
       |
       v
[syn::parse_file]     Rust parser (syn 2.0) produces syn::File AST
       |
       v
[Analyzer]            Static analysis: EvoLint EL001..EL023, memory model checks
       |
       v
[Frontend Parser]     syn AST -> ShaderModule (custom IR)
       |                  Processes items, imports, use-trees, loader
       v
[IR Transforms]       9 ordered transform passes (prologue injection, etc.)
       |
       v
[Name Interning]      Populate interner for fast NameId-keyed lookups
       |
       v
[Entry Point Split]   Clone module per entry point (parallel via rayon)
       |
       v
[Backend Codegen]     ShaderModule IR -> rspirv::Builder -> SPIR-V words
       |
       v
[Native Opt Passes]   22 SPIR-V optimization passes (on rspirv::dr::Module)
       |
       v
[SPIR-V Validation]   spirv-tools in-process validator
       |
       v
[spirv-opt]           External optimizer (tiered: skip / light / full)
       |
       v
[Manifest Gen]        ShaderManifest: entry points, descriptor sets, push consts,
       |              buffer access fingerprints, GBT resource accesses
       v
CompiledArtifact { name, spv: Vec<u32>, manifest }
```

---

## 2. Entry Point

The public API lives in `lib.rs`:

```rust
// libs/evo-shader-compiler/src/lib.rs

pub fn compile(source: &str) -> Result<Vec<CompiledArtifact>>
pub fn compile_with_options(source: &str, include_dirs: &[PathBuf], origin: Option<&Path>) -> Result<Vec<CompiledArtifact>>
pub fn compile_with_defines(source: &str, include_dirs: &[PathBuf], origin: Option<&Path>, defines: &[&str]) -> Result<Vec<CompiledArtifact>>
```

All three entry points follow the same flow:

1. **Preprocess** the source (strip `#ifdef` / `#ifndef` / `#else` / `#endif` directives).
2. Call `compile_source_flow()` which orchestrates the full pipeline.

`compile_source_flow()` (line 247 of `lib.rs`) is the internal orchestrator:

1. Eagerly index include-dir symbols for import suggestions.
2. Retrieve or build the **header base module** from `header_cache`.
3. `parse_and_analyze()` -- runs the analyzer then the frontend parser on a single `syn::parse_file` AST (shared, not parsed twice).
4. `transforms::run_all_transforms()` -- IR-level transforms.
5. Re-intern all names.
6. Split entry points, generate artifacts in parallel via `rayon::par_iter`.

Each entry point produces a separate `CompiledArtifact { name, spv, manifest }`.

---

## 3. Source Language

`.evo_shader` files use **Rust syntax** parsed by the `syn` crate. This is NOT rust-gpu -- it is a completely custom compiler that uses Rust as the surface language:

- Rust-like syntax: `fn`, `let`, `if`, `for`, `match`, `struct`, `enum`, `impl`, `use`, `const`, `static`
- Rust types mapped to GPU types: `f32`, `u32`, `i32`, `Vec2`, `Vec3`, `Vec4`, `UVec3`, `Mat4`, `Quat`, etc.
- Attributes use Rust attribute syntax: `#[spirv(...)]`, `#[evo_shader(...)]`, `#[repr(C)]`, `#[inline]`
- Imports via `use shared::shaders::noise;` resolve to actual `.rs` or `.evo_shader` files on disk
- SPIR-V concepts expressed as Rust idioms:
  - `&[u32]` = storage buffer (SSBO) read-only
  - `&mut [u32]` = storage buffer read-write
  - `as &PhysicalStorageBuffer<[u64]>` = BDA pointer cast
  - `shared_*` prefix = workgroup shared memory
  - `push` / `*_pc` naming = push constants (auto-inferred)

Example (from `terrain_gen.evo_shader`):

```rust
use shared::shaders::noise;
use shared::shaders::voxel::{pack_voxel, set_block};

#[evo_shader(preset = "TerrainGen", threads(16, 16, 1),
             worldgen_queue = "TERRAIN", signal_completion = "TERRAIN_DONE")]
pub fn terrain_gen(
    global_id: UVec3,
    local_id: UVec3,
    block_slots: &[u32],
    push: &shared::TerrainGenPushConsts,
    shared_noise_cache: &mut [f32; 256],
) {
    control_barrier();
    // ... terrain generation logic using gbt, voxels, etc.
    // (prologue variables auto-injected by worldgen_queue transform)
}
```

---

## 4. Preprocessing

**File**: `frontend/preprocessor.rs`

A simple line-by-line preprocessor that handles:

- `#ifdef VAR` / `#ifndef VAR` / `#else` / `#endif` (nested)
- **Comment-prefixed form** (recommended): `// #ifdef FOO` -- rust-analyzer treats these as comments so no red squiggles, but the preprocessor strips the `// ` prefix and processes normally.
- Skipped lines are replaced with empty lines to **preserve line numbers**.
- Defines are passed via `compile_with_defines()` (e.g., `PROFILER_ENABLED`).

---

## 5. Parsing (Frontend)

**Files**: `frontend/parser/mod.rs`, `frontend/parser/items.rs`, `frontend/parser/expressions.rs`, `frontend/parser/statements.rs`, `frontend/parser/types.rs`, `frontend/parser/attributes.rs`

### Step 1: syn::parse_file

The source is parsed once by `syn::parse_file()` into a `syn::File` AST. This AST is shared between the analyzer and the frontend parser to avoid a double-parse (~15ms savings).

### Step 2: AST to ShaderModule

`parse_items()` iterates over `syn::Item` nodes and calls `process_item()` for each:

| syn::Item          | ShaderModule field     | Notes                                    |
|--------------------|------------------------|------------------------------------------|
| `Item::Fn`         | `entry_points` or `functions` | Entry point if `#[spirv]` or `#[evo_shader]` attribute |
| `Item::Struct`     | `structs`              | With members, types, `#[repr(C)]` check  |
| `Item::Enum`       | `enums`                | Variants with optional payloads           |
| `Item::Const`      | `constants`            | Value expressions, `#[spirv(spec_constant)]` |
| `Item::Static`     | `static_vars`          | Storage class inference                   |
| `Item::Impl`       | `impls`                | Methods + associated constants            |
| `Item::Use`        | `module_aliases` + imported items | `resolve_imports()` via loader |
| `Item::Mod`        | (inline module items)  | Recursively processes contained items     |

**Skipping rules**:
- `#[cfg(not(target_arch = "spirv"))]` -- CPU-only code, skipped
- `const _: ... = ...` -- anonymous compile-time assertions, skipped
- Host-only intrinsics (`core::mem::size_of`, etc.) in const values -- skipped with warning

### Step 3: Imports and Module Resolution

**File**: `frontend/loader.rs`

The loader resolves `use` trees by:
1. Collecting import paths via `use_tree::collect_use_paths()`
2. Finding the corresponding file on disk (`find_module_file()` + include_dirs)
3. Parsing the file (with AST cache: `use_tree::get_cached_ast()`)
4. Processing its items into the current module
5. Tracking `visited_files` to prevent re-importing the same file

Glob imports (`use foo::*`) are **banned** in `.evo_shader` files (explicit named imports required) but allowed in library `.rs` files.

### Step 4: Hook Point Scanning

Before `syn::parse_file` strips comments, the parser pre-scans for `// @hook:name` comments and injects `Statement::HookPoint { name }` IR nodes into the corresponding function bodies. These are used by the mod transform system as injection points.

---

## 6. Static Analysis (Analyzer)

**File**: `analyzer.rs`

The analyzer runs a `syn::visit::Visit` pass over the raw AST (before IR conversion) and produces an `AnalysisReport` with:

- **EvoLint diagnostics** (EL001--EL023): compile-time checks for GPU correctness
- **Memory model validation**: long-lived pointers, generic pointers, storage class violations, SSA violations, returned pointers
- **Divergence taint analysis**: tracks variables depending on thread ID, warns about barriers/atomics in divergent branches

The analyzer uses a **ShaderVisitor** that maintains:
- Scope stack for symbol tracking
- Pointer variable tracking with storage class info
- Divergent variable set (taint propagation from `global_id`, `local_id`, etc.)
- Null-guard detection for entity_id patterns
- Per-function and per-file `#[allow(ELxxx)]` suppression

---

## 7. Custom Attributes

**File**: `frontend/parser/attributes.rs`

### `#[spirv(...)]` -- Low-Level SPIR-V Control

Applied to entry point functions, parsed by `parse_stage_attr()`:

| Key | Effect |
|-----|--------|
| `compute`, `vertex`, `fragment`, `mesh_ext`, `task_ext`, `geometry` | Sets shader stage |
| `threads(X, Y, Z)` | Workgroup size |
| `entry_point_name = "..."` | Override SPIR-V entry point name |
| `output_vertices = N`, `output_primitives_ext = N` | Mesh shader parameters |
| `output_triangles_ext` / `output_lines_ext` / `output_points` | Mesh output topology |

Applied to function parameters, parsed by `parse_arg_attributes()`:

| Key | Effect |
|-----|--------|
| `descriptor_set(N)`, `binding(M)` | Explicit descriptor layout |
| `push_constant` | Mark as push constant block |
| `spec_constant(id = N, default = V)` | Specialization constant |
| `uniform` | UBO storage class |
| `storage_buffer` | Explicit SSBO storage class |
| `non_writable` | Read-only buffer (NonWritable decoration) |
| `flat` | Flat interpolation |
| `engine_bind = "..."` | Engine-managed binding (resolved at load time) |
| `engine_buffer_init = "..."` | Buffer initialization hint |
| `global_invocation_id`, `local_invocation_id`, `workgroup_id`, etc. | SPIR-V BuiltIn decorations |

### `#[evo_shader(...)]` -- High-Level Declarative Shader

Parsed by `parse_evo_shader_attr()`. This is the recommended way to declare entry points:

```rust
#[evo_shader(preset = "TerrainGen", threads(16, 16, 1),
             worldgen_queue = "TERRAIN", signal_completion = "TERRAIN_DONE",
             priority = 100)]
pub fn my_shader(...) { ... }
```

| Key | Effect |
|-----|--------|
| `preset = "..."` | Auto-assigns descriptor bindings from preset definition |
| `threads = N` or `threads(X, Y, Z)` | Workgroup size |
| `entry_point_name = "..."` | Override SPIR-V name |
| `priority = N` | Exclusive preset conflict resolution (highest wins) |
| `worldgen_queue = "..."` | Triggers prologue injection (queue math, data_slot, coords) |
| `signal_completion = "..."` | Triggers epilogue injection (thread-0 guard + atomic_or status bit) |

### Preset System

**File**: `presets.rs`

Presets define canonical buffer binding layouts. When `preset = "TerrainGen"` is specified:
- Parameters are matched by **name** to `PresetBinding` entries (set, binding)
- Built-in variables are auto-decorated by name (e.g., `global_id` -> `BuiltIn(GlobalInvocationId)`)
- Push constants are auto-inferred from naming conventions (`push`, `*_pc`, `pc`)
- Workgroup shared memory is auto-inferred from `shared_*` prefix

### `#[evo_buffer(...)]`, `#[evo_dispatch(...)]`, `#[evo_push_constants]`

These are **manifest-level** attributes declared in `.evo_mod` files (not in `.evo_shader`), processed by the manifest extraction system (`manifest_extract/`). They declare:

- `#[evo_buffer]`: GBT buffer slots, sizes, and access patterns for the mod
- `#[evo_dispatch]`: GPU dispatch configurations (workgroup size, buffer bindings)
- `#[evo_push_constants]`: Push constant struct layout for a dispatch

The manifest codegen system (`manifest_codegen/`) generates `.evo_shader` source from these declarations.

---

## 8. IR Transforms

**File**: `transforms/mod.rs` and submodules

After parsing, 9+ ordered transforms run on the `ShaderModule` IR **in-place**. The pipeline order matters:

```
1. worldgen_prologue     transforms/worldgen.rs
2. worldgen_epilogue     transforms/worldgen.rs
3. gbt_prologue          transforms/gbt.rs
4. entity_prologue       transforms/entity.rs
5. voxel_ops_prologue    transforms/voxel_ops.rs
6. engine_config         transforms/engine_config.rs
7. evo_tick              transforms/evo_tick.rs
8. evo_selector          transforms/evo_selector.rs
9. evo_anim              transforms/evo_anim.rs
10. mod_transforms       transforms/mod_transforms.rs  (from mod manifests)
11. atomic_lint           transforms/atomic_lint.rs     (read-only lint pass)
```

### Transform Details

**worldgen_prologue** (`worldgen.rs`): For entry points with `worldgen_queue = "TERRAIN"`, injects ~20 lines of prologue that:
- Load `gbt` from `engine_pc.gbt_addr` (BDA root)
- Load `voxels`, `chunk_status`, `slot_to_coord` from GBT slots
- Compute `data_slot` from the worldgen queue
- Resolve chunk `coord`, `chunk_offset_x/z`, `voxel_base`
- Insert bounds check (`if global_id.x >= CHUNK_SIZE { return; }`)

**worldgen_epilogue** (`worldgen.rs`): For entry points with `signal_completion = "TERRAIN_DONE"`, appends:
- Thread(0,0) guard (only thread 0 writes the completion signal)
- `atomic_or(chunk_status[data_slot], 1 << TERRAIN_DONE_BIT)`

**gbt_prologue** (`gbt.rs`): For any entry point with `engine_pc` parameter that doesn't already have a `let gbt`, injects:
```rust
let gbt = engine_pc.gbt_addr as &PhysicalStorageBuffer<[u64]>;
```

**entity_prologue** (`entity.rs`): For Entity* presets, injects GBT loads for entity_core, entity_ext, and other entity buffers.

**evo_tick** (`evo_tick.rs`): For `#[evo_tick]` shaders, resolves `self.field` syntax into BDA member accesses on entity buffers, injects bounds checks, active checks, and write-back logic.

**mod_transforms** (`mod_transforms.rs`): Applies mod-declared shader transforms:
- **Inject**: Insert snippet at hook points or pattern locations
- **Replace**: Swap matched statements with snippet
- **AddFunction**: Append function definitions

All transforms are idempotent (check for existing variables before injecting).

---

## 9. SPIR-V Code Generation (Backend)

**Files**: `backend.rs`, `backend/codegen/`, `backend/intrinsics/`, `backend/types.rs`, `backend/context.rs`, `backend/tree_shaking.rs`, `backend/capabilities.rs`, `backend/validate.rs`

### Entry: `generate_spirv_module()`

Located in `backend.rs` (line 38). Orchestrates the full IR-to-SPIR-V translation:

1. **Pre-validation**: `validate::validate_module()` checks structural rules (e.g., RuntimeArray must be last struct member).

2. **Builder setup**: Creates `rspirv::dr::Builder` targeting SPIR-V 1.4.

3. **Capability scanning**: `capabilities::CapabilityScanner` walks the entire IR to detect required Vulkan capabilities:
   - `Shader` (always)
   - `MeshShadingEXT` (for Task/Mesh stages)
   - `PhysicalStorageBufferAddresses` (for BDA pointers)
   - `GroupNonUniform*` (for subgroup operations)
   - `Int8`, `Int16`, `Int64`, `Float16`, `Float64` (for non-standard types)
   - And corresponding extensions (`SPV_EXT_mesh_shader`, `SPV_KHR_physical_storage_buffer`, etc.)

4. **Memory model**: `PhysicalStorageBuffer64` when BDA is used, otherwise `Logical`. Always `GLSL450`.

5. **Type registry**: `TypeRegistry` (in `backend/types.rs`) maps `DataType` -> SPIR-V type IDs, handling:
   - Scalars, vectors, matrices
   - Structs with std430/std140 layout + Block decoration
   - RuntimeArrays, Pointers, Images, Samplers
   - Enum → u32 tag + per-variant padding structs

6. **Tree shaking**: `tree_shaking::collect_reachable_functions()` starts from entry points and traces call graphs to eliminate dead functions.

7. **Function codegen**: For each reachable function and entry point:
   - Declare SPIR-V function with correct signature
   - Process parameters (with decorations: BuiltIn, DescriptorSet, Binding, PushConstant, etc.)
   - Generate function body via `codegen/statements.rs` and `codegen/expressions.rs`

8. **Intrinsics**: Built-in functions are handled by specialized modules:

   | Module | Functions |
   |--------|-----------|
   | `intrinsics/math.rs` | `sin`, `cos`, `sqrt`, `clamp`, `mix`, `dot`, `cross`, `normalize`, `length`, `distance`, `pow`, `exp`, `log`, `fma`, `abs`, `min`, `max`, `floor`, `ceil`, `round`, `fract`, `step`, `smoothstep` |
   | `intrinsics/atomic.rs` | `atomic_add`, `atomic_sub`, `atomic_and`, `atomic_or`, `atomic_xor`, `atomic_min`, `atomic_max`, `atomic_exchange`, `atomic_compare_exchange`, `atomic_load` |
   | `intrinsics/construct.rs` | Vector/matrix constructors, `vec2()`, `vec3()`, `vec4()`, `uvec3()`, struct construction |
   | `intrinsics/pipeline.rs` | `control_barrier()`, `memory_barrier()`, `emit_mesh_tasks()`, `set_mesh_outputs()` |
   | `intrinsics/pointer.rs` | BDA pointer casts, `PhysicalStorageBuffer` loads/stores |
   | `intrinsics/subgroup.rs` | `subgroup_ballot()`, `subgroup_elect()`, `subgroup_broadcast()`, `subgroup_add()` |
   | `intrinsics/texture.rs` | `image_read()`, `image_write()`, `image_fetch()`, `sample()`, `texture_size()` |

9. **Constant folding**: `CodegenContext` maintains a `constant_map` that tracks compile-time constant values through expressions for folding.

10. **Debug info**: When `SHADER_LOGGING_ENABLED` is set, imports `NonSemantic.DebugPrintf` and emits `OpExtInst` for `printf()` / `debug_printf()` calls. When disabled (default), printf calls become no-ops.

11. **Warp divergence probing**: When `WARP_DIVERGENCE_PROBING` is set, injects `subgroupBallot` probes at branch points for GPU profiling.

---

## 10. Optimization Passes

### Native Passes (Always Run)

**File**: `optimization/mod.rs` and submodules

All passes implement `trait Pass { fn run(&self, module: &mut rspirv::dr::Module); }` and operate directly on the SPIR-V `Module` in-memory (via `rspirv`).

**Unconditional passes** (always run, in order):

| # | Pass | File | Description |
|---|------|------|-------------|
| 1 | `DeadBlockElimination` | `dead_blocks.rs` | Remove unreachable basic blocks |
| 2 | `BitcastElimination` | `bitcast.rs` | Simplify redundant bitcast chains |
| 3 | `RedundantConversionElimination` | `redundant_conversions.rs` | Remove no-op type conversions (e.g., i32->i32) |
| 4 | `AccessChainCombining` | `access_chain_combining.rs` | Merge nested OpAccessChain sequences |
| 5 | `LoadStoreVectorization` | `load_store_vectorization.rs` | Combine scalar loads/stores into vector ops |
| 6 | `ConstantBranchFolding` | `constant_branch_folding.rs` | Fold branches on compile-time constants |
| 7 | `DeadBlockElimination` | `dead_blocks.rs` | Second pass after constant folding |
| 8 | `DeadMemberElimination` | `dead_members.rs` | Remove unused struct members |
| 9 | `DeadBindingElimination` | `dead_bindings.rs` | Remove unused descriptor bindings |

**Extended passes** (gated behind `NATIVE_SPIRV_PASSES` flag, set via `[dev] native_spirv_passes = true`):

| # | Pass | File | Description |
|---|------|------|-------------|
| 10 | `InlineExhaustive` | `inline_exhaustive.rs` | Inline all non-recursive function calls |
| 11 | `SSARewrite` | `ssa_rewrite.rs` | Convert load/store patterns to SSA (mem2reg) |
| 12 | `DeadFunctionElimination` | `dead_functions.rs` | Remove functions unreachable from entry points |
| 13 | `UnifyConstants` | `unify_constants.rs` | Deduplicate identical constant definitions |
| 14 | `InsertExtractElimination` | `insert_extract_elim.rs` | Simplify OpCompositeInsert/Extract chains |
| 15 | `BlockMerge` | `block_merge.rs` | Merge single-predecessor/successor basic blocks |
| 16 | `LocalLoadStoreElimination` | `local_load_store_elim.rs` | Eliminate redundant local loads/stores |
| 17 | `ConstantPropagation` | `constant_propagation.rs` | Propagate known constant values |
| 18 | `Simplification` | `simplification.rs` | Algebraic simplifications (x*1=x, x+0=x, etc.) |
| 19 | `RedundancyElimination` | `redundancy_elimination.rs` | CSE (common subexpression elimination) |
| 20 | `CopyPropagation` | `copy_propagation.rs` | Replace copies with originals |
| 21 | `AggressiveDeadCodeElimination` | `dead_code.rs` | ADCE -- remove all unused instructions |
| 22 | `RemoveDuplicates` | `remove_duplicates.rs` | Remove duplicate type/decoration instructions |
| 23 | Strip debug names | (inline) | Clear `debug_names` section |
| 24 | `CompactIds` | `compact_ids.rs` | Renumber all IDs to be contiguous |

After all native passes, the SPIR-V header bound is recalculated to account for any new IDs.

---

## 11. Validation

**Function**: `validate_spirv()` in `lib.rs`

Uses `spirv-tools` in-process validator (no subprocess) with these options:
- `relax_block_layout = true` (permissive for std430)
- `uniform_buffer_standard_layout = true`
- `scalar_block_layout = true`

Validation rules:
- **Library modules** (no entry points): validation skipped
- **Release builds** with spirv-opt enabled: validation skipped (spirv-opt validates internally)
- **Release builds** with `FORCE_SPIRV_VALIDATION = true`: validation forced
- **Debug builds**: always validated

On failure: dumps the SPIR-V binary and disassembly to `/tmp/failed_spv_<hash>.spv` and `.spv.disasm` for debugging.

---

## 12. External Optimization (spirv-opt)

**Functions**: `optimize_spirv_tiered()`, `create_spirv_optimizer()` in `lib.rs`

A tiered strategy based on SPIR-V word count:

| Tier | Word Count | Strategy |
|------|-----------|----------|
| 0 | < 500 | Skip spirv-opt entirely (tiny shaders already optimal) |
| 1 | 500--5000 | Light pass list (10 passes) |
| 2 | > 5000 | Full pass list (30+ passes) |

**Full pass list** (`SPIRV_OPT_PASSES`, Tier 2):

```
MergeReturn, InlineExhaustive, UnifyConstant,
LocalAccessChainConvert, LocalSingleBlockLoadStoreElim,
LocalSingleStoreElim, LocalMultiStoreElim, SSARewrite,
ConditionalConstantPropagation, FoldSpecConstantOpAndComposite,
AggressiveDCE, DeadBranchElim, EliminateDeadFunctions,
EliminateDeadConstant, EliminateDeadMembers, DeadInsertElim,
DeadVariableElimination, CFGCleanup, BlockMerge, BlockMerge,
RedundancyElimination, LocalRedundancyElimination,
InsertExtractElim, CombineAccessChains, CopyPropagateArrays,
PrivateToLocal, Simplification, StrengthReduction, VectorDCE,
LoopInvariantCodeMotion, IfConversion, RemoveDuplicates, CompactIds
```

**Light pass list** (`SPIRV_OPT_PASSES_LIGHT`, Tier 1):

```
UnifyConstant, LocalSingleStoreElim, SSARewrite,
ConditionalConstantPropagation, AggressiveDCE, DeadBranchElim,
EliminateDeadFunctions, EliminateDeadConstant, BlockMerge, CompactIds
```

The optimizer is cached per-thread (`thread_local!`) to avoid re-registering passes when compiling entry points in parallel.

The `SKIP_SPIRV_OPT` global flag bypasses the external optimizer entirely (native passes only).

---

## 13. Manifest Generation

**File**: `metadata.rs`

After code generation, `metadata::generate_manifest()` produces a `ShaderManifest` containing:

```rust
pub struct ShaderManifest {
    pub entry_points: Vec<EntryPointMeta>,       // name, stage, workgroup_size
    pub descriptor_sets: FxHashMap<u32, DescriptorSetLayout>,  // set -> bindings
    pub push_constants: Vec<PushConstantRange>,
    pub source_hash: Option<String>,
    pub dependency_hashes: FxHashMap<String, String>,
    pub validated: bool,
    pub optimized: bool,
    pub buffer_access_patterns: Vec<BufferAccessFingerprint>,  // EL016-EL018
    pub gbt_resource_accesses: Vec<GbtResourceAccess>,         // auto resource usage
}
```

### Buffer Access Fingerprinting

**File**: `buffer_access_analyzer.rs`

Three complementary algorithms run at compile time:

1. **EL016 -- Access Pattern Fingerprinting**: Normalizes index expressions into canonical signatures. Cross-shader comparison detects off-by-one bugs (e.g., shader A writes `heightmap[slot*W^2+z*W+x]` but shader B reads `heightmap[slot*W^2+(z+1)*W+x]`).

2. **EL017 -- Contract-Based Validation**: Known GBT slots have declared index conventions (`GbtSlotContract`). Raw manual indexing on contract-protected buffers triggers a warning; the canonical accessor function should be used instead.

3. **EL018 -- Interval Analysis**: Abstract interpretation propagating `[min, max]` intervals through arithmetic. When the computed interval exceeds buffer bounds, a potential OOB is flagged.

### GBT Resource Access Summary

`gbt_resource_accesses` (derived from fingerprints) records which GBT slots each shader reads/writes. The runtime uses this to auto-generate `ResourceUsage` entries for pipeline barrier computation -- no manual `reads:` / `writes:` declarations needed in mod manifests.

---

## 14. GBT Integration

The GBT (Global Buffer Table) is EvoCraft's central mechanism for GPU buffer access. It is a flat array of `u64` BDA (Buffer Device Address) pointers stored in a push constant, giving every shader access to every engine buffer through a single indirection level.

### How `gbt[SLOT]` Compiles

At the source level, shaders access GBT buffers via BDA pointer casts:

```rust
// Auto-injected by gbt_prologue transform:
let gbt = engine_pc.gbt_addr as &PhysicalStorageBuffer<[u64]>;

// GBT slot access (in shader body):
let voxels = gbt[GBT_SLOT_VOXELS] as &mut PhysicalStorageBuffer<[u32]>;
```

This compiles to SPIR-V as:
1. `engine_pc.gbt_addr` -> `OpAccessChain` into the push constant block
2. `as &PhysicalStorageBuffer<[u64]>` -> `OpBitcast` to `PhysicalStorageBuffer` pointer
3. `gbt[SLOT]` -> `OpAccessChain` with the slot constant as index, then `OpLoad` (u64 address)
4. Second `as &mut PhysicalStorageBuffer<[u32]>` -> `OpConvertUToPtr` or `OpBitcast` to the target BDA pointer type
5. Subsequent `buffer[idx]` -> `OpAccessChain` + `OpLoad`/`OpStore` with `PhysicalStorageBuffer` storage class

### GBT Slot Cache

**File**: `libs/shared/src/gbt_slot_cache.rs`

Without persistence, dynamic slot assignment depends on mod load order. Adding/removing a mod shifts all subsequent slots, breaking compiled shaders.

The `GbtSlotCache` persists at `~/.config/evocraft/gbt_slot_cache.json`:
- Maps buffer names to slot indices
- On startup, pre-reserves all known mappings before new buffers are assigned
- New buffers get the next available slot (appended to cache)
- Cache is saved when modified (`dirty` flag)

The slot cache ensures that GBT slot indices in compiled shaders remain stable across mod changes.

---

## 15. Spec Constants and Template Shaders

### Specialization Constants

Constants can be declared as specialization constants:

```rust
#[spirv(spec_constant(id = 0, default = 0))]
const IDLE_MIN_TICKS: u32 = 180;
```

This emits an `OpSpecConstant` with the given `spec_id` and default value. At pipeline creation time, the runtime provides actual values via `VkSpecializationMapEntry`.

### Template Shaders (Behavior Codegen)

**File**: `manifest_codegen/behavior_gen.rs`

Template shaders are generated from declarative creature behavior definitions (`#[evo_creature_behavior]` in `.evo_mod` files):

1. `BehaviorGen::generate()` reads a `CreatureBehaviorDef` (idle/wander/flee/chase/attack parameters)
2. Emits a complete `.evo_shader` source file using spec constants for tunable values:
   - `IDLE_MIN_TICKS`, `WANDER_SPEED`, `FLEE_RANGE_SQ`, `CHASE_RANGE`, `ATTACK_DAMAGE`, etc.
3. The generated shader is compiled through the normal pipeline
4. At runtime, spec constants are filled from the creature definition

This allows a single compiled SPIR-V binary to serve multiple creature types with different behaviors, specialized at pipeline creation time.

---

## 16. Safety Guards (EvoLint)

### Compile-Time Lint Codes

**File**: `errors.rs`

The compiler defines 23 formal lint codes (EL001--EL023):

| Code | Severity | Description |
|------|----------|-------------|
| EL001 | Error | `entity_id` subtraction without null guard |
| EL002 | Warning | Hardcoded entity stride literal (`* 16`, `+ 8`) |
| EL003 | Error | BDA dereference without null check |
| EL004 | Warning | Atomic increment without overflow guard |
| EL005 | Warning | Division by non-constant variable (potential div-by-zero) |
| EL006 | Error | Barrier inside divergent branch (GPU deadlock) |
| EL007 | Warning | Push constants exceed 128B minimum guarantee |
| EL008 | Error | GPU struct missing `#[repr(C)]` |
| EL009 | Warning | Unused GBT slot import |
| EL010 | Warning | Generic push-constant struct name |
| EL011 | Warning | Assignment to undeclared variable (typo detection with Levenshtein) |
| EL012 | Warning | Narrowing integer cast (silent truncation) |
| EL013 | Error | Zero workgroup dimension |
| EL014 | Error | Mutable uniform buffer parameter (UBOs are read-only) |
| EL015 | Warning | `unsafe` block inside shader |
| EL016 | Error | Inconsistent buffer access pattern between shaders |
| EL017 | Warning | Raw manual index on contract-protected GBT slot |
| EL018 | Warning | Potential OOB buffer access (interval analysis) |
| EL019 | Error | Fragment shader mutable output missing location |
| EL020 | Warning | AUTO_GENERATED file modified (checksum mismatch) |
| EL021 | Warning | Raw numeric index in AUTO_GENERATED file |
| EL022 | Warning | Deprecated `custom_data[N]` usage |
| EL023 | Warning | Integer parameter bit-shift in branched helper (OpUndef risk) |

### Lint Configuration

Lints can be suppressed per-function (`#[allow(EL001)]`), per-file (`#![allow(EL001)]`), or globally via `LintConfig { allow, deny }`.

### Atomic Lint Pass

**File**: `transforms/atomic_lint.rs`

A read-only IR pass that warns about:
- `atomic_add/sub` without subsequent underflow/overflow guard (within 2 statements)
- Results used as array indices without bounds check
- Exceptions: debug/stats counters (heuristic: name contains "dbg"/"debug"/"stat"/"diag")

### Backend Validation

**File**: `backend/validate.rs`

Pre-codegen structural validation:
- `RuntimeArray` must be the last member of a struct

### Capability Validation

**File**: `backend/capabilities.rs`

`CapabilityScanner::validate()` checks that detected capabilities are compatible and no conflicting features are used.

---

## 17. Caching

### Header Cache

**File**: `header_cache.rs`

Library dependencies (`shared/`, etc.) are immutable during a compilation session. The header cache:
- Builds a base `ShaderModule` with all library items pre-imported on first use
- Caches by `(ordered lib roots, max modification timestamp)`
- Subsequent shaders clone the base module (cheap) and start with all headers present
- Eliminates ~14,000 redundant `import_items` calls

### Manifest Cache

**File**: `manifest_cache.rs`

Caches `ExtractedManifest` results for `.evo_mod` files:
- Key: `xxh64(source) XOR BACKEND_VERSION_STR XOR deps_fingerprint`
- Storage: one RON file per mod at `<cache_dir>/<mod_name>.manifest.ron`
- Invalidated when source, compiler version, or shared dependencies change

### Shader Compilation Cache

**File**: `lib.rs` (`BACKEND_VERSION_STR`)

```rust
pub const BACKEND_VERSION_STR: &str = env!("COMPILER_SOURCE_HASH");
```

Computed at build time by `build.rs` -- a hash of all compiler source files. Any change to the compiler automatically busts all shader caches. The shader cache (managed by the engine-runner, not the compiler crate) stores compiled `.spv` + manifests keyed by source hash + backend version + dependency hashes.

### AST Cache

**Files**: `use_tree.rs`, `item_index.rs`

- `get_cached_ast()` / `insert_cached_ast()`: caches parsed `syn::File` ASTs for imported files
- `get_cached_item_index()` / `insert_cached_item_index()`: caches item-kind indexes for fast lookup
- Not cleared between compilations (library files are immutable during a session)

---

## 18. Parallel Compilation

When a shader has multiple entry points, the compiler:

1. Builds a single `ShaderModule` containing all entry points
2. Clones the module per entry point (retaining only one)
3. Generates SPIR-V for each clone **in parallel** via `rayon::par_iter()`
4. Each thread uses its own `thread_local!` cached spirv-opt optimizer

This parallelism is safe because each clone is independent after the split. The header cache and AST cache are shared (read-only after warm-up, protected by `Mutex`).

---

## Architecture Summary

The evo-shader-compiler is a **custom shader compiler** that:

- Uses **Rust syntax** as the surface language (parsed by `syn`)
- Translates to a **custom IR** (`ShaderModule` with typed AST nodes)
- Applies **9+ domain-specific transforms** (worldgen prologue, entity resolution, mod injection)
- Generates **SPIR-V 1.4** via `rspirv` (not rust-gpu, not naga, not glslang)
- Runs **22+ native optimization passes** on the SPIR-V module
- Optionally runs **spirv-opt** with a tiered strategy (30+ passes for large shaders)
- Validates with **spirv-tools** and **23 custom lint rules** (EvoLint)
- Produces manifests with **buffer access fingerprints** for cross-shader validation
- Supports **modding** via hook points, shader transforms, and template generation
