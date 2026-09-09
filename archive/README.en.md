# DLSSG SM86

[简体中文](README.md) | English

**Copy `version.dll` and `dlssg_sm86.ini` beside the game's actual rendering executable, then launch the game normally.** No Python or PowerShell launcher is required at runtime.

The current Release (September 7, 2026) is the first milestone. The proxy embeds the matching original DLSSG 310.1 runtime, including its models and pipeline, and the SM86 backend. The default `Mode=Bundled` redirects DLSSG requests to this embedded implementation, regardless of the game's own DLSSG version.

On first use, the proxy extracts the matching files to `%LOCALAPPDATA%\DlssgSm86\bundles\<bundle-id>`, verifies their hashes, and loads them. Later launches reuse the cache; damaged cached files are restored automatically.

## Configuration file

The INI must be named `dlssg_sm86.ini` and placed beside the active `version.dll` or `winmm.dll`. **Fully exit and restart the game after changing it.** Settings are not reloaded while the process is running.

- Use `0` or `1` for Boolean settings. Lines starting with `;` are comments.
- `Mode` and `KernelImage` values are case-insensitive.
- Log directories, cache directories, and DLL paths may be absolute or relative to the proxy DLL / INI directory.
- Custom paths do not expand variables such as `%LOCALAPPDATA%` or `%TEMP%`. Enter the actual path, or leave `CacheDirectory` empty to use the default cache location.
- For normal use, keep `Mode=Bundled` and both compatibility switches at `0`.
- Update the proxy DLL and INI together when adopting new options.

## Complete default configuration

```ini
[General]
Enabled=1

[FrameGeneration]
MaxGeneratedFrames=3

[Logging]
Level=2
File=1
DebugOutput=0
EvaluateEvery=120
Directory=dlssg_sm86\logs

[Debug]
MarkGeneratedFrames=0
MarkerX=8
MarkerY=8
MarkerScale=2

[Compatibility]
KernelImage=Auto
ForceSM86Route=0
SimulateAmpere=0

[Runtime]
Mode=Bundled
CacheDirectory=
Path=

[Backends]
```

### [General]

| Option | Default | Behavior |
|---|---|---|
| `Enabled` | `1` | Enables DLSSG redirection, adaptation, capability reporting, and optional markers. `0` preserves the game's original DLSSG loading behavior. The proxy still forwards the system DLL's original exports. |

### [FrameGeneration]

| Option | Default | Behavior |
|---|---|---|
| `MaxGeneratedFrames` | `3` | Maximum additional generated frames advertised to the game. `0` preserves the runtime capability; `1` allows up to 2X, `2` up to 3X, and `3` up to 4X. |

**The game chooses the actual generated-frame count.** This setting does not independently add menu options or force 4X presentation. The backend limit is three additional frames. Accepted INI values from 4 to 16 are clamped to 3; values above 16 or invalid numbers cause configuration parsing to fail. Use 0–3 for normal operation.

### [Compatibility]

| Option | Default | Behavior |
|---|---|---|
| `KernelImage` | `Auto` | Selects `Auto`, `PTX`, or `Cubin`. An omitted setting also uses Auto. |
| `ForceSM86Route` | `0` | `0` automatically enables the adaptation route on physical SM86. `1` allows forced SM86 routing on other GPUs for validation; architectures below SM86 are still rejected. |
| `SimulateAmpere` | `0` | Adjusts the reported architecture for validation. Requires `ForceSM86Route=1`; it does not change the physical GPU. |

| KernelImage | Behavior when the SM86 route is active |
|---|---|
| `Auto` | Uses the precompiled SM86 cubin on physical SM86; uses SM86 PTX on other GPUs. |
| `PTX` | Uses SM86 PTX, which the driver JIT-compiles for the GPU. Also available on the RTX 3080 Ti. |
| `Cubin` | Requires physical SM86 and uses the precompiled cubin. Incompatible hardware is rejected; the default Bundled mode attempts to fall back to the original load request. |

**KernelImage selects the kernel format; it does not independently enable routing.** The RTX 3080 Ti is detected as SM86 automatically. Keep both compatibility switches at `0` when using Auto, PTX, or Cubin on this card. Devices without an active SM86 route use the runtime's original kernels.

This option does not change model weights, FP16 precision mode, or the advertised frame count. PTX requires driver support for the supplied PTX version and may incur JIT compilation on first use.

### [Logging]

| Option | Default | Behavior |
|---|---|---|
| `Level` | `2` | `0`: off; `1`: errors; `2`: adds configuration, loading, and capability information; `3`: adds kernel creation, Evaluate, and marker traces. |
| `File` | `1` | Enables file logging, subject to Level. |
| `DebugOutput` | `0` | Sends messages through Windows debug output for a debugger to receive. Does not display messages in the game. |
| `EvaluateEvery` | `120` | Sampling interval for normal Evaluate / marker messages, counted in Evaluate calls. `1` logs every call, `0` is treated as 120, and the maximum is 1,000,000. |
| `Directory` | `dlssg_sm86\logs` | Log directory; accepts another nonempty relative or absolute path. |

Files are named `loader_<PID>.jsonl` and `backend_<PID>.jsonl`. At Level 3, the first 12 Evaluate calls are logged; later normal calls follow `EvaluateEvery`. Failures are still logged at the error level. Multiple generated frames can require multiple Evaluate calls, so this interval is not a count of displayed game frames.

Level 2 is normally sufficient to confirm configuration and kernel selection. Level 3 produces more diagnostic output.

### [Debug]

