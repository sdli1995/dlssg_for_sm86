# DLSSG Native 0.2.3

[简体中文](README.md) | English

Windows x64 / D3D12. Install `version.dll` and `dlssg_sm86.ini` beside the actual game rendering executable after exiting the game and preserving the previous files.

The DLL contains the native C++ wrapper, SM75/SM86 PTX/Cubin, model and inference graph. It does not extract, load or memory-map the original frame-generation DLL. Installed NVIDIA NGX/NVAPI/CUDA driver interfaces are still required; CUDA Toolkit is unnecessary.

## What's new in 0.2.3

- **Performance**: verified exact SM86 kernels, fusions, image processing and CUDA buffer clearing are enabled by default; slower experimental paths are removed. On 3080 Ti, offline 4K 4X FG group time falls from 6.748 ms in Release 0.1.0 to 4.654 ms. Full timings and gameplay observations follow below.
- **Configuration**: the everyday INI has five settings. `HardwareBilinear=0` is the default exact mode; `1` enables optional approximate sampling on SM86 only.
- **Package**: SM75/SM86 routes share one package. Four alternate proxy entry points are available in `altnative`, and all five DLLs carry the project self-signature.
- **Compatibility fixes**: the Wukong typeless UI fix and initialization on first Evaluate from 0.2.2 are retained.

## Requirements

- **System and game**: Windows 10/11 x64, D3D12, and a game that can enable DLSS frame generation through this mod. The CPU and system RAM must still meet the game's own requirements.
- **GPU and route**: the SM86 route targets RTX 30-series GPUs; SM75 targets RTX 20-series GPUs. Physical validation currently uses RTX 3080 Ti. SM75 has been tested through forward PTX on that GPU; physical Turing/Cubin validation remains outstanding.
- **Driver and dependencies**: NVIDIA driver NGX/NVAPI/CUDA interfaces are required. These measurements used driver 591.86; that is a tested version, not a declared minimum. CUDA Toolkit and Python are unnecessary for playing.
- **VRAM**: allow for the game itself, additional FG resources and scene-dependent headroom. Windows and other applications also consume VRAM, so consider the memory budget available to the game.

### Additional VRAM by configuration

The following reference budgets use 0.2.3 on RTX 3080 Ti, driver 591.86 and PTX. The 27 additional cases cover SM86 exact, SM86 approximate and the SM75 route. Select by final **output resolution**: 4K output with DLSS Performance still uses the 4K row.

| Output resolution | 2X: estimated additional VRAM | 3X: estimated additional VRAM | 4X: estimated additional VRAM |
|---|---:|---:|---:|
| 1080p / 1920×1080 | About 320 MiB | About 330 MiB | About 340 MiB |
| 2K / 2560×1440 | About 490 MiB | About 510 MiB | About 520 MiB |
| 4K / 3840×2160 | About 700 MiB | About 740 MiB | About 770 MiB |

Values use the warmed-up process-local VRAM increase, subtract the benchmark's preloaded input and test-output allocation, then add one group of `M` 32-bit output buffers. They are rounded up to 10 MiB (1 GiB = 1024 MiB). These are steady-state estimates. Game resources, additional frames in flight, swapchains and larger pixel formats need further memory; **reserve extra headroom above these figures**. The table does not establish a game's minimum card capacity or peak usage. SM75 figures are route references measured on 3080 Ti.

Exact and approximate modes used the same VRAM in these measurements. The SM75 route differed by less than 1 MiB and uses the same rounded budget. 2X/3X/4X reuse resident inference resources, so lowering the multiplier mainly reduces output buffers and may not substantially reduce total VRAM usage.

