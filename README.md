# EvoCraft Documentation

Official documentation for [EvoCraft](https://github.com/quentin452/EvoCraft) — a GPU-driven voxel game engine with native modding.

## Contents

| Document | Description |
|----------|-------------|
| [RON API Reference](RON_API_REFERENCE.md) | Complete reference for `.evo_mod` manifest declarations |
| [Shader API Reference](SHADER_API_REFERENCE.md) | GPU shader system, `#[evo_buffer]`, `#[evo_dispatch]`, BDA |
| [Shader Compilation Pipeline](SHADER_COMPILATION_PIPELINE.md) | How `.evo_shader` files are compiled to SPIR-V |
| [Manifest Pipeline](MANIFEST_PIPELINE.md) | How mod manifests are parsed, validated, and executed |
| [Hot Reload Status](HOT_RELOAD_STATUS.md) | Live reload capabilities for shaders, textures, and manifests |

## Getting Started

EvoCraft mods are **declarative** — you write `.evo_mod` files (RON format) that declare blocks, entities, GPU services, and behaviors. The engine handles the rest.

```rust
#![evo_mod(
    name = "mod-my-first-mod",
    version = "0.1.0",
    description = "My first EvoCraft mod",
    evo_version = 3,
)]

blocks: {
    "MY_BLOCK": (
        hardness: 2.0,
        textures: ("my_block_top", "my_block_side", "my_block_bottom"),
    ),
}
```

## Related Repositories

- [EvoCraft](https://github.com/quentin452/EvoCraft) — Game engine
- [EvoCraft-ModsRegistry](https://github.com/quentin452/EvoCraft-ModsRegistry) — Community mod registry
- [EvoCraft-Issues](https://github.com/quentin452/EvoCraft-Issues) — Bug reports & feature requests
- [EvoCraft-CrashReports](https://github.com/quentin452/EvoCraft-CrashReports) — Automated crash reports
