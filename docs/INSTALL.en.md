# DLSSG SM86 fused build

[中文](INSTALL.md) · **English**

**Copy `version.dll`, `dlssg_sm86.ini` and the `dlssg_sm86\` folder from the package root next to the game's actual rendering EXE and launch as usual.** `version.dll` is the only proxy (hook DLL) to put there; if the game does not load it, delete it and try the proxies in `alternatives\` one at a time, in a fixed order (see "One proxy at a time" below). No Python and no PowerShell launcher at runtime. If this is your first install and you want to follow it step by step, read the installation manual `MANUAL.en.html` at the package root first (double-click it to open it in a browser); this document describes every setting in full.

The package is a set of files: the proxy `version.dll` at the root (with five alternative proxies and their instructions in `alternatives\`, not copied unless the game does not load `version.dll`) and, under `dlssg_sm86\`, the backend (`backend\sm86_backend.dll`), the frame-generation kernel pack (`kernels\fg_kernels.dll`: every kernel image, cubin + PTX, and its manifest; resources only, no code; without it frame generation does not start and the log says `kernel_pack_missing` and `fg_kernels_unavailable`), the matching stock DLSSG runtime (`runtime\nvngx_dlssg.dll`, which the default `Mode=Bundled` loads), `sr\` (Super Resolution, untouched while its switch is off: the snippet `sr\nvngx_dlss.dll` and the kernel pack `sr\sr_kernels.dll`, the INT8 kernel sets of RTX 30 / 20 as one resource-only DLL - without it the switch has no effect and `sr_gate` names the file it looked for; the `index.json` and `set_*` folders an older release left in `sr\` are no longer read and may be deleted), the XeSS-FG model pack `xefg\xefg_kernels.dll` (for `Engine=XeSS`: programs, weights and CUDA images; without it `Engine=XeSS` refuses frame generation, `xefg_unavailable`) and `logs\` (an `rr\` left by an older release belonged to the Ray Reconstruction route removed on 2026-10-09; it is never read and may be deleted). The proxy resolves these paths against **its own directory**, not the working directory, so the proxy, `dlssg_sm86.ini` and `dlssg_sm86\` go into the same folder: a proxy copied without `dlssg_sm86\` logs `backend_loaded{loaded:false}` or `bundled_runtime_unavailable` and the game keeps its own DLSS-G. **No runtime or backend is written to `%LOCALAPPDATA%` any more** (the one thing written there, on RTX 40 / 50 only, is the cache of compiled PTX, `%LOCALAPPDATA%\DlssgSm86\jit`, see "Driver version requirement: R580"): older releases extracted the runtime and the backend to `%LOCALAPPDATA%\DlssgSm86\bundles\<id>\` and loaded them from there; that step is gone. After upgrading from such a release you may delete `%LOCALAPPDATA%\DlssgSm86` by hand; the mod never deletes it (another version may still use it).

The factory INI is under "Normal use" below. Every diagnostic / compatibility / performance-experiment switch is still parsed and supported but is left out of the factory INI; when absent each takes a safe, validated default. The complete list is under "Advanced / diagnostic keys" further down.

## Runtime model (which runtime the package carries)

A release package carries exactly **one** runtime and the one backend that matches it, and `manifest.json`'s `runtime_model` says which. Since 2026-10-14 the default model is **310.9.16**; 310.9 is still buildable as the alternative:

| Model | Artifact | Bundled runtime | Max generated frames | Notes |
|---|---|---|---|---|
| **310.9.16** (default) | `dlssg-release-0.4.2-x64.zip` | `nvngx_dlssg.dll` 310.9.16.0 | 5 (6X) | The default since 2026-10-14. 68 of its 71 kernels are byte-identical to 310.9.1's; `Kernel_BlendCandidatesFused` and `Kernel_DownsampleRGBAndEstimateUIFused` changed and `Kernel_BlendCandidatesFGLite` is new (all three retargeted from the runtime's own sm_120 PTX to sm_86 / sm_75). At the default preset it is **bit-identical** to 310.9.1 (synthetic scene 56/56, two real captures 105/105 planes each). Adds **preset C (FG Lite)**, see the `Preset` subsection below. 34 variant rows + 3 image patches (P7 re-derived on 310.9.16's `Kernel_BlendCandidatesFused`, the other two shared with 310.9.1); the `CatmullRomWarp` rows sit on 310.9.1's old kernel and this model's pack has none (writing the key refuses frame generation, see "A switch that is on but cannot be carried out is refused" below). `Router=SM75` works (forward JIT on a 3070; the three new kernels have not run on a real Turing yet) |
| **310.9** (alternative) | `dlssg-release-0.4.2-x64-310.9.zip` | `nvngx_dlssg.dll` 310.9.1.0 | 5 (6X) | Native 6X; optimized kernels, cross-kernel fusions and image-kernel patches are all available (37 variant rows: 34 network variants + 3 image patches re-derived against 310.9.1 RVAs). `Router=SM75` (RTX 20) works (verified by forward JIT on a 3070 and on a real Turing, RTX 2080 Ti) |

Build with `powershell -File build.ps1` (default `-RuntimeModel 310.9.16`, output in `out\build-310.9.16` and `dist\Release-310.9.16`; `-RuntimeModel 310.9` builds the alternative). The runtime is NVIDIA material and does not ship with the source;
hand it to the build with `-Runtime310916 <path>` / `-Runtime3109 <path>` (`-DDLSSG_RUNTIME_310916=<path>` / `-DDLSSG_RUNTIME_3109=<path>` when configuring CMake directly).
**When the game ships 310.9.1 itself**: with the factory `[Runtime] Mode=Bundled` the proxy answers the game's request for `nvngx_dlssg.dll` with the bundled 310.9.16 and the game's own file is never loaded, so the game's version does not matter. `Mode=Auto` was removed in 0.4.0 (docs/COMPAT.md C86): it could install only when the game's file happened to be the bundled runtime, and otherwise left the game its own DLSS-G (no frame generation on RTX 30 / 20); an old INI's `Auto` now runs as `Bundled` with one `configuration_warning`. `Mode=Pinned` / `External` pointing at a 310.9.1 file is not recognised and the load is refused (`pinned_runtime_rejected` / `external_runtime_rejected` names it as 310.9.1.0). To run 310.9.1, use a `-RuntimeModel 310.9` build (its 310.9.1 hook table, kernel pack and paths are byte-for-byte what they were before the default changed). The factory INI's
`MaxGeneratedFrames=3` (4X) applies directly; for 6X **you have to set it to 5 yourself** (see below).

## One proxy at a time (`version.dll`, then the alternatives in order)

The package root has one proxy, `version.dll`, and nearly every game loads it. **Put exactly one of this project's proxies next to the EXE, always.** If the game does not load `version.dll`, exit the game, **delete it first**, then take the next one from `alternatives\` in this fixed order, one at a time, launching the game after each:

1. `version.dll` (package root, first choice)
2. `alternatives\winmm.dll`
3. `alternatives\dbghelp.dll`
4. `alternatives\dinput8.dll`
5. `alternatives\dxgi.dll` or `alternatives\d3d12.dll`: render-path proxies, on the D3D12 hot path and load-order sensitive, so a last resort, and never both (a Vulkan game does not load either)

The first four are utility DLLs, off the rendering hot path. All six names are the same loader (each loads the same backend from the same `dlssg_sm86\backend\`); only the file name and the forwarding target differ, so changing the name changes no behaviour. The instructions also ship with the package in `alternatives\README.md`.

How to tell that the proxy was loaded: set `[Logging] Level` to `2` for the test and turn frame generation on in the game. A `runtime_redirect` among the game's lines (its EXE name) in `dlssg_sm86\logs\dlssg_sm86.log` means it works, and the `proxies` field of the `configuration` record names the proxy doing the work, for example `{"active":"version.dll","standby":[]}`. No line with the game's EXE name in the log at `Level=2` with frame generation on most often means the game did not load this proxy: try the next one.

Should several of this project's proxies end up in one folder anyway, there is still never a second frame generation:

- The first proxy loaded in the process (the one whose DllMain runs first) takes a process-wide named marker and becomes **active**: it installs the `LoadLibrary` hook, reads the INI and loads the backend.
- Proxies of the same family loaded afterwards become **standby**: they still resolve and forward every export of their own system DLL (the forwarding is complete), but **install no hooks, read no INI and write no log**. `configuration.proxies` then reads like `{"active":"version.dll","standby":["winmm.dll"]}`.
- Which one loads first is the game's load order, not the file name, so do not set it up this way: keep one. When the full INI parse can see a standby, it logs one Level 1 `proxy_multiple{active,standby}` (since 2026-10-20; before, only the Level 2 `configuration.proxies` said so); the behaviour is unchanged.

**Upgrading from a four-proxy release**: 0.3.1 to 0.3.6 had users put all four of `version.dll`, `winmm.dll`, `dbghelp.dll` and `dinput8.dll` in the game folder. When upgrading to this release, delete the old `winmm.dll`, `dbghelp.dll` and `dinput8.dll`: if one of them is loaded first, it runs that old release's own embedded runtime and backend while the new `version.dll` only forwards (`docs/COMPAT.md` C80).

## Where the INI goes and when it takes effect

The configuration file is always named `dlssg_sm86.ini` and sits in the proxy DLL's directory (= the rendering EXE's directory). **Exit the game completely and restart it after editing**; there is no hot reload.

- Boolean switches take `0` or `1`. A line starting with `;` is a comment.
- Enumerated values such as `Mode` are case-insensitive.
- The log directory and the other paths accept absolute paths; a relative path is based on the proxy DLL / INI directory (**not the process's working directory**). A custom path does not expand environment variables such as `%LOCALAPPDATA%` or `%TEMP%`, so write the real path (the one exception is `[Runtime] Path` under `Mode=External`, which is expanded first). `[Runtime] CacheDirectory` was removed on 2026-10-21 (there has been no cache for a long time) and is not read.
- **A value written wrongly (an invalid number, a misspelled enum, out of range, a capture start key whose name is not recognised) is not replaced by a default and is not clamped** (2026-10-19, `docs/COMPAT.md` C97): the log gets one Level 1 `configuration_invalid{section,key,value,allowed,reason,effect}` and **the feature group the key belongs to is not enabled**:
  - `[General]`, `[FrameGeneration]`, `[DLSSG]`, `[XeSSFG]`, `[Compatibility]`, `[Runtime]` (except `SRMode` / `SRPath`) → frame generation is not enabled, exactly as with `[General] Enabled=0`: the game's request for `nvngx_dlssg.dll` is forwarded unchanged, `configuration.enabled` is `false` and `configuration.invalid` is the number of bad keys.
  - `[SuperResolution]` and `[Runtime] SRMode` / `SRPath` → super resolution is not enabled, as with `Mode=Off`.
  - `[Logging]` / `[Debug]` → only that diagnostic is off (`FrameTiming`, `CrashRecord`, `MfgProbe` each; any `Capture*` key turns the frame-generation capture off, any `CaptureSR*` key the super-resolution capture, `Marker*` the generated-frame marker). Exception: `[Logging] Level` / `File` / `Directory` / `EvaluateEvery` keep their default, because they govern the log that carries the record.

  For example `MaxGeneratedFrames=99`, `[DLSSG] Optimized=7` and `Router=typo` each leave frame generation not enabled (they used to become `3`, tier 2 and `Auto`); `[Logging] EvaluateEvery=0` logs one `configuration_invalid` and keeps the default 120 (it used to become 120 silently); `[Debug] MarkerScale=0` logs one and the marker is off. The architecture rewrite of `SpoofArchToGame` / `SpoofArchValue` is decided by the early read at start-up and does not change when another key's bad value switches frame generation off (see the `[Compatibility]` table).
- Only a `configuration_warning` (Level 2), switching nothing off: the old name `[Compatibility] ImageApprox` (`(honoured)` / `(ignored)`), the no-longer-read old places of `[DLSSG] Optimized` / `Preset` (written beside the new key or holding the default: `(no effect)`), `[DLSSG] Preset=Auto` (runs as `A`), keys with no effect in the current configuration, keys of the other engine (`ignored under Engine=…`), `[Runtime] Mode=Auto` (runs as `Bundled`). A removed key is not read: writing it is the same as not writing it, nothing is logged (2026-10-26, C113).
- `configuration_error` (the whole mod switches off and the game's original DLSSG loading is kept) is reserved for the case where **the whole configuration cannot be read at all** (out of memory, or `[Logging] Directory` / `[Debug] CaptureDirectory` / `[Runtime] Path` written as a string that cannot form a path). A wrong value is `configuration_invalid` instead.
- There is one backend: `dlssg_sm86\backend\sm86_backend.dll` beside the proxy (since 2026-10-22, `[Backends]` removed, `docs/COMPAT.md` C102). It is checked once, before the install: `DlssgMod_GetInfo` (the ABI and the runtime SHA-256 it serves) and the presence of this release's exports; a mismatch is `backend_unusable`, the load is refused and frame generation is not enabled. Nothing is negotiated after that.
- **A switch that is on but cannot be carried out is refused, never replaced by another path** (since 2026-10-20, `docs/COMPAT.md` C99). The two you are most likely to meet: **`[FrameGeneration] CatmullRomWarp=1` on the 310.9.16 package (the release) turns frame generation off** — the package has no row for it, the log has one error-level `detail_restore_row_unknown`, and frame generation does not run on without the key; **when `[SuperResolution] Mode` is on and super resolution cannot be installed** (the packaged `dlssg_sm86\sr\nvngx_dlss.dll` missing or wrong, a wrong `SRPath`, the backend unable to install the route), **the game's load of `nvngx_dlss.dll` fails and DLSS Super Resolution is unavailable in the game**, instead of the game's own stock DLSS running silently (error-level `sr_runtime_unavailable` / `external_runtime_rejected` / `sr_unsupported`).

## Normal use: the factory INI

The complete factory `dlssg_sm86.ini`:

```ini
[General]
Enabled=1

[FrameGeneration]
Engine=DLSS
MaxGeneratedFrames=3

[DLSSG]
Optimized=1
Preset=A

[XeSSFG]
FlowResolution=Middle
UseHudless=Auto

[Logging]
Level=1
Directory=dlssg_sm86\logs

[Runtime]
Mode=Bundled

