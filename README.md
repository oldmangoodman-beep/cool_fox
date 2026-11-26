# cool_fox

[![CI](https://github.com/DoubleGhosty/cool_fox/actions/workflows/ci.yml/badge.svg)](https://github.com/DoubleGhosty/cool_fox/actions/workflows/ci.yml)

Experimental compatibility layer and shader translation runtime for running Xbox Series (Scarlett / GDK) x64 executables natively on Windows D3D12 (AMD Radeon and NVIDIA GeForce).

> **Legal notice:** This repository contains no copyrighted game code, keys, assets, or executables. It does not provide or link to any third-party software, and does not instruct or encourage bypassing console protections. Use only with software you are legally entitled to run. All original code here is licensed under **GPL-3.0-or-later** (see [LICENSE](LICENSE) and [docs/asset-policy.md](docs/asset-policy.md)).

---

## Key Features & Recent Improvements

- **Pareto Memory Residency & 32 GB RAM Support:** Adaptive GPU memory residency limits, preventing WDDM paging exhaustion (`0x8007000E`) on 16–32 GB systems.
- **Pipeline & Descriptor Stability:**
  - `VICTORY_PIPELINE_WAIT_MS=0` & `VICTORY_LIBRARY_STORE_LATER=1` in launchers to eliminate 500 ms frame stalls and 200 ms+ pipeline-library save hitches.
  - Panic-safe pipeline worker queue (`Pending::make_with`) so a failed translation never hangs waiting threads.
  - `submit_split` dynamic descriptor slot release (`list::tables_done`) and `alias_of` reference pinning (`Held` guard) to prevent slot exhaustion and use-after-free during async texture aliasing.
  - DirectStorage staging buffer cleanup and idle list/upload trimming (`VICTORY_LIST_IDLE_TRIM_S`).
- **Shader & Ray Tracing Fixes:**
  - Shader LOD slot fix (leaves unused LOD slots undefined in array texture coordinate queries to pass DXIL validation).
  - Wave32 cascade lowering, scoped submerged shader lowering, and dynamic view address buffer indexing (`SLOT_VIEW_ADDRESSES`) for ray-tracing BVH descriptors.
  - Voxel visibility wave64 mask provenance recovery and blendshape pre-dispatch initialization.
- **NVIDIA & Video Synchronization:** Synchronous flush for decoded video frame copies (`VICTORY_THIN=0`, `VICTORY_JOIN_PARTS=0` by default in launchers for NVIDIA compatibility) and constant ambient lighting (`VICTORY_AMBIENT=constant:0.8`).
- **Wine / Proton & DXIL Robustness:** Code-generated root signatures for `vkd3d-shader`, single-latch DXIL loop CFG normalization, and 256 MiB pipeline worker stack.
- **Custom Resolution, Scaling & Framerate:**
  - `VICTORY_OUTPUT=1920x1080` (or `2560x1440`) and `VICTORY_RENDER_SCALE=0.667..1.0`.
  - `VICTORY_FRAME_INTERVAL_MS=16.667` (60 FPS) or `8.333` (120 FPS) while keeping simulation timing at 1.00x.
- **Optional Renderer Ports (Opt-in):** `VICTORY_PORT_CONSTANTS=1` and `VICTORY_PORT_CBV=1` (off by default in launchers for stability on 8 GB NVIDIA GPUs).

---

## How to Run

### Requirements
- Windows 10/11 x64
- CPU with AVX2 support
- Direct3D 12 GPU with Shader Model 6.6 and Mesh Shaders
- Rust toolchain (see `rust-toolchain.toml`) and MSVC C++ Build Tools (with Windows SDK)

### Build from Source

```powershell
git clone https://github.com/DoubleGhosty/cool_fox.git
cd cool_fox
cargo build --release --locked -p victory
```

### Launch

```powershell
$env:VICTORY_GAME_DIR = 'D:\path\to\your\title'
.\Play-VIctory.cmd
```

For 4 GB VRAM GPUs or low-memory systems:

```powershell
.\Play-VIctory-4GB.cmd
```

*(Note: `Play-VIctory-4GB.cmd` caps card buffers to `VICTORY_CARD_BUFFERS_MB=256` and targets `--console series-s`; on 8 GB+ GPUs, use `Play-VIctory.cmd` instead. On 16 GB RAM machines, ensure an adequate Windows pagefile is configured to avoid commit exhaustion.)*

### Diagnostics & Pre-warming

```powershell
copy victory.example.toml victory.toml      # set your local paths outside this repo
cargo build --locked --profile diag -p victory --features diagnostics
scripts\play.ps1 -Exe <path_to_executable>
```

`scripts\play.ps1 -Safe` disables newer experimental paths; `-RayTracing` enables ray-traced indirect lighting.

To pre-translate shaders or pre-warm cached pipelines ahead of time:

```powershell
target\diag\victory.exe shaders <archive_paths>
target\diag\victory.exe warm --used
```

Commands, debugging switches, and technical notes are documented in [docs/README-fxceless.md](docs/README-fxceless.md) and [docs/findings.md](docs/findings.md). Every `VICTORY_...` environment variable is also documented where it is read in the source.

---

## Architecture Highlights

- **Recorded Command Lists (`VICTORY_THIN`):** List calls are recorded on the title's threads and translated across worker threads.
- **Parallel Submission Prep (`VICTORY_ASYNC_SUBMIT`, `VICTORY_JOIN_PARTS`):** Frame preparation, AVX2 comparisons, descriptor records, and landings are parallelized and joined in submission order.
- **Write-Guard Relay (`VICTORY_RELAY`, `VICTORY_LAND`):** GPU-written buffers are read back by changed dwords on a dedicated relay thread.
- **On-Card Buffer Fills (`VICTORY_CARD_STEPS`):** Buffer fills depending on card-resident bytes execute on the GPU instead of stalling the CPU.
- **Dedicated Texture Copy Queue & Read-Back:** Textures upload on a dedicated copy queue; GPU-written textures read by the CPU are copied back cleanly.
- **Frame Record & Replay:** Per-pass timing and shader variant comparison via `VICTORY_RECORD` ([docs/replay.md](docs/replay.md)).

---

## Credits & Upstream

This repository builds on the work of many contributors across the VIctory ecosystem, preserving commit history and attribution:

- **[VIctory](https://github.com/Marvisak/VIctory)** by Marvisak and contributors: loader, HLE services, D3D12 backend, and shader recompiler.
- **[fxceless/victory-fork](https://github.com/fxceless/victory-fork)**: base fork and upstream pull requests.
- **codec-d / CGBJ**: `cool_fox` runtime and pareto residency architecture.
- **[avatarmanz/VIctory-7900XTX](https://github.com/avatarmanz/VIctory-7900XTX)**: multi-threaded command list recording, framerate unlocking, and renderer ports.
- **[DexNom/VIctory](https://github.com/DexNom/VIctory)**: ray tracing fixes, cube gradients, sparse textures, and Wave64/DXIL passes.
- **[Belobug/VIctory-Belobug-AMD](https://github.com/Belobug/VIctory-Belobug-AMD)**: ambient lighting, pipeline workers, and integer target blend protections.
- **[FQY55kai/ZJOU-VICTORY](https://github.com/FQY55kai/ZJOU-VICTORY)**, **[myokex/victory-amd](https://github.com/myokex/victory-amd)**, **Geckomacabre**, and **cxy0072019-sketch**.
- **[XCL](https://github.com/ivall/xcl)** (GPL-2.0-only) was studied as reference; no XCL code is included.

---

## License

GPL-3.0-or-later. See [LICENSE](LICENSE) and [docs/asset-policy.md](docs/asset-policy.md).