| Option | Default | Behavior |
|---|---|---|
| `MarkGeneratedFrames` | `0` | `1` draws a marker such as `FG 1/3` on generated output. Real-frame and Reset outputs are skipped. |
| `MarkerX` | `8` | Marker left edge in output-texture pixels, range 0–65535. |
| `MarkerY` | `8` | Marker top edge in output-texture pixels, range 0–65535. |
| `MarkerScale` | `2` | Scale from 1 to 8. The rectangle is `24 × scale` by `9 × scale` pixels, or 48×18 at the default scale. |

The marker must fit completely inside the output texture. An out-of-bounds marker fails and logs `marker_failed`. Markers modify generated-frame pixels: disable them for full-image numerical comparisons, or compare outside the marker region separately.

### [Runtime]

| Option | Default | Behavior |
|---|---|---|
| `Mode` | `Bundled` | Selects `Bundled`, `Auto`, or `Pinned`. Runtime Auto is independent of KernelImage Auto. |
| `CacheDirectory` | Empty | Uses `%LOCALAPPDATA%\DlssgSm86\bundles` when empty. A custom root receives a subdirectory for each bundle ID. |
| `Path` | Empty | Used only in Pinned mode; specifies the original `nvngx_dlssg.dll` to load. |

| Mode | Behavior |
|---|---|
| `Bundled` | Uses the embedded runtime and matching backend. Normal installations do not need Path or Backends entries. |
| `Auto` | Uses the original runtime requested by the game. Known hashes can select a matching backend; other versions need a matching external backend or retain their original behavior. |
| `Pinned` | Attempts to load Path with a matching backend. Missing or unsuitable files fall back to the original request; details are logged. |

Bundled uses the package's fixed 310.1 models, rather than automatically adopting models from newer game DLLs. Cache extraction, runtime loading, or backend installation failures log `runtime_selection_failed` and attempt the original request. A successful fallback does not establish that SM86 routing is active.

### [Backends]

Leave this section empty for normal Bundled use. Advanced Auto / Pinned setups can map an original runtime's complete SHA256 to its matching backend DLL:

```ini
[Backends]
; Replace the placeholder with the original runtime's complete SHA256.
; <runtime-sha256>=backends\matching_backend.dll
```

A mapping does not automatically adapt an unknown runtime; the backend must support that exact file. Explicit PTX / Cubin selection also requires an external backend that supports the kernel-selection extension. Unsupported selection logs `kernel_selection_unsupported` and rejects installation. Auto selection remains compatible with the original ABI 1 backend interface.

## Configuration examples

Replace the keys in the existing INI sections and restart the game. Keep other settings at their defaults.

### Use PTX JIT on the RTX 3080 Ti

```ini
[Compatibility]
KernelImage=PTX
ForceSM86Route=0
SimulateAmpere=0
```

Set KernelImage back to Auto, or explicitly to Cubin, to restore the precompiled path. Architecture simulation is not required.

### Enable generated-frame markers

```ini
[Debug]
MarkGeneratedFrames=1
MarkerX=8
MarkerY=8
MarkerScale=2
```

The denominator reflects the game's actual request. If the configured maximum is 3 but the game requests one generated frame, the marker reads `FG 1/1`.

## Confirm that the configuration is active

Check the Level 2 or Level 3 logs:

| Event or field | Meaning |
|---|---|
| Loader `configuration` | INI path, runtime mode, and requested kernel format. `requested_max` is the INI value before the backend limit; see `mfg_capability` for the advertised result. |
| Backend `install` | Preparation for installation. `actual_sm` identifies physical architecture; `active` is the planned SM86 route state; `image` is the selected format. This event occurs before hook installation finishes. |
| Loader `backend_install` | `status=0` means backend installation succeeded. Also check `install.active` to confirm that the SM86 route was enabled. |
| `image=ptx_sm86` / `cubin_sm86` | Selected SM86 PTX / cubin format. |
| `image=original` | Uses original runtime kernels without replacement. |
| `mfg_capability` | Original capability and the advertised maximum generated-frame count. |
| Level 3 `kernel_create` | Format and status of each kernel creation. |
| Level 3 `evaluate` / `frame_marker` | Actual generated-frame count, evaluation result, and marker index. |

Confirm both `install.active=true` and the corresponding `backend_install.status=0`. A selected image format alone does not prove that kernels were created or executed; inspect the subsequent `kernel_create` and `evaluate` results.

Invalid numbers, enumeration values, or combinations fail configuration parsing and can log `configuration_error`. Backend compatibility failures can appear as `install_failed`. Restore the default INI and restart to check again.

## Tested release and known limits

The tested platform is Windows x64 / D3D12 with an RTX 3080 Ti and driver 591.86. Offline Auto, PTX, and Cubin execution and output agreement on the same GPU were checked. Both games executed Auto/cubin at 2X and 4X; Wukong also covered disabling and re-enabling frame generation.

| User-reported gameplay | Off | 2X | 4X |
|---|---:|---:|---:|
| Black Myth: Wukong | 50 FPS | 80 FPS | 150 FPS |
| Cyberpunk 2077, path tracing | 35 FPS | 60 FPS | 100 FPS |

These approximate manual observations apply to the DLL identified in the [test record](docs/VALIDATION.md) (Chinese). The tester described the experience as relatively stable. Standardized frame-time, latency, and long-duration stability measurements have not been performed.

Small differences against fixed reference images remain a known issue: the maximum observed channel difference was 3/255. Strict validation remains `passed=false`; release acceptance is recorded separately in the manifest. New binaries require their own validation, and other hardware is not covered by these results.

## Uninstallation

Exit the game and remove the proxy DLL and INI added by this package. The cache can remain for other installations. Use only one proxy entry point; existing mods with the same filename require resolving the conflict. Arbitrary proxy chaining is not implemented.

See `LICENSE.md` and `THIRD_PARTY_NOTICES.txt` for license terms and third-party attribution.