[SuperResolution]
Mode=Off
```

**Two frame-generation engines (2026-10-14).** Keys are grouped by who reads them: `[General]`, `[FrameGeneration]`
(`Engine`, `MaxGeneratedFrames`), `[Logging]`, `[Runtime]` and `[SuperResolution]` are read under either engine;
`[DLSSG]` only under `Engine=DLSS`, `[XeSSFG]` only under `Engine=XeSS`. A key of the other engine written with a
non-factory value logs one `configuration_warning` (`default` `(ignored)`, `reason` `ignored under Engine=…`); factory
values do not, so the factory INI logs no warning under either engine. Of the advanced keys below, the `[FrameGeneration]`
`CatmullRomWarp` / `ImageApprox*` and the `[Compatibility]`
kernel keys (`KernelImage`, `Router`, `DisableFusions`, `ImagePatches`, `ImageApprox`)
**apply to the DLSS engine only**; the `[Compatibility]` process keys
(`SpoofArchToGame`, `SpoofArchValue`, `ForceSM86Route`, `SimulateAmpere`) are needed by both. `Engine=XeSS` runs Intel's
XeSS-FG model (model 15, which goes to 4X, so the factory `MaxGeneratedFrames=3` works as is; 2X is model 18's graph, see below) on this project's own runner (CUDA kernels recorded on the game's command list through NVAPI; the only runner since 2026-10-22, `[XeSSFG] Runner` removed) from `dlssg_sm86\xefg\xefg_kernels.dll`; Intel's
`libxess_fg.dll` is neither shipped nor loaded. NVIDIA's runtime still creates the feature and Streamline still paces
and presents; only the Evaluate is replaced. What XeSS cannot run is **never handed to DLSS**
(docs/COMPAT.md C94): a missing or damaged `xefg_kernels.dll` (its CUDA images included) is one
error-level `xefg_unavailable` at install and frame generation is refused (FG off); what is
known at CreateFeature - a declared back-buffer format other than R8G8B8A8 / B8G8R8A8 / R10G10B10A2 (e.g. RGBA16F, common
for HDR), CUDA objects that cannot be created - is one error-level `xefg_feature_refused` and that feature is refused (FG off; its `remedy` is written only where the cause has one: the declared format -> change it or `Engine=DLSS`; a D3D12 device already removed before XeSS-FG was created (`hresult` `0x887a0005` / `0x887a0006` / `0x887a0007`, plus `device_removed_reason`) -> says this is not a configuration problem; any other failure names `call` and `hresult` only, C107);
anything that fails after the feature exists (a missing input, a format or rect it cannot run, an
object / record / launch failure) is one error-level `xefg_kernel_fatal` naming the cause, then the process ends with
`0xE0D55001` (the C93 code). `MaxGeneratedFrames` is a requirement, not a ceiling (see the table below): above 3 with
`Engine=XeSS` the loader writes one Level 1 `max_generated_frames_unsupported{engine:"xess"}` and frame generation is not
enabled (as with `Enabled=0`); it is no longer clamped to 2X (`0` = the game's own count is accepted too).
The HUD-less image follows `[XeSSFG] UseHudless`: the factory `Auto` is the same as `0` since 2026-11-10 (C139) - the back
buffer is the network source and the whole image, UI included, is interpolated, whether the game supplies a HUD-less image
or not, so `xefg_hudless_unused` no longer appears; `1` is strict: a frame with no HUD-less image, or with it / the UI plane
at another size than the back buffer, is `xefg_kernel_fatal{kind:"hudless"}` and the process ends with `0xE0D55001` (C94);
`0` never uses it.

**The model** (a `[XeSSFG] Model` key from 2026-10-20 to 10-26, C100 / C105; one model and the key removed since 2026-10-26, C111): XeSS-FG runs Intel's model 15 (the multi-frame model) as ported by MFG-Unlocker 1.6.0 (dashdogy, MIT), ceiling 3 (`MaxGeneratedFrames` 1-3, 4X); a `Model` left in an old INI is not read (C113). **2X stays model 18**, exactly as the provider does: it swaps its 2X model for the ported one only for more than one generated frame. 3X / 4X take model 15's own multi-frame path: the optical flow (D01-D28) once per real frame, the synthesis (D29-D44) once per index i at t = i / (N + 1); it also reads the previous frame's depth (and, with HUD-less, the previous back buffer), kept in its own two-slot history, which 2X frames update too: the depth (and, with HUD-less, the back buffer) is copied, the network source is read from model 18's own history (read-only; the next multi-frame real frame first checks it is that frame, a mismatch is a reset), so 2X costs one or two more copies (RTX 3070 offline: 2X 4-5 % slower than the former `Model=18`; in game on a 5070 +1.7 %, bit-identical output). When the game switches the multiplier: 2X -> 3X / 4X continues the history, the first 2X frame after 3X / 4X is a reset (model 18's history did not see that frame). The shaders it shares with model 18 (D01, some trunk layers, the whole decoder and composite) run on model 18's CUDA kernels, the port's own trunk layers, D02, D28 and D30 on its own kernels, D31 and a few small dispatches (D03, D13-D17, D27, D29) run their DXIL shaders on the same command list (measured: DXIL is faster; D31's translation matches on 30 of 31 offline groups but is 1.8-1.9x slower), and D01 reuses the previous real frame's luma pyramid (model 18's way). Model 15's programs, weights and CUDA images **ship** inside the same `dlssg_sm86\xefg\xefg_kernels.dll` (since 2026-10-21, resource `MANIFEST15`; before that a separate, unshipped `xefg_m15.dll`): missing or different = `xefg_unavailable` at install (`reason` starting `model15_`), frame generation refused - never model 18 or DLSS in its place. Cost (RTX 3070, offline full-route replay, `Middle`, DIRECT, GPU ms per real frame with its generated frames): 4X 1440p 4.5-5.0 (DLSS default 5.3-5.4), 4K 10.3-10.8 (DLSS default 7.9-8.5); offline GPU time is not the game's frame rate. With jittered motion vectors (`DLSSG.MvecJittered=1`) the provider refuses this configuration at 3X / 4X (it has no program for it); this engine ports model 18's handling of the flag (both warp directions of a candidate position add (jp - jc) * (1 - t), jc / jp being this / the previous frame's jitter) to model 15's D31 at each index's own t: D31 then runs a CUDA kernel carrying that term, about +0.08 ms per generated frame at 1440p and +0.17 ms at 4K, no slower with an R10G10B10A2 back buffer (RTX 3070 offline), everything else unchanged; without jitter the output is byte-identical to before (C134). RTX 3070 offline: the full NGX path is byte-identical to the provider (`libxess_fg.dll` 1.3.1.78 + the port, run offline) at every index. Licence: 46 of the programs are derived from Intel's XeSS SDK programs and fall under Intel's XeSS SDK terms (THIRD_PARTY_NOTICES.txt).

**Kernels and routes on RTX 50 (2026-10-28, `docs/COMPAT.md` C132, C133)**: most XeSS-FG dispatches run this project's CUDA kernels and a few run the shaders' own DXIL (the pack's route table, measured on a 3070). RTX 40 / 50 have no cubin of this project and run the pack's shared PTX (compiled by the driver). That PTX is compute_75, which has only Turing's INT8 Tensor-Core instruction shape; Blackwell runs each of those as a half-empty m16n8k16, so the INT8 Tensor-Core kernels ran at about half speed there. On Blackwell (SM 10.0 and up) the 26 INT8 Tensor-Core kernels therefore use an image of their own in the pack (`<kernel>.bw`: compute_86 PTX, each pair of neighbouring 3x3 taps issued as one m16n8k32) and every other kernel keeps the shared PTX; D25 and model 15's D02 and D30 run their DXIL there (faster than CUDA). Both routes and both images write byte-identical output, so this changes speed only: RTX 5070, synthetic scene 3440x1440, alternating in one process against the previous setup (C132: shared PTX, D26, D31, D33..D38 on DXIL), FG GPU time per real frame 5.68 -> 4.31 ms at 4X, per generated frame 2.77 -> 1.89 ms at 2X; in Cyberpunk 2077 at 3440x1440 (against the C132 numbers) 5.81 -> 4.68 ms per real frame at 4X with 48.5 -> 52.0 real fps, and 2.43 -> 1.69 ms at 2X. The log's `xefg_cuda.blackwell_images` is how many of the model's kernels were created from a `.bw` image (21 for the 2X model and 26 for the multi-frame model on RTX 50, 0 elsewhere); `route_profile` is `blackwell` on RTX 50 and `pack` elsewhere. RTX 40 keeps the shared PTX and RTX 20 / 30 their cubins, unchanged.

There is only one switch that decides "how frame generation runs": `[DLSSG] Optimized` (`[FrameGeneration] Optimized` before 2026-10-14; the old place is no longer read since 2026-10-22). It is a **consistency tier**, with a single criterion:
**how far the generated image may move from the official NVIDIA runtime**. A higher tier is faster and further from the official image; every other knob follows from the tier,
and a normal user does not need to understand them one by one.

| Tier | Setting | Consistency guarantee | What this tier turns on |
|---|---|---|---|
| **0 stock** | `Optimized=0` | **Bit-identical to the official runtime** | No acceleration at all. **Note this is not "no kernel replacement"**: the SM86 route is still active, and what the backend substitutes are the **stock kernel images** extracted from the runtime (cubin on a real SM86 card, PTX on other architectures), numerically identical to the runtime. It is not the same as `KernelImage=Original` — which matters most on **Ampere**: 310.9.1's stock image kernels are sm_89 blobs that cannot be created on Ampere at all, so this path is the only way to get "stock numerics" on Ampere. |
| **1 bit-identical (the reference; default, recommended)** | `Optimized=1` | **Bit-identical to the official runtime** (measured on two real game captures, 320 images). Use this tier to compare against the official image bit for bit, or to decide whether something on screen comes from this project | Every acceleration that is **provably output-preserving**: 34 rewritten kernel variants + all cross-kernel fusions and the exact image-kernel patches (P1/P3/P4/P7/P8/P9/P10). Equivalent to the full optimized set + `ImagePatches=1, DisableFusions=0`. |
| **2 fast (lossy)** | `Optimized=2` | **No longer bit-identical**: 44–53 dB PSNR against the official numerics on real 4K 4X game captures, lower the faster the motion (worst frame 44.06 dB, the fastest stretch of a Resonance capture, RTX 3070); against ground truth only 0.02–0.13 dB below tier 1, and side by side on the tiles where the two differ most, no difference is visible (see "Tiers 2 / 3" below) | Tier 1 plus the lossy image-kernel rows (today one: A6, nearest-tap warp sampling in `Kernel_OutputPull`). 1.8 % (1080p) / 1.8 % (1440p) / 2.9 % (4K) faster per generated frame than tier 1 (RTX 3070, 4X, 0.4.0; 2.6 / 2.9 / 4.5 % before 0.4.0's P8/P9/P10 made tier 1 itself faster). |
| **3 fastest (lossy)** | `Optimized=3` | **No longer bit-identical**, the largest image-quality cost | Every lossy acceleration that is still faster: every lossy image-kernel row. Variants that are slower or show structural artifacts are permanently retired and are enabled at no tier. |

The tiers are **cumulative and monotone**: a higher tier only adds, it never withdraws an acceleration a lower tier turned on. To override one item individually, use
`ImagePatches` / `ImageApprox` / `ImageApproxMask` from "Advanced / diagnostic keys" below — they are applied after
the tier, so they can switch off something the tier turned on, or turn one item on at a lower tier.

The remaining factory keys:

| Key | Section | Default | Description |
|---|---|---|---|
| `Enabled` | `[General]` | `1` | `1` enables DLSSG redirection, adaptation and frame generation; `0` disables it: the game loads its own DLSSG unchanged (which on Ampere means no frame generation). The proxy still forwards the system DLL's original exports. |
| `Engine` | `[FrameGeneration]` | `DLSS` | Frame-generation engine: `DLSS` = NVIDIA's DLSS-G model (this project's SM86 / SM75 kernels); `XeSS` = Intel's XeSS-FG model (this project's own runner, model 15, 2X-4X; see above). `XeSS` is recorded as `configuration.engine{requested,effective,xessfg}` (absent for `DLSS` or no key) and the loader's `xefg_requested` record; the backend writes `xefg_engine` / `xefg_context` / `xefg_summary` (or `xefg_unavailable` / `xefg_feature_refused` / `xefg_kernel_fatal`). A misspelling is `configuration_invalid` and frame generation is not enabled. |
| `Optimized` | `[DLSSG]` | `1` | Consistency tier `0`–`3`, see the table above (factory `1`; `2` from 2026-10-04 to 10-21, and an old INI that writes `Optimized=2` still means tier 2). Not written, it is tier 0. **The old places** `[FrameGeneration] Optimized` (before 2026-10-14) and the older `[Compatibility] OptimizedKernels` are no longer read since 2026-10-22: written beside `[DLSSG] Optimized`, or holding `0` (what the absent new key runs anyway), each logs one `configuration_warning` (`(no effect)`); written alone with any other value it is `configuration_invalid` (`allowed` names `[DLSSG] Optimized`) and frame generation is not enabled, rather than running tier 0 silently. DLSS engine only. **A wrong or out-of-range value (such as `7`) is `configuration_invalid` and frame generation is not enabled** (it used to fall back to tier 2). |
| `MaxGeneratedFrames` | `[FrameGeneration]` | `3` | Generated frames per real frame, `0`–`5`: `5` = 6X, `3` = 4X, `2` = 3X, `1` = 2X, `0` = keep the runtime's and the game's own count. **A requirement, not a ceiling, and the highest-priority declaration** (2026-10-19): the runtime answers the game's `DLSSG.MultiFrameCountMax` with exactly this number, and the game's menu may still pick a lower multiplier; a driver-profile frame limit (private key `0x104C9A99`) cannot lower it — the backend hooks the runtime's DRS reader and drops that key (`mfg_drs_override`, Level 1 when the key was actually set); on a runtime build where that hook point is unknown (310.9.0) a lower profile limit refuses frame generation (`max_generated_frames_unsupported{reason:"driver_profile_limit"}`). What cannot be served is an error, never lowered: above `5` is `configuration_invalid`; above `3` with `Engine=XeSS` is `max_generated_frames_unsupported{engine:"xess"}` and frame generation is not enabled; above `1` on an RTX 40 running the runtime's own kernels (now only `KernelImage=Original`) is `max_generated_frames_unsupported{reason:"runtime_kernels"}` and frame generation is refused; a game that ships a generation-A runtime (≤ 3.8.x, an old Streamline plugin that drives 2X only) does not get frame generation whatever is written here (see "Which generation the game's own runtime is"). The factory value is `3` (4X): a game with dynamic MFG runs at whatever is written here by default, and `5` is too high for most people (public issues #497/#499), so 6X became **a manual edit when you want it** — set this line to `5`. A game that ships an older 4X Streamline plugin (generation B) stays at 4X whatever is written here (the plugin's own constant). |
| `Preset` | `[DLSSG]` | `A` | DLSS-G render preset, **pinned on every feature: what the INI says runs**, whatever the game's `DLSSG.UserInterfaceRecompositionEnabled` / `DLSSG.LiteCandidateBlendingEnabled` or the driver preset `0x10E41DF1` ask for (2026-10-22, `docs/COMPAT.md` C102). `A` (default, also when not written) = UI recomposition off, FG Lite off; `B` = UI recomposition on (cleaner HUD/UI inside generated frames) — **but B only takes effect when the game hands DLSS-G both a HUD-less image and a UI plane**, which most games do not, and then it looks like A; that case logs one Level 1 `preset_ineffective` (each feature handle is checked once, at its first Evaluate, naming the missing plane; COMPAT C87); `C` = FG Lite (310.9.16 only: −32…−54 % GPU time per generated frame, slightly lower image quality). `Auto` (the factory value before 2026-10-22, leaving it to the game / driver) was removed: written, it runs as `A` with one `configuration_warning`. `[DLSSG] Preset=` with an empty value is a wrong value: `configuration_invalid`, frame generation is not enabled. The old place `[Compatibility] Preset` is no longer read, by the same rule as `Optimized`'s old places: beside the new key, or holding `A` / `Auto`, it only warns; alone with `B` / `C` it is `configuration_invalid`. DLSS engine only. |
| `FlowResolution` | `[XeSSFG]` | `Middle` | XeSS-FG optical-flow resolution: `Coarse` / `Middle` / `Fine` (the XeSS-FG runtime's `0x410` / `0x408` / `0x208`); finer is slower. `CoarseAt4K` (opt-in, owner 10-17) = `Coarse` for a feature whose output has at least 3840 × 2160 pixels (8 294 400; total pixels, not height: 5120 × 2160 and 3840 × 2400 qualify, 2880 × 2160 does not), `Middle` otherwise, decided once per feature at CreateFeature from the output size the game declares and logged as one `xefg_flow_choice` (on an RTX 3070 at 4K, `Coarse` saves about 0.45–0.52 ms per generated frame over `Middle`; image quality in docs/COMPAT.md C95). XeSS engine only; the value is checked under either engine, and a misspelling is `configuration_invalid` with frame generation not enabled. |
| `UseHudless` | `[XeSSFG]` | `Auto` | `Auto` (default) = the same as `0` since 2026-11-10 (C139): the back buffer is the network source and the whole image, UI included, is interpolated, whether the game supplies a HUD-less image or not; `1` = strict: a frame without a usable HUD-less image is `xefg_kernel_fatal{kind:"hudless"}` and the process ends (`0xE0D55001`); `0` = never (same as `Auto`). Before 2026-10-19 the values were `0/1` with factory `1`; between 10-19 and 11-10 `Auto` decided per frame. XeSS engine only; a misspelling as above. |
| `Level` | `[Logging]` | `1` (`2` when absent) | `0` no logging; `1` errors only; `2` adds configuration, loading and capability information; `3` adds kernel creation, Evaluate and marker information on top. Every module of every process appends to one file, `<Directory>\dlssg_sm86.log` (by default `dlssg_sm86\logs\dlssg_sm86.log`); each line starts with the local time, `pid N`, the EXE name and a level letter (E = Level 1, I = 2, D = 3), then the JSON record, e.g. `2026-10-22 14:03:11.482  pid 12345  Cyberpunk2077.exe  E  {"event":...}`. To look at one game, filter by EXE name or pid (`Select-String`, `findstr`). The proxy writes a `log_session` record (exe, pid, proxy path and SHA-256, version) before each process's first record. The file is never emptied by the mod: delete it to start over. |
| `Directory` | `[Logging]` | `dlssg_sm86\logs` | Log directory; a relative path is based on the INI's directory. |
| `Mode` | `[Runtime]` | `Bundled` | Where the runtime comes from. `Bundled` = the packaged `dlssg_sm86\runtime\nvngx_dlssg.dll` (use this for a normal install; no version matching needed). `External` (`[Runtime] Path` names another location; its default is that same file) and `Pinned` are advanced, see below. `Auto` was removed in 0.4.0: written, it runs as `Bundled` with one `configuration_warning`. |
| `Mode` | `[SuperResolution]` | `Off` | DLSS Super Resolution, RTX 30 / RTX 20 only. `Off` = not touched, exactly as without the section; `M-INT8` / `L-INT8` = preset M / L pinned on every quality mode, run on this project's INT8 kernels (the mode is chosen in the game; Quality / Balanced / DLAA are slower than the game's own K); `M-FP16` / `L-FP16` = the same preset on NVIDIA's own kernels. Any other value leaves super resolution off. See the `[SuperResolution]` section below. |

A normal install can leave the factory INI alone (tier 1, output bit-identical to the official runtime). Set `Optimized` to `2` for a little more speed
without bit-identity, or to `0` to compare against stock numerics; nothing else needs touching. An upgrade that keeps
its old INI keeps the tier written there (0.4.0 shipped `Optimized=2`; **factory INIs from before 2026-10-14 write it
in `[FrameGeneration]` and `Preset` in `[Compatibility]`, two old places no longer read since 2026-10-22**: a lone
`[FrameGeneration] Optimized=1` is `configuration_invalid` and frame generation is not enabled - use the new factory INI or move both lines into `[DLSSG]`).

## RTX 40 / RTX 50 (since 0.4.0)

Since 0.4.0 this project's frame-generation kernels **also run on RTX 40 (sm_89) and RTX 50 (sm_120)**, at the tier
`Optimized` selects (factory tier 1). Before, the runtime used only its own kernels on those cards. Only GPUs of compute
capability 8.9 / 12.0 whose CUDA name carries `RTX` qualify (workstation boards such as the RTX 6000 Ada and the RTX PRO 6000
Blackwell included; data-centre boards without RTX in the name are not touched). Since 2026-10-19 this is no longer an option
that can be switched off: `[Compatibility] AccelerateNewerGpus` was removed
(not read), whether frame generation runs is decided by `[General] Enabled` alone, and when it runs it runs this
project's kernels. The only way left to run the runtime's own kernels on these cards is the diagnostic value
`KernelImage=Original` (on an RTX 40 that is 2X only: with `MaxGeneratedFrames` above 1 frame generation is refused, see the
table above).

- **The speed has not been measured on RTX 40 / 50 hardware yet.** The development machine has an RTX 3070: there the route
  ran end to end through a development switch (the same PTX, output bit-identical to the RTX 30 route); in 2026-09 on an RTX
  5070 these kernels (the version of that time - later changes were verified bit-exact on the RTX 3070 only) matched the
  official output bit for bit, but their time against the official kernels has never been measured. On an RTX 40 / 50 the one-click test kit adds an "official" column by itself (see the kit's README); until it has
  been run the gain is unknown and could be a loss.
- **The images are PTX**: the pack's sm_86-family PTX is JIT-compiled by the driver (there is no sm_89 / sm_120 cubin). The first
  frame-generation feature creation compiles it once: measured on an RTX 5070, a cold CreateFeature takes about 3.0-3.3 s in all
  (about 1.2 s for the 70 base kernels - on an RTX 50 the official runtime JIT-compiles as much PTX of its own - and about
  1.7-1.8 s for this project's 63 optimized rows at the time, 37 since 2026-10-21), and once the driver's compute cache (`%APPDATA%\NVIDIA\ComputeCache`, capped
  at 1 GiB) holds it about 0.13 s; after a driver update or an eviction it compiles again. On an RTX 40 the official runtime
  uses cubins and compiles nothing, so there all of it is new.
- **Image**: on RTX 50 tiers 0 / 1 are expected to be bit-identical to the official output (tier 2 is lossy as always). **On
  RTX 40 tiers 0 / 1 are not bit-identical to the official output either**: there the official runtime runs a different build
  (the Ada build of its image kernels, compiled for a product without multi-frame generation, and an sm_89 cubin network),
  while this project runs the RTX 50 build's algorithm; measured on an RTX 5070 the two builds are 55-82 dB apart (mostly on the
  UI-recomposition / candidate-blend path).
- **Multiplier**: the runtime answers the game's `DLSSG.MultiFrameCountMax` with `MaxGeneratedFrames`, on an RTX 40 too (this
  project's kernels step the generated frame's phase, `docs/COMPAT.md` C92); which modes the game's menu actually offers is up
  to the game and its Streamline plugin, and has not been checked on real RTX 40 hardware.
- **Unchanged**: every architecture answer the game, Streamline and
  the NGX core see (`SpoofArchToGame` rewrites nothing on these GPUs, and the snippet's minimum architecture is not rewritten as it
  is on RTX 30).
- **On failure** (since 2026-10-16, the same on every GPU): if the driver refuses any image of this project (a PTX compile or
  create failure) **the game crashes on purpose**; the log ends with one error-level `fg_kernel_fatal` (which kernel, why) and one
  `fg_gate_crash` (exception code `0xE0D55001`) - it never quietly switches to the runtime's own kernels. With the kernel pack
  `dlssg_sm86\kernels\fg_kernels.dll` missing or not this release's, frame generation **does not start**
  (`fg_kernels_unavailable`), again without the runtime's own kernels standing in (`docs/COMPAT.md` C93).
- Log: on GPUs other than RTX 30 / 20 the backend writes one `newer_gpu_route` (`engaged`, `reason`, `image`,
  `multi_frame_advertised` and more), visible at `Level=2`.

## INI key stability: three tiers

The loader reads 62 INI keys (`ini_keys` of `tests/golden/surface.json`; the `compat_surface` CTest pins every one,
see `docs/COMPAT.md` C4). They come in three tiers of promise, plus four old names (owner
decision 2026-10-05; the two-engine layout of 2026-10-14 added two, C84; three of them are no longer read since 2026-10-22, C102):

| Tier | Which keys | Promise |
|---|---|---|
| **1 frozen** | the 11 keys of the factory INI (`[SuperResolution] Mode` among them since 2026-10-10; `[FrameGeneration] Engine` and the `[DLSSG]` / `[XeSSFG]` sections since 2026-10-14; `[XeSSFG] Model` from 2026-10-23 to 10-26, removed by C111) | Section, name and the meaning of each value stay. After a rename the old spelling either records one `configuration_warning` naming the new one (when the new key is written too, or the old line holds what the absent new key runs anyway) or is `configuration_invalid` (when honouring it silently would run something else) - never silently something else: `Optimized` / `Preset` moved into `[DLSSG]` on 2026-10-14, and their old places follow this rule since 2026-10-22. |
| **2 kept compatible** | the 3 keys for players: `[Compatibility] SpoofArchToGame`, `[FrameGeneration] CatmullRomWarp`, `[Runtime] Path` (`[SuperResolution] Tier` and `[Compatibility] AccelerateNewerGpus` were in this tier, removed on 2026-10-19; `[XeSSFG] Runner` was in this tier too, removed on 2026-10-22) | Values may be added and behaviour improved, but nothing stops working silently: after a rename the old spelling either keeps working or records a `configuration_warning` naming what replaces it. Removal: see below. |
| **3 diagnostic** | everything else: keys for development, validation, capture, probes and measurement, the kernel format and family, the reported architecture value, the advanced runtime routes | May be renamed, change meaning or go between versions. |
| **old names** | `[FrameGeneration] Optimized` and `[Compatibility] OptimizedKernels` (old place / old name of `[DLSSG] Optimized`), `[Compatibility] Preset` (old place of `[DLSSG] Preset`), `[Compatibility] ImageApprox` (the old place of `[FrameGeneration] ImageApprox`) | `[Compatibility] ImageApprox` keeps its old meaning; writing it records a `configuration_warning` naming the new key (`(honoured)`; `(ignored)` when the new place is written too). The other three are no longer read since 2026-10-22: beside the new key, or holding the value the absent new key runs anyway (`Optimized=0`, `Preset=A` / `Auto`), one `configuration_warning` (`(no effect)`); alone with any other value, `configuration_invalid` (`allowed` names the new key) and frame generation is not enabled. |

Whatever the tier, removing a key fails `compat_surface`; the change that removes it updates
`tests/golden/surface.json` and adds a line to `tests/golden/README.md`: what went, why, and what replaces it.

**A removed key is not read** (since 2026-10-26, owner, C113; before, writing one logged a `configuration_warning`): writing it is exactly the same as not writing it - nothing is logged, warned or refused. What an old INI may still contain:

- `[FrameGeneration]`: `SkipRepeatedRealCopy` (2026-10-08, see "`SkipRepeatedRealCopy` (removed)" below), `ForceGeneratedFrames` (10-19), `DetailRestore` and `DetailRestoreA` / `B` / `C` / `D` / `Alpha` / `Gate` / `CasMul` / `WeightProbe` (10-21; mode A is now `CatmullRomWarp`, with the two values swapped: the old `DetailRestoreA=2` (five taps) = `CatmullRomWarp=1`), `ForcePluginFrames` (10-22).
- `[Compatibility]`: `AccelerateNewerGpus` (10-19), `HardwareBilinear` / `LaunchChains` / `PlainVariant` / `ChainBlock0` (10-21), `SM75Family` (10-22).
- `[XeSSFG]`: `Runner` (10-22), `Model` (10-26).
- `[Runtime]`: `RRMode` / `RRPath` (10-09), `CacheDirectory` (10-21); the whole `[RayReconstruction]` section (10-09); the whole `[Backends]` section (10-22).
- `[SuperResolution]`: `Int8`, `Int8L`, `Int8M`, `ForcePreset*` (10-03), `Tier` (10-19).
- `[Logging] DebugOutput` (10-22); `[Debug]`: `CaptureRR` / `CaptureFeature` (10-09), `QueueTrace` (10-21), `NgxLog` / `NgxLogLevel`, `CaptureInputs` / `CaptureOutputs` / `CaptureRingSlots` / `CaptureVramMB`, `CaptureSRFrames` (the old alias of `CaptureSR`) / `CaptureSRSkip` / `CaptureSRRingSlots` / `CaptureSRMemoryMB` / `CaptureSRVramMB` / `CaptureSRAsyncCopy` (10-22).

Removed **values** still warn: `[DLSSG] Preset=Auto` (2026-10-22, runs as `A`) and `[Runtime] Mode=Auto` (0.4.0, runs as `Bundled`) each log one `configuration_warning` - a change in a tier-1 key's values, by owner decision. On 2026-10-19 `[XeSSFG] UseHudless=1` became the strict setting (the old behaviour became `Auto`, and the factory value became `Auto`), likewise by owner decision; since 2026-11-10 `Auto` is the same as `0` (C139).

| Section | Tier 1 (11) | Tier 2 (3) | Tier 3 (44) | Old names (4) |
|---|---|---|---|---|
| `[General]` | `Enabled` |  |  |  |
| `[DLSSG]` | `Optimized`, `Preset` |  |  |  |
| `[XeSSFG]` | `FlowResolution`, `UseHudless` |  |  |  |
| `[FrameGeneration]` | `Engine`, `MaxGeneratedFrames` | `CatmullRomWarp` | `ImageApprox`, `ImageApproxMask` | `Optimized` (old place of `[DLSSG] Optimized`, no longer read) |
| `[Compatibility]` |  | `SpoofArchToGame` | `DisableFusions`, `ForceSM86Route`, `ImagePatches`, `KernelImage`, `Router`, `SimulateAmpere`, `SpoofArchValue` | `ImageApprox` (the old place of `[FrameGeneration] ImageApprox`), `OptimizedKernels` (the old name of `Optimized`, no longer read), `Preset` (the old place of `[DLSSG] Preset`, no longer read) |
| `[Logging]` | `Level`, `Directory` |  | `CrashRecord`, `EvaluateEvery`, `File`, `FrameTiming` |  |
| `[Runtime]` | `Mode` | `Path` | `SRMode`, `SRPath` |  |
| `[SuperResolution]` | `Mode` |  | `Auxiliary`, `KernelImage`, `KernelSet`, `KernelSetSM`, `ObserveFrames` |  |
| `[Debug]` |  |  | `Capture`, `CaptureAsyncCopy`, `CaptureDirectory`, `CaptureLean`, `CaptureMemoryMB`, `CaptureNoStall`, `CaptureSR`, `CaptureSRBursts`, `CaptureSRDedup`, `CaptureSRDirectory`, `CaptureSRLean`, `CaptureSRMaxGB`, `CaptureSRNoStall`, `CaptureSRStartKey`, `CaptureSRStartOnCreate`, `CaptureSRStride`, `CaptureSkip`, `CaptureStartKey`, `CaptureStartOnReset`, `MarkGeneratedFrames`, `MarkerScale`, `MarkerX`, `MarkerY`, `MfgProbe` |  |


## Advanced / diagnostic keys (not in the factory INI)

The keys below are **still parsed and supported by the loader** (development, validation, monitoring, capture and the MFG probe all need them); they are simply not in the factory INI. **When absent each takes its safe / best default**, so a normal install does not need to write any of them. To use one, add the section and key to the INI (add a `[SectionName]` line yourself if the section is missing).

> **Note: the defaults below are already the fastest / safest set.** Turning it on **reduces performance** and is only for diagnostics: `DisableFusions>0`.

### [FrameGeneration]

| Key | Values | Default | Description |
|---|---|---|---|
| `ImageApprox` | `0/1` | **set by the tier** (`1` from tier ≥2) | Explicit switch for the lossy image-kernel rows, overriding the tier's answer: `1` at tier 1 adds only the lossy image rows, `0` at tier 2/3 keeps only the bit-identical part. Tier 0 builds no row at all, so `1` there does nothing (and logs a `configuration_warning`). `[Compatibility] ImageApprox` is its old place: still read, and writing it logs a `configuration_warning` naming this key. |
| `ImageApproxMask` | bitmask | **set by the tier** (tier 2 = the default set, tier 3 = all) | Names exactly which lossy image-kernel rows are enabled, one bit per fixed slot. Slots are **assigned once and never recycled**: a retired row leaves its slot empty, so a mask value written down in an old document always means the same set of rows; an unassigned bit in the mask is ignored and logs one `image_approx_mask_unassigned`. When one kernel has several rows, only the row with the lowest bit number is enabled and the other logs `image_approx_row_shadowed`. The armed rows, slots and resource ids are all in the backend's `image_approx` record. |
| `CatmullRomWarp` | `0/1/2` | **`0`** (no tier turns it on) | Resamples the two **motion-warped taps** of `Kernel_BlendCandidatesFused` with **Catmull-Rom** instead of the sampler's bilinear. `1` = the **five-tap** row (the separable 4×4 with the four corner taps dropped and renormalised; 5 hardware-filtered taps per candidate) - the measured value for money and the one recommended value; `2` = the exact separable 4×4 (**nine taps**, 9 per candidate), no more detail energy for twice the cost, kept for completeness rather than recommended. It **adds no decision logic** (a pure resampling change, no "if the check passes, then" branch), and at zero displacement Catmull-Rom reduces to the centre texel, so a still image is unchanged by construction. Measured on an RTX 5070 (Cyberpunk 4K capture): the fine-structure energy ratio `E_gen/E_real` rises from **0.897 to 0.953**, about **+2.5 %** per generated frame at 1080p / 1440p and about **+4 %** at 4K (+20 % per launch for that one kernel, K39; +42 % with nine taps); **the HUD tiles improve too**, but **the improvement visible to the eye is slight**. It deliberately **departs** from the official output (no longer bit-identical), so no `Optimized` tier includes it. **It needs `Optimized` ≥1 and `[Compatibility] ImagePatches` not 0**: it is an image-patch row (built on the exact patch P7), tier 0 builds no row and `ImagePatches=0` switches every image-patch row off; in those configurations it does nothing, the loader logs a `configuration_warning` and the backend's `catmull_rom_warp` record says `enabled:false` with a `disabled_reason`. At `0` the output is bit-identical to the official runtime. When the kernel pack has no row for it (**the 310.9.16 model's pack has no CatmullRomWarp row**), one error-level `detail_restore_row_unknown` and frame generation is refused (since 2026-10-20; before, it simply did nothing). |

### [Logging]

| Key | Values | Default | Description |
|---|---|---|---|
| `File` | `0/1` | `1` | `1` writes log files; `0` disables file output (still governed by `Level`). |
| `EvaluateEvery` | `1`–`1000000` | `120` | Sampling interval for the normal Evaluate / marker log lines (counted in Evaluate calls). `1` logs every one; `0` or out of range is `configuration_invalid` and this key keeps the default 120 (it governs the log that carries the record). At Level 3 the first 12 are always logged. |
| `FrameTiming` | `0/1` | `0` | `1`: one line per real frame in `frame_timing.txt` under `Directory` (plain text, one appended session per process) - the CPU frame interval, the GPU time of that frame's super-resolution Evaluate and of all its frame-generation Evaluates (D3D12 timestamps) - and one summary per second including the GPU utilisation, whatever `Level` / `File` say. `0` or absent: nothing is loaded or hooked for it. An invalid value writes one `configuration_invalid` and the frame-timing file is off. See "The frame-timing file" below. |
| `CrashRecord` | `0/1` | `1` | When the game crashes (an unhandled exception), leave one `fg_gate_crash` record in `dlssg_sm86.log` (the crashing process's last line): the exception code, the address, the faulting module and offset, the thread, and whether the module is ours. Record only: the filter is a `SetUnhandledExceptionFilter` that chains to the previous one, so the crash itself, the exit code and the game's own crash reporter are unaffected; no vectored handler. `0` turns it off (for example if you suspect it conflicts with a game's crash reporter). Present only once the backend is installed (a super-resolution-only process has none); a malformed value logs one `configuration_invalid` and the crash record is off. What it cannot cover: `docs/COMPAT.md` C89. |

#### The frame-timing file `frame_timing.txt` (`[Logging] FrameTiming=1`)

A process with `FrameTiming=1` **appends** to **one** plain-text file, `frame_timing.txt` in `[Logging] Directory` (by default `dlssg_sm86\logs`, the folder of `dlssg_sm86.log`, so a zipped log folder carries it), from its first DLSS Evaluate on (since 2026-10-22; before that one `frames_<pid>.jsonl` per process). Each process writes one session that opens with a `# ==== session` line (pid, exe, start time, GPU), then the INI path, the timestamp frequency, the column legend and the column names. It is written with super resolution only, frame generation only, or both; with `[General] Enabled=0` too, where it times the game's own DLSS / DLSS-G - the other half of an A/B against this project. A Vulkan game is not timed; one `# vulkan` line says so. The file is never emptied by the mod: delete it to start over; two processes timing at the same time interleave in 0.1 s blocks.

