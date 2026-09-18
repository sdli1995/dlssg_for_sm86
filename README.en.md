# DLSSG for SM86 (proxy) - 0.3.4 Version

[中文](README.md) · **English**

Enables NVIDIA DLSS Frame Generation (DLSS-G) on RTX 30-series (SM86) and RTX 20-series (SM75). Windows x64 / D3D12; the runtime files are `version.dll` and `dlssg_sm86.ini`.

## Changes in this release

### 0.3.4

- Fixes the 0.3.3 crash on RTX 30 (GPU driver reset at startup or once DLSS is on; Forza Horizon 6 shows FHC01; issues #535 / #538 / #540 / #542). Cause: with the NVIDIA App DLSS override or an NGX OTA model update active, the DLSS Super Resolution model also received the GPU-architecture rewrite this project makes for Streamline, took a path that does not belong to an RTX 30 and hung the GPU. NVIDIA's own components now always get the real architecture; the rewrite applies to Streamline and the game only. On 0.3.3, just replace `version.dll`.

### 0.3.3

- RTX 20 / 30: the GPU-architecture rewrite shown to the game is now armed at game start and reports RTX 50. Games on Streamline 2.8 (e.g. Final Fantasy VII Rebirth) used to decide "this GPU does not support DLSS-G" during startup and drop the frame-generation plugin (issues #509 / #528); the rewrite is now in place before that check, and the game's 3X / 4X / 6X options unlock as on an RTX 50.
- `Optimized=1` no longer skips the repeated real-frame copy within a group (`SkipRepeatedRealCopy`; off at every level, set it to `1` yourself if wanted). It had never been validated in a live game, and a flicker report followed on 0.3.2 (issue #532). Level `1` remains bit-identical to the official image.

### 0.3.2

- Part of the inference kernels rewritten (310.9 build): the generated image now matches the official DLSS-G exactly (bit-identical in tests on an RTX 3080 Ti and an RTX 5070; an RTX 2080 Ti's output is bit-identical to the 3080 Ti's), no longer lossy, with a small speed-up (0–8% on the 3080 Ti).
- Optimization levels reworked: `[FrameGeneration] Optimized` is now `0`–`3`. `0` stock kernels, no acceleration; `1` every acceleration, image bit-identical to the official one (factory default); `2` adds lossy image kernels, faster, about 50 dB PSNR or better against the official image (310.9 build only); `3` everything lossy, fastest. See [`docs/INSTALL.en.md`](docs/INSTALL.en.md).

### 0.3.1

- Fixed frame generation not enabling on RTX 20-series (Turing). RTX 20 now works with the factory `dlssg_sm86.ini` like RTX 30, including 6X (310.9 build, `MaxGeneratedFrames=5`, game plugin permitting).
- Factory `MaxGeneratedFrames` is now `3` (4X); set `5` for 6X.
### 0.3.0

- Reverted from the native build to the proxy build. The native approach (a self-built NGX host) had game-compatibility problems that were hard to fix; this build uses a proxy DLL around the unmodified factory runtime, leaving the game's calls to NGX unchanged, which is more compatible.
- The 310.9 runtime adds 6X (`MaxGeneratedFrames` ceiling raised from 3 to 5). On games that themselves support Dynamic MFG, selecting "Dynamic / Auto" frame generation reaches 6X.
- Optimized kernel set (~19–32% in the offline benchmark), a two-switch factory INI, capture/replay, and diagnostic logging — see below and `docs/`.

## Statement & roadmap

- Kernel-level work on frame generation (DLSS-G) has essentially reached the best this project can do at this stage; kernel optimization is paused, and later releases will carry compatibility and bug fixes only.
- The next release focuses on INT8 optimization of several Transformer super-resolution models, to raise the base frame rate on RTX 30 / RTX 20.
- Vulkan support is hard to continue in the project's current state: the wrapper layer itself still has weak game compatibility and little coverage testing, and adding Vulkan on top would only bring more compatibility problems. If you need it, use one of the community patch builds.

## Requirements

- OS/game: Windows 10/11 x64, D3D12.
- GPU: RTX 30-series (SM86) or RTX 20-series (Turing / SM75). The full offline benchmark was run on a 3080 Ti and development validation on a 3070; RTX 20 was confirmed working on a 2080 Ti, whose generated image for the same inputs is bit-identical to the 3080 Ti's; performance is not yet measured.
- Driver: an NVIDIA driver with the NGX / NVAPI / CUDA interfaces; tested on 591.86 and 610.74. The cubins need roughly R580+; older drivers fall back to PTX automatically (one extra JIT on the first frame only).
- No CUDA Toolkit and no Python.

Both release zips install the same way; only the embedded runtime and the ceiling differ. The 310.9 build matches the 310.1 build at 4X and below, and additionally supports 6X. The root directory contains the latest DLSSG version, 310.9, while 310.1 is the older version.


## Extra VRAM by configuration

Frame generation's own local VRAM delta (after warm-up minus before feature creation, rounded up to 10 MiB, 310.9 build, optimized). It tracks the output resolution only and is independent of the multiplier: 2X and 6X use the same amount; the multiplier costs no extra VRAM.

| Aspect | Output resolution | Extra VRAM |
|---|---|---|
| 16:9 | 720p | ~230 MiB |
| 16:9 | 1080p | ~350 MiB |
| 16:9 | 1440p | ~540 MiB |
| 16:9 | 4K / 3840×2160 | ~810 MiB |
| 21:9 | 2560×1080 | ~440 MiB |
| 21:9 | 3440×1440 | ~700 MiB |
| 21:9 | 5120×2160 | ~1050 MiB |
| 32:9 | 3840×1080 | ~610 MiB |
| 32:9 | 5120×1440 | ~990 MiB |
| 4:3 | 1920×1440 | ~430 MiB |

The same height at different widths is similar (it tracks output pixel count). This is frame generation's own increment, excluding the game itself and the synthetic inputs.

## Install & upgrade

1. Exit the game completely.
2. Go to the game's rendering-EXE directory (e.g. Black Myth: Wukong is `...\b1\Binaries\Win64\`).
3. Copy `version.dll` and `dlssg_sm86.ini` into it; if a `version.dll` already exists, back it up first. If a game does not load `version.dll`, use one of the other proxy names in `alternatives\` (pick the DLL the game actually loads — e.g. `winmm.dll` / `dxgi.dll` / `dbghelp.dll`).
4. Launch the game, enable DLSS Frame Generation in the graphics settings, and select 2X / 3X / 4X (up to 6X on the 310.9 build where the game supports it).
5. Upgrade: exit the game and overwrite `version.dll`; `dlssg_sm86.ini` usually needs no change.
6. Uninstall: overwrite `version.dll` with the backed-up original (or delete it) and delete `dlssg_sm86.ini`.

The factory `dlssg_sm86.ini` keeps only two decisive switches: `[FrameGeneration] Optimized` (optimization level `0`–`3`: `0` stock kernels, no acceleration; `1` every acceleration, image bit-identical to the official one, factory default; `2`/`3` lossy but faster) and `[FrameGeneration] MaxGeneratedFrames` (factory `3` = 4X; `5` = 6X on the 310.9 build only; the actual count is requested by the game and clamped to the runtime's ceiling). Every other diagnostic/compatibility knob takes a safe default and is omitted; the full list is in [`docs/INSTALL.en.md`](docs/INSTALL.en.md).

## Antivirus & signing

The release proxy DLLs (`version.dll`, `winmm.dll`, and each proxy in `alternatives\`) are code-signed. A self-signed certificate only verifies the signer's identity and file integrity; it gives no default Windows trust, so Windows SmartScreen may still prompt "unknown publisher" on first run — that is a reputation prompt, not an antivirus detection. The certificate is self-signed as `CN=DLSSG for SM86 (self-signed)`, SHA-1 thumbprint `85BA66762F851E49148D706915D09026281418E6`; verify the signer and thumbprint via the file's Properties → Digital Signatures tab or `signtool verify /pa`.

## Performance (RTX 3080 Ti, offline benchmark)

RTX 3080 Ti, driver 591.86, SM86, measured 2026-09-13. The unit is GPU milliseconds for the whole generation group per real frame, shared preprocessing included; 4 rounds × 256 groups each, median of the per-round medians, configs interleaved within a round to share thermal drift. The table is the 310.9 build, common 16:9 resolutions: stock kernels (`Optimized=0`) versus optimized kernels (`Optimized=1`). Reduction is `(stock − optimized) / stock` on un-rounded data.

This table measures frame generation's GPU compute cost; it is not an in-game FPS gain — how to estimate displayed FPS from it is in the next section. The optimized kernel set itself is unchanged since this measurement (the same 63 variants + image patches + cross-kernel fusions); 0.3.2 only swaps the images of the 26 kernels new in the 310.9 build, which measured another 0–8% faster on the same card, so these numbers are conservative for this release.

| Resolution | Multiplier | Stock (ms) | Optimized (ms) | Reduction |
|---|---|---|---|---|
| 720p | 2X | 1.135 | 0.779 | 31.4% |
| 720p | 3X | 1.766 | 1.307 | 26.0% |
| 720p | 4X | 2.405 | 1.839 | 23.5% |
| 720p | 5X | 3.043 | 2.372 | 22.0% |
| 720p | 6X | 3.680 | 2.907 | 21.0% |
| 1080p | 2X | 1.389 | 0.949 | 31.7% |
| 1080p | 3X | 2.011 | 1.482 | 26.3% |
| 1080p | 4X | 2.641 | 2.021 | 23.5% |
| 1080p | 5X | 3.269 | 2.563 | 21.6% |
| 1080p | 6X | 3.888 | 3.099 | 20.3% |
| 1440p | 2X | 2.143 | 1.491 | 30.4% |
| 1440p | 3X | 3.156 | 2.324 | 26.4% |
| 1440p | 4X | 4.163 | 3.160 | 24.1% |
| 1440p | 5X | 5.206 | 4.008 | 23.0% |
| 1440p | 6X | 6.197 | 4.849 | 21.8% |
| 4K | 2X | 2.605 | 1.987 | 23.7% |
| 4K | 3X | 4.091 | 3.214 | 21.4% |
| 4K | 4X | 5.577 | 4.442 | 20.4% |
| 4K | 5X | 7.066 | 5.667 | 19.8% |
| 4K | 6X | 8.557 | 6.902 | 19.3% |

The gain is larger at lower resolution and lower multiplier (closer to launch/latency bound). The 310.1 build is close to the above at 2X–4X. Data for 21:9 / 32:9 / 4:3 and a second "sum of per-Evaluate spans" table are under `docs/evidence/`. Group times at 3X and above can be bimodal (present in every implementation), so the median may sit between the two peaks; raw min/mean are in the data JSON.

## Estimating FPS after frame generation

First, in the same scene, output resolution, DLSS upscaling mode, and quality settings, turn frame generation off and read the frame rate `F_off`. Convert to a base frame time with `1000 / F_off`, then take the whole-group generation time `T_FG` (ms) from the table above for that resolution, multiplier, and kernel setting.

```text
base frame time    T_base (ms) = 1000 / F_off
group time         T_group (ms) ≈ T_base + T_FG
real-frame/group rate G (grp/s) ≈ 1000 / T_group
output frame rate  F_out (FPS)  ≈ G × M = 1000 × M / (1000 / F_off + T_FG)
```

`M` is the multiplier (2X…6X → 2…6). A group is one real frame plus `M − 1` generated frames; `T_FG` already covers the whole group and the shared preprocessing, so do not multiply it by `M − 1`. With generation on, the real-frame/group rate `G` is lower than `F_off`.

Example: with frame generation off, ~**50 FPS** (20 ms base frame time), using the **4K 4X** `T_FG` from the table:

| Kernels | T_FG (ms) | group time (ms) | group rate (grp/s) | estimated FPS |
|---|---|---|---|---|
| Stock | 5.577 | 25.577 | 39.1 | 156.4 FPS |
| Optimized | 4.442 | 24.442 | 40.9 | 163.7 FPS |

For other resolutions/multipliers use the matching row and your own measured `F_off` for that setting. This is a rough estimate — the base render time plus the generation-group cost; GPU contention, synchronization, CPU overhead, frame caps, and the monitor refresh rate all affect the real result, and the estimate is not guaranteed to equal a counter reading or the actual displayed rate.

## 6X

6X (5 generated frames per real frame) is NVIDIA's DLSS 4.5 Dynamic Multi Frame Generation. The runtime embedded in the 310.9 build supports it and this project makes it run on Ampere; whether you get 6X depends on the game:

- The game itself supports 6X (ships a newer Streamline frame-gen plugin and offers 6X or Dynamic MFG in its menu): install the 310.9 build with `MaxGeneratedFrames=5`; 6X runs stably in testing.
- The game only supports 4X (ships an older 4X plugin, as most current games do): the ceiling is set by the game's plugin and this project cannot raise it to 6X. That plugin sizes its internal present queue for 4X at init, so forcing extra frames overruns it and disables frame generation or crashes. Use 4X for these games.

## Real-world feedback

Black Myth: Wukong, Cyberpunk 2077, and FH6 run 4X normally in testing; Resonance A Plague Tale Legacy runs 6X by default on the 310.9 build with "Dynamic / Auto" frame generation. Image quality depends on the base frame rate: at a low base rate, generated frames can show breakup and edge artifacts, more so at higher multipliers (6X is more demanding than 4X), and some games need graphics settings lowered, to raise the base rate, even at 4X for a good result. This is inherent to frame generation when there is little frame-rate headroom, not a defect of this project.

## Diagnostics & boundaries

- Logs go to `dlssg_sm86\logs\` in the game directory (`loader_<PID>.jsonl` / `backend_<PID>.jsonl`). `[Logging] Level=1` (default) records errors only; use `2` or `3` when investigating.
- If frame generation does nothing, check `backend_*.jsonl` for an `install` line with `route active=true`; if it is absent, the driver/runtime usually did not match — the reason is logged and the factory path is used.
- This release optimizes frame generation's GPU compute cost; do not read the offline time reduction as an in-game FPS gain — the real frame-rate change depends on the game and where the bottleneck is.
- All INI keys, log fields, and capture/replay are in [`docs/INSTALL.en.md`](docs/INSTALL.en.md) and [`docs/CAPTURE.md`](docs/CAPTURE.md).

## SM75 source & credits

- Coldwood1026 for the RTX 20-series / SM75 adaptation (the kernel family behind the experimental SM75 route — see `THIRD_PARTY_NOTICES.txt`).
- NVIDIA for the DLSS-G runtime, models, and pre/post-processing (embedded, unmodified).

## License & third-party

- The project source is GPLv3.
- The embedded `nvngx_dlssg.dll` (310.1 SHA prefix `c989c0eb…`, 310.9.1 SHA prefix `ff6e90eb…`), the extracted/recompiled kernel resources, and `assets/kernels/sm75/` are NVIDIA and upstream third-party material, not re-licensed under GPL — see `THIRD_PARTY_NOTICES.txt`.
