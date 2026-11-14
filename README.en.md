# DLSSG for SM86 (proxy build) - 0.4.2

[中文](README.md) · **English**

Enables NVIDIA DLSS Frame Generation (DLSS-G) on RTX 30-series (SM86) and RTX 20-series (SM75), and accelerates DLSS Super Resolution with INT8 on RTX 30 / 20. Windows x64 / D3D12; the runtime files are `version.dll`, `dlssg_sm86.ini` and the `dlssg_sm86\` directory.

> **Upgrading from 0.3.x? Read this first: 0.4.0 is a major upgrade. The file layout and the install steps changed; do not copy it over the old version.**
>
> 1. Remove the old version completely: quit the game, then delete every proxy DLL of this project from the game directory (0.3.1–0.3.6 had you copy four, `version.dll`, `winmm.dll`, `dbghelp.dll` and `dinput8.dll`; delete all four), `dlssg_sm86.ini`, and the whole `dlssg_sm86\` directory.
> 2. Copy the three items of the new version: `version.dll`, `dlssg_sm86.ini` and the whole `dlssg_sm86\` directory. The runtime, backend, kernel pack, XeSS model pack and super-resolution files all live in `dlssg_sm86\`; copying only the DLL does nothing.
> 3. Only one proxy at any time: the root has only `version.dll`; if the game does not load it, swap in one from `alternatives\` (swap, do not add).
> 4. Use the new `dlssg_sm86.ini`: keys moved (`Optimized` and `Preset` are now in `[DLSSG]`, `[XeSSFG]` is new) and `[Runtime] Mode=Auto` was removed. The old places of `Optimized` / `Preset` are no longer read: written only there with a non-default value (such as an old INI's `[FrameGeneration] Optimized=1`), frame generation is not enabled and one `configuration_invalid` is logged; redo your changes in the new INI.
>
> Step-by-step: "Install & upgrade" below and the manual `MANUAL.en.html`.

## Changes in this release

### 0.4.2

- **XeSS-FG performance fixed on RTX 50 (Blackwell)**: 0.4.1 shipped compute_75 PTX to RTX 40 / 50, and ptxas turns every `m8n8k16` into one full `m16n8k16` on sm_120 and throws half of it away (144 instructions against 72 for the trunk convolutions, registers 79 → 124) - the tensor-core kernels were about twice as slow there as the Ampere shape. 0.4.2 builds 26 kernels as their own `.bw` images (compute_86 PTX with 3x3 taps paired into `m16n8k32`), and the dispatches that are faster as DXIL on RTX 50 are now only D25 and model 15's D02 / D30. Cyberpunk 2077 at 3440x1440 (RTX 5070): 4X 6.54 → 4.68 ms per real frame, 46.5 → 52.0 fps; 2X 2.43 → 1.69 ms (57.8 → 59.8 fps); DLSS 4X on the same card is 3.98 ms.
- **XeSS-FG handles jittered motion vectors (`DLSSG.MvecJittered=1`)**: 3X / 4X no longer end with `xefg_kernel_fatal{unsupported_input}` - D31 runs four `jt_` kernels carrying model 18's jitter term `D = (jp - jc) * (1 - t)`, bit-identical to the plain path when the term is zero. It costs about +0.08 ms per generated frame at 1440p RGBA8 and +0.17 ms at 4K.
- **4K optimisations**: model 18's D24 / model 15's D29 MV splat is now this project's own CUDA kernel, and the scatter buffers are cleared only over the prefix this or the last scatter can write. Another 3.0-5.6 % per real frame at 4K, byte-identical output.
- **New `dlssg_sm86ini_Configurator.html`**: a single-file, dependency-free INI configurator - per-section editing of the common keys, a live INI preview, loading an existing INI (file picker or drag and drop), download / copy, Chinese-English switching, and built-in checks (`MaxGeneratedFrames` 1-5, `Engine=XeSS` 1-3 only, a stale `Model=` key, and so on). Ships with the release package.
- **`[XeSSFG] UseHudless=Auto` now behaves as `0`**: the back buffer is the network source and the whole image, UI included, is interpolated whether the game supplies a HUD-less image or not (`xefg_hudless_unused` no longer appears); `1` stays strict (no usable HUD-less image means `xefg_kernel_fatal{hudless}`). The key's values and factory value are unchanged (`Auto`).
- Version 0.4.2; `README` / `MANUAL` / `INSTALL` / `COMPAT` updated in step (`COMPAT` gains C139).

### 0.4.1

- XeSS-FG engine (optional):
  - Multi-frame generation (MFG): `[FrameGeneration] Engine=XeSS` goes to 2X / 3X / 4X (`MaxGeneratedFrames` 1-3). The model is Intel's model 15 as ported by MFG-Unlocker 1.6.0: 2X still runs model 18's graph (4-5 % more than before), 3X / 4X run model 15's own multi-frame path with the i-th generated frame of a real frame at t = i / (N + 1). The `[XeSSFG] Model` key is removed (one model); an old INI that writes it gets one warning.
  - Tensor-core CUDA backend: the port's trunk layers (3x3 INT8 convolutions) are hand-written INT8 IMMA kernels (`mma.sync.m16n8k16`) with `cp.async` double buffering; the whole graph runs through this project's own D3D12 + NVAPI CUDA runner on the game's command list and never loads Intel's `libxess_fg.dll`.
  - `[XeSSFG] FlowResolution=CoarseAt4K`: `Coarse` automatically when the output has at least 3840 x 2160 pixels, `Middle` at 1440p.
  - `[XeSSFG] UseHudless` is now three-state: `Auto` (factory, decided per frame) / `1` (strict: a frame without a usable HUD-less image is refused) / `0` (never).
  - On RTX 30 it is 8-15 % faster than DLSS-G at 1440p and below, and 27-30 % slower from 4K up (see "Performance").
- `[SuperResolution] Mode` is now explicit presets: `Off` (factory) / `M-INT8` / `L-INT8` / `M-FP16` / `L-FP16`; 0.4.0's `Mode=On` is removed, and an invalid value is `configuration_invalid` with super resolution not enabled.
- `[DLSSG] Preset` is A / B / C only: `Auto` is removed (it runs as `A`), factory `A`.
- Keys removed and no longer read: `Optimized` / `Preset` in their old places, `[Compatibility] AccelerateNewerGpus` / `ForceGeneratedFrames` / `SM75Family` / `HardwareBilinear` / `LaunchChains` / `PlainVariant`, `[Backends]`, and more; writing one logs a warning and the factory value applies. RTX 40 / 50 always run this project's kernels.
- The R580 driver floor is now hard: a CUDA 13 cubin the driver refuses is an error that refuses frame generation (`driver_below_floor` / `kernel_image_unavailable`), no longer a quiet fall back to PTX.
- Super-resolution INT8 first-frame hitch fixed: the twelve weight-image uploads moved off the first observed Evaluate (that Evaluate's CPU time used to be 5-6 ms higher than the same preset in FP16).
- An explicit switch that cannot be carried out is refused: a `[Runtime] Path` / `SRPath` naming a wrong file, a missing packaged file, `KernelImage=Cubin` on RTX 40 / 50 and the like are refused with the reason in the log, never replaced by another path.
- Logging: one text log, `dlssg_sm86\logs\dlssg_sm86.log` (0.4.0's per-process `frames_<pid>.jsonl` is gone); `[Logging] FrameTiming=1` writes `frame_timing.txt`.


### 0.4.0

- Super-resolution INT8 (RTX 30 / 20 only): with `[SuperResolution] Mode=On`, the super-resolution modes the game runs with preset L or M (Performance and Ultra Performance in most games) use this project's INT8 kernels; no preset is changed. On an RTX 3070 one super-resolution evaluate takes L −6..−16 %, M −9..−22 % GPU time against the same preset in FP16. Factory `Mode=Off`. (`Mode=On` was removed on 2026-10-19; the values are now `M-INT8` / `L-INT8` / `M-FP16` / `L-FP16`.)
- DLSS-G preset C (FG Lite, optional): `[DLSSG] Preset=C`. Each generated frame skips the second network and its post-processing; on RTX 3070 real captures the GPU time per generated frame is −29..−34 % at 2X and about −45 % at 4X, with slightly lower image quality (mostly on the HUD and at occlusion edges). Factory is still `Preset=Auto` (`Auto` was removed on 2026-10-22; the factory value is now `A`).
- XeSS-FG engine (optional): `[FrameGeneration] Engine=XeSS` generates frames with Intel's XeSS-FG model: Intel's model 15 (as ported by MFG-Unlocker 1.6.0) goes to 3X / 4X with model 18's graph at 2X; the one model since 2026-10-26, the `[XeSSFG] Model` key is removed (no longer read); `[XeSSFG] FlowResolution` (`Coarse` / `Middle` / `Fine`), `UseHudless`. Intel's `libxess_fg.dll` is neither shipped nor loaded: the model runs on this project's own CUDA kernels. On RTX 30 it is slower than DLSS-G, see "Known issues". Factory is still `Engine=DLSS`.
- The bundled runtime is now `nvngx_dlssg.dll` 310.9.16 (bit-identical output to 310.9.1 under the default preset). The version the game ships does not matter.
- New INI layout: the DLSS-G-only `Optimized` and `Preset` moved to `[DLSSG]`, the XeSS-only keys are in `[XeSSFG]`; the old locations still work and log one `configuration_warning` (not read since 2026-10-22, see point 4 above). `[Runtime] Mode=Auto` was removed: written, it runs as `Bundled` with one warning.
- The release root carries a single proxy, `version.dll`; the other five are in `alternatives\`, one at a time (0.3.1–0.3.6 had you copy four; delete the extra ones when upgrading, see "Install & upgrade").
- RTX 40 / 50: frame generation also runs this project's kernels (PTX, compiled by the driver); `[Compatibility] AccelerateNewerGpus=0` switches back to NVIDIA's own kernels (that key was removed on 2026-10-19 and is no longer read: RTX 40 / 50 always run this project's kernels). Not yet run on real hardware.
- `[Logging] FrameTiming=1`: one line per real frame in the log directory (frame interval, super-resolution and frame-generation GPU time), for performance reports.
- New installation manual `MANUAL.en.html` (Chinese `MANUAL.html`).

### 0.3.6

- New option `[FrameGeneration] CatmullRomWarp=1` (default 0): the two motion-compensated samples taken for a generated frame switch from bilinear to Catmull-Rom, so fine structure (railings, text, distant window panes) is somewhat clearer in generated frames, at a cost of about +3% per generated frame (1080p / 1440p) to +4% (4K); with it on the image is no longer bit-identical to the official runtime. Measured offline on 11 real captures from 6 games, fine-structure energy rises by about 5 percentage points and the flicker metric improves along with it; the visible improvement is slight. It is not part of any `Optimized` tier, and the 310.1 build does not support it. There are also experimental `DetailRestore` keys (modes C and D); the measured gain is small or comes with side effects, so enabling them is not recommended - see `docs/INSTALL.md`.
- The log now records the version, hash and generation of the DLSS-G runtime the game itself ships (`runtime_redirect` / `game_runtime_generation`). Games on an early Streamline (1.x–2.4, shipping DLSS-G 3.x) can only do 2X: that generation of the plugin never writes a generated-frame count to the runtime, which no setting of this project can change; the log now says so directly. Streamline 1.x versions used to be recorded as 1.0.0.0 throughout; fixed.
- With NVIDIA App's DLSS override enabled, the frame-generation component is supplied by the NGX model store, and this project takes it over as usual; the new `ngx_override_active` record marks that case. The over-the-air Streamline plugin in the model store is now recognised as well (`ForcePluginFrames` used to be silently ineffective in such processes).
- The release package now carries all four proxies (`version` / `winmm` / `dbghelp` / `dinput8`) and the INI at its root, as `alternatives/README.md` describes; `310.1\` carries an INI too.
- Removed an NVAPI stub that was never called (`SetRawScgPriority`, present in none of the runtimes, in the NGX core or in Streamline).
- `docs/INSTALL.md` additions: what `Preset` means and when it takes effect, why `Optimized=2/3` "looks no different", how to attribute image artefacts, and the order in which to rule out full-screen flicker.

### 0.3.5

- Fixes a defect present since 0.3.0: when the game re-creates the frame-generation feature (menu changes, a resolution or settings change, toggling frame generation — some games re-create it often while running), the driver recycles kernel handles, and a recycled handle could still be bound to the previous kernel's optimized variant, so the wrong kernel could be launched afterwards — seen as corrupted generated frames, or a random crash / display-driver reset after playing for a while (`Optimized=0` is not affected). In an offline test with 15 re-creations on a 3070, the old build produced wrong frames in three runs out of four; the new build is bit-identical every time.
- Logging: `fg_gate_exit` is new (written when the process exits normally); a log without it means the game was killed or crashed.

### 0.3.4

- Fixes the 0.3.3 crash on RTX 30 (the display driver resets when the game starts or when DLSS is switched on; Forza Horizon 6 reports it as FHC01). Cause: when the NVIDIA App's DLSS override or an NGX online update is in effect, the DLSS super-resolution model is loaded from the NGX model directory, and it also received the GPU-architecture rewrite this project makes for Streamline, so on RTX 30 it took kernel paths that do not belong to this card and hung the GPU. NVIDIA's own components (the NGX model directory, the driver modules) now always get the real architecture; the rewrite applies to Streamline and the game only.
- Logging: `arch_spoof_applied` is now one record per calling module, with the full path, and `arch_spoof_excluded` is new; three small fixes (a missing system export no longer makes the proxy fail to load #10 / #65, `backend_install -100` now says which reason it was, and hardware-accelerated GPU scheduling being off is reported on its own #513).

### 0.3.3 (preview)

- `[FrameGeneration] Optimized=1` no longer turns on `SkipRepeatedRealCopy`, and no tier does; write `SkipRepeatedRealCopy=1` yourself if you want it. It is bit-identical in synthetic scenes and offline replays, but its exactness rests on an assumption outside our kernels (how the game's own plugin submits the Evaluates of one group), it had never been validated in a live game, and a flicker report on 0.3.2 followed (Zenless Zone Zero, public issue #532). Tier `1` remains bit-identical to the official runtime.
- RTX 20-series (Turing): the NVAPI GPU-architecture rewrite shown to the game is now armed at game start (our own `DllMain`, or when `nvapi64.dll` loads). Streamline 2.x decides "this GPU does not support DLSS-G" inside `slInit`, before the DLSS-G runtime is loaded and before earlier versions had armed the rewrite, which is why a 2080 Ti in Final Fantasy VII Rebirth (Streamline 2.8) lost the frame-generation plugin. RTX 30 needs it too: an RTX 3060 Laptop shows exactly the same symptom in the same game (public issue #509; the 0.2.4 native build reported Ada to every GPU, which is why that version worked), so a Turing or an Ampere answer is now rewritten to Blackwell (RTX 50: both builds do multi-frame generation, and games and Streamline unlock the 3X / 4X / 6X options on that architecture) while Ada / Blackwell are left alone.
- Diagnostics: the log field `arch_spoof_installed.trigger` gained the values `dllmain` / `nvapi_load` / `dependency_load`.

### 0.3.2

- Part of the inference kernels rewritten (310.9 build): the generated image now matches the official DLSS-G exactly (bit-identical in tests on an RTX 3080 Ti and an RTX 5070; an RTX 2080 Ti's output is bit-identical to the 3080 Ti's), no longer lossy, with a small speed-up (0–8% on the 3080 Ti).
- Optimization levels reworked: `[FrameGeneration] Optimized` is now `0`–`3`. `0` stock kernels, no acceleration; `1` every acceleration, image bit-identical to the official one (factory default); `2` adds lossy image kernels, faster, about 50 dB PSNR or better against the official image (310.9 build only); `3` everything lossy, fastest. See [`docs/INSTALL.en.md`](docs/INSTALL.en.md).

### 0.3.1

- Simpler install: the package root now ships four proxies (`version.dll`, `winmm.dll`, `dbghelp.dll`, `dinput8.dll`) — **copy all of them next to the game EXE, there is nothing to pick**. Whichever one the game loads first runs the mod; the others go on standby and only forward their exports to the matching system DLL, so frame generation is never installed twice. Only if none of the four is loaded, take `dxgi.dll` or `d3d12.dll` (one of them) from `alternatives\`.
- The factory `MaxGeneratedFrames` changes from `5` to `3` (4X). A game with dynamic MFG runs at whatever ceiling is written here by default, and `5` turned out to be too high for most people (public issues #497 / #499). For 6X use the 310.9 build and set this line to `5` yourself; a game whose own Streamline plugin is 4X stays at 4X either way.
- A bad value in the INI no longer switches the whole mod off: that key falls back to its own default and logs one `configuration_warning` (visible at `Level=2` since 2026-10-23, C106; `Level=1` before), every other key keeps working (changed on 2026-10-19: a wrong value is `configuration_invalid` and the feature the key belongs to is not enabled, see the factory INI paragraph below). The same for a switch the backend cannot do (for example `Preset=B` on the 310.1 build): those flags are **stripped and the install goes ahead**, logging `kernel_selection_unsupported{stripped_flags}` instead of refusing outright.
- RTX 20-series (Turing) works again, with the factory `dlssg_sm86.ini` and no edits: the kernel family is selected from the physical GPU and resolves to the SM75 family. Confirmed on an RTX 2080 Ti in The Witcher 3 next-gen DX12 at 2X (that game's own Streamline plugin caps at 2X); 4X / 6X are decided by the game's own plugin, as on RTX 30. Driver R580 or newer required (a hard requirement since 2026-10-26, see `docs/INSTALL.en.md` "Driver version requirement: R580").
- 0.3.0 could not enable frame generation on any RTX 20: the GPU architecture it reported to the NGX core was not the card's real architecture, so the core refused the frame-generation feature (public issues #491 / #492). Nothing about RTX 30 numerics or performance changes with this fix.
- The two RTX 20-related keys, `SM75Family` and `SpoofArchToGame`, are not in the factory INI; their defaults are the right values. See [`docs/INSTALL.en.md`](docs/INSTALL.en.md) if you need them.
- Diagnostics for "frame generation will not turn on": at `[Logging] Level=2` the log now records which gate refused it — hardware-accelerated GPU scheduling (HAGS), driver version, the NGX core's answer to the capability query, and the `CreateFeature` result. These records are named `fg_gate_*`.
- Image quality and performance on RTX 20 are not yet measured: this release only shows that frame generation runs there (thousands of Evaluates with no failure and no fallback). There are no FPS numbers and no image comparison against RTX 30.

### 0.3.0

- Reverted from the native build to the proxy build. The native approach (a self-built NGX host) had game-compatibility problems that were hard to fix; this build uses a proxy DLL around the unmodified factory runtime, leaving the game's calls to NGX unchanged, which is more compatible.
- The 310.9 runtime adds 6X (`MaxGeneratedFrames` ceiling raised from 3 to 5). On games that themselves support Dynamic MFG, selecting "Dynamic / Auto" frame generation reaches 6X.
- Optimized kernel set (~19–32% in the offline benchmark), a two-switch factory INI, capture/replay, and diagnostic logging — see below and `docs/`.

## Statement & roadmap

- Kernel-level work on frame generation (DLSS-G) has essentially reached its best at this stage and kernel optimization is paused; later releases focus on compatibility and bug fixes.
- INT8 for the super-resolution presets L / M ships in this release (RTX 30 / 20).
- The XeSS-FG engine: model 15, 2X-4X, no performance work; performance and HDR (RGBA16F) back buffers are listed under "Known issues".
- Vulkan support is hard to continue in the project's current state: the wrapper layer's own game compatibility is not good enough yet, test coverage is thin, and Vulkan only adds more compatibility problems. Users who need it can use the community patch versions.

## Requirements

- OS and games: Windows 10/11 x64, D3D12.
- GPU: RTX 30-series (SM86) or RTX 20-series (Turing / SM75). Development validation and all in-game tests on an RTX 3070; the full 0.4.0 test kit also ran on an RTX 3080 Ti; RTX 20 was tested on a 2080 Ti for frame generation (0.3.x) and one super-resolution INT8 smoke run. RTX 40 / 50 work too: frame generation takes this project's PTX route, not yet run on real hardware.
- Driver: an NVIDIA driver **R580 or newer** (CUDA 13.0); tested on 591.86, 610.74, 610.88 and 617.14. On an older driver frame generation is not enabled and the log has one `driver_below_floor`.
- No CUDA Toolkit and no Python.

| Release zip | Packaged runtime | Ceiling | `version.dll` size | SHA-256 |
|---|---|---|---|---|
| `dlssg-release-0.4.0-x64.zip` | 310.9.16.0 | 6X | 853,280 | `7b3f90435be4f94348bf8d128c84871c7f904c704f7690601a263277c327a7b2` |

The proxy DLL is about 0.85 MB; the runtime, the backend and the packs are files under `dlssg_sm86\`: the backend `backend\sm86_backend.dll` 2.1 MB (no kernels in it), the frame-generation kernel pack `kernels\fg_kernels.dll` 18.3 MB (required), the runtime `runtime\nvngx_dlssg.dll` 7.5 MB, the XeSS model pack `xefg\xefg_kernels.dll` 16.9 MB (programs, weights and CUDA images of model 18 and model 15), and super resolution's `sr\nvngx_dlss.dll` 74.2 MB and `sr\sr_kernels.dll` 106.6 MB (a pair, used only when `[SuperResolution] Mode` is not `Off`).

## Extra VRAM by configuration

Frame generation's own local VRAM delta (after warm-up minus before feature creation, rounded up to 10 MiB, 310.9 build, optimized, measured with 0.3.x). It tracks the output resolution only and is independent of the multiplier: 2X and 6X use the same amount; the multiplier costs no extra VRAM.

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

The same height at different widths is similar (it tracks output pixel count). This is frame generation's own increment, excluding the game itself and the synthetic inputs. With `Engine=XeSS` the dedicated VRAM in use is about 410–530 MiB higher than with DLSS-G (RTX 3070, 4K output, five games).

## Install & upgrade

New to mods? Follow the installation manual at the package root step by step: `MANUAL.en.html` (Chinese: `MANUAL.html`; double-click it to open it in a browser). The short version:

1. Exit the game completely.
2. Go to the directory of the EXE the game actually renders with (e.g. Black Myth: Wukong is `...\b1\Binaries\Win64\`). If you are not sure which one it is: while the game runs, open Task Manager's Details page, right-click the game's process → "Open file location".
3. Copy `version.dll`, `dlssg_sm86.ini` and the `dlssg_sm86\` folder from the package root into it; back up any same-named DLL that is already there. The runtime, the backend, the kernel packs, the XeSS model pack and the super-resolution snippet live in `dlssg_sm86\` and the proxy itself is under a megabyte, so **copying the DLL without the folder does nothing at all**. `alternatives\` is not copied normally.
4. Launch the game, enable DLSS Frame Generation in the graphics settings, and select 2X / 3X / 4X (for 6X see "6X" below: it needs `MaxGeneratedFrames` set to `5` by hand). On RTX 20 / 30, a frame-generation option that used to be grayed out and can now be turned on means it is installed.
5. To confirm that the proxy is loaded (the only way on RTX 40 / 50, where frame generation was available anyway): set `[Logging] Level` to `2`, play a little with frame generation on and exit; lines with the game's EXE name in `dlssg_sm86\logs\dlssg_sm86.log` mean it was loaded (the file keeps earlier runs too; look at this one). The factory `Level=1` writes a log only on errors, so an empty `logs`, or none at all, is normal.
6. If the game does not load `version.dll` (no line with its EXE name in `dlssg_sm86.log` at `Level=2` with frame generation on): exit the game, delete it and put the proxies from `alternatives\` next to the EXE one at a time, in the order `winmm.dll` → `dbghelp.dll` → `dinput8.dll` → `dxgi.dll` → `d3d12.dll`, trying each. Only ever one at a time; `dxgi.dll` and `d3d12.dll` sit on the render path, so they come last and never together. See `alternatives\README.md`.
7. Upgrade: exit the game, delete the old proxy DLL, `dlssg_sm86.ini` and the `dlssg_sm86\` folder first, then copy the three new items; make the settings you had changed again in the new INI. Releases 0.3.1–0.3.6 had you copy four proxies (`version.dll`, `winmm.dll`, `dbghelp.dll`, `dinput8.dll`); delete all four when upgrading: when several of this project's proxies sit in one folder, the first one the process loads runs the mod, and if that is an old one, the old version runs. Coming from 0.3.x you can also delete `%LOCALAPPDATA%\DlssgSm86` — nothing is written there any more.
8. Uninstall: delete `version.dll` (or the alternative you switched to), `dlssg_sm86.ini` and the `dlssg_sm86\` folder, and put back any DLL you backed up.

Switches in the factory `dlssg_sm86.ini` a player may change:

- `[DLSSG] Optimized`: consistency tier `0`–`3`. Factory `1` = every acceleration that stays bit-identical (since 2026-10-21; 0.4.0 shipped `2`); `2` = plus the lossy image kernels: no longer bit-identical to the official image, with no visible difference from `1` side by side against ground truth, 1–3 % faster per generated frame; `0` = stock numerics; `3` = every lossy acceleration. In `[FrameGeneration]` before 2026-10-14; the old place is no longer read (see point 4 above).
- `[DLSSG] Preset`: factory `A`; what is written here runs, whatever the game or a driver profile asks for (`Auto` was removed; written, it runs as `A` with a warning); `B` = UI recomposition forced on (effective only when the game hands over both a HUD-less image and a UI plane); `C` = FG Lite (faster, slightly lower quality), also runs this project's accelerated kernels (per the `Optimized` tier) whatever DLSS-G version the game ships.
- `[FrameGeneration] MaxGeneratedFrames`: factory `3` = 4X; `5` = 6X. A requirement, not a ceiling: it is told to the game as written (a driver profile cannot lower it), and the game's menu may pick a lower multiplier; where it cannot be served (above 5, above 3 with `Engine=XeSS`) frame generation is not enabled and the log says why (`max_generated_frames_unsupported`), never lowered quietly.
- `[FrameGeneration] Engine`: factory `DLSS`; `XeSS` = Intel's XeSS-FG model (model 15; 2X runs model 18's graph): `MaxGeneratedFrames` `1`-`3`, so the factory `3` works as is; the `[XeSSFG] Model` key was removed on 2026-10-26 (written, it is not read). `[XeSSFG] FlowResolution` (factory `Middle`; finer is slower) and `UseHudless` (factory `Auto` = the same as `0`: the whole image, UI included, is interpolated; `1` = always required, a game that supplies none ends with an error) apply to XeSS only.
- `[SuperResolution] Mode`: factory `Off`. On RTX 30 / 20, `M-INT8` / `L-INT8` = preset M / L on every super-resolution quality mode (DLAA included), run on this project's INT8 kernels, the mode chosen in the game; `M-FP16` / `L-FP16` = the same preset on NVIDIA's own kernels (the comparison). On Quality / Balanced / DLAA, M or L is slower than the game's own K; the gain is in Performance / Ultra Performance. These five values only: anything else (0.4.0's `On` included) leaves super resolution off with one `configuration_invalid`; the old key `Tier` is not read. No effect on RTX 40 / 50.

The other entries in the factory file (`Enabled`, logging, the runtime source `[Runtime] Mode=Bundled`) are normally left alone; further diagnostic/compatibility keys are not in the file and take safe defaults — the full list is in [`docs/INSTALL.en.md`](docs/INSTALL.en.md). A value written wrongly is not replaced by a default: the log gets one `configuration_invalid` and the feature the key belongs to is not enabled (a frame-generation key → no frame generation, as `Enabled=0`; a super-resolution key → no super resolution; a logging / debug key → only that diagnostic is off).

## Antivirus & signing

Every DLL of this project in the package is code-signed: `version.dll` at the root, the five alternative proxies in `alternatives\`, and under `dlssg_sm86\` the backend and three resource packs (frame-generation kernels `fg_kernels.dll`, super-resolution kernels `sr_kernels.dll`, XeSS model `xefg_kernels.dll`), 10 in total (NVIDIA's runtime and super-resolution snippet in `dlssg_sm86\runtime\` and `dlssg_sm86\sr\` keep NVIDIA's own signature and are not touched). A self-signed certificate only verifies the signer's identity and file integrity; it gives no default Windows trust, so Windows SmartScreen may still prompt "unknown publisher" on first run — that is a reputation prompt, not an antivirus detection. The certificate is self-signed as `CN=DLSSG for SM86 (self-signed)`, SHA-1 thumbprint `85BA66762F851E49148D706915D09026281418E6`; verify the signer and thumbprint via the file's Properties → Digital Signatures tab or `signtool verify /pa`.

## Performance

Every number below comes from this project's own timing (a D3D12 timestamp before and after each Evaluate, or the `[Logging] FrameTiming` log). Offline GPU milliseconds measure the cost of frame generation / super resolution itself; they are not an in-game frame-rate gain.

### Frame generation: real game captures (RTX 3070, offline replay)

RTX 3070, driver 617.14, the 0.4.0 test kit's full run (run_all). Inputs recorded in the games (Cyberpunk 2077, Black Myth: Wukong) replayed through the whole route (proxy → backend → NVIDIA runtime 310.9.16 or the XeSS engine); the first 4 real frames of each run dropped, median over all frames of 4 rounds. GPU milliseconds per generated frame; in brackets the change against DLSS default (negative = faster). DLSS default = 0.4.0's factory INI (`Optimized=2`, `Preset=Auto`; the factory tier is `1` since 2026-10-21, 1–3 % slower per generated frame, see the RTX 3080 Ti table below); stock kernels = `Optimized=0`.

**2X**

| Capture | DLSS default | Preset C | XeSS Middle | XeSS Coarse | XeSS Fine | Stock kernels |
|---|---:|---:|---:|---:|---:|---:|
| Cyberpunk 2077 1440p | 2.554 | 1.694 (−33.7 %) | 2.974 (+16.4 %) | 2.344 (−8.2 %) | 3.828 (+49.9 %) | 3.639 (+42.4 %) |
| Black Myth: Wukong 1440p | 2.581 | 1.745 (−32.4 %) | 3.266 (+26.5 %) | 2.634 (+2.0 %) | 4.164 (+61.3 %) | 3.657 (+41.7 %) |
| Cyberpunk 2077 4K | 3.557 | 2.499 (−29.7 %) | 6.379 (+79.3 %) | 5.059 (+42.2 %) | 8.296 (+133.2 %) | 4.407 (+23.9 %) |
| Black Myth: Wukong 4K | 3.756 | 2.679 (−28.7 %) | 6.974 (+85.7 %) | 5.682 (+51.3 %) | 9.025 (+140.3 %) | 4.560 (+21.4 %) |

**4X** (per generated frame = the sum of a real frame's 3 Evaluates ÷ 3): Cyberpunk 2077 4K default 2.532 ms, preset C 1.392 ms (−45.0 %); Black Myth: Wukong 4K default 2.733 ms, preset C 1.517 ms (−44.5 %).

### Frame generation: real game captures (RTX 3080 Ti, offline replay)

RTX 3080 Ti, driver 610.88, a full run of the same test kit (a tester's machine); method as above. During these two timing stages the GPU held 1665 MHz (below the 1950 MHz of the synthetic timing), so the absolute values are conservative; the comparisons hold.

**2X**

| Capture | DLSS default | Preset C | XeSS Middle | XeSS Coarse | XeSS Fine | Stock kernels |
|---|---:|---:|---:|---:|---:|---:|
| Cyberpunk 2077 1440p | 1.651 | 1.032 (−37.5 %) | 2.235 (+35.4 %) | 1.786 (+8.2 %) | 2.791 (+69.1 %) | 2.381 (+44.2 %) |
| Black Myth: Wukong 1440p | 1.663 | 1.062 (−36.1 %) | 2.359 (+41.9 %) | 1.911 (+14.9 %) | 2.936 (+76.6 %) | 2.391 (+43.8 %) |
| Cyberpunk 2077 4K | 2.171 | 1.473 (−32.2 %) | 4.640 (+113.7 %) | 3.668 (+68.9 %) | 5.908 (+172.1 %) | 2.860 (+31.7 %) |
| Black Myth: Wukong 4K | 2.275 | 1.577 (−30.7 %) | 4.763 (+109.3 %) | 3.822 (+68.0 %) | 6.060 (+166.3 %) | 2.935 (+29.0 %) |

**4X**: Cyberpunk 2077 4K default 1.567 ms, preset C 0.822 ms (−47.5 %); Black Myth: Wukong 4K default 1.666 ms, preset C 0.897 ms (−46.2 %).

### Super-resolution INT8 against the same preset in FP16 (RTX 3070, offline)

Same test kit. GPU time of one super-resolution Evaluate in ms: median of 64 measured frames per run, median of 3 rounds. Same runtime (310.7.129), same preset; only INT8 against FP16. The two kernel families correspond to The Witcher 3 / Resonance / Forza Horizon 6 / Spider-Man (mvlo) and Black Myth: Wukong / Hogwarts Legacy (mvhi).

| Family | Output | Mode | L FP16 | L INT8 | L change | M FP16 | M INT8 | M change |
|---|---|---|---:|---:|---:|---:|---:|---:|
| mvlo | 3840×2160 | Performance | 7.443 | 6.459 | −13.2 % | 6.065 | 4.905 | −19.1 % |
| mvlo | 2560×1440 | Performance | 3.415 | 2.993 | −12.4 % | 2.787 | 2.294 | −17.7 % |
| mvlo | 1920×1080 | Performance | 1.965 | 1.736 | −11.7 % | 1.615 | 1.353 | −16.2 % |
| mvlo | 1920×1080 | Ultra Performance | 1.258 | 1.179 | −6.3 % | 1.066 | 0.967 | −9.3 % |
| mvhi | 3840×2160 | Performance | 7.305 | 6.127 | −16.1 % | 5.914 | 4.623 | −21.8 % |
| mvhi | 2560×1440 | Performance | 3.346 | 2.845 | −15.0 % | 2.719 | 2.171 | −20.2 % |
| mvhi | 1920×1080 | Performance | 1.934 | 1.656 | −14.4 % | 1.581 | 1.278 | −19.2 % |
| mvhi | 1920×1080 | Ultra Performance | 1.235 | 1.106 | −10.4 % | 1.057 | 0.914 | −13.5 % |

### Super-resolution INT8 against the same preset in FP16 (RTX 3080 Ti, offline)

Same test kit and method, GPU at 1665 MHz.

| Family | Output | Mode | L FP16 | L INT8 | L change | M FP16 | M INT8 | M change |
|---|---|---|---:|---:|---:|---:|---:|---:|
| mvlo | 3840×2160 | Performance | 5.338 | 4.578 | −14.2 % | 4.341 | 3.455 | −20.4 % |
| mvlo | 2560×1440 | Performance | 2.484 | 2.151 | −13.4 % | 2.028 | 1.644 | −18.9 % |
| mvlo | 1920×1080 | Performance | 1.465 | 1.333 | −9.0 % | 1.208 | 1.047 | −13.4 % |
| mvlo | 1920×1080 | Ultra Performance | 1.000 | 0.948 | −5.2 % | 0.849 | 0.791 | −6.9 % |
| mvhi | 3840×2160 | Performance | 5.262 | 4.355 | −17.2 % | 4.252 | 3.239 | −23.8 % |
| mvhi | 2560×1440 | Performance | 2.437 | 2.045 | −16.1 % | 1.991 | 1.545 | −22.4 % |
| mvhi | 1920×1080 | Performance | 1.441 | 1.272 | −11.8 % | 1.186 | 0.994 | −16.1 % |
| mvhi | 1920×1080 | Ultra Performance | 0.979 | 0.892 | −8.9 % | 0.848 | 0.746 | −12.0 % |

### In game (RTX 3070, 4K output, 2X)

RTX 3070, driver 617.14, logs of `[Logging] FrameTiming=1`, median of a 60 s still window after entering the world, tier 2, XeSS at `Middle`. Per generated frame = frame-generation GPU time; real frame rate = frames the game renders (the displayed rate is twice that).

| Game | DLSS default (ms) | XeSS (ms) | XeSS real fps | Preset C (ms) | Preset C real fps |
|---|---:|---:|---:|---:|---:|
| Cyberpunk 2077 | 3.755 | 6.733 (+79 %) | −8.6..−9.7 % | 2.748 (−27 %) | +3.6 % |
| Black Myth: Wukong | 4.035 | 7.308 (+81 %) | −11.0 % | 3.085 (−24 %) | +4.3 % |
| The Witcher 3 | 3.973 | 7.166 (+80 %) | −15.0 % | 2.974 (−25 %) | −0.8 % |
| A Plague Tale: Resonance | 3.943 | 6.968 (+77 %) | −28.1 % | 2.947 (−25 %) | +9.5 % |
| Forza Horizon 6 | 3.856 | 6.847 (+78 %) | −8.0 % | 2.803 (−27 %) | +7.4 % |

Cyberpunk 2077: path tracing + ray reconstruction, DLSS Ultra Performance; Black Myth: Wukong: DLSS 50 %. How the real frame rate moves depends on the game's bottleneck: in The Witcher 3 preset C saves 1 ms per generated frame and the frame rate does not change.

### Super-resolution INT8 in game (RTX 3070, The Witcher 3)

RTX 3070, driver 617.14, frame generation off, frame rate over a still window after entering the world; the baseline is the same preset in FP16 (`Mode=M-FP16` / `L-FP16`), INT8 and FP16 alternated 2–4 runs each, means in the table, gain from the medians. The preset M rows use the 0.4.0 shipped kernel sets (measured 10-10); the L rows were measured 10-09 (this family's L set has not changed since).

| Output | Mode (render) | Preset | FP16 fps | INT8 fps | Gain | Per frame |
|---|---|---|---:|---:|---:|---|
| 3840×2160 | Performance (1920×1080) | M | 47.15 | 50.85 | +7.85 % | 21.21 → 19.67 ms (−1.54) |
| 2560×1440 | Performance (1280×720) | M | 85.48 | 91.34 | +6.86 % | 11.70 → 10.95 ms (−0.75) |
| 1920×1080 | Performance (960×540) | M | 122.14 | 128.35 | +5.08 % | 8.19 → 7.79 ms (−0.40) |
| 3840×2160 | Ultra Performance (1280×720) | L | 63.56 | 66.13 | +4.0 % | 15.73 → 15.12 ms (−0.61) |
| 2560×1440 | Ultra Performance (853×480) | L | 109.42 | 111.80 | +2.2 % | 9.14 → 8.94 ms (−0.19) |
| 1920×1080 | Ultra Performance (640×360) | L | 145.83 | 147.57 | +1.2 % | 6.86 → 6.78 ms (−0.08) |

Every gain but the last exceeds twice the same-configuration repeat spread; 1080p Ultra Performance L is within one run's spread, but all 4 INT8 runs are above all 4 FP16 runs. Image quality: preset M flickers slightly more than FP16 on still content (frame-to-frame stability of still pixels in game 61.8 vs 62.5 dB) with PSNR against ground truth no lower than FP16; preset L is within 0.6 dB of FP16.

### Whole generation group GPU time (RTX 3080 Ti, synthetic scene)

RTX 3080 Ti, driver 610.88, the 0.4.0 test kit (a tester's machine). GPU milliseconds of the whole frame-generation group per real frame (the span of one group's Evaluates in one submission): median of 256 groups per run, median of 4 rounds; reduction = (tier0 − current) / tier0. tier0 = stock kernels (`Optimized=0`), tier1 = bit-identical acceleration (`1`, the factory default), tier2 = the lossy tier (`2`, 0.4.0's factory default). The same kit's consistency checks match the RTX 3070 reference bit for bit: frame generation 18/18, real captures 28/28, super resolution 32/32, super resolution `Mode=On` 4/4.

| Resolution | Multiplier | tier0 (ms) | tier1 (ms) | Reduction | tier2 (ms) | Reduction |
|---|---|---:|---:|---:|---:|---:|
| 1080p | 2X | 1.364 | 0.926 | 32.1 % | 0.917 | 32.8 % |
| 1080p | 3X | 1.942 | 1.423 | 26.7 % | 1.400 | 27.9 % |
| 1080p | 4X | 2.520 | 1.914 | 24.0 % | 1.888 | 25.1 % |
| 1080p | 5X | 3.089 | 2.411 | 21.9 % | 2.375 | 23.1 % |
| 1080p | 6X | 3.677 | 2.911 | 20.8 % | 2.867 | 22.0 % |
| 1440p | 2X | 2.094 | 1.459 | 30.3 % | 1.443 | 31.1 % |
| 1440p | 3X | 3.030 | 2.238 | 26.1 % | 2.205 | 27.2 % |
| 1440p | 4X | 3.963 | 3.008 | 24.1 % | 2.960 | 25.3 % |
| 1440p | 5X | 4.900 | 3.786 | 22.7 % | 3.719 | 24.1 % |
| 1440p | 6X | 5.841 | 4.558 | 22.0 % | 4.472 | 23.4 % |
| 4K | 2X | 2.472 | 1.901 | 23.1 % | 1.864 | 24.6 % |
| 4K | 3X | 3.792 | 3.002 | 20.8 % | 2.924 | 22.9 % |
| 4K | 4X | 5.112 | 4.106 | 19.7 % | 3.987 | 22.0 % |
| 4K | 5X | 6.437 | 5.212 | 19.0 % | 5.054 | 21.5 % |
| 4K | 6X | 7.760 | 6.319 | 18.6 % | 6.123 | 21.1 % |

The gain is larger at lower resolution and lower multiplier (closer to launch/latency bound).

## Estimating FPS after frame generation

First, in the same scene, output resolution, DLSS upscaling mode, and quality settings, turn frame generation off and read the frame rate `F_off`. Convert to a base frame time with `1000 / F_off`, then take the whole-group generation time `T_FG` (ms) from the table above for that resolution, multiplier, and kernel setting.

```text
base frame time    T_base (ms) = 1000 / F_off
group time         T_group (ms) ≈ T_base + T_FG
real-frame/group rate G (grp/s) ≈ 1000 / T_group
output frame rate  F_out (FPS)  ≈ G × M = 1000 × M / (1000 / F_off + T_FG)
```

`M` is the multiplier (2X…6X → 2…6). A group is one real frame plus `M − 1` generated frames; `T_FG` already covers the whole group and the shared preprocessing, so do not multiply it by `M − 1`. With generation on, the real-frame/group rate `G` is lower than `F_off`.

Example: with frame generation off, ~**50 FPS** (20 ms base frame time), using the RTX 3080 Ti **4K 4X** `T_FG` from the table:

| Kernels | T_FG (ms) | group time (ms) | group rate (grp/s) | estimated FPS |
|---|---|---|---|---|
| Stock (tier0) | 5.112 | 25.112 | 39.8 | 159.3 FPS |
| Factory default (tier1) | 4.106 | 24.106 | 41.5 | 165.9 FPS |
| Lossy tier (tier2) | 3.987 | 23.987 | 41.7 | 166.8 FPS |

For other resolutions/multipliers use the matching row and your own measured `F_off` for that setting. This is a rough estimate — the base render time plus the generation-group cost; GPU contention, synchronization, CPU overhead, frame caps, and the monitor refresh rate all affect the real result, and the estimate is not guaranteed to equal a counter reading or the actual displayed rate.

## 6X

6X (5 generated frames per real frame) is NVIDIA's DLSS 4.5 Dynamic Multi Frame Generation. The bundled runtime supports it and this project makes it run on Ampere (with `Engine=DLSS` only).

**The factory `MaxGeneratedFrames` is `3` (4X); 6X is a manual edit.** A game with dynamic MFG runs at whatever is written there by default, and `5` was too high for most people (public issues #497 / #499); setting it back to `5` gets 6X at any time. Whether you actually get 6X still depends on the game:

- The game itself supports 6X (ships a newer Streamline frame-gen plugin and offers 6X or Dynamic MFG in its menu): set `MaxGeneratedFrames=5`; 6X runs stably in testing.
- The game only supports 4X (ships an older 4X plugin, as most current games do): the ceiling is set by the game's plugin and this project cannot raise it to 6X. That plugin sizes its internal present queue for 4X at init, so forcing extra frames overruns it and disables frame generation or crashes. Use 4X for these games.

## Real-world feedback

Black Myth: Wukong, Cyberpunk 2077, and Forza Horizon 6 run 4X normally in testing; A Plague Tale: Resonance runs 6X by default with "Dynamic / Auto" frame generation. 0.4.0 ran DLSS default, preset C and XeSS (2X, 4K output) in these four games and The Witcher 3 on an RTX 3070 with no crash, TDR or device removal, and the HUD intact in screenshots. Image quality depends on the base frame rate: at a low base rate, generated frames can show breakup and edge artifacts, more so at higher multipliers, and some games need graphics settings lowered, to raise the base rate, even at 4X for a good result. This is inherent to frame generation when there is little frame-rate headroom, not a defect of this project.

## Known issues

- XeSS-FG (`Engine=XeSS`) is optional; the default is still DLSS-G. On RTX 30 it is much slower per generated frame than DLSS-G (4K: RTX 3070 +77..+86 %, about +3 ms; RTX 3080 Ti offline +109..+114 %), the real frame rate is 8..15 % lower, and 28 % lower in A Plague Tale: Resonance (the cause of the extra wait is not yet found); it uses about 0.4–0.5 GB more VRAM; `Engine=XeSS` goes to 4X (`MaxGeneratedFrames` 1-3); with 4 or more frame generation is not enabled (`max_generated_frames_unsupported`). The image-quality difference between HUD-less on / off has not been compared frame by frame. `[Debug] Capture` records nothing under XeSS.
- Cost of XeSS-FG (model 15; RTX 3070, offline full-route replay, `Middle`, DIRECT, GPU ms per real frame together with its generated frames): 4X at 1440p 4.5-5.0 ms against DLSS default 5.3-5.4 ms (8-15 % faster); at 4K 10.3-10.8 ms against 7.9-8.5 ms (27-30 % slower); preset C is the fastest everywhere. 2X runs model 18's graph and costs 4-5 % more than the former `Model=18` (in game on a 5070: +1.7 %, bit-identical output) (it also copies the previous depth, and the previous back buffer with a HUD-less image). Offline GPU time is not the game's frame rate. Black Myth: Wukong at 4K (R10G10B10A2, HUD-less) in game, real frame rate: 3X 32.0 fps (DLSS default 34.0), 4X 28.5 fps (DLSS default 31.5). Image quality was compared with skip-frame truth on 4 captures only: at the indices near the previous real frame (t = 1/4, 1/3) it ghosts less than model 18 + t, but not in every scene.
- What XeSS cannot run is never handed to DLSS-G: a declared back buffer that is not R8G8B8A8 / B8G8R8A8 / R10G10B10A2 (e.g. RGBA16F, common for HDR; Intel's own XeSS-FG does not accept that format either) or XeSS objects that cannot be created refuse the frame-generation feature (FG off, error-level `xefg_feature_refused`); a problem found only while running (no back buffer, depth, motion vectors or interpolation output, mismatched sizes, a record failure) is one error-level `xefg_kernel_fatal` naming the cause, and the game exits with `0xE0D55001`. The factory `UseHudless=Auto` is the same as `0` since 2026-11-10 (C139): the whole image, UI included, is interpolated and `xefg_hudless_unused` no longer appears; under `UseHudless=1` a HUD-less image or UI plane whose size differs from the back buffer's (or none at all) is `xefg_kernel_fatal` and the game exits with `0xE0D55001`.
- `[DLSSG] Preset=B` (UI recomposition forced on) only takes effect when the game provides both a HUD-less image and a UI plane, which most games do not; then the log (at the factory level) has one `preset_ineffective` record naming what is missing, and the image is the same as without `Preset=B`.
- A game that ships a generation-A DLSS-G runtime (≤ 3.8.x, an old Streamline plugin that can drive 2X only; for example The Witcher 3 next-gen, Bright Memory, Marvel's Spider-Man Remastered, Atomic Heart, Delta Force) does not get frame generation since 2026-10-19: its request is forwarded unchanged (error-level `game_runtime_generation`); before, these games got 2X.
- Preset C has no comparison against the official preset C on RTX 40 / 50; in The Witcher 3 preset C saves 1 ms per generated frame but the frame rate does not change (the bottleneck is elsewhere).
- `[FrameGeneration] CatmullRomWarp` has no kernel row in the 310.9.16 package: writing it refuses frame generation (error-level `detail_restore_row_unknown`; before 2026-10-20 it simply had no effect). The experimental `DetailRestore*`, `[Compatibility] LaunchChains` / `PlainVariant` / `ChainBlock0` / `HardwareBilinear`, `[Runtime] CacheDirectory` and `[Debug] QueueTrace` were removed on 2026-10-21: since 2026-10-26 they are not read (writing one is the same as not writing it).
- An explicit switch that cannot be carried out is refused, never replaced by another path (2026-10-20): a `[Runtime] Path` / `SRPath` naming a wrong file, a missing packaged runtime or super-resolution snippet (before: the packaged one, or the game's own, answered instead — on RTX 40 / 50 that was native 2X), `KernelImage=Cubin` on RTX 40 / 50 (`kernel_image_unavailable`); a super-resolution install failure refuses that `nvngx_dlss.dll` load, so DLSS is unavailable in the game rather than running the stock FP16. How game menus behave after such a refusal has not been observed in a real game.
- Super resolution: `Mode` pins M / L on every quality mode, so Quality / Balanced / DLAA are slower than the game's own K (accepted by the owner); 0.4.0's `Mode=On` (accelerating only the modes the game already ran with L / M) is removed.
- RTX 20: 310.9.16's three new kernels and XeSS were verified only by forward JIT of the sm_75 kernels on the RTX 3070 and have not run on a real Turing card; super-resolution INT8 had one smoke run on a 2080 Ti. RTX 40 / 50: the frame-generation PTX route has not run on real hardware.
- When a game exits abnormally (e.g. force-closed), the backend writes no XeSS summary record (`xefg_summary`; diagnostics only).

## Diagnostics & boundaries

- Logs go to `dlssg_sm86\logs\dlssg_sm86.log` in the game directory: every module of every process appends to this one file, each line starting with the local time, `pid N`, the EXE name and a level letter (E = Level 1, I = 2, D = 3), then the JSON record, e.g. `2026-10-22 14:03:11.482  pid 12345  Cyberpunk2077.exe  E  {"event":...}`. To look at one game, filter by EXE name or pid (`Select-String`, `findstr`). With `FrameTiming=1` there is also the plain-text `frame_timing.txt`, one appended session per process. `[Logging] Level=1` (default) records errors only; use `2` or `3` when investigating.
- If frame generation does nothing, check the log for `install.active=true` and `backend_install.status=0`; if it is absent, the driver/runtime usually did not match — the reason is logged; when the install fails, the `nvngx_dlssg.dll` load is refused and frame generation is not enabled (`runtime_selection_failed`) instead of using the game's own runtime.
- Frame generation on but no frame generated: look for `foreign_fg_mod` in the log — if it is there, another frame-generation mod (rtx-encore, OptiScaler, an old ReShade companion add-on, ...) is installed in the game too; keep only one. When the game crashes, the `fg_gate_crash` record at the end of the log names the faulting module (`ours` says whether it is this project's code); please attach it to reports.
- Do not read an offline time reduction as an in-game FPS gain — the real frame-rate change depends on the game and where the bottleneck is.
- All INI keys, log fields, and capture/replay are in [`docs/INSTALL.en.md`](docs/INSTALL.en.md) and [`docs/CAPTURE.md`](docs/CAPTURE.md).

## SM75 source & credits

- Coldwood1026 for the RTX 20-series / SM75 adaptation (the source of the SM75 kernel family — see `THIRD_PARTY_NOTICES.txt`).
- NVIDIA for the DLSS-G runtime, the DLSS super-resolution snippet, models, and pre/post-processing (bundled, unmodified).
- Intel for the XeSS-FG model (shaders and weights, MIT license, see `THIRD_PARTY_NOTICES.txt`).
- MFG-Unlocker 1.6.0 (Michael Robles / dashdogy, MIT) for the model-15 port XeSS-FG runs and the frame-count unlock; 46 of its programs are derived from Intel's and fall under Intel's XeSS SDK terms. The authors of the "XessFG Model 15 Final" package found the relabel of model 15 as model 16.

## Source & building

- The C++ source of the proxies, the backend, the tests and the tools is in the source repository and builds with
  `build.ps1` (default runtime model 310.9.16: `-Runtime310916` for the runtime, `-RuntimeModel 310.9` with `-Runtime3109` for the developer alternative; `-KernelPack` for the kernel pack); the steps,
  tests and rules are in that repository's `CONTRIBUTING.en.md`.
- The kernel pack `dlssg_sm86\kernels\fg_kernels.dll`, the super-resolution kernel pack, the XeSS model pack and the NVIDIA runtimes are not in the source; take them from the
  release of the same version. Without the kernel pack, `Engine=DLSS` frame generation is refused (logged as
  `kernel_pack_missing` and `fg_kernels_unavailable`); the runtime's own kernels are never used in its place. Without the XeSS model pack, `Engine=XeSS` refuses frame generation (`xefg_unavailable`, FG off).

## License & third-party

- The project's C++ source is released under the MIT license (`LICENSE.md`, Copyright (c) 2026 sdli1995). The kernel images shipped with it are provided in binary form only and are not covered by the MIT license.
- The bundled `nvngx_dlssg.dll` (310.9.16, SHA prefix `cadee1aa…`) ships unmodified as a file under `dlssg_sm86\runtime\`, and the super-resolution snippet `nvngx_dlss.dll` (310.9.16) under `dlssg_sm86\sr\`. The kernel images in `dlssg_sm86\kernels\fg_kernels.dll` and `dlssg_sm86\sr\sr_kernels.dll` are extracted, recompiled or rewritten from NVIDIA's kernels (the sm_75 family imported from Coldwood1026), are provided in binary form only and are not covered by the MIT license. All of these are NVIDIA and upstream third-party material, not re-licensed with the project source — see `THIRD_PARTY_NOTICES.txt`.
- The shaders and weights in `dlssg_sm86\xefg\xefg_kernels.dll` come from Intel XeSS-FG and ship under Intel's MIT license; the notice is in `THIRD_PARTY_NOTICES.txt`. The model-15 programs in the same DLL come from MFG-Unlocker 1.6.0 (MIT): 46 of them are derived from Intel's XeSS SDK programs and fall under Intel's XeSS SDK terms, outside MFG-Unlocker's MIT License and this project's `LICENSE.md`. Intel's `libxess_fg.dll` is not included.