One line per real frame (a frame the game rendered): with frame generation on, that is one group of frame-generation Evaluates with `MultiFrameIndex` 1..N, together with the super-resolution Evaluate before it; without frame generation, one super-resolution Evaluate. Data rows start with the frame number, in fixed columns, `-` where a value does not exist; every other line starts with `#`:

| Line | When | What |
|---|---|---|
| `# ==== session …` and the four after it | before each process's first frame | pid, exe, start time (UTC), GPU name; the INI; timestamp frequency and list type; column legend; column names |
| `# context from frame N: …` | before the first frame, then on every change | super resolution: render (input) and output size (from the Evaluate's own `DLSS.Render.Subrect.Dimensions` / `Color` and `Output`, C108), preset, `kernels` (`int8` = this project's INT8 kernels, `native` = this project's route is in place and NVIDIA's kernels ran, `game` = the game's own DLSS); frame generation: multiplier, generated-frame output size (the `DLSSG.OutputInterpolated` subrect or resource), `route` (`ours` / `runtime`), tier, kernel image |
| data row | every real frame | `frame` number, `t_ms` ms since the first timed Evaluate, `dt_ms` CPU interval since the previous frame (the game's real frame time), `sr_ms` super-resolution GPU ms, `fg_ms` GPU ms of all frame-generation Evaluates of the frame summed, `n` generated frames, `fgw_ms` GPU span from the first frame-generation Evaluate's start to the last one's end, `gdt_ms` interval to the previous frame on the GPU timeline, `src_ms` / `fgc_ms` CPU ms inside the two kinds of EvaluateFeature call; `flags`: `x:sr` / `x:fg` that stamp came late or was lost (the value is `-`), `srn=N` N super-resolution Evaluates in the frame, `orphan` a group whose `MultiFrameIndex` 1 was not seen |
| `# second S …` | once per second | `frames`, `fps`; `dt` / `sr` / `fg` / `gdt` four numbers each: avg p50 p95 max in ms; `n`; `gpu` and `fb` utilisation |
| `# end …` | last line of a session that exited cleanly | frame count and counters; a session without it ended in a kill or a crash (every frame completed before a kill is there: the file is written every 0.1 s, and at exit the remaining frames and the open second are written) |

Example (RTX 3070, `sm86_scene` 1080p frame generation 4X, tier 1; rows elided):

```
#
# ==== session  pid 22288  exe sm86_scene.exe  started 2026-10-21T18:58:09.203Z  gpu NVIDIA GeForce RTX 3070
# ini D:\Games\Example\dlssg_sm86.ini
# GPU times: D3D12 timestamps around the NGX core's EvaluateFeature (1000000000 Hz, direct list); CPU times: QueryPerformanceCounter
# t_ms CPU ms since the first timed Evaluate; dt_ms ... (the column legend, one line)
#    frame       t_ms     dt_ms    sr_ms    fg_ms  n   fgw_ms    gdt_ms   src_ms   fgc_ms  flags
# context from frame 0: fg x4 1920x1080 route ours tier 1 image cubin_sm86
         0      0.000         -        -    1.247  3   46.473         -        -    4.811  -
         1    123.156   123.156        -    2.957  3   45.715    97.260        -    2.600  -
         2    218.845    95.689        -    2.823  3   45.722    95.205        -    1.304  -
# second 0  frames 10  fps 9.83  dt 101.713 98.867 123.156 123.156  fg 2.683 2.828 2.957 2.957  n 3  gdt 98.719 97.260 104.561 104.561  gpu 13%  fb 2%
# end  frames 30  late 0  lost 0  capped no  timed_evaluates 90  failed_evaluates 0  vulkan_evaluates 0  untimed 0  fg_launches_in_bracket 4275  fg_launches_other_list 0  fg_launches_outside 0
```

Reading it:

- `dt_ms` is the game's real frame time; `1000 / dt` is the real frame rate (the displayed rate is that times the frame-generation multiplier). `gpu` near 100 % means GPU-bound, and then less `sr` / `fg` time turns directly into frame rate; `gpu` clearly below 100 % with a low frame rate means CPU-bound (or a frame cap), and GPU time saved does not show up as frame rate. `gpu` is the whole GPU, every process, sampled once per second through NVAPI (the counter the usual overlays show).
- `sr_ms` / `fg_ms` is the difference of two timestamps this project records into the game's command list around the NGX core's EvaluateFeature: every GPU command the runtime records for that Evaluate is inside (for frame generation, also every kernel this project substitutes or fuses). Not inside: Streamline's own work outside Evaluate, the gaps between the Evaluates of one group (only in `fgw_ms`), presentation and frame pacing, the game's own rendering. Work running at the same time on other queues makes it longer, which is a real cost.
- Cost (measured on an RTX 3070): about 3-8 us more GPU time per Evaluate (two timestamps and one read-back copy; 13-20 us per real frame at 4X frame generation, +0.3-0.5 %), about 7-13 us more CPU time per super-resolution Evaluate. No pixel changes: super-resolution and frame-generation outputs are byte-identical with and without it.
- Size: about 100 bytes per row and 0.2 KB per summary; about 22 MB per hour at 60 fps. A process's data rows stop at 64 MiB (one `# cap` line says so), then only the summaries go on.

Reading tool: `python scripts\perf\frame_timing_report.py <frame_timing.txt> [<other.txt>]` reads the last session of each file by default (`--all` every session, `--pid N` one process), prints a summary per session (real fps, `dt` / `sr` / `fg` avg / p50 / p95, GPU utilisation, the contexts); with exactly two sessions, one more block compares B with A, every difference in ms and in percent. `--skip-seconds N` leaves out the first N seconds (a loading screen).

### [Debug]

| Key | Values | Default | Description |
|---|---|---|---|
| `MarkGeneratedFrames` | `0/1` | `0` | `1` draws a marker with the actual index, such as `FG 1/3`, on the generated output, to tell whether an artifact is on a generated frame or a real one. Real frames and Reset outputs are skipped. The marker changes generated-frame pixels, so turn it off for whole-image numeric comparisons. |
| `MarkerX` / `MarkerY` | `0`–`65535` | `8` / `8` | Top-left corner of the marker (in output-texture pixels). |
| `MarkerScale` | `1`–`8` | `2` | Marker scale factor; the rectangle is `24×scale` wide and `9×scale` high. A wrong value in any of the `Mark*` keys (such as `MarkerScale=0`) is `configuration_invalid` and the generated-frame marker is off. |
| `Capture` | `0`–`100000` | `0` | How many Evaluates to record — parameters, inputs and outputs — for offline replay (GCR). `0` installs no hooks at all and costs nothing. The count is in Evaluate calls (2 per real frame at 3X, 5 per real frame at 6X). About 48 MB each at 1080p RGBA8. Full mechanism in `docs/CAPTURE.md`. |
| `CaptureDirectory` | path | `dlssg_sm86\capture` | Capture directory, relative to the game EXE's directory. |
| `CaptureSkip` | `0`–`1e8` | `0` | How many Evaluates to skip first (to get past loading screens / cutscenes). |
| `CaptureStartOnReset` | `0/1/2` | `1` | Which frame recording starts on. `2`: wait only for "the first Evaluate right after a feature was created", the only starting point at which a replay can be bit-identical; `1`: a `DLSSG.Reset` frame or the first frame of a newly created feature, whichever comes first; `0`: start immediately. With `1`/`2`, when 3 minutes pass without a hit nothing is recorded. |
| `CaptureStartKey` | key name / VK code | `0` | Key-triggered starting point: `F9` (F1–F24), a single letter or digit, or a virtual-key code (`120` / `0x78`). Once set, after `CaptureSkip` is satisfied it keeps waiting until this key is pressed in the game, then applies the `CaptureStartOnReset` rule from that moment (`0` = start recording on the Evaluate during which it was pressed, `start_reason=key`; `1`/`2` = wait for a Reset / a newly created feature from the key press on, with the 3-minute timeout counted from the key press). This is the way to land a capture on the moment "the character is actually moving"; `capture_waiting_for_key` is logged while waiting, `capture_start_key` on the press. A key name that is not recognised (and an invalid value of any `Capture*` key) is `configuration_invalid`: this capture is off, nothing else is affected. |
| `CaptureNoStall` | `0/1` | `0` | `1`: the game thread never waits. The whole capture is buffered in system memory: the ring is as deep as the requested count, the readback buffers are built ahead of time by the writer thread while it waits for the starting point, and after recording ends the files finish in the background (`capture_finishing` → `capture_done`), so what gets recorded is consecutive frames. A group's memory is estimated as a whole from the sizes learned during prewarm (the first group, or any group with nothing in flight, is always let through; when the budget is smaller than one group it degenerates into recording one group at a time), and when the budget is short the whole group is skipped (a group = all generated frames of one real frame, `MultiFrameIndex` 1..count) — a plane or a frame is never dropped in the middle of a group. `0`: a 4-slot ring, and when the writer cannot keep up the game thread waits at most 2 seconds (at 4K the game drops to a few frames). Events `capture_prewarm` / `capture_group_skipped`, manifest `capture.taken_groups` / `skipped_groups` / `skipped_groups_memory`. On replay the first group after each gap only warms the runtime's previous-frame history, and `capture_compare.py` excludes it from the summary by default. |
| `CaptureMemoryMB` | `0`–`1048576` | `0` | Readback memory budget (pooled + in flight), in MiB; `0` = half the physical memory available at install time. Readback heaps live in system memory and use no VRAM. |
| `CaptureAsyncCopy` | `0/1` | `1` | `1`: moves the PCIe transfer off the rendering timeline. The game's own command list only does a VRAM→VRAM `CopyResource` (into a staging texture), and a separate `COPY` queue then moves the staging texture into the readback buffer in parallel with the game's rendering; the `COPY` queue first `Wait`s on the fence of the game's submission, and `collect()` waits on the `COPY` queue's own fence. `0`: the readback copy stays in the game's command list (the old path, measured at +122% per generated frame at 4K on a 3070). Events `capture_plane_inline` (one plane fell back to the old path) and `capture_async_copy_unavailable` (this machine cannot create the `COPY` queue/list, so everything falls back to the old path). |
| `CaptureLean` | `0/1` | `0` | `1`: read back only what cannot be reconstructed. Inside one group, the input resources handed over by an Evaluate with index>1 are exactly the same as at index 1 (54/54 texture planes byte-for-byte identical across three Wukong captures), so they are recorded only as a reference to the index-1 file; `DLSSG.OutputReal` is the runtime's copy of `DLSSG.Backbuffer` (18/18) and is recorded as a reference too. A referenced plane's json carries `stored=false` and `source=<file relative to the capture root>`, and `sm86_replay` and `capture_compare.py` follow it. At 4X this roughly halves the bytes per group and cuts the readback PCIe time by the same amount. The 16-byte buffer parameters are always recorded in full. |
| `MfgProbe` | `0/1/2` | `0` | `1` logs every set and get of the frame-count parameters, with the calling module name, slot, value and context, to work out who decides "how many frames are generated"; `2` adds a trace of the Streamline interaction (`mfg_state` and others). It reuses the capture's parameter-store hooks but also works on its own with `Capture=0`. Events and how to read them in `docs/MFG_PROBE.md`. |

#### DLSS Super Resolution input capture (`docs/CAPTURE_SR.md`)

**Entirely independent** of the frame-generation capture above: it hooks the NGX core rather than the DLSS-G runtime,
so **it also works in games that never load DLSS-G** (all it needs is DLSS Super Resolution or DLAA on in the game;
frame generation need not be on and the GPU need not be an Ampere), with any DLSS version (2.x / 3.x / 310.x), and
whether the game calls NGX directly or through the UE plugin / Streamline `sl.dlss`. It records the colour / depth /
motion-vector / exposure inputs and every parameter of DLSS Super Resolution (NGX feature 1) per frame, to calibrate and
validate the INT8 Super Resolution work on real content. With `CaptureSR=0` the whole mechanism does not exist (the
backend is not even loaded for it). A game that uses **Ray Reconstruction** (feature 13) does not create feature 1;
then nothing is recorded, and the backend logs one `capture_sr_feature{stage:"other_feature"}` giving the reason.

**`[General] Enabled=0` does not turn it off**: that key turns off the frame-generation part (the kernel replacement),
and this capture is not part of it. `Enabled=0` + `CaptureSR=N` is the "inputs only" deployment: the packaged DLSS-G
runtime file is not even opened, `nvngx_dlssg.dll` goes to the system unchanged (no `runtime_redirect`),
`DlssgMod_Install` is not called (no GPU detection, no kernel replacement, no fusion), the NVAPI architecture report is
left alone and there is no `turing_host`. The loader logs one `capture_sr_only` that lists what it did not do, and
`capture_sr_requested` carries the three fields `mod_enabled` / `runtime_extracted` / `dlssg_untouched`. The CTest
`capture_sr_disabled_mod` pins this path.

| Key | Values | Default | Description |
|---|---|---|---|
| `CaptureSR` | `0`–`4096` | `0` | How many DLSS Super Resolution frames to record **per burst** (Super Resolution calls Evaluate once per real frame). `0` = off. The old alias `CaptureSRFrames` was removed on 2026-10-22 and is not read; write `CaptureSR`. |
| `CaptureSRDirectory` | path | `dlssg_sm86\capture_sr` | An absolute path, or one relative to the game EXE's directory. |
| `CaptureSRStride` | `1`–`100000` | `1` | Within a burst, record 1 frame of every N Evaluates. Calibration needs **variety of content**, not a contiguous slice: at 60 fps a stride of 30 spreads 100 frames over 50 seconds of play instead of 100 nearly identical frames in 1.7 seconds. With `>1` the burst is no longer contiguous, and `source.consecutive` in `pack.json` is `false`. |
| `CaptureSRBursts` | `0`–`4096` | `1` | The most bursts in one session; one burst = one press of the start key. `1` = the traditional "record once and stop"; `0` = unlimited: press as often as you like, until `CaptureSRMaxGB` is reached or the game exits. Each burst gets a `burst_NNNN\` subdirectory, all listed in `bursts[]` of the one `manifest.json`. |
| `CaptureSRMaxGB` | `0`–`4096` | `0` | Stop once the session has written this many GiB of **pixels** (`params.json` and the planes' `.json` files do not count). `0` = no limit. With unlimited bursts this is the **only** bound, so with `CaptureSRBursts=0` and this key at 0 the backend puts in an **8 GiB default limit** for you and logs one `capture_sr_options`. Reaching the limit logs `capture_sr_done{stage:"budget"}`; what was recorded is usable as usual. |
| `CaptureSRDedup` | `0/1` | `1` | `1`: a plane whose content (SHA-256) was already written in this session is written only once, and later jsons carry `stored=false` + a `source` pointing to it (the same convention as the frame-generation capture's lean references). The 1×1 exposure texture, a mask that does not change and static planes are therefore stored once per session. The offline tools follow the references. |
| `CaptureSRStartKey` | key name / VK code | `Insert` | Key-triggered starting point: `Insert` / `Delete` / `Home` / `End` / `PageUp` / `PageDown` / `Pause` / `ScrollLock`, `F1`–`F24`, a single letter or digit, or a virtual-key code (`45` / `0x2D`). `0` = no key. The default is Insert rather than "none": calibration wants the real scenes the player walks into, and the three automatic starting points all land on menus or fixed counts. With several bursts **every burst needs a press of its own** (the key has to be released first, otherwise holding it for half a second would open every burst at once). `capture_sr_waiting_for_key` is logged while waiting. A key name that is not recognised (and an invalid value of any `CaptureSR*` key) is `configuration_invalid`: the super-resolution capture is off. |
| `CaptureSRStartOnCreate` | `0/1` | `0` | `1`: after the key, keep waiting until the first frame of "a newly created Super Resolution feature" — DLSS Super Resolution keeps all of its temporal state inside the feature, only that frame has an empty history, and only a recording that starts there lets an offline replay reproduce the game's own output (in the game: switch DLSS off and on again in the settings). **Meaningful only with "one burst + stride 1"**; under any other setting the backend switches it off and logs one `capture_sr_options` giving the reason (a session with a stride or several bursts is not a contiguous slice of one feature's history anyway). When 3 minutes pass without such a frame nothing is recorded (`capture_sr_done{stage:"start_timeout"}`). |
| `CaptureSRLean` | `0/1` | `1` | `1` (default): the game's own `Output` is not read back. It can be regenerated by replaying the inputs, and at 4K it is the largest plane; `0` records it (to reconcile against the game's own output). |
| `CaptureSRNoStall` | `0/1` | `1` | `1` (default): the game thread never waits for the disk writes; the readback buffers are built ahead of time, and the files finish in the background after recording; when memory or slots run short, a whole frame is skipped (`capture_sr_skipped`). `0`: a 4-slot ring, and the game thread waits at most 2 seconds (what is recorded is always complete, but the game stutters). |

The readback memory budget and the `COPY`-queue switch belong to a pool **shared** by the two captures and set by **whichever is configured first** (the fixed values above on the Super Resolution side, `CaptureMemoryMB` / `CaptureAsyncCopy` on the frame-generation side); usually
that is Super Resolution (it is installed when `nvngx_dlss.dll` loads). Events: `capture_sr_requested` / `_unsupported` /
`capture_sr_only` (loader), `capture_sr_hooks` / `_install` / `_options` / `_feature` / `_prewarm` / `_waiting_for_key` /
`_start` / `_burst` / `_frame` / `_skipped` / `_done` (backend).

Once on disk there are three offline steps (on the developer's side, all with the Python standard library only):
lossless compression with every SHA-256 verified, tabulating a set of sessions (by kernel family, resolution, preset,
DLSS version), and conversion into input packs for the probe replay (one pack per burst if wanted). The capture format
is in `docs/CAPTURE_SR.md`.

### [Compatibility]

| Key | Values | Default | Description |
|---|---|---|---|
| `OptimizedKernels` | — | — | The oldest **old name** of `[DLSSG] Optimized`, no longer read since 2026-10-22: written beside `[DLSSG] Optimized`, or holding `0`, it logs one `configuration_warning` (`(no effect)`); alone with any other value (such as an old INI's `OptimizedKernels=1`) it is `configuration_invalid` and frame generation is not enabled - write `[DLSSG] Optimized=1` instead. |
| `KernelImage` | `Auto/PTX/Cubin/Original` | `Auto` | Kernel image format, see the table below. **The driver rejecting the chosen image is an error, there is no PTX retry** (since 2026-10-26; see "Driver version requirement: R580"). An explicit `Cubin` where this install has no cubin at all — RTX 40 / 50 (this project ships no sm_89 / sm_120 cubin) or a kernel pack without cubins — logs one error-level `kernel_image_unavailable{reason}` and frame generation is refused (since 2026-10-20; before, the PTX ran silently); `Auto` is "the cubin where there is one, else the PTX", decided once at install. `Original` installs the hooks but replaces no kernel at all (a local stock-numerics reference; **this cannot be used on Ampere** — 310.9.1's stock image kernels are sm_89 and fail to create on Ampere, so use `Optimized=0` + `KernelImage=PTX` for a local reference). On RTX 40 / 50 it is the only way to run the runtime's own kernels; on an RTX 40 those kernels do 2X only, so with `MaxGeneratedFrames` above 1 frame generation is refused (`max_generated_frames_unsupported{reason:"runtime_kernels"}`). |
| `Router` | `Auto/SM86/SM75` | `Auto` | Kernel family. `Auto` picks it from the physical GPU (SM86 and above go SM86, Turing goes SM75). **SM75 is experimental**. At `Optimized=1` it creates all 37 variant rows (34 network variants + 3 image patches re-derived against 310.9.1 RVAs) with cross-kernel fusions and image patches fully on, and logs one `sm75_route_limits` (`variant_rows=37`, `image_patches_partial` empty). On 3070 forward JIT it is bit-identical to the SM86 stock route (including 6X); **on a real Turing (RTX 2080 Ti)** the sm_75 cubins match their PTX and tier 0 matches tier 1 bit for bit, and for the same inputs the output is bit-identical to an RTX 3080 Ti's (synthetic rotate scene, 32/32; both differ from the official output on an RTX 5070 by the same bytes — the hardware accumulation difference between Blackwell and Turing/Ampere). Performance on Turing is not measured yet. The SM75 stock kernel family comes from Coldwood1026 (see `THIRD_PARTY_NOTICES.txt`). |
| `Preset` | — | — | The **old place** of `[DLSSG] Preset`, no longer read since 2026-10-22: written beside `[DLSSG] Preset`, or holding `A` / `Auto`, it logs one `configuration_warning` (`(no effect)`); alone with `B` / `C` it is `configuration_invalid` and frame generation is not enabled - write `[DLSSG] Preset=B` / `C` instead. |
| `SpoofArchToGame` | absent / `0/1` | **absent = automatic** | Governs **only the gate the game — or Streamline on its behalf — puts on the architecture**. **This is a tri-state key**: **not written** (the factory INI has no such line) = automatic and **`1`** = explicit install the redirect at exactly the same moment, the earliest of the five triggers below, on every GPU; **`0`** = never installed (that answer has to be known before the full INI parse, so at startup the loader makes one separate kernel32-only `GetPrivateProfileStringW` read of `[General] Enabled` and this key; a malformed or empty value counts as automatic), logging one `arch_spoof_disabled{redirect_installed}` on Turing. **That early read decides for the whole process** (2026-10-19): when the full parse reads a different `Enabled` / `SpoofArchToGame` / `SpoofArchValue`, it only logs one Level 1 `spoof_policy_mismatch{peek,parsed}` and changes nothing; when another key's bad value switches frame generation off (`configuration_invalid`), the rewrite stays as the early read decided. The only difference between automatic and an explicit `1` is the **log**: automatic releases its `arch_spoof_*` records only once the host has proved to be a Turing (the first real rewrite, or the backend's Turing verdict plus `turing_host_defaults`), so **a non-Turing card running the factory INI still produces not a single `arch_spoof_*` record**. Why it has to be this early: **Streamline 2.x decides the frame-generation plugin is "not supported on this platform" inside `slInit`** — `sl.dlss_g` defaults its minimum architecture to AD100, `sl.common` compares every adapter's `NvAPI_GPU_GetArchInfo` architecture against it and evicts the plugin (`Ignoring plugin 'sl.dlss_g' since it is not supported on this platform`), which happens before a D3D device exists and long before the NGX core asks us for the DLSS-G snippet, so anything armed "once the runtime is loaded" is too late (measured on an RTX 2080 Ti with FF7 Rebirth / Streamline 2.8.0.0: writing `1` and omitting the line gave the identical outcome, the plugin absent from the process either way). Mechanism: `nvapi64.dll`'s export-table entry for `nvapi_QueryInterface` is redirected to a 14-byte trampoline in that module's own section padding, so `NvAPI_GPU_GetArchInfo` reports a newer architecture to the outside (which one is the next row, `SpoofArchValue`: by default Turing is told Blackwell `0x1b0` and Ampere Ada `0x190`; no byte of NVIDIA code is changed; **NVIDIA's own components always see the real architecture**: `_nvngx.dll`, `nvngx_*`, `nvapi*`, our own `sm86_backend.dll`, the driver components under `\DriverStore\FileRepository\`, and **the snippets in the NGX model store** - with the NVIDIA App's "DLSS override" active, or after an NGX OTA update, the core loads the super-resolution / ray-reconstruction / frame-generation snippets from paths like `C:\ProgramData\NVIDIA\NGX\models\dlss\versions\<n>\files\160_E658700.bin`, which carry no `nvngx_` in the name; 0.3.3 rewrote the answer for those as well, the SR snippet took its `arch >= 0x180` kernel paths and hung the GPU (`DXGI_ERROR_DEVICE_HUNG` right after the SuperSampling feature is created, before frame generation is ever asked for; issue #535, and the startup crashes #538 / #540 / #542), so since 0.3.4 every component under `\NVIDIA\NGX\models\` other than `sl_*`, and any `.bin` module, sees the truth, while `models\sl_*` - Streamline's own OTA plugins - is exactly what the rewrite is for and is still rewritten). The log carries one `arch_spoof_applied` per **distinct** calling module (with a new `path` field giving the full path) and one `arch_spoof_excluded` per distinct excluded module, so it shows who asked and which answer it got. **It only rewrites when the real answer is Turing (`0x160`) or Ampere (`0x170`)** (Ampere was added in 0.3.3: an RTX 3060 Laptop showed the same symptom as the 2080 Ti in FF7 Rebirth, issue #509): on other architectures the redirection is installed but does nothing, logging one `arch_spoof_inert{real_arch,rewritten_archs}`; a `0` found by the full parse (i.e. the INI was unreadable at startup) no longer stops it rewriting (until 2026-10-19 it did, which changed what the game was told mid-session); it is only `spoof_policy_mismatch`. `arch_spoof_installed` carries `trigger` and `default`: the **three early triggers** `dllmain` (`nvapi64.dll` was already mapped when we were) / `nvapi_load` (our LoadLibrary hook caught `nvapi64.dll`'s own load) / `dependency_load` (`nvapi64.dll` arrived as another module's static import and was found mapped after some load) — only these three are ahead of `slInit`; the **two fallbacks** `settings` (the INI read, i.e. the game's first `nvngx_dlssg.dll` request) / `backend` (after the backend install). `default=true` means the automatic setting made the decision and says nothing about how early it was armed. **It has nothing to do with the NGX core's `0xbad0000b`**: that one is the core comparing the GPU architecture against the minimum architecture the runtime exports (see the `fg_gate_create_feature` row below), which no NVAPI spoof can change. |
| `SpoofArchValue` | `Auto/Ada/Blackwell` | `Auto` | Which architecture the rewrite reports. `Auto` (the default) tells **RTX 30 (Ampere) Ada (`0x190`) and RTX 20 (Turing) Blackwell (`0x1b0`)**. With the Ada answer on an RTX 3070, Cyberpunk 2077, The Witcher 3 and Forza Horizon 6 still offer 3X / 4X and run 4X when chosen (the multiplier comes from the backend, not from this answer), and Streamline's online-update components load their Ada builds, which clearly reduces the on-and-off stutter Cyberpunk shows at 2X. On RTX 20 an Ada answer passes the frame-generation gate but leaves only 2X on screen (measured on real RTX 20 hardware, issues #527 / #528), so Turing keeps Blackwell. `Ada` / `Blackwell` pin that value for both generations: `Blackwell` is the 0.3.3–0.4.0 answer on RTX 30, so it can be written back for comparison (both also accept `0x190` / `0x1b0`). **This value was never the cause of the 0.3.3 crash**: what crashed was NVIDIA's own DLSS Super Resolution model receiving the rewritten architecture — with the NVIDIA App's "DLSS override" or an NGX OTA update in place the core loads it out of the NGX model store under a name like `160_E658700.bin`, which a leaf-name exclusion rule cannot catch. On a 3070 both the Ada and the Blackwell answer hang the GPU, and neither does once that snippet is excluded, so the fix is excluding NVIDIA's own components by location (`src/common/spoof_policy.h`, see the row above), not a smaller architecture. Read at startup with kernel32 alone like `SpoofArchToGame`, and that read decides for the whole process; a malformed value in the full parse is `configuration_invalid` (frame generation not enabled, the rewrite still as the early read decided). `configuration.spoof_arch_value`, `arch_spoof_installed{spoof_arch_value,reported_arch_turing,reported_arch_ampere}` and `arch_spoof_applied.reported_arch` (432 / 400) say what was actually used. |
| `ForceSM86Route` | `0/1` | `0` | `1` allows forcing the SM86 backend on a non-SM86 GPU for validation (anything below SM86 is still refused). This is the validation configuration: it installs the RTX 30 architecture bridges as well, and is not the same thing as the default RTX 40 / 50 route. |
| `SimulateAmpere` | `0/1` | `0` | `1` adjusts the reported architecture for validation (the runtime itself is told Ampere); requires `ForceSM86Route=1` as well. It does not change the physical GPU. The default RTX 40 / 50 route neither needs nor installs it. Written without `ForceSM86Route=1` it is `configuration_invalid` and frame generation is not enabled (C114). |
| `DisableFusions` | `0`–`63` | `0` | **Diagnostic bitmask (turning bits on makes it slower)** that disables one class of the optimized set's cross-kernel fusion each: 1 decoder upscale+add+1×1, 2 decoder 1×1 pair, 4 `k_initial_merge`+convPre, 8 conv0+pooling, 16 residual chain, 32 `k_central_block`+block1 conv0. The default 0 = every fusion on. |
| `ImagePatches` | `0/1` | **set by the tier** (`1` from tier ≥1) | `0` disables the **exact** PTX-level patches on the image-processing kernels (needs tier ≥1). The patches do not change semantics and are bit-identical. The lossy rows are governed by `ImageApprox` and are not affected; **the `CatmullRomWarp` rows are image-patch rows too, so `ImagePatches=0` switches them off as well** (that key then logs a `configuration_warning`). Tier 0 builds no row, so `1` there does nothing (and logs one too). |

### [SuperResolution]

**The DLSS Super Resolution route (NGX feature 1)**, independent of frame generation. **Since 2026-10-19 it does one thing: a preset override** (owner decision, `docs/COMPAT.md` C96; mechanism and validation in `docs/SR.md`). The player surface is one key, `Mode`, with five values. **The factory INI says `Mode=Off`, which is exactly the section left out: the process is byte-for-byte the same as a build without this code** (`nvngx_dlss.dll` is not intercepted, the 74 MB snippet is not hashed, no hook is installed, there is not a single `sr_*` record, and `configuration` does not gain even one field).

**While Ray Reconstruction is on in the game, this section does nothing.** Ray Reconstruction replaces Super Resolution: the game creates NGX feature 13 (DLSS-D, one network does both the denoising and the upscaling) and **never creates feature 1**, so `Mode` has nothing to act on; the quality mode still sets the render resolution, and what runs is the game's own Ray Reconstruction (this project does not touch it; next section). Turn Ray Reconstruction off in the game's settings and the Super Resolution route is back. Measured in Cyberpunk 2077: a whole path-tracing + Ray Reconstruction session has not one `sr_*` record and no feature 1.

| Key | Values | Default | Description |
|---|---|---|---|
| `Mode` | `Off` / `M-INT8` / `L-INT8` / `M-FP16` / `L-FP16` | **`Off`** | `Off` = not touched. The other four replace the Super Resolution snippet the game asks for with the packaged **310.9.16** (`dlssg_sm86\sr\nvngx_dlss.dll`) and **pin** M or L on **all six quality modes** (DLAA, Ultra Quality, Quality, Balanced, Performance, Ultra Performance); which mode is in use is the player's choice in the game's menu. `X-INT8` runs this project's static INT8 kernels (M's 12 network kernels; for L it is 10 on RTX 30 / RTX 20 — the first and the last layer stay on the runtime's own FP16 kernels, because INT8 is slower for those two layers on these two generations, `docs/SR.md` section 3 — plus, on D3D12, the bit-exact post / downsample rewrites); `X-FP16` is identical except that the kernels are NVIDIA's own — **the A/B control for the INT8 value of the same letter** (same runtime, same preset, only the kernels differ). **These five values only**: anything else (`On` of 2026-10-10 to 10-19 included) is an invalid value, logs one level-1 `configuration_invalid{section,key,value,allowed,effect:"super_resolution_not_enabled"}`, and super resolution is not enabled (the same as `Off`). |

**INT8 pays off in Performance and Ultra Performance**: measured on a 3070, GPU time of one Evaluate (INT8 against FP16 of the same runtime and preset, 1080p–4K × four modes × both families): M −9.4…−23.2 % (−0.10…−2.12 ms), L −6.5…−16.3 % (−0.08…−1.89 ms). **Quality / Balanced / DLAA are pinned to M / L as well**, where games mostly use K: on a 3070 K is 1.24–3.66× faster than M and 1.48–4.77× faster than L (Resonance in Quality mode 47.5 → 38.1 fps), so `X-INT8` there is slower than the game's own K — accepted by the owner (2026-10-19); for frame rate pick Performance / Ultra Performance in the game.

**The pinned preset wins over every other source**: the game's preset hints (Streamline's `0x8000000B/C/D`, which 310.9.16 drops anyway) are not read; an active NVIDIA App preset override is overridden too, logging one `sr_preset_drs_conflict` per process (C69). What existed until 2026-10-19 — `Mode=On` (no pin, INT8 only where the game already ran L / M), `Tier` (pin one mode only), hint restore, following the NVIDIA App, and refusing a second preset in one session (`sr_route_preset_refused`) — is removed: a process only ever has one preset.

**In-game A/B**: same scene, same mode in the game, `Mode=M-INT8` against `Mode=M-FP16` (or `L-INT8` against `L-FP16`); the two runs differ only in the kernels. Take the numbers from `[Logging] FrameTiming=1` (`scripts/perf/frame_timing_report.py`). Do not use `Mode=Off` as the baseline: that is the game's own runtime with its own preset, which differs in more than the kernels.

**Preset and quality mode are two separate dimensions**: the quality mode sets the render/output resolution ratio, the preset then picks the weights and the network. **Pinning a preset does not change the render resolution.**

**It takes effect only on RTX 30 and RTX 20.** The first time `nvngx_dlss.dll` is loaded, before anything is replaced, the proxy asks the GPU once (CUDA driver API: compute capability and name) and reads the index of the packaged kernel pack `sr_kernels.dll`: it steps in only when an NVIDIA GPU is sm_86 or sm_75, **has `RTX` in its name**, and the package has a set for it. **In every other case — RTX 50, RTX 40, GTX 16 and the Turing T series (sm_75 as well, but without tensor cores), MX570, no CUDA, a pack that cannot be read — it behaves exactly like `Mode=Off`**: the snippet is not replaced, no hook is installed, and the log gets one warning-level `sr_gate` giving the reason (`reason:"device_not_rtx"` for a GTX 16 class card; the `super_resolution` field that `configuration` copies from the INI is still there, written when the INI was read). That gate only asks whether the machine has a GPU on the list. The GPU the game actually renders on is judged again, by LUID, at the first CreateFeature (2026-10-19): when it is not on the list (for example a machine with an RTX 30 and also an RTX 40 / GTX 16, the game running on the latter), under an INT8 Mode (`M-INT8`, `L-INT8`) every feature given INT8 is refused the same way as with a missing kernel pack: one Level 1 `sr_kernels_unavailable{reason:"device_not_on_list"}`, and those CreateFeature calls return `0xbad00001`; FP16 Modes are unaffected. The other exception (C93): under an INT8 Mode on an RTX 30 / 20 this project serves, with no usable set for it in the pack (missing / damaged), the snippet is replaced and the route installed all the same, and a feature that would run INT8 is refused at its creation (`sr_kernels_unavailable` / `sr_feature_refused`) rather than quietly run on FP16. RTX 40 / 50 are excluded because their own FP8 path is faster than our INT8, so stepping in gains nothing (no configuration of the product runs the RTX 50 sm_120a verification sets, and since 2026-10-11 they are not in the released pack, see the `KernelSetSM` row below). RTX 20 is on the list, but its host weight widths have not been measured on a real card yet.

**Removed keys**: `Int8`, `Int8L`, `Int8M`, `ForcePreset`, `ForcePresetDLAA` / `ForcePresetQuality` / `ForcePresetBalanced` / `ForcePresetPerformance` / `ForcePresetUltraPerformance` / `ForcePresetUltraQuality` (2026-10-03), and `Tier` (2026-10-19). They are not read (since 2026-10-26; before, each logged one `configuration_warning`) — **so an old INI's `Int8=1` no longer turns anything on, without a `Mode` line it is Off, and `Tier=Performance` no longer limits a pin to Performance**.

**Diagnostic keys** (not in the factory INI and not needed by players; none of them can make the route step in on a GPU the check above turns away):

| Key | Values | Default | Description |
|---|---|---|---|
| `Auxiliary` | 0/1 | 1 | On a feature that runs INT8 (`X-INT8`), also replace the bit-exact rewrites of post / downsample. **D3D12 only**: the Vulkan route refuses them unconditionally (COMPAT C57 — on that API what goes in is no longer the runtime's own bits, and the post kernel writes the temporal history, so the difference feeds back every frame until pixels saturate); the log says `auxiliary_allowed:false` and `auxiliary_refused`. |
| `KernelImage` | `Auto`/`Ptx`/`Cubin` | `Auto` | Which image family to try first. Every kernel in a set has two: `.nvapi.fatbin` (PTX, JIT-compiled by the driver) and `.cubin.nvapi.fatbin` (a native cubin for this architecture, no JIT). **`Auto` and `Cubin` both prefer the cubin**; `Ptx` never looks at it. Under `Auto` a set without a cubin image uses PTX (no record); an explicit `Cubin` where the set chosen for this device lacks the cubin of a kernel the preset replaces (or `KernelSetSM` chose another architecture's set) is one error-level `sr_kernel_image_unavailable`, nothing is built and every feature given INT8 fails its CreateFeature (`sr_feature_refused`, `0xbad00001`), never PTX in its place (2026-10-22); the driver rejecting the chosen image (cubin or PTX) is `sr_kernel_fatal{kind:"kernel"}` and the process ends (C93; since 2026-10-26 a rejected cubin too, with no PTX retry). **That is why the cubin image exists**: with PTX only, the game's first feature creation (and every rebuild after a quality-mode change) makes the driver compile 12 kernels of about 600 KB each. The diagnostic switch `DLSSG_FORCE_CUBIN_FAIL=1` breaks only the cubin images, `DLSS_SM86_FORCE_IMAGE_FAIL=1` breaks **every** image; both reach this fatal path. |
| `ObserveFrames` | 0–64 | 1 | How many frames to observe, checking every launch against its contract, before replacing starts. `1` (the default) observes frame 1 and replaces from frame 2. `0` is for measurement: replacing starts on the first frame and each launch still passes its own contract first, but that frame's all-or-nothing is given up (measured on a 3070: 8 of 8 frames replaced, against the last 7 with `1`). |
| `KernelSet` | path | `dlssg_sm86\sr` | Where the kernel-set **pack** is. The default directory holds the released kernel pack `sr_kernels.dll` (one resource-only DLL with the four sets of RTX 30 / 20, one per (architecture, input family); the backend reads them all and picks by the GPU's compute capability); it ships as a file rather than embedded in the proxies (about 106 MB, and a release has six proxy names). A directory holding `sr_kernels.dll` means that pack, even when an older release's `index.json` is still beside it; any other directory is read as a directory pack (one `index.json` + set folders, how research sets are pointed at, by the same rules as before); the value may also be a pack DLL itself. A relative path is relative to the proxy's own directory. **The device gate reads this too**: no set for this GPU = Off. The device gate reads only the pack's gate summary (the INDEX resource of `sr_kernels.dll`, or `gate_summary` in a directory pack's `index.json`), so a directory holding a single set, a directory pack generated before 2026-10-04 (no summary), or a release missing `sr_kernels.dll` is also Off, with the reason in `sr_gate`'s `sets` (`kernel_set_source` says whether a pack or a directory was read). A damaged pack is refused whole by the backend (`sr_kernel_set{loaded:false}`, `source.reason` says which check), and under an INT8 Mode a feature given INT8 is refused at creation (`sr_feature_refused`, C93). |
| `KernelSetSM` | `Auto`/`75`/`86`/`120` | `Auto` | **For diagnostics / measurement; keep `Auto` for playing.** Runs another architecture's kernel set on this machine (forward JIT). `Auto` uses the GPU's own compute capability, the only normal value. **While an override is in effect the image family is forced to PTX** (a cubin is one architecture's machine code and the driver will not take it), logging one `sr_kernel_image{forced_ptx:true}`; **when the pack has no set for that architecture the whole route is refused** rather than quietly switched back to this machine's set (`sr_kernel_set_override{applied:false}`, `route.disabled_reason="kernel_set_override_missing"`, `replaced` always 0); on a foreign architecture the host weight-buffer width gate is waived and counted per launch (`route.binding_waivers`), while the rest of the contract (entry names, dedicated kernel digests, block, geometry) holds as usual. The device gate judges **the GPU itself**, so `KernelSetSM=120` on a 3070 passes the gate; the released kernel pack has no sm_120a set (since 2026-10-11), so the route is refused with `kernel_set_override_missing` before any kernel is built and an INT8 feature is refused at creation (C93). With `KernelSet` pointing at a directory pack that has one, the sm_120a set is selected - **and then cannot run**: PTX JIT-compiles forward only, the driver refuses the sm_120a PTX and the first creation is `sr_kernel_fatal` (C93 / C103; measured on a 3070, 2026-10-10). A card can run only its own or an **older** architecture's set this way: a 3070 runs `75`, a 2080 Ti nothing but its own `75` (`86` there fails at kernel creation the same way), and no RTX 30 / 20 runs `120`. **Why it exists**: the weight images of `set_sm_75` and `set_sm_86` are byte-for-byte identical and only the machine code differs, so one RTX 30 running once with `Auto` and once with `75` proves the 12 sm_75 kernels compute the same thing — while doing the same on an RTX 50 proves nothing (Blackwell accumulates differently from Ampere/Turing). Details in `docs/SR.md` section 8. |

Before install: a GPU not on the list = forwarded unchanged with one `sr_gate`; a missing bundled snippet or a hash mismatch = this load is refused (`sr_runtime_unavailable`, C99). Once an INT8 Mode is selected, a set that cannot be used (pack missing / damaged, no set for the GPU or the input family) or a render GPU not on the list (above) refuses that feature per C93 (its CreateFeature returns `0xbad00001`), and any later failure is `sr_kernel_fatal`; nothing quietly falls back to FP16.

**Games with LDR (tone-mapped colour) input are not accelerated, by design**: there is no calibration content for this family, so the route declines explicitly before building any kernel and logs one normal-level `sr_family{chosen:false, by_design:true, family:"mvlo_ldr"}`; every mode still has the Mode's preset pinned, on NVIDIA's own kernels (the same as X-FP16) — that is NVIDIA's own code, not a fault, and the log has no error-level record either (see the input-family section of `docs/SR.md`).

**On Vulkan games** (Red Dead Redemption 2 and the like) this Super Resolution route **holds as well**: the same runtime, the same kernel set, the same `Mode`. Kernels are replaced by a different mechanism (five function pointers in the snippet's own Vulkan entry-point table are changed instead of patching code), but for the user there are only three differences: ① **a Vulkan game can only use the four utility proxy names** (`version` at the root, or `winmm` / `dbghelp` / `dinput8` from `alternatives\`) — `dxgi.dll` / `d3d12.dll` are never loaded under a Vulkan back end; ② if the game creates its `VkDevice` without the `VK_NVX_binary_import` extension, one error-level `sr_vk_entrypoints{installed:false}` is logged and under an INT8 Mode a Vulkan feature given INT8 is refused at creation (`sr_feature_refused`; that decision happens before anything we can step into, and no knob changes it); when hooking that entry-point table itself fails (`sr_vk_install{installed:false}`), under an INT8 Mode every Vulkan feature given INT8 is refused (`sr_feature_refused`, `route_hooks_unavailable`, `0xbad00001`), and D3D12 games are unaffected; ③ **`[Debug] Capture` and `[Debug] CaptureSR` work on D3D12 only**, and the whole `[FrameGeneration]` section is not supported on Vulkan in this round either. The log fields that identify this API are `api:"vulkan"` on `sr_install` / `sr_exit`, `sr_vk_install`, `sr_vk_entrypoints`, `sr_device{api:"vulkan"}`, `sr_resource_hook{api:"vulkan"}`, and `fg_gate_hooks_vk`, which appears only the first time Vulkan NGX is used. **So far this route has only static and GPU-free evidence**; see gap G13 in `docs/COMPAT.md`.

### [RayReconstruction] (removed)

**Since 2026-10-09 this project does not touch Ray Reconstruction** (NGX feature 13; owner decision, the reason is in
`docs/RR.md`). The former `Mode=Mixed` replaced the Ray Reconstruction snippet the game asked for with a packaged 310.9.0.0,
pinned preset F and ran a mixed INT8 kernel set; measured in Cyberpunk 2077 it ran level with the game's own Ray
Reconstruction (4K Ultra Performance, frame generation 4X: 26.86 against 26.89 fps), so it did nothing for players and the
whole route was removed. `nvngx_dlssd.dll` - next to the game, or in the model store under an NVIDIA App override - is now
**loaded unchanged**: not intercepted, not hashed, not hooked, not one `rr_*` record - exactly what the factory `Mode=Off`
did (with `[Debug] CaptureSR` written, its load still arms the Super Resolution capture, as `nvngx_dlss.dll`'s does, so the
log can say that the game uses feature 13). The package no longer has `dlssg_sm86\rr\`; one left by an older install is never
read and may be deleted.

**Turning Ray Reconstruction on replaces Super Resolution**: the game creates only feature 13, and none of the
`[SuperResolution]` settings apply meanwhile (start of the previous section); to test Super Resolution INT8, turn Ray
Reconstruction off in the game.

**Old keys**: every key of this section and `[Runtime] RRMode` / `RRPath`, `[Debug] CaptureRR` / `CaptureFeature` are not read (since 2026-10-26, C113; before, read only to warn): `Mode=Mixed` in an old INI does nothing.

### [Runtime]

| Key | Values | Default | Description |
|---|---|---|---|
| `Path` | path | `dlssg_sm86\runtime\nvngx_dlssg.dll` | Used by `Mode=External` and `Mode=Pinned`; points at an `nvngx_dlssg.dll`. **Its default is the very file `Bundled` uses**, so `Mode=External` without `Path` is the same as `Bundled` (as with `SRPath`). |
| `SRMode` | `Bundled`/`External`/`Pinned` | follows `Mode` | The same choice for the DLSS *Super Resolution* snippet (`Auto` was removed with `Mode=Auto` in 0.4.0: it runs as `Bundled` with a warning). **It follows `Mode` by default**: whoever moves an NVIDIA snippet elsewhere, pins it or turns it off almost always wants both. `Bundled` means the file the package puts beside the proxy, `dlssg_sm86\sr\nvngx_dlss.dll` (74 MB, deliberately not embedded — a release has six proxy names), SHA-256 verified before anything is patched; when it is missing or wrong, `sr_runtime_unavailable` is logged and **the `nvngx_dlss.dll` load is refused**: super resolution is not enabled (since 2026-10-20; before, the game got its own DLSS back — under an INT8 Mode that is the stock FP16). |
| `SRPath` | path | `dlssg_sm86\sr\nvngx_dlss.dll` | Used by `SRMode=External` and `SRMode=Pinned`. A hash mismatch logs `external_runtime_rejected{feature:"super_resolution"}` (`sr_pinned_runtime_rejected` for `Pinned`) and **the load is refused**: super resolution is not enabled (since 2026-10-20; before, it fell back to `Bundled` with a `configuration_warning`). |

`Mode` values: `Bundled` (a normal install: the packaged `dlssg_sm86\runtime\nvngx_dlssg.dll` and the packaged backend); `External` (the runtime lives elsewhere, named by `Path` — see below); (`Auto` - load the stock runtime the game asked for - was removed in 0.4.0; an old value runs as `Bundled` with one `configuration_warning`, COMPAT C86); `Pinned` (use the runtime named by `Path`, which needs a matching backend). A failure to load or to install (`backend_unusable`, `install_failed`, a backend ceiling below `MaxGeneratedFrames`, ...) **no longer hands the game its own `nvngx_dlssg.dll`** (2026-10-19; `External` / `Pinned` too): the loader logs `runtime_selection_failed{action:"frame_generation_not_enabled: the nvngx_dlssg.dll load is refused"}`, the load fails and frame generation is not enabled - on RTX 40 / 50 that means no frame generation rather than the runtime's own 2X.
On the super-resolution side, when the game - or an NVIDIA App override - already maps 310.9.16, nothing needs replacing under any mode: only the hooks go in, and `sr_runtime_adopted` is logged.

**`Mode=External`** names where the runtime is:

```ini
[Runtime]
Mode=External
Path=dlssg_sm86\runtime\nvngx_dlssg.dll
SRMode=External
SRPath=dlssg_sm86\sr\nvngx_dlss.dll
```

A path is absolute, or **relative to the proxy DLL's own directory** — not to the working directory, which Steam, a
shortcut's "Start in" and engines that `chdir()` into their data folder all change, while the proxy sits where the user
put it, so only it makes one install resolve to the same file every time (every relative path, `[Logging] Directory`
included, already follows this rule). `%VARIABLES%` are expanded (for `External` only; `Pinned` takes its path literally). The file must hash to the
runtime this build's backend serves; otherwise the loader records
`external_runtime_rejected{feature,path,sha256,reason,action:"frame_generation_not_enabled: …"}` and **the load is
refused** — neither the bundled nor the game's own runtime answers in its place (since 2026-10-20; before, it fell back
to `Bundled` with a `configuration_warning`) — and when the file is the runtime of another release of this
project it is named rather than only hashed (`runtime_version` / `expected_version`, e.g. "it is 310.1.0.0, and this
build's backend serves 310.9.1.0"). `Mode=External` without `Path` is no longer a
configuration to correct: `Path` defaults to the packaged file, so it resolves exactly as `Bundled` does (as
`SRPath` always has), and the `configuration_warning` it used to cause is gone. Which of the four
answered a request is `runtime_redirect.selected_source` / `sr_runtime_redirect.selected_source`
(`bundled`/`external`/`pinned`/`game_own`), plus `sha256` of the file that answered, in all four modes; `source` still
says only where the request came from: the game directory / the NGX model store / the driver store. When the
packaged runtime is missing or is not this build's, `Bundled` has nothing to fall back to: one error-level
`bundled_runtime_unavailable{path,sha256,runtime_sha256,found_version,reason,remedy}` per process, and **the load is
refused**: frame generation is not enabled — never a silent slide into `Auto`, and the game's own `nvngx_dlssg.dll` is
not handed back either (since 2026-10-20; before, it was, which on an RTX 40 / 50 is the runtime's own kernels at 2X). The usual cause is
copying the proxy DLLs without the `dlssg_sm86\` folder. The check runs **before** the load: a module whose hash
does not match is never mapped.

### [Backends] (removed)

Removed on 2026-10-22 (C102): the backend is the one beside the proxy, and `Pinned` / `External` can only name this build's runtime. An INI that still has the section: not read (since 2026-10-26, C113). The optional export `DlssgMod_SupportedRouteFlags` and its flag negotiation (`kernel_selection_unsupported`) went with it.

### KernelImage behavior

| KernelImage value | Behavior once the route is active |
|---|---|
| `Auto` | Uses the precompiled cubin when the physical architecture matches the selected kernel family, otherwise PTX. Falls back to PTX automatically when the driver rejects a cubin. |
| `PTX` | Always uses the selected kernel family's PTX, JIT-compiled by the driver; works on a 3080 Ti too. |
| `Cubin` | **Prefers** the precompiled cubin and requires the physical architecture to match the kernel family **exactly**, otherwise the install fails (`install_failed`), the `nvngx_dlssg.dll` load is refused and frame generation is not enabled (before 2026-10-19 it fell back to the original loading). When the architecture matches but the driver rejects the cubin it still falls back to PTX (better a JIT than no frame generation). **The cubins are in the packaged kernel pack `dlssg_sm86\kernels\fg_kernels.dll`** (beside the PTX; the backend DLL carries no kernel at all): when that file is not there, frame generation is refused (the backend logs `kernel_pack_missing` + `fg_kernels_unavailable`; since 2026-10-16, C93, whatever `KernelImage` says); when the pack is there but has no cubins, `Cubin` logs one error-level `kernel_image_unavailable` and frame generation is refused as well (since 2026-10-20, C99). Before, both cases were treated as `Auto`. |
| `Original` | **Diagnostic / reference**: installs the hooks but replaces **not a single kernel**, and enables no optimization / fusion / image patches. Mutually exclusive with `PTX`/`Cubin` and independent of `Router`. **Not usable on Ampere.** |

What `KernelImage` × `Router` actually selects:

| Router / physical GPU | `Auto` | `PTX` | `Cubin` |
|---|---|---|---|
| SM86 family, physical SM86 (3070/3080 Ti) | `cubin_sm86` | `ptx_sm86` | `cubin_sm86` |
| SM86 family, physical SM89/SM120 (the default since 0.4.0, not switchable since 2026-10-19) | `ptx_sm86` | `ptx_sm86` | refused (`kernel_image_unavailable`) |
| SM86 family, physical SM89/SM120 (`ForceSM86Route` forced-route validation) | `ptx_sm86` | `ptx_sm86` | refused |
| SM75 family, physical SM75 (RTX 20-series) | `cubin_sm75` | `ptx_sm75` | `cubin_sm75` |
| SM75 family, physical SM86 and above (forward-JIT validation) | `ptx_sm75` | `ptx_sm75` | refused |

The SM75 family and the SM86 family are **bit-identical** under forward JIT on an RTX 3070 (both the stock family and the optimized set: 240/240 images across the box and rotate scenes at 7 resolutions, plus **another 80/80 with `--multi 5` (6X)**; the MFG phase (the `i/(N+1)` centroid) is exactly the same on both routes; splitting `m16n8k16` into two `m16n8k8` adds no rounding step that differs from stock). Of 310.9.1's 70 kernels, 44 use the sm_75 stock family shared with 310.1, and the sm_75 images of the 26 new kernels are rewritten by this project from its own sm_86 PTX. The optimized set on SM75 is **not one row short**: all 37 variant rows (34 network variants + 3 image-patch rows) have an sm_75 image. The retired 310.1 build was bit-identical in the same way (240/240 images).

The numerics were verified bit-identical to the SM86 route by `.target sm_75` forward JIT on an RTX 3070; on **real Turing (a physical RTX 20, 2026-09-15/17)** the whole `sm_75` cubin family creates and runs with 0 fallbacks, cubin and PTX are bit-identical, tier 0 and tier 1 are bit-identical, and it works in game. Timings on Turing are not yet quantified. Details in `docs/VALIDATION.md`.

#### Driver version requirement: R580

| Path | Minimum driver | Reason |
|---|---|---|
| everything: frame generation, XeSS-FG, super-resolution INT8, cubin and PTX | **R580** (CUDA 13.0; this round measured on 617.14) | every cubin is produced by CUDA 13.0.2's `ptxas` (ELF ABI 8; since 2026-10-26 there is no 12.x cubin), the PTX is ISA 8.5 / 8.7 |

The backend asks the CUDA driver once at install which CUDA version it implements (`cuDriverGetVersion`; R580 = 13.0).
**Below R580 nothing is installed**: frame generation and XeSS-FG log one error-level
`driver_below_floor{cuda_driver,floor:"R580 (CUDA 13.0)",driver}` and the loader then refuses the DLSS-G runtime load (no frame
generation in the game); an INT8 super-resolution Mode logs one `sr_kernels_unavailable{reason:"driver_below_floor"}` and
every feature given INT8 fails its CreateFeature (`0xbad00001`); FP16 Modes are unaffected. There is no "old driver, use
PTX" path (`docs/COMPAT.md` C127; before 2026-10-26 a cubin the driver refused was retried as the same kernel's PTX).

**Which image a card runs is one rule for frame generation, super resolution and XeSS-FG (since 2026-10-24, `docs/COMPAT.md` C109)**: decided once per process from the physical GPU's SM - SM 7.5 (RTX 20) the sm_75 cubins, SM 8.6 (RTX 30) the sm_86 cubins, any other SM (8.9 on RTX 40, 12.x on RTX 50, ...) PTX, JIT-compiled by the driver. Nothing is probed: a cubin is never handed to a card it was not chosen for. `[Compatibility] Router` / `KernelImage` and super resolution's `KernelImage` / `KernelSetSM` still act on that answer; super resolution serves SM 7.5 / 8.6 only (its device gate) and does not intervene on RTX 40 / 50. Before 2026-10-24 XeSS-FG offered its first kernel as `.75`, `.86`, `.ptx` in turn and took the first family the driver accepted: an RTX 40 ran the sm_86 cubins (now PTX), and a refused cubin silently moved on to the next family without a record.

From R580 on, the driver rejecting the chosen image (cubin or PTX) is that module's error: frame generation / super resolution `fg_kernel_fatal` / `sr_kernel_fatal{kind:"kernel"}` and the process ends (C93); XeSS-FG `xefg_feature_refused`, that feature's CreateFeature is refused (C94). No other image is tried. The diagnostic variable `DLSSG_FORCE_CUBIN_FAIL=1` breaks every cubin and forces this path (for validation only, all three).

**No kernel is compiled inside the game any more** (2026-10-26, `docs/COMPAT.md` C128):
- **Within a process**: every time the runtime creates its feature (menu ↔ game, resolution / setting changes, FG toggles) it releases its ~65 CU modules and builds them again. The modules and functions built from this project's images now stay for the process and are handed back, per (device, image), to every later create; the runtime's release of them is answered with success (`feature_create`'s `modules_reused` / `functions_reused` / `releases_kept`). Measured on an RTX 3070: re-creating a feature's modules went from 4.8 ms (cubin) / 1229 ms (PTX, driver cache disabled) to 0.02-0.03 ms.
- **Across processes, only on RTX 40 (the PTX family; not on RTX 50, see the end of this item)**: each PTX image is compiled once per machine and driver - at backend install, on background threads, by the driver's own CUDA linker (`nvcuda.dll` cuLink) - and stored under `%LOCALAPPDATA%\DlssgSm86\jit` (file name = the PTX's SHA-256 + physical SM + driver); later processes hand NvAPI that cubin directly. A first CreateFeature that comes before the compile is finished waits for it. One `jit_cache{batch,images,disk_hits,compiled,wall_ms}` per batch. A cache file that does not verify (truncated, digest mismatch, not an ELF) or that the driver rejects is fatal and names the file (delete it; the next start compiles it again) - never a quiet recompile, never PTX in its place. A new driver is a new key; old files are simply not used. **RTX 50 (Blackwell, physical SM 10.0 and up) does not use this cache** (2026-10-27, `docs/COMPAT.md` C131): there a cubin cuLink compiles samples D3D12 textures differently from NvAPI's own compile of the same PTX, and both engines' generated frames come out wrong. The backend logs one `jit_cache_off`, hands NvAPI the PTX as before, and the driver compiles it at the process's first CreateFeature (about 2 s for frame generation, about 9 s for XeSS-FG, less when the driver's own compute cache hits); the modules are then kept for the process.

#### `SkipRepeatedRealCopy` (removed)

This key made the backend swallow the runtime's full-resolution `CopyResource(Backbuffer → DLSSG.OutputReal)` when `MultiFrameIndex > 1`. It was removed on 2026-10-08: the runtime makes that copy only when the caller hands `DLSSG.OutputReal` over, and every Streamline `sl.dlss_g` build examined (2.6.10–2.14.1, nine builds) hands it over at index 1 only and passes null after it, so in a Streamline game there is no repeated copy to skip (captures of 5 games: `OutputReal` null at index ≥ 2 in 300/300; a live Witcher 3 run: 3486 Evaluates, 0 skipped). Savings of tens to hundreds of microseconds per real frame existed only in this project's own `sm86_scene` / `sm86_bench`, which passed `OutputReal` at every index until 2026-10-08 (since then only at index 1, as Streamline does). Since 2026-10-26 the key is not read.

#### Tiers 2 / 3: the lossy image rows, and what "no visible difference" means

Tiers 2 / 3 arm **the only kernel images in this project that are deliberately not bit-identical** to the official
runtime (the `approx` rows of the kernel pack's manifest).

**Tier 2 (the factory default from 2026-10-04 to 2026-10-21; the factory tier is 1 again since) arms one row only**: A6, the two warp taps of `Kernel_OutputPull`
lowered from a hand-written bilinear to nearest-point (mask `0x20`). Measured:

- **Against the official numerics**: on an RTX 3070, three real 4K 4X game captures, the tier-2 replay against the
  tier-0 replay (tier 0 is the official runtime's numerics), per generated frame, worst / median PSNR: Resonance
  44.06 / 48.06 dB (the fastest motion of the three, 250–500 px of displacement per real frame, the lowest values in the
  stretch of Evaluates 46–90), Forza Horizon 6 49.59 / 51.34 dB, Black Myth: Wukong 49.87 / 52.09 dB; on an RTX 5070
  against the official DLSS-G output: Cyberpunk 2077 49.8 / 53.5 dB, Black Myth: Wukong 53.2 / 56.3 dB. The deviation
  grows with the motion in the image and does not accumulate with history: replaying the same capture from a feature
  created halfway through, the per-frame difference stays within ±0.5 dB.
- **Against ground truth**: on the skip-frame ground truth of five captures, the whole frame is 0.02–0.13 dB below
  tier 1 (worse on all five); of 94,648 tiles, 229 are more than 1 dB worse than tier 1 and 36 more than 1 dB better.
  Side by side on the tiles where the two differ most (picked in both directions), the fastest stretch of Resonance and
  Forza Horizon 6 included, **no difference is visible**: where tier 1 breaks up, tier 2 breaks up the same way, in the
  same places.
- **Speed**: RTX 3070, 4X, per generated frame 1.8 % (1080p) / 1.8 % (1440p) / 2.9 % (4K, 2207 → 2142 µs;
  `sm86_bench --hooks 0`, 0.4.0) faster than tier 1. Before 0.4.0 it was 2.6 / 2.9 / 4.5 % (2379 → 2272 µs, measured
  with per-launch stamps and one real-frame copy per index); the gap is smaller because 0.4.0's exact patches speed up
  tier 1's `Kernel_OutputPull` more: P8/P9 by 58 µs, of which A6 (already at 40 registers) gains only the P8 half; P10
  by another 20 µs on the bench, where A6, with only two fetches left to hoist, is flat to slightly slower at 1080p /
  1440p (+0.3 … +2 µs per generated frame) and 3 µs faster at 4K - on real 4K game captures both tiers gain about the
  same from P10 (4–9 µs on this kernel). The kernel itself is about 16 % faster (5070), but it is only one part of a
  generated frame, so a small change in frame rate is what to expect.

**This tier shipped (2026-10-04 to 10-21) because "side by side, no difference is visible", not because it is "close to
the official output"; on 2026-10-21 the owner moved the factory tier back to 1.** The criterion of the tiers is still "how far from the official output", and by that criterion tier 2 is
44 dB from the official output in the fastest motion, so it is not bit-identical and is no longer described as "about
50 dB or better"; the project's owner judged it side by side by eye to have no visible difference from tier 1, for
3–5 % per generated frame. To compare against the official image bit for bit, use tier 1.

Tier 3 adds two experimental rows: A1 (`Kernel_WarpBlendingWeights`, 40.5 / 46.8 dB and 53.0 / 58.9 dB; that kernel
is 12 % faster at 4K and only 0.9 % at 1080p, and it is 2 % of a generated frame) and A5
(`Kernel_EstimateIntermMvecsScatter`, the scatter radius clamped to 3, 52.9 / 62.7 dB and 56.0 / 67.5 dB; the gain on
the synthetic scenes is 0, it depends entirely on the image content and **has never been measured on game content**; it
is also the MFG phase kernel, so with it on the interpolation phase has to be checked again). `ImageApproxMask` names
rows one by one.

**On RTX 20 (SM75) both tiers work the same way**: every row has an sm_75 image, and the backend arms them whatever
the kernel family.

How to confirm what it actually armed (set `[Logging] Level` to `2` temporarily and look at the backend's records in the log):

- One `image_approx`: `enabled`, `optimized_tier`, `mask` and `mask_from` (given by the INI or the tier default),
  `rows`, `armed[]` (each row's entry name, label `A1`/`A5`/`A6`, slot and resource id), `exact_patches_replaced`.
  `enabled` is true only for rows that will actually run: when they were asked for but will not run (tier 0,
  `KernelImage=Original`, the route not enabled), `enabled=false` and `disabled_reason` says why.
- One `optimized_create` per armed row, with `row="image_approx"`, `approx="A6"`, `entry`, `image` (`cubin_sm86` /
  `ptx_sm86` / `cubin_sm75` / `ptx_sm75`) and `resource`.
- `install`'s `optimized_tier` / `optimized_tier_name` and `optimized_knobs.image_approx` say what the tier finally
  resolved to.

When these records are all there and neither the image nor the frame rate changes noticeably, the tier is doing what it
should — it has not failed to take effect.

#### Preset: the A / B / C presets (UI recomposition, FG Lite)

`[DLSSG] Preset` (`[Compatibility] Preset` before 2026-10-14) is **this mod's own INI key**, not the setting of that name in NVIDIA Inspector / the NVIDIA
App — changing the preset in those tools does not go through this key; this key pins `A` / `B` / `C` on every
feature, over the driver's / NVIDIA App's preset override and the game's own parameters; since 2026-10-22 there is no value that lets the driver speak (`Auto` was removed and runs as `A`).

The DLSS frame-generation "render preset A / B" in the driver / NVIDIA App corresponds to exactly one boolean inside 310.9.1: UI recomposition (`DLSSG.UserInterfaceRecompositionEnabled`). `A` = the UI is baked into the color history and interpolated with it (the HUD is smeared by the motion vectors); `B` = the HUD-less color and the "UI color + alpha" are interpolated separately (a static HUD is not dragged along). `B` needs the game to provide both `DLSSG.HUDLess` and `DLSSG.UI`; otherwise the image is the same as A while two full-resolution FP16 surfaces are wasted (about 33 MB at 1080p, about 133 MB at 4K). One `preset_pinned` is logged per feature creation.

**Preset C (FG Lite, 310.9.16 and later, opt-in).** 310.9.16 adds a "lite candidate blending" switch (game parameter
`DLSSG.LiteCandidateBlendingEnabled`, driver preset `0x10E41DF1 = 3`): a generated frame skips the DL2 network and its
post-processing and runs `Kernel_BlendCandidatesFGLite` plus two small kernels instead, with UI recomposition off.
`Preset=C` pins both override groups, UI recomposition and Lite (`preset_pinned` gains `lite_*` fields). Measured on the
RTX 3070 (`sm86_bench`, tier 2): −32…−36 % per generated frame at 2X and −46…−54 % at 4X (4X: 1080p 0.946 → 0.437 ms,
4K 2.167 → 1.181 ms). Image quality against the skip-frame truth: −0.20 / −0.27 dB over the whole frame, −1.84 dB on the
HUD blocks (Cyberpunk, UI recomposition forced off), with more detail energy (sharper, blockier at occlusions). **There is
no official reference**: the official runtime does not run on RTX 30, so a bit-exact comparison of preset C against it
needs an RTX 40 / 50 card; so far only self-consistency between this project's routes is shown (cubin / PTX / SM75 /
tier 0 bit-identical). The factory value is `A`; C applies only when written. 310.9.16 itself turns Lite on when the game sets
`DLSSG.LiteCandidateBlendingEnabled` or the driver preset is `0x10E41DF1 = 3`; the backend pins the Lite group to the
INI's value on every feature (0 under `A` / `B`), and `preset_pinned` says what the game / driver asked for
(`lite_game_value` / `lite_drs_value`) and what ran (`lite_effective`). When the runtime's override groups are not laid
out as the backend knows them, `preset_layout_unexpected` is logged and frame generation is refused.

**Whether the two planes are handed over is up to the game, not to this key.** The two 4K captures on the development
machine are one of each: Black Myth: Wukong hands over both, and `Preset=B` really changes the image; Cyberpunk 2077
hands over only `DLSSG.HUDLess` and not `DLSSG.UI`, so `Preset=B` is a no-op there. To find out which kind your game is,
record a capture with `scripts\game\dlssg_diag_capture.cmd`: the line `HUD 平面:` ("HUD planes") in the report's
`== 抓取 ==` (capture) section says exactly which planes the game handed over (the report is in Chinese). **Only a
capture can give this line**: no record in the log of a normal run answers it.

**It is not the fix for fine-structure flicker.** `Preset` decides only which path the UI / HUD takes; the
interpolation of the scene content itself (foliage, railings, thin lines) is not affected by it. It addresses the HUD /
floating-UI judder kind of artifact; see the next section.

#### Known image artifacts and where they come from

**Fine-structure flicker (foliage, grass, railings, text), ghosting at occlusion edges and artifacts along the left and
right borders of the image, smearing while the view turns**: these are what the runtime's own interpolation algorithm
does, not something this project introduced. The criterion is a bit-for-bit comparison, **made at tier 1** — tier 1 is
the bit-identical reference: at `Optimized=1`, the offline replay of two real 4K game captures (120 Evaluates in all)
produces output **bit-identical to the official NVIDIA runtime's output on the same inputs**: `differing_pixels 0`,
`max_abs_diff 0`, all 320 planes. The same inputs produce the same pixels, so this kind of artifact exists on the
official path just the same. Tier 2 (the factory tier from 2026-10-04 to 10-21) replaces only one more image kernel (A6) on top of tier 1; its output
is no longer bit-identical, but side by side its difference from tier 1 is not visible, and this kind of artifact
appears on both tiers looking the same and in the same places ("Tiers 2 / 3" above). If you suspect an artifact comes
from this project, make sure `Optimized` is `1` (the factory value) and look again. Several reporters agree that **it is more visible at
higher multipliers**; but "a low base frame rate" is not the only factor — some see it at a base frame rate above 90,
and at 155 fps under 4X (public issues #61 / #493). The only user-side mitigation with a closed-loop report is lowering
`MaxGeneratedFrames` (issue #82, 2X), while issue #171 reports it still there at 2X.

**Whole-screen flicker / strobing is a different matter**, unrelated to the image content. The cases closed in the
public repository came from: the RTSS / MSI Afterburner overlay (issue #99), another mod that changes the image
(volumetric fog, issue #523), and the swap-chain mode (gone after switching to windowed and back once, or with V-Sync on;
issue #60 / #68 and several more). These are closures the reporters confirmed themselves, not measurements on the
development machine. Issue #532 (Zenless Zone Zero, 0.3.2) is **not closed**: in the public thread the reporter never
confirmed the effect of any change. `SkipRepeatedRealCopy`, suspected at the time, does nothing in a Streamline game:
the game ships `sl.dlss_g` 2.11.1 (the version comes from another user's log in the same thread), and the 2.11.1 build
checked on this machine, like the other 8 builds, hands `OutputReal` over at index 1 only. Provided the game's copy is
NVIDIA's signed original, the flicker did not come from this key; its cause has not been located. For whole-screen
flicker, rule things out in this order: turn the overlay off → keep only this mod, removing other mods that change the
image → switch to windowed once / turn V-Sync on.

**HUD / floating-UI judder** takes another path; see the `Preset` section above.

**Only a capture can really settle what an artifact is.** Record one while it is happening, as `docs/CAPTURE.md`
describes (`scripts\game\dlssg_diag_capture.cmd`, press Insert), and a second one from the same camera position with
`Optimized=0` as the control; they can then be replayed offline on a supported GPU and compared pixel by pixel against
the official runtime's output on the same inputs: present in the official output too = a property of the runtime's
algorithm; absent from the official output = a defect of this project, with a pixel-level starting point to locate it.
No image-quality capture has been received so far (`docs/COMPAT.md` gap G8).

**Image-quality switches in third-party forks** (for example `QualityValidWarp`) change NVIDIA's own candidate-blend
kernel, so their output is no longer the official runtime's, and the three upstream projects have measured its image
quality zero times. This project **implemented it, measured it and withdrew it**: on 11 real game captures and 14
configurations, the non-HUD area showed no measurable improvement in 13 of 14, the HUD tiles got worse, and on the one
capture that really flickers the 4X alternation got worse as well. So this project has **no** key of this kind. A
resampling change in the same kernel that **adds no decision logic** is another matter; see `CatmullRomWarp`.

## How to confirm the configuration took effect

Check in a Level 2 or 3 log (the factory `Level=1` records errors only; set it to `2` or `3` temporarily when verifying the configuration):

| Event / field | Meaning |
|---|---|
| loader: `configuration` | The INI as read this run, the runtime mode, the kernel-format request, plus `optimized_tier`/`optimized_tier_name` (the parsed consistency tier), every knob that tier resolves to (`optimized_kernels`/`image_patches`/`image_approx`/`image_approx_mask`), `enabled` (whether the frame-generation group ended up on), `warnings` (the number of `configuration_warning` records), `invalid` (the number of `configuration_invalid` records, written only when there are any) and `proxies` (`{active, standby[]}`, which proxy is doing the work). |
| loader: `configuration_invalid` (Level 1) | A key's value is invalid or out of range (since 2026-10-19, C97): `section`/`key`/`value`/`allowed`/`reason`/`effect`. It is not replaced by a default and not clamped; the feature group the key belongs to is not enabled: a frame-generation section → frame generation is not enabled (as `Enabled=0`, the request forwarded unchanged); `[SuperResolution]` and `[Runtime] SRMode` / `SRPath` → super resolution is not enabled (as `Mode=Off`); `[Logging]` / `[Debug]` → only that diagnostic is off (`Level` / `File` / `Directory` / `EvaluateEvery` keep their default). `effect` says which. One record per bad key. |
| loader: `configuration_warning` (Level 2; Level 1 until 2026-10-23, C106) | A warning that switches nothing off: `section`/`key`/`value`/`default`/`reason`/`effect`. The kinds: `[Runtime] Mode=Auto` (runs as `Bundled`), `[DLSSG] Preset=Auto` (runs as `A`, `default` `a`), a key of the other engine (`ignored under Engine=…`), the old name `[Compatibility] ImageApprox` (`default` is `(honoured)` / `(ignored)`, `reason` names the new key) and the no-longer-read old places of `[DLSSG] Optimized` / `Preset` (beside the new key or holding the default: `(no effect)`) and a key that does nothing in this configuration (`default` is `(no effect)`: `CatmullRomWarp` at tier 0 or under `ImagePatches=0`; `ImagePatches=1` / `ImageApprox=1` / `DisableFusions` at tier 0; `ImageApproxMask` with no lossy row armed); `effect` says which. A removed key is not read and never appears here (2026-10-26, C113). |
| loader: `configuration_error` (Level 1) | The whole configuration could not be read at all (a path that cannot be formed, out of memory); the mod switches off and the game's original loading is kept. **A wrong value does not get here** — that is `configuration_invalid`. |
| loader: `diagnostic_environment` (Level 1) | Right after `configuration`, only when a behaviour-changing diagnostic environment variable is set: `names`, `values`. Release builds still honour these variables (`DLSSG_FAKE_HOST_SM`, `DLSSG_FAKE_TURING_HOST`, `DLSSG_FORCE_CUBIN_FAIL`, `DLSSG_FORCE_KERNEL_FAIL`, `DLSS_SM86_FORCE_IMAGE_FAIL`, `DLSSG_CAPTURE_WRITER_DELAY_MS`), so a log with this record is not a normal run. `DLSSG_SM75_SLOTS`, `DLSSG_MFG_ARCH_ANY`, `DLSSG_FUSION_BARRIERS` and the log-only `DLSSG_GPU_CALLERS`, `DLSSG_CU_CHAIN_LOG`, `DLSSG_MFG_DIAG` were removed on 2026-10-21, `DLSSG_SR_SKIP_CU_HOOKS`, `DLSSG_XEFG_CUDA_IMAGE` and `DLSSG_XEFG15_DXIL` on 2026-10-22 (C102); none is read any more. |
| loader: `spoof_policy_mismatch` (Level 1) | The kernel32-only early read at start-up and the full parse read different values of `[General] Enabled` / `SpoofArchToGame` / `SpoofArchValue`: `peek`, `parsed`. The early read's decision holds for the whole process; this only records the difference and changes nothing. |
| loader / backend: `max_generated_frames_unsupported` (Level 1) | `MaxGeneratedFrames` cannot be served, so frame generation is not enabled or is refused, never lowered: `engine:"xess"` (above XeSS-FG's ceiling of 3); `reason:"runtime_kernels"` (an RTX 40 running the runtime's own kernels, i.e. `KernelImage=Original`); `reason:"driver_profile_limit"` (a lower driver-profile limit on a runtime build whose hook point is unknown). With `max_generated_frames`, `supported`, `effect`, `remedy`. |
| backend: `mfg_drs_override` | One per process: the backend hooks the runtime's DRS reader and drops the driver profile's frame limit (private key `0x104C9A99`); `MaxGeneratedFrames` wins. `present`, `value`, `max_generated_frames`. Level 1 when the key was actually set (`present:true`), Level 2 otherwise. |
| loader: `proxy_forward_incomplete` (Level 1) | The system DLL in this machine's System32 does not export every name in the proxy's forwarding table (a Windows version older than the build machine's, or Wine/Proton): `library`/`resolved`/`missing`/`missing_names`. Those forwarders answer `ERROR_PROC_NOT_FOUND`; **all other forwarding and the mod itself are unaffected**. A single missing name used to fail DllMain, so the game would not start (`0xc0000142`) and left no log at all. |
| loader: `runtime_redirect` | Who answered this request for `nvngx_dlssg.dll`: `mode` (`bundled`/`external`/`pinned`), **`selected_source` (the same thing in four values: `bundled`/`external`/`pinned`/`game_own`; after a fallback it names the one that actually runs)**, `requested`, **`source` (where the request came from: `game_dir` the game's own install directory / `ngx_model_store` the NGX model store = the NVIDIA App's DLSS override is on / `driver_store` the copy in the driver store / `other`, a bare file name counting as `other`; the path it actually resolved to is in `game_runtime.path`/`resolved_by`)**, `selected`, `game_runtime_used`, and **`game_runtime` — the identity of the file that was replaced (the game's own copy)**: `path`/`resolved_by`/`bytes`/`sha256`, `file_version` (the one to trust) and its two sources `file_version_fixed` (the version resource's fixed block) / `file_version_string` + `product_version_string` (the `StringFileInfo` strings), `snippet_version` (the immediate of `NVSDK_NGX_GetSnippetVersion`, `0x01360901` = 310.9.1), `generation` (`A`/`B`/`unknown`) and `generation_marker` (which criterion decided it). When it cannot be read it is just `available=false` + `reason`, and the redirect goes ahead as usual. |
| loader: `external_runtime_rejected` (Level 1) | The file named by `[Runtime] Mode=External` / `SRMode=External` cannot be used: `feature` (`frame_generation` / `super_resolution`), `key`, `configured` (as written in the INI), `path` (the full path after expansion and resolution), `sha256`, `expected_sha256`/`expected_version`, `runtime_version` (a file it knows is named, e.g. "it is 310.1.0.0"), `reason`, `action` (`frame_generation_not_enabled: …` / `super_resolution_not_enabled: …`): the load is refused, nothing falls back (since 2026-10-20; before, `bundled_runtime_used` / `bundled_snippet_used` plus one `configuration_warning`). |
| loader: `bundled_runtime_unavailable` (Level 1) | One per process: `[Runtime] Mode=Bundled`, but `dlssg_sm86\runtime\nvngx_dlssg.dll` **is not there** (most often: only the proxy DLLs were copied) or **is not this build's copy**. Carries `path` (which file it looked at), `sha256` (the digest read, `null` when the file is missing), `runtime_sha256` / `runtime_version` (the one that should be there), `found_version` (the runtime of another release of this project is named) and `remedy`. The `nvngx_dlssg.dll` load is then refused; no other file is tried and the game's own copy is not handed back (since 2026-10-20). |
| loader: `proxy_multiple` (Level 1) | More than one of this project's proxies is loaded in the process (since 2026-10-20): `active`, `standby[]`. The behaviour is unchanged: the active one is the mod, every standby only forwards. Delete the extra ones. |
| backend: `preset_ineffective` (Level 1) | `Preset=B` is pinned, but the game's Evaluate hands over no `DLSSG.HUDLess` and / or no `DLSSG.UI`: `hudless`, `ui`, `missing[]`, `handle`, `evaluate`, `effect` (the runtime's own behaviour applies; generated frames are the same as without `Preset=B`). Each feature handle is checked and recorded once, at its first Evaluate (since 2026-10-19 no re-check every 64 Evaluates; `preset_effective` is gone). See COMPAT C87. |
| loader: `preset_unavailable` (Level 1) | Only on the `-RuntimeModel 310.9` developer build: its runtime 310.9.1 has no FG Lite, so `Preset=C` cannot take effect; one per process, and the load is refused: frame generation is not enabled (since 2026-10-20; before, the runtime's own preset ran). Release packages are 310.9.16 and never log it. |
| loader: `backend_loaded` | One per process, written on success and on failure: `path` (`dlssg_sm86\backend\sm86_backend.dll`, resolved against the proxy's own directory), `sha256` (the actual digest of this backend, **the only answer to "which backend is answering"**), `loaded`/`load_error`, the backend's own `abi`/`id`/`runtime_sha256`, and the `expected_runtime_sha256`/`expected_runtime_version` this package expects. |
| backend: `kernel_pack` | One per process, only in installs that substitute kernels: which kernel pack was used and whether it matches this runtime. `path`, `manifest` (the pack's id), `model`/`schema`/`fusion_abi`, `images`/`image_bytes`, `cubins` (whether the pack carries cubins at all), `kernels`/`variants`/`patches`/`approx_rows`/`detail_rows`, `runtime_blobs`/`kernels_matched`/`kernels_unused` (how many of the loaded runtime's kernels were matched by identity, `docs/PACK_FORMAT.md`), `scan_ms`, `selected_image` → `image` (with no cubin in the pack `cubin_sm86` resolves to `ptx_sm86`; **that is not a refusal**: no error record). Replaced `kernel_cubin_pack` on 2026-10-04. |
| backend: `kernel_pack_missing` / `kernel_pack_rejected` (Level 1) | One per process: `dlssg_sm86\kernels\fg_kernels.dll` is not there (`missing`), or cannot be used (`rejected`; `reason` is one of `manifest_missing`/`manifest_parse`/`tool`/`schema`/`fusion_abi`/`model`/`structure`/`detail_constants`/`missing_resource`/`resource_size`/`target`/`runtime_coverage`, `detail` says which item). Carries `path`, `open_error`, `manifest`, `target_sm`/`actual_sm`, `effect:"route_inactive"` and `remedy`. The install then does not enable the route (`install.active=false`), the game runs normally and **frame generation is refused** (since 2026-10-16: one `fg_kernels_unavailable` as well, and every frame-generation feature creation fails with `fg_feature_refused`) - on no GPU does the runtime's own kernel set stand in. The usual cause: only the proxies and the backend were copied, not `dlssg_sm86\kernels\`, or the pack comes from another release (another runtime model). |
| loader: `sr_gate.kernel_set_source`; backend: `sr_kernel_set.source` (Level 1 on a refusal) | Where the Super Resolution kernel sets came from (since 2026-10-11): `pack` = the released kernel pack `dlssg_sm86\sr\sr_kernels.dll`, `directory` = a directory pack named by `KernelSet`. The backend's `source` also carries `path`, `outcome` (`accepted` / `missing` / `rejected`) and, on a refusal, `reason` / `detail` (one of `unmappable`/`manifest_missing`/`manifest_parse`/`tool`/`schema`/`runtime`/`structure`/`missing_resource`/`resource_size`/`index`/`set`, `docs/PACK_FORMAT.md` part 3). Without the pack the device gate does not let anything through (`sr_gate`'s one row names `sr_kernels.dll` as the file it looked for), the same as `Mode=Off`; with a damaged pack whose index is intact the snippet is still replaced and the route refuses the whole pack, and under an INT8 Mode a feature given INT8 is refused at creation (`sr_feature_refused`, C93). The usual cause: `nvngx_dlss.dll` was copied without `sr_kernels.dll`. |
| backend: `sr_resource_hook{installed:false}` (Level 1) | The hook through which Super Resolution reads weight-buffer widths was not installed: another module (usually an overlay) hooked `ID3D12Device`'s `CreateCommittedResource` / `CreatePlacedResource` first. The width check cannot run, the GPU architecture is the only gate left, and Super Resolution runs as usual; not a fault of this project or the game. |
| loader: `game_runtime_generation` | One per process: which contract generation the game's own runtime belongs to, and what this replacement loses because of it: `generation`, `sl_dlss_g_version` (the version of the Streamline plugin in the process, **read from `StringFileInfo`**, so SL 1.x shows `1.5.6.0` rather than the fixed block's `1.0.0.0`), `effect`, `lost[]`, and **`sl_dlss_g_path` + `plugin_match` (`leaf`/`ota`/`ota_override`/`none`) — whether that plugin is the game's own or the NVIDIA OTA copy that Streamline ≥ 2.11 loads from the NGX model store** (the multiplier ceiling is that plugin's own constant; Forza Horizon 6 / Resonance on the development machine both run the OTA 2.14.0). **Generation A is Level 1 (error level)**: since 2026-10-19 a generation-A game does not get frame generation, `action` says `frame_generation_not_enabled` and the request is forwarded unchanged; generation B / unknown is Level 2. |
| loader: `ngx_override_active` (Level 1) | One per process, only when `runtime_redirect.source=ngx_model_store`: **the user turned on the NVIDIA App's "DLSS Override - Frame Generation"** (or an NGX OTA update arrived), so the NGX core asks for the snippet from `C:\ProgramData\NVIDIA\NGX\models\dlssg\versions\<n>\files\<arch>_<appId>.bin`. Fields `feature`, `requested`, `store_version` (the model store's version directory, equal to the snippet's own version constant; `20318464` = 310.9.0), `snippet_sha256`. **We take over that snippet as usual**, but the same switch usually also replaces Streamline's frame-generation plugin with NVIDIA's OTA version (`\ngx\models\sl_dlss_g_0\`), and the multiplier ceiling is that plugin's own constant — neither this mod nor the INI can change it (see `plugin_match`). |
| backend: `install` | `actual_sm` the physical architecture; `active` whether the route is enabled this run; `image` the selected format; `target_sm`/`router` the resolved kernel family; `optimized_tier`/`optimized_tier_name` and `optimized_knobs` (the whole set of knobs that tier actually landed on); `kernel_pack` the kernel pack's outcome (`accepted`/`missing`/`rejected`, `not_needed` when this install substitutes no kernel). |
| backend: `image_approx` | How the lossy image rows resolved — `enabled`, `disabled_reason`, `optimized_tier`, `mask`/`mask_from` (given by the INI or the tier default), `armed[]` (each row's entry name, slot and resource id), `rows_shadowed`, `exact_patches_replaced`. The `catmull_rom_warp` and `image_patches` records carry `enabled` (true only for rows that will run) and `disabled_reason` (why a requested one will not) the same way. |
| backend: `fg_kernel_fatal` / `sr_kernel_fatal` (Level 1) | Since 2026-10-16 (`docs/COMPAT.md` C93): a kernel of this project that was selected failed after the feature existed (frame generation: the driver refused an image, the runtime created a kernel the pack does not know, an optimized row was refused, ...; super-resolution INT8: a broken contract, a weight image that cannot be built, ...). The game then **ends on purpose**; the last line is `fg_gate_crash` (exception code `0xE0D55001`, `dlssg_kernel_fatal`). It never switches back to the runtime's own kernels. Please send the log. (A foreign CU entry name used to be `cu_function_unrouted` and a refused image on RTX 40 / 50 `newer_gpu_route_fallback`; both are gone.) |
| backend: `fg_kernels_unavailable` / `fg_feature_refused`, `sr_kernels_unavailable` / `sr_feature_refused` (Level 1) | This project's kernels were selected but the kernel pack is missing, damaged or not this release's: frame generation (or the INT8 of that super-resolution feature) **does not start**, CreateFeature returns `0xbad00001`, and the runtime's own kernels do not stand in. Put back `dlssg_sm86\kernels\` / `dlssg_sm86\sr\` from the release, or choose the runtime's own kernels explicitly (`KernelImage=Original`; super resolution `Mode=Off` or an FP16 Mode). Super resolution has one more reason: `reason:"device_not_on_list"` = the GPU the game renders on is not on the list (the `[SuperResolution]` section). |
| backend: `variants_unavailable_sm75` (Level 1) | SM75-route rows missing from the kernel pack at the chosen tier (the released pack has them all): frame generation is refused (CreateFeature returns `0xbad00001`). |
| backend: `optimized_create` | One per variant / image row substituted: `row` says whether it is a `variant` (a rewritten variant), an `image_patch` (an exact patch) or an `image_approx` (a lossy row), `approx` gives a label such as `A1`/`A5`/`A6`, plus `entry`, `image`, `resource`, `status`. This is the record that says which rows a tier actually armed. |
| loader: `backend_install` | `status=0` means the backend installed successfully; combine with `install.active` to tell whether kernel replacement is enabled. `-100` = the backend is unusable, with the reason in the next row. `kernel_pack` (since 2026-10-04) is the backend's verdict on the kernel pack in this install, reported to the loader through `DlssgMod_GetInfo`: `outcome` (`accepted`/`missing`/`rejected`/`not_needed`), `found` (whether the file mapped), the `model`/`schema`/`manifest` its manifest states (filled in for a refused pack too - a pack of another runtime model says here which model it is), `reason` (as in `kernel_pack_rejected`); `null` when the backend was refused (`-100`). |
| loader: `backend_unusable` (Level 1) | The record right after a `backend_install` that reports `-100`, saying which kind it is: `loaded=false` + `load_error` (**the most common: `dlssg_sm86\backend\sm86_backend.dll` was not copied over, or an antivirus / Windows "Smart App Control" blocked it**, which shows on the game's side as `Bad Image 0xc0e90002`; the `backend_loaded` record says where it looked), an export of this release missing, an ABI mismatch (one install mixing the proxies and the backend of two releases), or a backend that does not match the runtime SHA of the module being installed. The `remedy` field says what to do. |
| `image=ptx_sm86` / `cubin_sm86` / `original` | The SM86 kernel format that was selected, or no replacement. |
| backend: `driver_below_floor` (Level 1) | Since 2026-10-26: the driver is older than R580 (CUDA 13.0); frame generation and XeSS-FG are not installed (`cuda_driver`, `floor`; `driver` is NvAPI's driver version when available). Update the driver. |
| backend: `jit_cache` | RTX 40 (the PTX family; RTX 50 does not use the cache) only: one batch of PTX images through the cubin cache (`batch`, `images`, `disk_hits`, `compiled`, `wall_ms`, `directory`, `key`). |
| backend: `jit_cache_off` | One per process, RTX 50 (physical SM 10.0 and up) only: the PTX cubin cache is not used (`sm`, `reason`, `effect`) and NvAPI compiles the PTX (C131). The expected state, not an error. |
| backend: `preset_pinned` | `[DLSSG] Preset` took effect; one per feature: `requested` (`A`/`B`/`C`), what the game and the driver gave (`game_value` / `drs_value`, plus `lite_game_value` / `lite_drs_value` on 310.9.16), what runs (`effective`, `lite_effective`) and `overrode_runtime`. |
| backend: `drs_dlssg` | One per process (on the first `PopulateParameters`): **the seven DLSS-G keys of the NVIDIA driver's per-game profile**, read only. `profile`/`profile_source` (`application` = this EXE's own profile, `current_global` = the global one)/`profile_settings`, `keys{<id>:{name,consumer,value,location,predefined}}`, and, picked out separately, `plugin_limit` (`0x104D6667`, the "requested frame count" the Streamline plugin reads) and `runtime_limit` (the private key `0x104C9A99`, which the runtime reads itself on every Evaluate). **The unit is generated frames** (the NVIDIA App's `4x` is written as `3`). A key never written is `value:null` + `status:-160` (`NVAPI_SETTING_NOT_FOUND`), **not 0** — on an untouched machine all seven keys are like that. `plugin_limit` is the plugin's own request, and a game picking a lower multiplier from it is allowed; `runtime_limit` no longer lowers `MaxGeneratedFrames` since 2026-10-19 (see `mfg_drs_override`; on a runtime build whose hook point is unknown it is `max_generated_frames_unsupported`). |
| backend: `mfg_probe` / `mfg_summary` | Records from `[Debug] MfgProbe` (`ForceGeneratedFrames` was removed); how to read them in `docs/MFG_PROBE.md`. |
| Level 3: `kernel_create` / `evaluate` | The format and status actually used for each kernel creation; the actual generated count, the result. |

To confirm the route installed, check both `install.active=true` and the matching `backend_install.status=0`. `image=ptx_sm86` on its own does not mean a kernel was created or executed; look at the later `kernel_create` / `evaluate` as well.

### Which generation the game's own runtime is: generation-A games do not get frame generation

DLSS-G changed its whole contract between `3.8.1 → 310.1`, and **the generation of the `nvngx_dlssg.dll` a game ships
decides how many frames that game can generate at most**:

- **Generation B (310.x)**: it has the `DLSSG.MultiFrameCount` / `MultiFrameCountMax` pair of parameters, i.e.
  multi-frame generation (MFG). `MaxGeneratedFrames` really takes effect in these games (at most 5 = 6X).
- **Generation A (1.0.1 – 3.8.x; of the 13 runtimes the games in the development machine's Steam library ship, 5 are
  this generation: The Witcher 3, Bright Memory, Marvel's Spider-Man Remastered, Atomic Heart, Delta Force)**: that
  generation **has no parameter named `DLSSG.MultiFrameCount` at all**, so the game's own Streamline plugin never
  writes it and the runtime keeps its default of 1.

**Since 2026-10-19 a generation-A game does not get frame generation** (owner decision: these old plugins kept bringing
compatibility problems nobody could pin down), whatever `MaxGeneratedFrames` says: the game's request for
`nvngx_dlssg.dll` is forwarded unchanged (as with `[General] Enabled=0`), with one error-level `game_runtime_generation`
per process whose `action` is `frame_generation_not_enabled`. On RTX 30 / 20 that means these games have no frame
generation. Before, we took them over and they ran (The Witcher 3 DX12, shipping Streamline 1.5.6 + `nvngx_dlssg.dll`
3.1.13.0: three sessions of 6009 / 7352 / 28757 Evaluates, 96/96 planes bit-identical to the official runtime), but at 2X
only, with `MaxGeneratedFrames` unable to change it, and without UI recomposition (`Preset=B`), Reflex Warp, dynamic MFG and
the per-frame "do not interpolate this frame" switch (generation A expresses it as `DLSSG.EnableInterp`, which 310.x does
not read) - which contradicts "`MaxGeneratedFrames` is a requirement".

How to read it in the log: `generation` of `game_runtime_generation` = `A` / `B` / `unknown`, the file that was examined in
`game_runtime` (criterion in `generation_marker`); generation A is error level, so the factory `Level=1` shows it too. A file
whose generation cannot be recognised (`unknown`) is taken over as usual. `scripts\game\dlssg_diag.cmd` and
`python scripts\analysis\sm75_report.py` both turn it into a one-sentence conclusion (in Chinese).

### Why frame generation will not turn on: `fg_gate_*` (always on, no INI key)

When `install` looks entirely normal after installation but there is not a single `kernel_create` / `evaluate`, either the game never created the DLSS-G feature or the creation failed before the runtime built its first kernel — two things our old log could not tell apart, let alone explain. The decision is made entirely above the runtime: Streamline's own gates (**hardware-accelerated GPU scheduling (HAGS) off** → `eFeatureNotSupportedHWSchedulingDisabled`, or a driver below what the plugin requires), the NGX core's final answer to `GetFeatureRequirements` (the core adds its own GPU / driver / HAGS judgement on top of the runtime's answer), the NGX capability parameters the plugin reads, and the game's own UI switch. `src/backend/ngx/fg_gate.cpp` observes only these four places, and all five records are visible at Level 2 (only a failed CreateFeature is error level and is recorded at Level 1 as well):

| Event (backend) | Content / how to read it |
|---|---|
| `fg_gate_environment` / the module versions in `mfg_modules` | Each module's `file_version` is **the one to trust**: when the version resource's fixed block and `StringFileInfo` disagree (Streamline 1.x's fixed block always says `1.0.0.0`; the real version is only in the strings), the string's is recorded, with the two extra fields `file_version_fixed`/`file_version_string` showing the disagreement. When the two agree there is only `file_version`, exactly as in older logs. |
| `fg_gate_environment` | One record at install time. `hags` (`KMTQAITYPE_WDDM_2_7_CAPS` from `D3DKMTQueryAdapterInfo`: `supported`/`enabled`/`enabled_by_default`, `state=on\|off\|unsupported`), `driver` (`NvAPI_SYS_GetDriverAndBranchVersion`, e.g. `610.74` / `r610_00`, plus the file version of a loaded `nvapi64.dll`), `windows` (`RtlGetVersion`), `ngx_core` (`_nvngx.dll` path + version), `streamline` (the loaded `sl.interposer.dll` / `sl.common.dll` / `sl.dlss_g.dll` / `sl.dlss.dll`, looked up with `GetModuleHandle` only, never loaded). **`hags.state=off` is basically the answer.** Every item is optional; when it cannot be obtained it is written as `available=false` plus a reason, and it never makes the install fail. |
| `fg_gate_hags_disabled` (Level 1) | The one item of the record above that is **both a Windows setting the user can change and enough on its own to keep frame generation off in every game**: HAGS is supported on this machine but switched off. Since the factory `Level=1` does not write `fg_gate_environment` at all, this one is recorded separately at error level and shows at any log level; the `remedy` field says how to turn it on. (Public issue #513: someone spent three hours tracking this down.) |
| `fg_gate_hooks` / `fg_gate_hooks_deferred` | What is hooked are the **public exports of the NGX core `_nvngx.dll`** (`GetFeatureRequirements` / `GetCapabilityParameters` / `GetParameters` / `CreateFeature`), each of the four reporting `attached`. When the core is not loaded yet, `LoadLibrary` is watched first (`fg_gate_hooks_deferred`) and the hooks go in when it appears; the record also carries the `ngx_core` version once. |
| `fg_gate_requirements` | Each `GetFeatureRequirements` in the core (up to 8). `feature_supported` is the value **the caller actually received**, `decoded` unpacks it bit by bit (0 supported / 1 GPU unsupported / 2 driver version unsupported / 4 OS version unsupported / 8 hardware scheduling off), and there are also `min_hw_architecture`, `min_os_version`, `caller` (calling module + offset within it) and `adapter_description`. Do not confuse it with the backend's own `feature_requirements`: that one is only the runtime's answer to our bridge, and the core still adds its own judgement on top. **`status_decoded` / `status_reason`** say which path of the runtime's own function the status came from: 310.x's `GetFeatureRequirements` can return only three values — `1` (`success`), `0xbad00002` (`nvapi_enum_physical_gpus_failed`, the **only** one of its five checks that fails the whole call) and `0xbad00005` (`input_is_nullptr`). **`success` does not mean "supported"**: an OS version that is too old, a failed `NvAPI_GPU_GetArchInfo` and a failed `getDeviceID` all **return success and write the verdict into the flags**, so look at `decoded`, not at the status. Any other value (`0xbad0000b`, for example) means the call was not answered by a 310.x runtime. |
| `fg_gate_capability` | After each capability query (up to 4), reads back `FrameGeneration.Available` / `.NeedsUpdatedDriver` / `.MinDriverVersionMajor` / `.MinDriverVersionMinor` / `.FeatureInitResult`, `DLSSG.MultiFrameCountMax`, `DLSSG.ReflexWarp.Available`, and `SuperSampling.Available` as a control. `Available=0` or `NeedsUpdatedDriver=1` = the plugin refused before anything on our side ever ran. `.FeatureInitResult` is the NGX status the core recorded for this feature: `3134193675` = `0xBAD0000B` (`NVSDK_NGX_Result_FAIL_OutOfDate`), which means exactly the same as the `CreateFeature` failure in the next row, except that the core recorded it back in the capability-query stage, and once the plugin sees `Available=0` it never calls `CreateFeature` again (measured on an RTX 2080 Ti in The Witcher 3). |
| `fg_gate_create_feature` | Each `CreateFeature` in the core (up to 16): `feature_id` (DLSS-G = 11), `status`, `succeeded`, `handle`, `caller`, and for DLSS-G also `parameters` (`multi_frame_count` / `multi_frame_count_max` / `width` / `height`). **A failure is error level**; `0xbad0000b` (`NVSDK_NGX_Result_FAIL_OutOfDate`) = the core took the GPU architecture it read from NVAPI during its own initialization, compared it against `NVSDK_NGX_GetGPUArchitecture` exported by the DLSS-G runtime (snippet) (= the minimum architecture the runtime declares), and refused. The backend reports that export as the **physical architecture** (`turing_host.gpu_architecture_export`, `0x160` on Turing; it used to report Ada `0x190`, which was exactly why frame generation would not turn on on real Turing). `SpoofArchToGame` governs the other gate (the game's / Streamline's own) and does not help here: the redirect is now installed at game start (`trigger=dllmain`/`nvapi_load`/`dependency_load`), but what the core compares here is not NVAPI's answer, it is the minimum architecture the snippet exports. |
| `fg_gate_summary` | One record, on the first Evaluate (`reason=first_evaluate`) or at process exit (`process_exit`): the counts of requirements / capability / create and the status of the last creation. `create_calls=0` with an entirely normal `install` = the game never asked for the feature. |
| `fg_gate_exit` | One record, written only in `DllMain(DLL_PROCESS_DETACH)`: `evaluates` (how many Evaluates ran in all), `features` (how many frame-generation features were created), `last_evaluate_age_ms` (how long before the process exit the last Evaluate ran), `clean:true`. **A process that is killed or crashes never gets here, so a log that does not end with this record means "not a normal exit"** — the summary above is written on the first Evaluate, so on its own it cannot tell "played and quit" from "died while playing". Builds of older versions have no `fg_gate` records at all; do not read that as a crash. |
| `fg_gate_crash` (Level 1) | **The last record of a process that crashed** (`[Logging] CrashRecord`, on by default): `code` / `name` (e.g. `0xc0000005` `access_violation`), `address`, `module` (full path of the faulting module) + `offset`, `role` (`backend` / `proxy` = our code, `ours:true`; `runtime` = the NVIDIA runtime we hook; `other` = the game or another module; `unknown` = not inside any image), `thread`, `av{access,target}` for an access violation, `chained` (the previous handler was called afterwards), `rearmed`. At most one per process. **`fg_gate_summary`, no `fg_gate_exit` and no `fg_gate_crash`**: the process was killed, or crashed where this cannot see (the game's own crash reporter handled it first, `__fastfail`, a debugger attached; `docs/COMPAT.md` C89). |
| `foreign_fg_mod` (Level 1) | **Another frame-generation mod is in the process** (`docs/COMPAT.md` C88): one record per module, scanned after the first frame-generation `CreateFeature` and again when new modules load. `module`, `kind` (`rtx_encore` / `dlssg_transfusion` / `mfg_unlock` / `optiscaler` / `dlssg_to_fsr3` / `dlssg_sm86_other_copy` (an older build or companion add-on of ours) / `reshade_fg_addon` / `second_backend` (a second `sm86_backend.dll`) / `second_dlssg_runtime` (a DLSS-G runtime other than the one we loaded) / `ngx_wrapper` (a non-NVIDIA module exporting the NGX entry point)), `matched` (how it was recognised: file name / version string / a config file next to it / an export name), `version`, `when`. Logged only; nothing is changed. Two frame-generation engines in one game commonly end with frame generation "on" and not one frame generated: keep one. Nothing is written when there is no other mod. |
| `kernel_handle_recycled` | At most one per process: the driver handed a kernel handle from `old_rva` over to `new_rva` (`had_variants` = it was still bound to the previous kernel's variant set at the time). This is normal driver behaviour, and every creation rebuilds what the handle means, so it is **not a problem** in itself; the totals are in `evaluate.optimized.handles_recycled` / `stale_bindings_cleared` and in `feature_create`. |
| `stale_variant_rejected` (Level 1) | **Should never appear**: the last guard before a launch stopped a variant row that belongs to another kernel, which means some creation path still does not reset the handle. If you see it, please send the log to the developers. |

The `== fg gate ==` section of `python scripts\analysis\sm75_report.py <log directory>` puts the records above on one page, and the VERDICT gets an extra `feature created` line: PASS when creation succeeded, FAIL (with the status) when it failed, and NOT RUN when it was never called, naming the most likely gate from the evidence.

## Advanced configuration examples

The examples below are for diagnostics / validation only; a normal install needs none of them. Just add the section and key to the factory INI (add the `[Compatibility]` / `[Debug]` section yourself if it is missing).

**Force PTX JIT on a 3080 Ti (automatic detection uses the cubin; write it only to diagnose):**

```ini
[Compatibility]
KernelImage=PTX
```

**Validating the SM86 PTX path on an RTX 5070 (a development validation configuration):**

```ini
[Compatibility]
KernelImage=PTX
ForceSM86Route=1
SimulateAmpere=1

[Logging]
Level=3
```

**Taking a local stock-numerics reference:** just use the factory INI and change `Optimized` to `0` (do not use `KernelImage=Original` on Ampere).

**Turning on generated-frame markers (to tell whether an artifact is on a generated frame or a real one):**

```ini
[Debug]
MarkGeneratedFrames=1
```

**Allowing 6X:**

```ini
[FrameGeneration]
MaxGeneratedFrames=5
```

## Installation notes and uninstalling

Put one proxy in the game folder: `version.dll` from the package root (first choice); if the game does not load it, go through `alternatives/winmm.dll` → `alternatives/dbghelp.dll` → `alternatives/dinput8.dll`, **one at a time, deleting the previous one first** (how to tell is in "One proxy at a time" above). Each loads the same backend from the same `dlssg_sm86\backend\`; only the file name and forwarding target differ. **The `dlssg_sm86\` folder has to go over with it**: the proxy itself is only the loader.

The render-path proxies `alternatives/dxgi.dll` and `alternatives/d3d12.dll` (usable, but riskier) sit on the D3D12 hot path and are load-order sensitive: use them only when none of the four above was loaded by the game, as a last resort, and only one of the two. The instructions ship with the package in `alternatives/README.md`.

Every proxy forwards all of its own exports to the real DLL of the same name in `System32`, and intercepts only the loading of `nvngx_dlssg.dll`.

Code signing the release DLLs is optional: `build.ps1 -SignCert <pfx> -SignPass <pw>` (or `DLSSG_SIGN_CERT`/`DLSSG_SIGN_PASS`) applies a SHA-256 Authenticode signature to every DLL of this project (`version.dll` at the root, the five proxies in `alternatives\`, and the backend and the three resource packs under `dlssg_sm86\`), leaving NVIDIA's runtime and snippet untouched; without a certificate the artifacts are unsigned. The steps, how to make a throwaway test certificate, and why a self-signed certificate gives no default trust are in `docs/SIGNING.md`.

Windows x64 / D3D12 for now. This package's build checks and optional GPU validation status are in `manifest.json` / `validation.json`, the architecture in `docs/ARCHITECTURE.md` and capture and replay in `docs/CAPTURE.md`.

To uninstall, exit the game and remove the one proxy you put there (`version.dll`, or the alternative you switched to), `dlssg_sm86.ini` and the `dlssg_sm86\` folder (the logs are in it too). Ownership of the original NVIDIA DLL and the kernel resources is in `THIRD_PARTY_NOTICES.txt`; the SM75 GPU resources come from Coldwood1026's RTX 20-series porting work, gratefully acknowledged.