**Insufficient VRAM or exceeding the Windows-assigned memory budget can cause occasional stutters and frame-time spikes, even when average FPS looks normal.** Lower texture quality, output resolution or ray tracing, and reduce background VRAM usage to leave room for scene changes and resource loading. [Microsoft video-memory budget guidance](https://learn.microsoft.com/en-us/windows/win32/api/dxgi1_4/nf-dxgi1_4-idxgiadapter3-queryvideomemoryinfo)

## Installation and upgrade

Requirements: Windows x64, a D3D12 game and NVIDIA drivers. Python and CUDA Toolkit are unnecessary for playing. The current model is 310.1; Vulkan is planned for a later release.

1. Exit the game. For an upgrade, back up this project's previous proxy DLL and INI to a separate directory and remove its old proxy from the game directory. Preserve other mods' files.
2. Find the actual rendering EXE. For Black Myth: Wukong, this is `D:\SteamLibrary\steamapps\common\BlackMythWukong\b1\Binaries\Win64`, containing `b1-Win64-Shipping.exe`.
3. Copy **one proxy DLL and `dlssg_sm86.ini`** beside that EXE. The default is the root `version.dll`. Alternatives are `altnative/winmm.dll`, `dinput8.dll`, `winhttp.dll` and `dxgi.dll`: select a name the game loads, preserve its filename, and keep only one proxy from this package installed.
4. A 3080 Ti uses `Router=SM86, KernelImage=PTX`; Turing/SM75 uses `Router=SM75, KernelImage=PTX`. The INI is shared by all entry points.
5. Restart, enable DLSS frame generation and select the multiplier in the game. `MaxGeneratedFrames=3` permits up to three generated frames, or 4X total; the game selects the actual count.

Every proxy contains the complete native runtime. Preserve conflicting DLLs owned by other mods and select a different available entry point. Do not mix this package with upstream SM75 proxy/injector/backend files.

Default exact sampling is `HardwareBilinear=0`; optional approximate sampling is `1`, applies only to SM86, and may change generated pixels. Restart after editing the INI. See [configuration](docs/NATIVE_INI.md).

For loading diagnostics, temporarily set `Logging.Level=2` and inspect `dlssg_sm86/logs` beside the EXE. If no project log appears, check the EXE directory and whether the game loads your chosen DLL. Restore `Level=1` for normal use. Keep the game's original DLSSG files. To uninstall, exit the game and remove your selected project proxy and INI; restore your backup to roll back.

## Antivirus false positives and signing

This mod uses a system DLL proxy and LoadLibrary hooks to integrate with the game. Such behavior may trigger heuristic false positives. Native integration removes extraction and manual mapping of the original feature DLL, while the integration hooks remain necessary. The relevant security vendor must review the specific detection to determine whether it is a false positive.

All five DLLs are signed with the **DLSSG Native Project self-signed certificate**, visible under Digital Signatures in Windows file properties. The signature verifies signer identity and file integrity; **it does not establish default Windows trust or guarantee the absence of antivirus alerts**. An untrusted certificate chain, a SmartScreen reputation warning and a malware detection are separate checks. Self-signed files can still receive SmartScreen warnings. [Microsoft SmartScreen guidance](https://learn.microsoft.com/en-us/windows/apps/package-and-deploy/smartscreen-reputation)

If a detection occurs, first check the download source and the release ZIP against its `.sha256` sidecar. Record the security product, detection name, definition version and detected DLL's SHA256, then request a false-positive review from that vendor. For Microsoft Defender, use [Microsoft file analysis](https://www.microsoft.com/en-us/wdsi/filesubmission). Matching hashes and signatures do not replace the vendor's detection assessment.

## Unified baseline: Release 0.1.0 → Native 0.2.3

Every row uses the **2026-09-07 `dist/Release/version.dll`**, SHA256 `03d445237d519ac48cd9226278a0f07aecd7ac597697697eb64404e1d51b3c5a`, as the baseline. Both modes were measured with the unsigned 0.2.3 DLL `f5715c29…`; the release adds a signature with an identical PE image digest. All nine conditions were freshly measured on RTX 3080 Ti / SM86, driver 591.86.

Times are **GPU milliseconds for the entire frame-generation group per real frame**, including shared preprocessing. 2X/3X/4X generate 1/2/3 frames. Exact is the default (`HardwareBilinear=0`); approximate sampling is optional (`1`).

| Resolution | Multiplier | Release 0.1.0 (ms) | 0.2.3 exact (ms) | Reduction | 0.2.3 approximate (ms) | Reduction |
|---|---:|---:|---:|---:|---:|---:|
| 1080p | 2X | 1.528 | 0.996 | 34.82% | 0.985 | 35.55% |
| 1080p | 3X | 2.253 | 1.594 | 29.26% | 1.574 | 30.11% |
| 1080p | 4X | 2.977 | 2.185 | 26.61% | 2.157 | 27.54% |
| 2K / 1440p | 2X | 2.477 | 1.632 | 34.12% | 1.616 | 34.77% |
| 2K / 1440p | 3X | 3.674 | 2.591 | 29.49% | 2.557 | 30.40% |
| 2K / 1440p | 4X | 4.882 | 3.545 | 27.39% | 3.493 | 28.45% |
| 4K | 2X | 3.138 | 2.090 | 33.38% | 2.048 | 34.74% |
| 4K | 3X | 4.994 | 3.407 | 31.77% | 3.326 | 33.39% |
| 4K | 4X | 6.748 | 4.654 | 31.03% | 4.535 | 32.80% |

Reduction is `(Release time − current time) / Release time`, calculated before rounding. Times are medians of run medians. Each condition has four rounds, a Release run before and after each round, and alternating exact/approximate order. Each run measures 256 groups: 144 runs in total. Clocks were not locked; no outliers were removed.

All modes use identical synthetic Wukong-format inputs, PTX, a HIGH=100 compute queue, and independent submission of each Evaluate. A 1.5-second load soak is followed by Reset and 64 warm-up frames. Old Release Auto/Cubin is explicitly overridden to PTX. Its original 310.1 feature DLL is loaded explicitly with a matching payload hash; initialization, extraction and loading costs are outside timing.

Post-timing exact outputs match Release byte for byte. Approximate mode preserves real frames and alpha while changing generated RGB. The group GPU span includes submission gaps between Evaluate calls. Rendering, Present, uploads and readbacks are excluded; these reductions are not measured game FPS gains.

## Estimating FPS with frame generation

First disable frame generation in the **same scene, at the same output resolution, DLSS Super Resolution mode and graphics settings**, and measure `F_off`. Convert it to a base frame time with `1000 / F_off`, then select the whole-group frame-generation time `T_FG` in milliseconds from the table above for your resolution, multiplier and exact/approximate mode.

```text
Base frame time T_base (ms) = 1000 / F_off
Frame-group time with FG T_group (ms) ≈ T_base + T_FG
Real-frame / group rate G (groups/s) ≈ 1000 / T_group
Total FPS with FG F_out ≈ G × M
                      = 1000 × M / (1000 / F_off + T_FG)
```

`M` is the total multiplier: 2, 3 or 4 for 2X, 3X or 4X. A group contains one real frame and `M − 1` generated frames. **The table's `T_FG` already includes all generated frames and shared preprocessing; do not multiply it by `M − 1` again.** The real-frame/group rate `G` with FG is lower than the no-FG rate `F_off` in this estimate.

For example, **50 FPS without FG** gives a **20 ms** base frame time. Using the RTX 3080 Ti / SM86 **4K 4X** group timings above:

| Configuration | FG group time T_FG (ms) | Estimated frame-group time (ms) | Estimated real-frame / group rate (groups/s) | Estimated total FPS |
|---|---:|---:|---:|---:|
| Release 0.1.0 | 6.748 | 26.748 | 37.4 | 149.5 |
| 0.2.3 default exact (HardwareBilinear=0) | 4.654 | 24.654 | 40.6 | 162.2 |
| 0.2.3 optional approximate (HardwareBilinear=1) | 4.535 | 24.535 | 40.8 | 163.0 |

For 1080p, 1440p or 2X/3X, use the matching row and your own measured `F_off` at those settings. These timings were measured on 3080 Ti / SM86; other GPUs or the SM75 route need their own group timings.

This is a **rough additive estimate** of base rendering time plus FG overhead. GPU resource contention, synchronization, CPU overhead, frame caps and display refresh rate affect actual results; estimated FPS need not match either an FPS counter or the rate of frames actually displayed.

## Black Myth: Wukong gameplay feedback

User-reported approximate readings from the same scene on **RTX 3080 Ti, 4K output, DLSS Performance, Full Ray Tracing off, all graphics settings at Cinematic**. With FG disabled, performance is about **50 FPS**. With **4X FG**:

| State | Real-frame / group rate | Total FPS including generated frames |
|---|---:|---:|
| Before optimization | About 36 groups/s | About 144 |
| After this optimization | About 40 groups/s | About 160 |

Both rates improve by about **11.1%**: **+4 groups/s and +16 FPS**. These are the user's gameplay observations, separate from the calculated estimates; no new automated game test was run for this documentation update. The result is close to the default exact estimate of about 162 FPS, but the 31.03% reduction in offline FG time does not translate directly into the same percentage increase in game FPS.

## Configuration

The supplied INI has five keys: `Router=SM86`, `KernelImage=PTX`, `HardwareBilinear=0`, `MaxGeneratedFrames=3`, and `Logging.Level=1` (errors only). The game chooses the actual multiplier up to 4X. Restart after changes.

Use `Router=SM75` for Turing or SM75 forward-PTX testing. On a 3080 Ti, SM75 requires PTX or Auto. Cubin requires an exact physical SM/router match. SM75 keeps its own kernels and original clear operations; SM86 fusions, CUDA clearing and approximate sampling do not apply to that route. Physical Turing/Cubin validation is still outstanding.

Default and approximate presets are in `config/presets`. Obsolete optimization keys in older INI files are ignored. See [INI details](docs/NATIVE_INI.md).

## Scope

The 0.2.2 typeless UI fix and initialization on first Evaluate are retained. Explicit forward/reverse clip matrices are required. This release supports up to three generated frames; 6X/dynamic multipliers, Reflex Warp and automatic Reflex matrix lookup are not implemented. The caller owns queue submission, synchronization and presentation. Offline GPU savings are not measured game FPS gains.

Set `Logging.Level=2` or `3` for diagnostics; logs are written to `dlssg_sm86/logs`. SM75 forward PTX has been checked on 3080 Ti; physical Turing/Cubin and extended gameplay validation remain outstanding. Keep the game's original DLSSG files. Remove this package's DLL and INI to uninstall.

## SM75 attribution and thanks

Thanks to **Coldwood1026** for the RTX 20-series / SM75 adaptation. GPU assets come from [dlssg_for_sm75](https://github.com/Coldwood1026/dlssg_for_sm75), formerly `dlssg_for_sm86`, pinned to [c60c2aa…](https://github.com/Coldwood1026/dlssg_for_sm75/commit/c60c2aa363c7e66a523122aa5cec9c884658ad5f). This project loads and schedules the GPU resources through its own native host. See `THIRD_PARTY_NOTICES.txt` for provenance and licenses.

## Next release plan

1. **Vulkan support**: resource integration, interoperability and synchronization, tested on both SM75 and SM86 routes.
2. **Update the model to the latest DLSSG**: pin the latest available version and hashes when implementation starts; adapt the model/graph and evaluate quality, memory and GPU time.

The current release remains D3D12 with the 310.1 model. See the [roadmap](docs/ROADMAP.md).
