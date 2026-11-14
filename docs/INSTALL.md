# DLSSG SM86 融合版

**中文** · [English](INSTALL.en.md)

**把发布包根目录的 `version.dll`、`dlssg_sm86.ini` 和 `dlssg_sm86\` 这个文件夹复制到游戏实际渲染 EXE 旁，照常启动。** `version.dll` 是唯一要放的代理（钩子 DLL）；游戏不加载它时，删掉它，按固定顺序一次换一个 `alternatives\` 里的代理（见下面“代理只放一个”）。运行时无需 Python 或 PowerShell 启动器。第一次安装、想一步一步照着做的，先看发布包根目录的安装手册 `MANUAL.html`（双击用浏览器打开）；本文是全部设置的完整说明。

发布树长这样，每一样东西都是一个文件，代理本身只有半兆、不内嵌任何载荷：

```
<游戏 EXE 目录>\
  version.dll                    代理（只有加载器）；EXE 旁只放这一个代理
  dlssg_sm86.ini
  dlssg_sm86\
    backend\sm86_backend.dll     本包这个模型的后端（已签名）
    kernels\fg_kernels.dll       帧生成内核包：全部内核镜像（cubin + PTX）与清单，只有资源、没有代码；缺了它帧生成打不开（日志 kernel_pack_missing、fg_kernels_unavailable）
    runtime\nvngx_dlssg.dll      配套的原版 DLSSG 运行库，`Mode=Bundled` 用的就是它
    sr\nvngx_dlss.dll            超分 snippet（DLSS 310.9.16），开关关着时完全不碰
    sr\sr_kernels.dll            超分内核包：RTX 30 / 20 的 INT8 内核集（四套），只有资源、没有代码；缺了它超分开关不生效（日志 sr_gate 写明找的是这个文件）
    xefg\xefg_kernels.dll        XeSS-FG 模型包（Engine=XeSS 用）：程序、权重与 CUDA 镜像；缺了它 Engine=XeSS 拒绝帧生成（xefg_unavailable）
                                 （老版本的 sr\ 里还有 index.json 与 set_* 目录，新版不再读它们，可以删）
                                 （老版本还有 rr\，是 2026-10-09 删除的光线重建路线的；留着的话不会被读取，可以删）
    logs\                        日志
```

发布包里另有 `alternatives\`：`winmm.dll`、`dbghelp.dll`、`dinput8.dll`、`dxgi.dll`、`d3d12.dll` 五个备用代理（与 `version.dll` 是同一份加载器）和一份说明 `README.md`。平时不用复制，只在游戏不加载 `version.dll` 时取其中一个换上。

代理按**自己所在目录**解析这些路径（不是进程工作目录），所以代理、`dlssg_sm86.ini` 与 `dlssg_sm86\` 要放进同一个目录；把代理单独拷过去、`dlssg_sm86\` 留在原地，日志里会是 `backend_loaded{loaded:false}` 或 `bundled_runtime_unavailable`，游戏保留它自己的 DLSS-G。

**不再往 `%LOCALAPPDATA%` 写运行库或后端**（RTX 40 上唯一写的是 PTX 编译结果的缓存（RTX 50 不用它，C131） `%LOCALAPPDATA%\DlssgSm86\jit`，见“驱动版本要求：R580”）。老版本会把运行库和后端释放到 `%LOCALAPPDATA%\DlssgSm86\bundles\<id>\` 再从那里加载；现在没有这一步（那也是杀软和 Windows「智能应用控制」最容易拦的一步）。从老版本升级的人可以手动删掉 `%LOCALAPPDATA%\DlssgSm86`，mod 自己不会去删它——那个目录里可能还有别的版本在用的东西。

出厂 INI 见下面“常规使用”。所有诊断 / 兼容性 / 性能实验开关仍然被解析和支持，但不在出厂 INI 里；缺失时各自取安全且经过验证的默认值，完整清单见后面的“高级 / 诊断键”。

## 运行库模型（包里是哪个运行库）

发布包只带**一个**运行库和配套的一个后端，`manifest.json` 的 `runtime_model` 写明是哪一个。2026-10-14 起默认模型是 **310.9.16**，310.9 仍可构建、作为备选：

| 模型 | 产物 | 随包运行库 | 最大生成帧 | 备注 |
|---|---|---|---|---|
| **310.9.16**（默认） | `dlssg-release-0.4.2-x64.zip` | `nvngx_dlssg.dll` 310.9.16.0 | 5（6×） | 2026-10-14 起的默认。71 个内核里 68 个与 310.9.1 逐字节相同；`Kernel_BlendCandidatesFused`、`Kernel_DownsampleRGBAndEstimateUIFused` 有改动、`Kernel_BlendCandidatesFGLite` 是新的（三个都从运行库自己的 sm_120 PTX 重定向到 sm_86 / sm_75）。默认预设下与 310.9.1 **逐位相同**（合成场景 56/56、两份真实抓取各 105/105 个平面）。多一个**预设 C（FG Lite）**，见下面 `Preset` 一节。变体行 34 + 3 个图像补丁（P7 按 310.9.16 的 `Kernel_BlendCandidatesFused` 重新推导，另两个与 310.9.1 共用）；`CatmullRomWarp` 的行挂在 310.9.1 的旧内核上，这个模型的内核包里没有（写了就拒绝帧生成，见下面「开了做不到就拒绝」）。`Router=SM75` 可用（3070 前向 JIT 已验证；新三个内核在真 Turing 上还没跑过） |
| **310.9**（备选） | `dlssg-release-0.4.2-x64-310.9.zip` | `nvngx_dlssg.dll` 310.9.1.0 | 5（6×） | 原生 6×；优化内核、跨内核合并与图像内核补丁均可用（37 个变体行：34 个网络变体 + 3 个按 310.9.1 RVA 重新推导的图像补丁）。`Router=SM75`（RTX 20）可用（3070 前向 JIT 与真 Turing（RTX 2080 Ti）都已验证） |

构建方式：`powershell -File build.ps1`（默认 `-RuntimeModel 310.9.16`，产物在 `out\build-310.9.16`、`dist\Release-310.9.16`；`-RuntimeModel 310.9` 构建备选）。运行库是 NVIDIA 的材料，不随源码分发，
用 `-Runtime310916 <路径>` / `-Runtime3109 <路径>`（直接配置 CMake 时是 `-DDLSSG_RUNTIME_310916=<路径>` / `-DDLSSG_RUNTIME_3109=<路径>`）交给构建。
**游戏自己带的是 310.9.1 时**：出厂 `[Runtime] Mode=Bundled` 下代理把游戏对 `nvngx_dlssg.dll` 的请求换成随包的 310.9.16，游戏自己那份根本不加载，所以与游戏带哪个版本无关。`Mode=Auto` 在 0.4.0 删除（`docs/COMPAT.md` C86）：它只在游戏那份恰好就是随包运行库时才装得进去，其余情况游戏保留自己的 DLSS-G（RTX 30 / 20 上就没有帧生成）；旧 INI 里的 `Auto` 现在按 `Bundled` 运行，并记一条 `configuration_warning`。`Mode=Pinned` / `External` 指向 310.9.1 的文件认不出，这次加载被拒绝（`pinned_runtime_rejected` / `external_runtime_rejected` 会写明“它是 310.9.1.0”）。要跑 310.9.1，用 `-RuntimeModel 310.9` 的构建（它的 310.9.1 钩子表、内核包与路径和改默认之前逐字节相同）。出厂 INI 里的
`MaxGeneratedFrames=3`（4×）直接生效；想要 6× 要**自己把它改成 5**（见下）。

## 代理只放一个（`version.dll`，不行再按顺序换）

发布包根目录只有一个代理 `version.dll`，绝大多数游戏都会加载它。**任何时候 EXE 旁只放一个本项目的代理。** 游戏不加载 `version.dll` 时，退出游戏、**先删掉它**，再从 `alternatives\` 按下面的固定顺序一次换一个，每换一个启动一次游戏：

1. `version.dll`（根目录，首选）
2. `alternatives\winmm.dll`
3. `alternatives\dbghelp.dll`
4. `alternatives\dinput8.dll`
5. `alternatives\dxgi.dll` 或 `alternatives\d3d12.dll`：渲染路径代理，在 D3D12 热路径上、加载顺序敏感，只作最后手段，两者不要同时放（Vulkan 游戏不加载这两个）

前四个是工具类 DLL，不在渲染热路径上。六个名字是同一份加载器（都从同一个 `dlssg_sm86\backend\` 加载同一个后端），区别只有文件名和转发目标，换名字不改变行为；挑选说明也随包放在 `alternatives\README.md`。

怎么判断代理被加载了：把 `[Logging] Level` 临时改成 `2`，进游戏打开帧生成。日志 `dlssg_sm86\logs\dlssg_sm86.log` 里这个游戏 EXE 名的行中有 `runtime_redirect` 即已生效，`configuration` 记录里的 `proxies` 写明是哪个代理在干活，例如 `{"active":"version.dll","standby":[]}`。`Level=2` 下开了帧生成，日志里也没有这个 EXE 名的行，最常见的原因就是这个代理没被游戏加载，换下一个。

同一个目录里万一有好几个本项目的代理，也不会装出两份帧生成：

- 进程里第一个被加载的那个代理（DllMain 先跑的）拿到一个进程内的命名标记，成为 **active**：它挂 `LoadLibrary` 钩子、读 INI、加载后端。
- 之后加载的同族代理成为 **standby**：照常解析并转发自己那份系统 DLL 的全部导出（转发是完整的），但**不装钩子、不读 INI、不写日志**。`configuration.proxies` 形如 `{"active":"version.dll","standby":["winmm.dll"]}`。
- 谁先被加载由游戏的加载顺序决定，不由文件名决定，所以不要这样放，只留一个。完整解析 INI 时看得见 standby，就记一条 Level 1 `proxy_multiple{active,standby}`（2026-10-20 起；此前只在 Level 2 的 `configuration.proxies` 里），行为不变。

**从四代理的版本升级**：0.3.1 到 0.3.6 让用户把 `version.dll`、`winmm.dll`、`dbghelp.dll`、`dinput8.dll` 四个都放进游戏目录。升级到本版时把旧的 `winmm.dll`、`dbghelp.dll`、`dinput8.dll` 删掉：它们若先被游戏加载，跑的是那个旧版本自己内嵌的运行库与后端，新的 `version.dll` 只做转发（`docs/COMPAT.md` C80）。

## INI 放在哪里、何时生效

配置文件固定命名为 `dlssg_sm86.ini`，放在代理 DLL 所在目录（= 渲染 EXE 目录）。**修改后完全退出并重新启动游戏**，当前没有热重载。

- 布尔开关填写 `0` 或 `1`。以 `;` 开头的行为注释。
- `Mode` 等枚举值不区分大小写。
- 日志目录等路径可以填写绝对路径；相对路径以代理 DLL / INI 所在目录为基准（**不是进程工作目录**）。自定义路径不会展开 `%LOCALAPPDATA%`、`%TEMP%` 等环境变量，请填写实际路径（唯一的例外是 `Mode=External` 下的 `[Runtime] Path`，那一个会先展开）。`[Runtime] CacheDirectory` 2026-10-21 删除（早就没有缓存了），不读。
- **某个键的值写错（非法数字、枚举拼错、越界、抓取起点键写了认不出的键名）不会被换成默认值，也不钳位**（2026-10-19，`docs/COMPAT.md` C97）：日志记一条 Level 1 的 `configuration_invalid{section,key,value,allowed,reason,effect}`，**这个键所属的功能整组不启用**：
  - `[General]`、`[FrameGeneration]`、`[DLSSG]`、`[XeSSFG]`、`[Compatibility]`、`[Runtime]`（`SRMode` / `SRPath` 除外）→ 帧生成不启用，与 `[General] Enabled=0` 相同：游戏对 `nvngx_dlssg.dll` 的请求原样转发，`configuration.enabled` 为 `false`，`configuration.invalid` 是写错的键数。
  - `[SuperResolution]` 与 `[Runtime] SRMode` / `SRPath` → 超分不启用，与 `Mode=Off` 相同。
  - `[Logging]` / `[Debug]` → 只关那一项诊断（`FrameTiming`、`CrashRecord`、`MfgProbe` 各自关；任何 `Capture*` 键关帧生成抓取，任何 `CaptureSR*` 键关超分抓取，`Marker*` 关生成帧标记）。例外：`[Logging] Level` / `File` / `Directory` / `EvaluateEvery` 保留默认值，因为记这条记录的就是它们管的那份日志。

  例如 `MaxGeneratedFrames=99`、`[DLSSG] Optimized=7`、`Router=typo` 都让帧生成不启用（以前分别被换成 `3`、档位 2、`Auto`）；`[Logging] EvaluateEvery=0` 记一条 `configuration_invalid`、按默认 120 记（以前静默按 120）；`[Debug] MarkerScale=0` 记一条、标记关。`SpoofArchToGame` / `SpoofArchValue` 的架构改写由启动时那次早读决定，别的键写错让帧生成不启用时它不变（见 `[Compatibility]` 表）。
- 只记 `configuration_warning`（Level 2）、不关任何东西的：旧键名 `[Compatibility] ImageApprox`（`(honoured)` / `(ignored)`）、`[DLSSG] Optimized` / `Preset` 不再读取的旧位置（与新键同写或写的是默认值时，`(no effect)`）、`[DLSSG] Preset=Auto`（按 `A` 运行）、在当前配置下不起作用的键、另一个引擎的键（`ignored under Engine=…`）、`[Runtime] Mode=Auto`（按 `Bundled` 运行）。已删除的键不读：写了与没写相同，不记录（2026-10-26，C113）。
- `configuration_error`（整个 mod 关闭、保留游戏原始的 DLSSG 加载）只留给**整份配置根本读不了**的情况（内存不足、`[Logging] Directory` / `[Debug] CaptureDirectory` / `[Runtime] Path` 写成了无法构成路径的字符串）。值写错走的是 `configuration_invalid`。
- 后端只有一个：代理旁边的 `dlssg_sm86\backend\sm86_backend.dll`（2026-10-22 起，`[Backends]` 删除，`docs/COMPAT.md` C102）。安装前只核对一次：`DlssgMod_GetInfo`（ABI 与它服务的运行库 SHA-256）和本发布的导出是否齐全，不符记 `backend_unusable`，这次加载被拒绝、帧生成不启用；之后不再逐项协商。
- **开了做不到就拒绝，不换路径**（2026-10-20，`docs/COMPAT.md` C99）。两个最可能碰到的：**在 310.9.16 包（发布包）上写 `[FrameGeneration] CatmullRomWarp=1`，帧生成不启用**——包里没有这一行，日志记一条错误级 `detail_restore_row_unknown`，不会"照常插帧、只是这个键不生效"；**`[SuperResolution] Mode` 开着而超分装不上**（随包的 `dlssg_sm86\sr\nvngx_dlss.dll` 缺失或不对、`SRPath` 指错、后端装不上路线）**时，游戏这次加载 `nvngx_dlss.dll` 会失败，游戏里的 DLSS 超分不可用**，而不是悄悄跑游戏自带的原厂 DLSS（日志错误级 `sr_runtime_unavailable` / `external_runtime_rejected` / `sr_unsupported`）。

## 常规使用：出厂 INI

出厂 `dlssg_sm86.ini` 的完整内容：

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

**两个插帧引擎（2026-10-14）。** 键按“对谁生效”分节：`[General]`、`[FrameGeneration]`（`Engine`、`MaxGeneratedFrames`）、
`[Logging]`、`[Runtime]`、`[SuperResolution]` 两个引擎都读；`[DLSSG]` 只有 `Engine=DLSS` 读，`[XeSSFG]` 只有
`Engine=XeSS` 读。另一个引擎的键写成了非出厂值，日志记一条 `configuration_warning`（`default` 是 `(ignored)`，
`reason` 是 `ignored under Engine=…`）；出厂值不告警，所以出厂 INI 在两个引擎下都是零告警。下面“高级 / 诊断键”里
`[FrameGeneration]` 的 `CatmullRomWarp` / `ImageApprox*`
与 `[Compatibility]` 的内核类键（`KernelImage`、`Router`、`DisableFusions`、`ImagePatches`、`ImageApprox`）
**只对 DLSS 引擎生效**；`[Compatibility]` 的
进程类键（`SpoofArchToGame`、`SpoofArchValue`、`ForceSM86Route`、`SimulateAmpere`）两个引擎都需要。`Engine=XeSS` 用本项目
自己的运行器（CUDA 内核，经 NVAPI 录在游戏的命令列表上；2026-10-22 起只有这一种，`[XeSSFG] Runner` 删除）跑 Intel XeSS-FG 模型（model 15，到 4X，出厂的 `MaxGeneratedFrames=3` 直接可用；2X 是 model 18 的图，见下），着色器与权重在 `dlssg_sm86\xefg\xefg_kernels.dll`；不带、也不加载 Intel 的
`libxess_fg.dll`。CreateFeature 仍由 NVIDIA 运行库完成，帧节奏与呈现仍归 Streamline，只换 Evaluate。XeSS 跑不了时**不改跑 DLSS**
（`docs/COMPAT.md` C94）：`xefg_kernels.dll` 缺失或损坏（含其中的 CUDA 镜像）在安装时记一条错误级 `xefg_unavailable`，帧生成被拒绝（插帧关）；CreateFeature 时就能判定的——游戏声明的后缓冲格式不是 R8G8B8A8 / B8G8R8A8 /
R10G10B10A2（如 HDR 常用的 RGBA16F）、CUDA 对象建不出来——记一条错误级 `xefg_feature_refused`，该特性被拒绝（插帧关；`remedy` 只在原因有补救时才写：声明的格式不对 → 换格式或 `Engine=DLSS`；D3D12 设备在 XeSS-FG 创建之前已被移除（`hresult` 为 `0x887a0005` / `0x887a0006` / `0x887a0007`，另记 `device_removed_reason`）→ 写明这不是配置问题；其它失败只写 `call` 与 `hresult`，C107）；
特性建成之后才出现的（缺输入、跑不了的格式或矩形、对象 / 录制 / 启动失败）记一条错误级 `xefg_kernel_fatal` 写明原因，进程以
`0xE0D55001` 结束（C93 的同一个码）。`MaxGeneratedFrames` 是要求而不是上限（见下表）：`Engine=XeSS` 时写了 4 以上，
加载器记一条 Level 1 的 `max_generated_frames_unsupported{engine:"xess"}`，帧生成不启用（与 `Enabled=0` 相同），不再钳到 2X（`0` = 按游戏自己的帧数，也接受）。
HUD-less 画面按 `[XeSSFG] UseHudless`：出厂 `Auto` = 与 `0` 相同（2026-11-10 起，C139）：后缓冲作为网络输入，整幅画面（含 UI）
一起插值，游戏给不给 HUD-less 画面都一样，因此不会再出现 `xefg_hudless_unused`；`1` = 严格要求，哪一帧没有 HUD-less 画面、
或它 / UI 平面与后缓冲尺寸不同，就是 `xefg_kernel_fatal{kind:"hudless"}`，进程以 `0xE0D55001` 结束（C94）；`0` = 从不用（与 `Auto` 相同）。

**模型（2026-10-20 到 10-26 是 `[XeSSFG] Model` 键，C100 / C105；2026-10-26 起只有一个模型、键删除，C111）**：XeSS-FG 跑的是 MFG-Unlocker 1.6.0（dashdogy，MIT）移植的 Intel model 15（多帧模型），上限 3（`MaxGeneratedFrames` 1–3，4X）；旧 INI 里留下的 `Model` 不读（C113）。**2X 仍是 model 18**——提供者本身就是这样：它只在多于一张生成帧时把 2X 模型换成移植的模型。3X / 4X 走 model 15 自己的多帧路径：每个真实帧跑一次光流（D01–D28），每个序号 i 在 t = i / (N + 1) 跑一次合成（D29–D44）；它多读上一帧的深度（开 HUD-less 时还有上一帧的后缓冲），历史是它自己的两组槽，2X 帧也照样更新：深度（开 HUD-less 时还有后缓冲）照拷，网络源直接读 model 18 自己的历史（只读；下一个多帧真实帧先核对它正是那一帧，不符就 reset），所以 2X 只多一到两次拷贝（RTX 3070 离线 2X 比原来的 `Model=18` 慢 4–5 %，5070 游戏内 +1.7 %，输出逐位相同）。倍率在游戏里切换时：2X → 3X / 4X 接着历史，3X / 4X 之后的第一个 2X 帧是一次 reset（model 18 的历史没见过那一帧）。与 model 18 共用的着色器（D01、部分主干层、整个解码与合成）跑 model 18 的 CUDA 内核，移植版自己的主干层、D02、D28、D30 跑它自己的内核，D31 与几个小派发（D03、D13–D17、D27、D29）在同一条命令列表上跑 DXIL（量过：DXIL 更快；D31 的翻译在离线 31 组里 30 组逐位一致，但慢 1.8–1.9 倍），D01 复用上一真实帧的亮度金字塔（model 18 的做法）。model 15 的程序、权重与 CUDA 镜像**随包**，在同一个 `dlssg_sm86\xefg\xefg_kernels.dll` 里（2026-10-21 起，资源 `MANIFEST15`；此前是不随包的 `xefg_m15.dll`）：缺失 / 不符 = 安装时 `xefg_unavailable`（`reason` 以 `model15_` 开头），帧生成被拒，不会改跑 model 18 或 DLSS。开销（RTX 3070，离线完整路线回放，`Middle`、DIRECT，每个真实帧连同生成帧的 GPU 毫秒）：4X 1440p 4.5–5.0（DLSS 默认 5.3–5.4），4K 10.3–10.8（DLSS 默认 7.9–8.5）；离线 GPU 时间不等于游戏帧率。游戏给的是带抖动的运动矢量（`DLSSG.MvecJittered=1`）时，提供者在 3X / 4X 拒绝这个配置（没有对应程序）；本引擎把 model 18 处理这个标志的那一项（候选位置的两个扭曲方向都加 (jp − jc)·(1 − t)，jc / jp 是本帧 / 上一帧的抖动）移植到 model 15 的 D31，在各序号自己的 t 上跑：这时 D31 改跑带这一项的 CUDA 内核，每个生成帧 1440p 约 +0.08 ms、4K 约 +0.17 ms，R10G10B10A2 后缓冲不变慢（RTX 3070 离线），其余不变；不带抖动时输出与之前逐字节相同（C134）。RTX 3070 离线：完整 NGX 路径对提供者（`libxess_fg.dll` 1.3.1.78 + 移植，离线运行）每个序号逐字节相同。许可：其中 46 个程序由 Intel 的 XeSS SDK 程序改来，归 Intel XeSS SDK 条款（见 `THIRD_PARTY_NOTICES.txt`）。

**RTX 50 上的内核与路线（2026-10-28，`docs/COMPAT.md` C132、C133）**：XeSS-FG 的大部分 dispatch 跑本项目的 CUDA 内核，少数跑着色器自己的 DXIL（包里的路线表，按 3070 实测定）。RTX 40 / 50 没有本项目的 cubin，跑包里共用的 PTX（驱动编译）；这份 PTX 是 compute_75，只有 Turing 的 INT8 张量核心指令形状，Blackwell 把每条当作一半空着的 m16n8k16 来跑，INT8 张量核心内核在那里只有一半速度。所以在 Blackwell（SM ≥ 10.0）上，26 个 INT8 张量核心内核改用包里专为它准备的一份镜像（`<内核>.bw`：compute_86 PTX，3×3 卷积相邻两个抽头合成一条 m16n8k32），其余内核照旧用共用的 PTX；D25 与 model 15 的 D02、D30 在那里跑 DXIL（比 CUDA 快）。两条路线、两份镜像的输出逐字节相同，只影响速度：RTX 5070 合成场景 3440×1440，与此前（C132：共用 PTX，D26、D31、D33..D38 跑 DXIL）同一进程交替比较，4X 每个真实帧的插帧 GPU 时间 5.68 → 4.31 ms，2X 每张生成帧 2.77 → 1.89 ms；赛博朋克 2077 3440×1440 游戏内（与 C132 那时的数字比）4X 每个真实帧 5.81 → 4.68 ms、真实帧率 48.5 → 52.0 fps，2X 2.43 → 1.69 ms。日志 `xefg_cuda.blackwell_images` 是这个模型从 `.bw` 镜像建的内核数（RTX 50 上 2X 模型 21、多帧模型 26，其它显卡 0），`route_profile` 在 RTX 50 上是 `blackwell`、其它显卡是 `pack`。RTX 40 照旧跑共用的 PTX，RTX 20 / 30 照旧跑 cubin，都不变。

只有一个决定“怎么跑帧生成”的开关：`[DLSSG] Optimized`（2026-10-14 之前是 `[FrameGeneration] Optimized`，旧位置 2026-10-22 起不再读取）。它是一个**一致性档位**，判据只有一条：
**允许生成的画面偏离官方 NVIDIA 运行库多远**。档位越高越快、离官方画面越远；其余所有旋钮都由档位决定，
普通用户不需要逐个理解。

| 档位 | 设置 | 一致性保证 | 这一档打开了什么 |
|---|---|---|---|
| **0 原厂（stock）** | `Optimized=0` | **与官方运行库逐位一致** | 什么加速都不做。**注意这不是“不替换内核”**：SM86 路线仍然生效，后端替换进去的是从运行库里提取出来的**原厂内核镜像**（真 SM86 卡用 cubin，其它架构用 PTX），数值与运行库完全一致。它不等于 `KernelImage=Original`——在 **Ampere** 上尤其重要：310.9.1 的原厂图像内核是 sm_89 blob，在 Ampere 上根本创建不出来，所以 Ampere 上的“原厂数值”只能靠这条路径。 |
| **1 逐位一致（参照档；默认，推荐）** | `Optimized=1` | **与官方运行库逐位一致**（两份真实游戏抓取、320 张图实测）。要和官方画面逐位对照、或判断某个画面现象是不是本项目引入的，用这一档 | 全部**可证明不改变输出**的加速：34 个重写内核变体 + 全部跨内核合并、精确图像内核补丁（P1/P3/P4/P7/P8/P9/P10）。等价于优化集全开 + `ImagePatches=1, DisableFusions=0`。 |
| **2 快（有损）** | `Optimized=2` | **不再逐位一致**：真实游戏 4K 4X 抓取上对官方数值 PSNR 44–53 dB，运动越快越低（RTX 3070 上 Resonance 最快的那段最差 44.06 dB）；对真值只比档位 1 低 0.02–0.13 dB，两者差得最多的区块并排看不出差别（见下面“档位 2 / 3”一节） | 档位 1 + 有损图像内核行（目前只有一行：A6，`Kernel_OutputPull` 的 warp 取样改为最近点）。每生成帧比档位 1 快 1.8 %（1080p）/ 1.8 %（1440p）/ 2.9 %（4K）（RTX 3070，4X，0.4.0；之前是 2.6 / 2.9 / 4.5 %——0.4.0 的 P8/P9/P10 让档位 1 自己快得更多）。 |
| **3 最快（有损）** | `Optimized=3` | **不再逐位一致**，画质代价最大 | 全部仍然更快的有损加速：每一行有损图像内核。慢的或有结构性伪影的变体已永久退役，任何档位都不会启用。 |

档位是**累加且单调**的：更高的档位只会再加东西，不会撤回低档位的加速。想逐个覆盖某一项，用下面“高级 / 诊断键”
里的 `ImagePatches` / `ImageApprox` / `ImageApproxMask`——它们在档位
之后生效，可以把档位打开的东西再关掉，也可以在低档位上单独打开某一项。

其余出厂键：

| 配置项 | 节 | 默认值 | 说明 |
|---|---|---|---|
| `Enabled` | `[General]` | `1` | `1` 启用 DLSSG 重定向、适配和帧生成；`0` 关闭：游戏原样加载它自己的 DLSSG（在 Ampere 上就意味着没有帧生成）。代理仍转发系统 DLL 的原有导出。 |
| `Engine` | `[FrameGeneration]` | `DLSS` | 插帧引擎：`DLSS` = NVIDIA DLSS-G 模型（本项目的 SM86 / SM75 内核）；`XeSS` = Intel XeSS-FG 模型（本项目自己的运行器，model 15，2×–4×，见上）。写 `XeSS` 时 `configuration` 记录多一个 `engine{requested,effective,xessfg}` 字段（写 `DLSS` 或不写时没有），加载器记 `xefg_requested`，后端记 `xefg_engine` / `xefg_context` / `xefg_summary`（或 `xefg_unavailable` / `xefg_feature_refused` / `xefg_kernel_fatal`）。拼错是 `configuration_invalid`，帧生成不启用。 |
| `Optimized` | `[DLSSG]` | `1` | 一致性档位 `0`–`3`，见上表（出厂 `1`；2026-10-04 到 10-21 出厂是 `2`，写着 `Optimized=2` 的旧 INI 照旧是档位 2）。不写时是档位 0。**旧位置** `[FrameGeneration] Optimized`（2026-10-14 之前）与更早的 `[Compatibility] OptimizedKernels` 2026-10-22 起不再读取：与 `[DLSSG] Optimized` 同写、或写的是 `0`（新键不写时本来就是档位 0）时各记一条 `configuration_warning`（`(no effect)`）；单独写了别的值是 `configuration_invalid`（`allowed` 指向 `[DLSSG] Optimized`），帧生成不启用，不会悄悄按档位 0 跑。**值写错或超范围（如 `7`）是 `configuration_invalid`，帧生成不启用**（以前回退到档位 2）。只对 `Engine=DLSS` 生效。 |
| `MaxGeneratedFrames` | `[FrameGeneration]` | `3` | 每个真实帧生成几帧，`0`–`5`：`5`=6×，`3`=4×，`2`=3×，`1`=2×，`0`=保留运行库与游戏自己的帧数。**这是要求、不是上限，优先级最高**（2026-10-19）：运行库答给游戏的 `DLSSG.MultiFrameCountMax` 就是这个数，游戏菜单仍可以选更低的倍率；驱动 profile 里的帧数限制（私有键 `0x104C9A99`）压不低它——后端钩住运行库的 DRS 读取、去掉这个键（`mfg_drs_override`，这个键真的被设过时是 Level 1）；钩点未知的运行库构建（310.9.0）上，profile 的限制低于它时帧生成被拒绝（`max_generated_frames_unsupported{reason:"driver_profile_limit"}`）。做不到时一律报错、不降：`5` 以上是 `configuration_invalid`；`Engine=XeSS` 时 `4` 以上记 `max_generated_frames_unsupported{engine:"xess"}`，帧生成不启用；RTX 40 跑运行库自己的内核（现在只有 `KernelImage=Original`）时 `2` 以上记 `max_generated_frames_unsupported{reason:"runtime_kernels"}`，帧生成被拒绝；游戏自带代 A 运行库（≤ 3.8.x、旧 Streamline 插件只驱动 2×）时不论写几帧生成都不启用（见“游戏自带的运行库是哪一代”）。出厂值是 `3`（4×）：自带 Dynamic MFG 的游戏默认会直接跑到这里给出的数，`5` 对多数人偏高（公开 issue #497/#499），所以 6× 改成**按需手动开**——把这一行改成 `5` 即可。自带旧版 4X Streamline 插件（代 B）的游戏无论写几都只有 4×（插件自己的常量）。 |
| `Preset` | `[DLSSG]` | `A` | DLSSG 渲染预设，**每个特性都钉住：INI 写什么就跑什么**，游戏的 `DLSSG.UserInterfaceRecompositionEnabled` / `DLSSG.LiteCandidateBlendingEnabled` 与驱动预设 `0x10E41DF1` 都改不了它（2026-10-22，`docs/COMPAT.md` C102）。`A`（默认，不写也是 `A`）= 关 UI 重组、关 FG Lite；`B` = 开 UI 重组（生成帧里 HUD/UI 更干净）——**但 B 只在游戏同时向 DLSS-G 交出 HUD-less 图和 UI 平面时才生效**，多数游戏不提供、此时与 A 相同；这种情况日志记一条 Level 1 的 `preset_ineffective`（每个特性句柄只在第一次 Evaluate 查一次，写明缺哪张平面），见 COMPAT C87；`C` = FG Lite（只有 310.9.16：每生成帧 GPU 时间 −32…−54 %，画质略低）。`Auto`（2026-10-22 之前的出厂值，让游戏 / 驱动决定）已删除：写了按 `A` 运行并记一条 `configuration_warning`。`[DLSSG] Preset=`（写了空值）算写错：`configuration_invalid`，帧生成不启用。旧位置 `[Compatibility] Preset` 不再读取，规则与 `Optimized` 的旧位置相同：与新键同写、或写的是 `A` / `Auto` 只告警，单独写 `B` / `C` 是 `configuration_invalid`。只对 `Engine=DLSS` 生效。 |
| `FlowResolution` | `[XeSSFG]` | `Middle` | XeSS-FG 光流分辨率：`Coarse` / `Middle` / `Fine`（XeSS-FG 运行库的 `0x410` / `0x408` / `0x208`），越细越慢；`CoarseAt4K`（可选，业主 10-17）= 输出像素数 ≥ 3840 × 2160（8 294 400，按总像素而不是高度：5120 × 2160、3840 × 2400 算，2880 × 2160 不算）的特性用 `Coarse`，其余用 `Middle`，每个特性在 CreateFeature 时按游戏声明的输出尺寸定一次，记一条 `xefg_flow_choice`（RTX 3070 上 4K `Coarse` 比 `Middle` 每生成帧少约 0.45–0.52 ms，画质见 `docs/COMPAT.md` C95）。只对 `Engine=XeSS` 生效；值在两个引擎下都检查，拼错是 `configuration_invalid`，帧生成不启用。 |
| `UseHudless` | `[XeSSFG]` | `Auto` | `Auto`（默认）= 与 `0` 相同（2026-11-10 起，C139）：后缓冲做网络输入，整幅画面（含 UI）一起插值，不管游戏给不给 HUD-less 画面；`1` = 严格：没有可用的无 HUD 画面的帧是 `xefg_kernel_fatal{kind:"hudless"}`、进程结束（`0xE0D55001`）；`0` = 不用（与 `Auto` 相同）。2026-10-19 之前取值是 `0/1`、出厂 `1`；10-19 到 11-10 之间 `Auto` 是逐帧自动。只对 `Engine=XeSS` 生效；拼错同上。 |
| `Level` | `[Logging]` | `1`（不写时是 `2`） | `0` 关闭日志；`1` 只记错误；`2` 增加配置、加载和能力信息；`3` 再增加内核创建、Evaluate 和标记信息。所有模块、所有进程都追加写同一个文件 `<Directory>\dlssg_sm86.log`（默认 `dlssg_sm86\logs\dlssg_sm86.log`）；每行先是本地时间、`pid N`、EXE 名和级别字母（E = Level 1 / I = 2 / D = 3），再是 JSON 记录，例如 `2026-10-22 14:03:11.482  pid 12345  Cyberpunk2077.exe  E  {"event":...}`；只看一个游戏时按 EXE 名或 pid 过滤（`Select-String` / `findstr`）。代理在每个进程的第一条记录前写一条 `log_session`（exe、pid、代理路径与 SHA-256、版本）。文件不会自己清空，要从头开始就删掉它。 |
| `Directory` | `[Logging]` | `dlssg_sm86\logs` | 日志目录，相对路径以 INI 目录为基准。 |
| `Mode` | `[Runtime]` | `Bundled` | 运行库来源。`Bundled` = 随包的 `dlssg_sm86\runtime\nvngx_dlssg.dll`（普通安装用这个，无须版本匹配）。`External`（`[Runtime] Path` 指定另一个位置，默认值就是上面那个文件）/ `Pinned` 是高级用法，见下。`Auto` 在 0.4.0 删除：写了按 `Bundled` 运行并记一条 `configuration_warning`。 |
| `Mode` | `[SuperResolution]` | `Off` | DLSS 超分，只在 RTX 30 / RTX 20 上生效。`Off` = 不介入，与没有这一节完全相同；`M-INT8` / `L-INT8` = 所有画质档都钉预设 M / L、跑本项目的 INT8 内核（档位在游戏里选；画质 / 平衡 / DLAA 档比游戏自己的 K 慢）；`M-FP16` / `L-FP16` = 同一预设、NVIDIA 自己的内核。别的值超分不启用。见下面 `[SuperResolution]` 一节。 |

一般安装保留出厂 INI 不动即可（档位 1，画面与官方运行库逐位一致）。想再快一点、接受不逐位一致把 `Optimized` 改成
`2`，想对比原厂数值改成 `0`，其它一律不用碰。从老版本升级时保留了旧 INI 的，仍是旧 INI 里写的档位
（0.4.0 出厂是 `Optimized=2`；**2026-10-14 之前的出厂 INI 把它写在 `[FrameGeneration]`、把 `Preset` 写在 `[Compatibility]`，
这两个旧位置 2026-10-22 起不再读取**：单独写着的 `[FrameGeneration] Optimized=1` 是 `configuration_invalid`、帧生成不启用，换成新的出厂 INI 或把这两行搬进 `[DLSSG]`）。

## RTX 40 / RTX 50（0.4.0 起）

0.4.0 起本项目的帧生成内核**也在 RTX 40（sm_89）和 RTX 50（sm_120）上运行**，档位照样由 `Optimized` 决定（出厂档位 1）。
此前这两代卡上运行库只用它自己的内核。只认计算能力 8.9 / 12.0、CUDA 显卡名带 `RTX` 的卡（包括 RTX 6000 Ada、RTX PRO 6000
Blackwell 这类工作站卡；名字不带 RTX 的数据中心卡不受影响）。2026-10-19 起这不再是可关的选项：`[Compatibility] AccelerateNewerGpus`
已删除（不读），帧生成开不开只由 `[General] Enabled` 决定，开着就跑本项目的内核。
唯一还能让运行库自己的内核在这两代卡上跑的是诊断值 `KernelImage=Original`（RTX 40 上它只能 2X：`MaxGeneratedFrames` 大于 1 时帧生成被拒绝，见上表）。

- **速度还没有在 RTX 40 / 50 实机上量过。** 开发机只有 RTX 3070：这条路线在 3070 上用开发开关完整跑过（同一份 PTX，输出与
  RTX 30 路线逐位相同）；2026-09 在 RTX 5070 上，这套内核（当时的版本，之后的改动只在 RTX 3070 上验证过逐位一致）与官方输出
  逐位一致，但对官方内核的耗时差一次都没有量过。
  一键测试包在 RTX 40 / 50 上会自动多测一列"官方"（见测试包 README），量出来之前收益未知，也可能更慢。
- **镜像是 PTX**：包里 sm_86 族的 PTX 由驱动即时编译（包里没有 sm_89 / sm_120 cubin）。第一次创建帧生成特性要多编译一遍，
  RTX 5070 上实测冷启动整次 CreateFeature 约 3.0–3.3 秒（约 1.2 秒是 70 个基础内核——RTX 50 上官方运行库自己也要即时编译
  同样多的 PTX——约 1.7–1.8 秒是本项目当时的 63 个优化行，2026-10-21 起是 37 个），之后命中驱动的计算缓存（`%APPDATA%\NVIDIA\ComputeCache`，上限 1 GiB）
  只要约 0.13 秒；驱动更新或缓存被挤掉后会再编一次。RTX 40 上官方运行库用的是 cubin，不需要编译，所以这几秒全是新增的。
- **画面**：RTX 50 上档位 0 / 1 预期与官方逐位一致（档位 2 照旧有损）；**RTX 40 上档位 0 / 1 也不会与官方逐位一致**——官方在
  RTX 40 上跑的是另一套构建（为不做多帧生成的产品编的 Ada 版图像内核与 sm_89 cubin 网络），本项目跑的是 RTX 50 那一套的算法，
  两者在 RTX 5070 上实测相差 55–82 dB（差异集中在 UI 重组 / 候选混合路径）。
- **倍率**：运行库答给游戏的 `DLSSG.MultiFrameCountMax` 是 `MaxGeneratedFrames`，RTX 40 上也是（本项目的内核逐帧推进生成帧的
  相位，`docs/COMPAT.md` C92）；游戏菜单实际给几档由游戏与它的 Streamline 插件决定，RTX 40 实机上没有验证过。
- **不变的**：不改对游戏、Streamline、NGX 核心的任何架构答复（`SpoofArchToGame`
  在这两代卡上照旧什么都不改写；也不像 RTX 30 那样改写运行库对 NGX 核心声明的最低架构）。
- **出错时**（2026-10-16 起，所有显卡一样）：驱动拒绝本项目的任何一个镜像（PTX 编译或创建失败）时**游戏直接崩溃**，日志最后是
  一条错误级 `fg_kernel_fatal`（哪个内核、什么原因）和一条 `fg_gate_crash`（异常码 `0xE0D55001`）——不会悄悄改用运行库自己的内核。
  内核包 `dlssg_sm86\kernels\fg_kernels.dll` 缺失或不配套时帧生成**打不开**（`fg_kernels_unavailable`），也不会改用运行库自己的
  内核（`docs/COMPAT.md` C93）。
- 日志：后端在 RTX 30 / 20 以外的卡上记一条 `newer_gpu_route`（`engaged`、`reason`、`image`、`multi_frame_advertised` 等），
  `Level=2` 可见。

## INI 键的稳定性：三级

加载器读的 INI 键一共 62 个（`tests/golden/surface.json` 的 `ini_keys`，CTest `compat_surface` 逐个钉住，见
`docs/COMPAT.md` C4）。它们按对外的承诺分三级，另有四个旧名（2026-10-05 业主定；2026-10-14 两个插帧引擎
的布局又加了两个，C84；2026-10-22 起其中三个不再读取，C102）：

| 级 | 哪些键 | 承诺 |
|---|---|---|
| **1 冻结** | 出厂 INI 里的 11 个键（2026-10-10 起含 `[SuperResolution] Mode`；2026-10-14 起含 `[FrameGeneration] Engine` 与 `[DLSSG]`、`[XeSSFG]` 两节；`[XeSSFG] Model` 2026-10-23 到 10-26，C111 删除） | 节、键名和取值的含义都不变。真要改名，旧写法要么只记一条 `configuration_warning` 指向新写法（新键也写了，或写的是新键不写时本来就跑的值），要么是 `configuration_invalid`（照旧写法会跑出别的东西时），从不悄悄改跑别的：2026-10-14 `Optimized` / `Preset` 搬进 `[DLSSG]`，旧位置 2026-10-22 起就按这条处理。 |
| **2 保持兼容** | 写给玩家的 3 个键：`[Compatibility] SpoofArchToGame`、`[FrameGeneration] CatmullRomWarp`、`[Runtime] Path`（`[SuperResolution] Tier` 与 `[Compatibility] AccelerateNewerGpus` 原在这一级，2026-10-19 删除；`[XeSSFG] Runner` 也原在这一级，2026-10-22 删除） | 可以增加取值、改进行为，但不会悄悄失效：改名时旧写法要么继续生效，要么记一条 `configuration_warning` 说明替代写法。删除见下面。 |
| **3 诊断** | 其余全部：开发、验证、抓取、探针与测量用的键，内核格式与内核族、架构上报值、运行库路线的高级写法 | 版本之间可以改名、改义或删除。 |
| **旧名** | `[FrameGeneration] Optimized` 与 `[Compatibility] OptimizedKernels`（`[DLSSG] Optimized` 的旧位置 / 旧名）、`[Compatibility] Preset`（`[DLSSG] Preset` 的旧位置）、`[Compatibility] ImageApprox`（`[FrameGeneration] ImageApprox` 的旧位置） | `[Compatibility] ImageApprox` 仍按原意生效，写了就记一条 `configuration_warning` 指向新键（`(honoured)`；新位置也写了时 `(ignored)`）。另外三个 2026-10-22 起不再读取：与新键同写、或写的是新键不写时的值（`Optimized=0`、`Preset=A` / `Auto`）记一条 `configuration_warning`（`(no effect)`）；单独写了别的值是 `configuration_invalid`（`allowed` 写明改成哪个新键），帧生成不启用。 |

不论哪一级，删掉一个键都会让 `compat_surface` 失败；删掉它的改动要同时更新 `tests/golden/surface.json`，并在
`tests/golden/README.md` 写一行：删了什么、为什么、用什么代替。

**删除的键不读**（2026-10-26 起，业主，C113；此前写了记一条 `configuration_warning`）：写了与没写完全相同，不记录、不告警、不判无效。旧 INI 里可能还留着的：

- `[FrameGeneration]`：`SkipRepeatedRealCopy`（2026-10-08，见下面「`SkipRepeatedRealCopy`（已删除）」）、`ForceGeneratedFrames`（10-19）、`DetailRestore` 与 `DetailRestoreA` / `B` / `C` / `D` / `Alpha` / `Gate` / `CasMul` / `WeightProbe`（10-21；模式 A 现在是 `CatmullRomWarp`，两个值对调：旧的 `DetailRestoreA=2`（五抽头）= `CatmullRomWarp=1`）、`ForcePluginFrames`（10-22）。
- `[Compatibility]`：`AccelerateNewerGpus`（10-19）、`HardwareBilinear` / `LaunchChains` / `PlainVariant` / `ChainBlock0`（10-21）、`SM75Family`（10-22）。
- `[XeSSFG]`：`Runner`（10-22）、`Model`（10-26）。
- `[Runtime]`：`RRMode` / `RRPath`（10-09）、`CacheDirectory`（10-21）；`[RayReconstruction]` 整节（10-09）；`[Backends]` 整节（10-22）。
- `[SuperResolution]`：`Int8`、`Int8L`、`Int8M`、`ForcePreset*`（10-03）、`Tier`（10-19）。
- `[Logging] DebugOutput`（10-22）；`[Debug]`：`CaptureRR` / `CaptureFeature`（10-09）、`QueueTrace`（10-21）、`NgxLog` / `NgxLogLevel`、`CaptureInputs` / `CaptureOutputs` / `CaptureRingSlots` / `CaptureVramMB`、`CaptureSRFrames`（`CaptureSR` 的旧别名）/ `CaptureSRSkip` / `CaptureSRRingSlots` / `CaptureSRMemoryMB` / `CaptureSRVramMB` / `CaptureSRAsyncCopy`（10-22）。

删除的**值**照旧告警：`[DLSSG] Preset=Auto`（2026-10-22，按 `A` 运行）与 `[Runtime] Mode=Auto`（0.4.0，按 `Bundled` 运行）各记一条 `configuration_warning`；这是第 1 级键取值的改动，业主决定。2026-10-19 `[XeSSFG] UseHudless` 的 `1` 改成严格档（原来的行为改叫 `Auto`，出厂值随之改成 `Auto`），同样是业主决定；2026-11-10 起 `Auto` 改成与 `0` 相同（C139）。

| 节 | 第 1 级（11） | 第 2 级（3） | 第 3 级（44） | 旧名（4） |
|---|---|---|---|---|
| `[General]` | `Enabled` |  |  |  |
| `[DLSSG]` | `Optimized`、`Preset` |  |  |  |
| `[XeSSFG]` | `FlowResolution`、`UseHudless` |  |  |  |
| `[FrameGeneration]` | `Engine`、`MaxGeneratedFrames` | `CatmullRomWarp` | `ImageApprox`、`ImageApproxMask` | `Optimized`（`[DLSSG] Optimized` 的旧位置，不再读取） |
| `[Compatibility]` |  | `SpoofArchToGame` | `DisableFusions`、`ForceSM86Route`、`ImagePatches`、`KernelImage`、`Router`、`SimulateAmpere`、`SpoofArchValue` | `ImageApprox`（`[FrameGeneration] ImageApprox` 的旧位置）、`OptimizedKernels`（`Optimized` 的旧名，不再读取）、`Preset`（`[DLSSG] Preset` 的旧位置，不再读取） |
| `[Logging]` | `Level`、`Directory` |  | `CrashRecord`、`EvaluateEvery`、`File`、`FrameTiming` |  |
| `[Runtime]` | `Mode` | `Path` | `SRMode`、`SRPath` |  |
| `[SuperResolution]` | `Mode` |  | `Auxiliary`、`KernelImage`、`KernelSet`、`KernelSetSM`、`ObserveFrames` |  |
| `[Debug]` |  |  | `Capture`、`CaptureAsyncCopy`、`CaptureDirectory`、`CaptureLean`、`CaptureMemoryMB`、`CaptureNoStall`、`CaptureSR`、`CaptureSRBursts`、`CaptureSRDedup`、`CaptureSRDirectory`、`CaptureSRLean`、`CaptureSRMaxGB`、`CaptureSRNoStall`、`CaptureSRStartKey`、`CaptureSRStartOnCreate`、`CaptureSRStride`、`CaptureSkip`、`CaptureStartKey`、`CaptureStartOnReset`、`MarkGeneratedFrames`、`MarkerScale`、`MarkerX`、`MarkerY`、`MfgProbe` |  |


## 高级 / 诊断键（不在出厂 INI 里）

下面这些键**仍然被 loader 解析和支持**（开发、验证、监控、抓取、MFG 探针都要用），只是不在出厂 INI 里。**缺失时各自取安全 / 最佳默认值**，所以正常安装不需要写它们。要用时把对应的节和键补进 INI（节不存在就自己加一个 `[节名]`）。

> **注意：以下开关默认已经是最快 / 最安全的一组。** 打开它会**降低性能**，只用于诊断：`DisableFusions>0`。

### [FrameGeneration]

| 配置项 | 值 | 默认 | 说明 |
|---|---|---|---|
| `ImageApprox` | `0/1` | **档位决定**（档位 ≥2 为 `1`） | 有损图像内核行的显式开关，覆盖档位的答案：在档位 1 上写 `1` 就只加有损图像行，在档位 2/3 上写 `0` 就只留逐位一致的部分。档位 0 不建任何行，在那里写 `1` 不起作用（记一条 `configuration_warning`）。`[Compatibility] ImageApprox` 是它的旧位置：仍被读取，写了就记一条 `configuration_warning` 指向这个键。 |
| `ImageApproxMask` | 位掩码 | **档位决定**（档位 2 = 默认集，档位 3 = 全部） | 精确指定启用哪几行有损图像内核，每一位对应一个固定槽位。槽位**一次分配、永不回收**：退役的行留下空位，所以老文档里写下的掩码值永远指同一批行；掩码里出现未分配的位会被忽略并记一条 `image_approx_mask_unassigned`。同一个内核有多行时只启用位号最小的那一行，另一行记 `image_approx_row_shadowed`。armed 的行、槽位与资源号都在后端 `image_approx` 记录里。 |
| `CatmullRomWarp` | `0/1/2` | **`0`**（任何档位都不打开） | 把 `Kernel_BlendCandidatesFused` 里两次**运动 warp 取样**从取样器的双线性换成 **Catmull-Rom**。`1` = 丢掉四个角点再归一化的**五抽头**档（每候选 5 次硬件过滤取样）——实测的性价比档，也是唯一推荐的值；`2` = 精确的可分离 4×4（**九抽头**，每候选 9 次），能量并不更高而代价翻倍，留着是为了完整而不是为了推荐。**不引入任何判据**（纯重采样改动，没有"校验通过就怎样"这类分支），而且位移为 0 时 Catmull-Rom 退化成取中心纹素，所以车停着的画面按构造不变。5070 实测（赛博朋克 4K 抓取）：细结构能量比 `E_gen/E_real` 从 **0.897 抬到 0.953**，1080p / 1440p 每生成帧约 **+2.5 %**、4K 约 **+4 %**（K39 这一个内核逐 launch +20 %，九抽头是 +42 %）；**HUD 区块也变好**，但**肉眼能看出的改善很轻微**。按设计**偏离**官方输出（不再逐位一致），所以不进任何 `Optimized` 档位。**它需要 `Optimized` ≥1，且 `[Compatibility] ImagePatches` 不是 0**：它是一行图像补丁行（建在精确补丁 P7 的 PTX 上），档位 0 不建任何行，`ImagePatches=0` 关掉所有图像补丁行；这两种配置下写它什么都不做，loader 记一条 `configuration_warning`，后端的 `catmull_rom_warp` 记录写 `enabled:false` 与 `disabled_reason`。`0` 时输出与官方运行库逐位一致。内核包里没有它要的那一行时（**310.9.16 模型的包没有 CatmullRomWarp 行**）记一条错误级 `detail_restore_row_unknown`，帧生成被拒绝（2026-10-20 起；此前只是不生效）。 |

### [Logging]

| 配置项 | 值 | 默认 | 说明 |
|---|---|---|---|
| `File` | `0/1` | `1` | `1` 写日志文件；`0` 关闭文件输出（仍受 `Level` 控制）。 |
| `EvaluateEvery` | `1`–`1000000` | `120` | 正常 Evaluate / 标记日志的采样间隔（按 Evaluate 调用计数）。`1` 记录每次；`0` 或越界是 `configuration_invalid`，这个键保留默认 120（它管的就是记这条记录的日志）。Level 3 下前 12 次总是记录。 |
| `FrameTiming` | `0/1` | `0` | `1`：每个真实帧在 `Directory` 下的 `frame_timing.txt`（普通文本，每个进程追加一段）写一行——CPU 帧间隔、这一帧超分 Evaluate 的 GPU 时间、它全部帧生成 Evaluate 的 GPU 时间（D3D12 时间戳），每秒一行汇总（含 GPU 占用率），与 `Level` / `File` 无关。`0` 或不写：什么都不加载、不装钩子。值非法记一条 `configuration_invalid`，帧时间文件关。见下面「帧时间文件」。 |
| `CrashRecord` | `0/1` | `1` | 游戏崩溃（未处理的异常）时在 `dlssg_sm86.log` 里留一条 `fg_gate_crash`（崩溃进程的最后一行）：异常码、地址、出事的模块与偏移、线程，以及是不是我们的模块。只记录：装的是链式调用原处理器的 `SetUnhandledExceptionFilter`，崩溃本身、退出码、游戏自己的崩溃报告器都不受影响；不用 VEH。`0` 关掉（例如怀疑它与某个游戏的崩溃报告器冲突时）。只在后端装上之后才有（只开超分的进程没有）；值非法记一条 `configuration_invalid`，崩溃记录关。覆盖不到的情形见 `docs/COMPAT.md` C89。 |

#### 帧时间文件 `frame_timing.txt`（`[Logging] FrameTiming=1`）

写了 `FrameTiming=1` 的进程在第一次 DLSS Evaluate 之后往 `[Logging] Directory`（默认 `dlssg_sm86\logs`，与 `dlssg_sm86.log` 同一个目录，打包日志时会一起带上）里的**一个**普通文本文件 `frame_timing.txt` **追加**（2026-10-22 起；此前是每个进程一个 `frames_<pid>.jsonl`）：每个进程一段，以 `# ==== session` 一行开头（进程号、exe、开始时间、显卡），接着 INI 路径、时间戳频率、列说明与列名。只开超分、只开帧生成、两个都开都写；`[General] Enabled=0` 时也写，量的是游戏自己的 DLSS / DLSS-G，可以直接与开着本项目的一次做 A/B。Vulkan 游戏不计时，一行 `# vulkan` 说明。文件不会自己清空：要从头开始就删掉它；两个进程同时计时，它们的行会按 0.1 秒一块交错。

一个真实帧（游戏自己渲染的一帧）一行：开着帧生成时是一组 `MultiFrameIndex` 1..N 的帧生成 Evaluate，这一帧之前的超分 Evaluate 也算在它里面；不开帧生成时是一次超分 Evaluate。数据行以帧号开头、列宽固定，`-` 表示没有；其余行都以 `#` 开头：

| 行 | 何时 | 内容 |
|---|---|---|
| `# ==== session …` 及其后四行 | 每个进程的第一帧之前 | 进程号、exe、开始时间（UTC）、显卡名；INI；时间戳频率与列表类型；列说明；列名 |
| `# context from frame N: …` | 第一帧之前，之后每次变化 | 超分：渲染（输入）与输出分辨率（取自这次 Evaluate 自己的 `DLSS.Render.Subrect.Dimensions` / `Color` 与 `Output`，C108）、预设、`kernels`（`int8` = 本项目的 INT8 内核，`native` = 本项目的路线在、跑的是 NVIDIA 的内核，`game` = 游戏自己的 DLSS）；帧生成：倍数、生成帧的输出分辨率（`DLSSG.OutputInterpolated` 的子矩形或资源）、`route`（`ours` / `runtime`）、档位、内核镜像 |
| 数据行 | 每个真实帧 | `frame` 帧号、`t_ms` 距第一次计时的毫秒、`dt_ms` 与上一帧的 CPU 间隔（游戏的真实帧时间）、`sr_ms` 超分的 GPU 毫秒、`fg_ms` 这一帧全部帧生成 Evaluate 的 GPU 毫秒之和、`n` 生成帧数、`fgw_ms` 第一个帧生成 Evaluate 开始到最后一个结束的 GPU 跨度、`gdt_ms` 与上一帧在 GPU 时间线上的间隔、`src_ms` / `fgc_ms` 两种 EvaluateFeature 调用里的 CPU 毫秒；`flags`：`x:sr` / `x:fg` 该项时间戳迟到或丢了（值写 `-`）、`srn=N` 这一帧有 N 次超分 Evaluate、`orphan` 没见到 `MultiFrameIndex` 1 的一组 |
| `# second S …` | 每秒一行 | `frames`、`fps`，`dt` / `sr` / `fg` / `gdt` 各四个数：avg p50 p95 max（毫秒），`n`，`gpu` 与 `fb` 占用率 |
| `# end …` | 正常退出时这一段的最后一行 | 帧数与计数器；没有这一行就是进程被杀或崩溃（被杀时之前完成的帧都在：每 0.1 秒写一次盘，退出时把剩下的帧与这一秒的汇总写完） |

例（RTX 3070，`sm86_scene` 1080p 帧生成 4X、档位 1；中间省略）：

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

怎么读：

- `dt_ms` 是游戏的真实帧时间，`1000 / dt` 是真实帧率（显示帧率再乘帧生成倍数）。`gpu` 接近 100 % 是 GPU 瓶颈，这时 `sr` / `fg` 变小会直接变成帧率；`gpu` 明显低于 100 % 而帧率不高，是 CPU（或帧率限制）瓶颈，GPU 时间省下来也不会变成帧率。`gpu` 是整张卡、所有进程，NVAPI 每秒采一次（常用 overlay 显示的就是这个数）。
- `sr_ms` / `fg_ms` 是本项目在 NGX 核心的 EvaluateFeature 调用前后录进游戏命令列表的两个时间戳之差：运行库在这次 Evaluate 里录的全部 GPU 工作都在里面（帧生成还包括本项目替换、融合的内核）。不在里面：Streamline 在 Evaluate 之外的工作、同一组 Evaluate 之间的空隙（只在 `fgw_ms` 里）、呈现与节奏控制、游戏自己的渲染。同时跑在别的队列上的工作会让它变长，那是真实代价。
- 代价（RTX 3070 实测）：每次 Evaluate 的 GPU 时间多约 3–8 µs（两个时间戳加一次回读拷贝；4X 帧生成每真实帧多 13–20 µs，即 +0.3–0.5 %），超分每次 Evaluate 的 CPU 时间多约 7–13 µs。不改任何像素：开与不开，超分与帧生成的输出逐字节相同。
- 大小：一行约 100 字节，每秒汇总约 0.2 KB；60 fps 每小时约 22 MB。每个进程的数据行写到 64 MiB 为止（一行 `# cap` 说明），之后只写每秒汇总。

读：`python scripts\perf\frame_timing_report.py <frame_timing.txt> [<另一个.txt>]` 默认读每个文件的最后一段（`--all` 全部、`--pid N` 某个进程），给每段一个汇总（真实帧率，`dt` / `sr` / `fg` 的 avg / p50 / p95，GPU 占用率，各段上下文）；正好两段时再给 B 对 A 的差，毫秒与百分比都列。`--skip-seconds N` 去掉开头 N 秒（加载画面）。

### [Debug]

| 配置项 | 值 | 默认 | 说明 |
|---|---|---|---|
| `MarkGeneratedFrames` | `0/1` | `0` | `1` 在生成输出上绘制 `FG 1/3` 等实际序号标记，用于区分毛刺出现在生成帧还是真实帧。真实帧和 Reset 输出跳过。标记会改动生成帧像素，做全图数值比较时请关闭。 |
| `MarkerX` / `MarkerY` | `0`–`65535` | `8` / `8` | 标记左上角坐标（输出纹理像素）。 |
| `MarkerScale` | `1`–`8` | `2` | 标记缩放倍率；矩形为 `24×scale` 宽、`9×scale` 高。`Mark*` 这几个键任何一个写错（如 `MarkerScale=0`）都是 `configuration_invalid`，生成帧标记关。 |
| `Capture` | `0`–`100000` | `0` | 录多少次 Evaluate 的参数、输入和输出，供离线重放（GCR）。`0` 完全不装任何钩子，性能影响为零。数的是 Evaluate 次数（3× 每真实帧 2 次、6× 每真实帧 5 次）。1080p RGBA8 约 48 MB/次。完整机制见 `docs/CAPTURE.md`。 |
| `CaptureDirectory` | 路径 | `dlssg_sm86\capture` | 抓取目录，相对游戏 EXE 目录。 |
| `CaptureSkip` | `0`–`1e8` | `0` | 先跳过多少次 Evaluate（跳过加载画面 / 过场）。 |
| `CaptureStartOnReset` | `0/1/2` | `1` | 从哪一帧开始录。`2`：只等“刚创建完特性之后的第一次 Evaluate”，是重放能逐位一致的唯一起点；`1`：`DLSSG.Reset` 帧或新建特性的第一帧，先到先算；`0`：立刻开始。`1`/`2` 等满 3 分钟不到就什么都不录。 |
| `CaptureStartKey` | 键名 / VK 码 | `0` | 按键触发起点：`F9`（F1–F24）、单个字母或数字，或虚拟键码（`120` / `0x78`）。设了之后 `CaptureSkip` 满足后继续等，直到在游戏里按下这个键，再从那一刻应用 `CaptureStartOnReset` 的规则（`0` = 按下的那次 Evaluate 立刻开始录，`start_reason=key`；`1`/`2` = 从按键起再等 Reset / 新建特性，3 分钟超时从按键算起）。这是让抓取落在“人物正在运动”那一刻的办法；等待期间记 `capture_waiting_for_key`，按下记 `capture_start_key`。认不出的键名（以及任何 `Capture*` 键的非法值）是 `configuration_invalid`，这套抓取关，其余不受影响。 |
| `CaptureNoStall` | `0/1` | `0` | `1`：游戏线程永远不等。整份抓取缓冲在系统内存里：环和请求的次数一样深、回读缓冲由写盘线程在等待起点期间提前建好、录满之后文件在后台写完（`capture_finishing` → `capture_done`），所以录到的是连续的帧；组的内存按预热时学到的尺寸整组估算（第一组或没有东西在飞时总是放行，预算小于一组时退化成一次只录一组），预算不够时整组跳过（组 = 同一真实帧的全部生成帧，`MultiFrameIndex` 1..count），绝不在组中间丢平面或丢帧。`0`：4 槽环、写盘跟不上时游戏线程最多等 2 秒（4K 下游戏会降到几帧）。事件 `capture_prewarm` / `capture_group_skipped`，manifest `capture.taken_groups` / `skipped_groups` / `skipped_groups_memory`。重放时每个间隙后的第一组只用来暖运行库的上一帧历史，`capture_compare.py` 默认把它排除在汇总外。 |
| `CaptureMemoryMB` | `0`–`1048576` | `0` | 回读内存预算（池里的 + 在飞的），MiB；`0` = 安装时可用物理内存的一半。回读堆在系统内存，不占显存。 |
| `CaptureAsyncCopy` | `0/1` | `1` | `1`：把 PCIe 传输从渲染时间线上挪走。游戏自己的命令列表里只做显存→显存的 `CopyResource`（拷进一张暂存纹理），再由一条独立的 `COPY` 队列把暂存纹理搬进回读缓冲，和游戏渲染并行；`COPY` 队列先 `Wait` 游戏那次提交的 fence，`collect()` 等的是 `COPY` 队列自己的 fence。`0`：回读拷贝留在游戏的命令列表里（旧路径，3070 实测 4K 每生成帧 +122 %）。事件 `capture_plane_inline`（某个平面退回旧路径）、`capture_async_copy_unavailable`（这台机器建不出 `COPY` 队列/列表，全部退回旧路径）。 |
| `CaptureLean` | `0/1` | `0` | `1`：只回读不能重建的数据。同一组内 index>1 的 Evaluate 交出的输入资源和 index 1 完全相同（三份悟空抓取 54/54 个纹理平面逐字节一致），所以只记成对 index 1 文件的引用；`DLSSG.OutputReal` 是运行库对 `DLSSG.Backbuffer` 的拷贝（18/18），也记成引用。被引用的平面 json 里 `stored=false`、`source=<相对抓取根目录的文件>`，`sm86_replay` 与 `capture_compare.py` 会跟过去。4X 下每组字节约减半，回读的 PCIe 时间同步减少。16 字节的 buffer 参数总是照录。 |
| `MfgProbe` | `0/1/2` | `0` | `1` 记录每一次对帧数参数的 set 与 get，带调用方模块名、槽位、值和上下文，用来判断“生成几帧”是谁决定的；`2` 另加 Streamline 交互追踪（`mfg_state` 等）。复用抓取的参数存储钩子，但 `Capture=0` 时也能单独工作。事件与判定见 `docs/MFG_PROBE.md`。 |

#### DLSS 超分输入抓取（`docs/CAPTURE_SR.md`）

和上面那套帧生成抓取**完全独立**：它钩的是 NGX 核心而不是 DLSS-G 运行库，**在从来不加载 DLSS-G 的游戏里也能用**
（只要游戏开了 DLSS 超分或 DLAA，帧生成不必打开，显卡也不必是 Ampere），任何 DLSS 版本（2.x / 3.x / 310.x）、
游戏直接调 NGX 还是经 UE 插件 / Streamline `sl.dlss` 都一样。录的是 DLSS 超分（NGX feature 1）每帧的
颜色 / 深度 / 运动矢量 / 曝光输入和全部参数，给 INT8 超分工作做真实内容的标定与验证。`CaptureSR=0` 时整套机制不存在
（连后端都不会为它加载）。用**光线重建**（Ray Reconstruction，feature 13）的游戏不创建 feature 1，这时什么都录不到，
后端会记一条 `capture_sr_feature{stage:"other_feature"}` 说明原因。

**`[General] Enabled=0` 不会关掉它**：那个键关的是帧生成那套（替换内核的部分），而这套抓取不属于它。
`Enabled=0` + `CaptureSR=N` 就是“只采集输入”的部署：随包的 DLSS-G 运行库文件连打开都不打开、`nvngx_dlssg.dll` 原样交给系统
（没有 `runtime_redirect`）、不调 `DlssgMod_Install`（不检测 GPU、不替换内核、不做融合）、不改 NVAPI 架构上报、
不做 `turing_host`。loader 会记一条 `capture_sr_only` 把这份“没做什么”的清单写出来，`capture_sr_requested` 带
`mod_enabled` / `runtime_extracted` / `dlssg_untouched` 三个字段。CTest `capture_sr_disabled_mod` 钉住这条路。

| 键 | 取值 | 默认 | 说明 |
|---|---|---|---|
| `CaptureSR` | `0`–`4096` | `0` | **每段**录多少帧 DLSS 超分（超分每个真实帧只调一次 Evaluate）。`0` = 关闭。旧的同义别名 `CaptureSRFrames` 2026-10-22 删除，不读：改写成 `CaptureSR`。 |
| `CaptureSRDirectory` | 路径 | `dlssg_sm86\capture_sr` | 绝对路径，或相对于游戏 EXE 目录。 |
| `CaptureSRStride` | `1`–`100000` | `1` | 段内每 N 次 Evaluate 录 1 帧。标定要的是**内容多样性**而不是连续切片：60 fps 下步长 30 把 100 帧摊到 50 秒的游玩，而不是 1.7 秒里几乎一样的 100 帧。`>1` 时这一段不再连续，`pack.json` 的 `source.consecutive` 为 `false`。 |
| `CaptureSRBursts` | `0`–`4096` | `1` | 一局最多录几段；一段 = 按一次起录键。`1` = 传统的“录一次就结束”，`0` = 不限：想按几次按几次，直到写满 `CaptureSRMaxGB` 或游戏退出。每段一个 `burst_NNNN\` 子目录，都登记在同一份 `manifest.json` 的 `bursts[]` 里。 |
| `CaptureSRMaxGB` | `0`–`4096` | `0` | 整局写到多少 GiB **像素**就停（`params.json` 与各平面 `.json` 不算在内）。`0` = 不限。这是不限段数时**唯一**的界，所以 `CaptureSRBursts=0` 而这个键是 0 时后端会替你放一个 **8 GiB 默认上限**并记一条 `capture_sr_options`。到上限时记 `capture_sr_done{stage:"budget"}`，已经录下的照常可用。 |
| `CaptureSRDedup` | `0/1` | `1` | `1`：整局里内容（SHA-256）相同的平面只写一份，后面的 json 写 `stored=false` + `source` 指过去（和帧生成抓取的 lean 引用同一套约定）。1×1 曝光纹理、没变的 mask、静态平面因此每局只存一次。离线工具会跟着引用走。 |
| `CaptureSRStartKey` | 键名 / VK 码 | `Insert` | 按键触发起点。可写 `Insert` / `Delete` / `Home` / `End` / `PageUp` / `PageDown` / `Pause` / `ScrollLock`、`F1`–`F24`、单个字母或数字，或虚拟键码（`45` / `0x2D`）。`0` = 不用按键。默认不是“无”而是 Insert：标定要的是玩家走到的真实画面，而三种自动起点都落在菜单或固定计数上。多段时**每段都要重新按一次**（按键必须先松开，否则按住半秒会一次开完所有段）。等待期间记 `capture_sr_waiting_for_key`。认不出的键名（以及任何 `CaptureSR*` 键的非法值）是 `configuration_invalid`，超分抓取关。 |
| `CaptureSRStartOnCreate` | `0/1` | `0` | `1`：按键之后继续等，直到“刚新建的超分特性”的第一帧——DLSS 超分把全部时域状态留在特性内部，只有那一帧的历史是空的，也只有从那里起录，离线重放才可能复现游戏自己的输出（在游戏里就是去设置里把 DLSS 关掉再打开）。**只对“一段 + 步长 1”有意义**，其它设置下后端把它关掉并记一条 `capture_sr_options` 说明原因（一个带步长或多段的会话本来就不是某个特性历史的连续切片）。等满 3 分钟不到就什么都不录（`capture_sr_done{stage:"start_timeout"}`）。 |
| `CaptureSRLean` | `0/1` | `1` | `1`（默认）：不回读游戏自己的 `Output`。它能靠重放输入重新生成，而且是 4K 下最大的一个平面；`0` 才录（要和游戏自己的输出对账时）。 |
| `CaptureSRNoStall` | `0/1` | `1` | `1`（默认）：游戏线程永远不等写盘，回读缓冲提前建好，录满后在后台写完；内存或槽位不够时整帧跳过（`capture_sr_skipped`）。`0`：4 槽环 + 游戏线程最多等 2 秒（录到的一定完整但会卡）。 |

回读内存预算与 `COPY` 队列开关属于两套抓取**共用**的池，由**先配置的那一份**决定（SR 一侧是上面那组固定值，帧生成一侧是 `CaptureMemoryMB` / `CaptureAsyncCopy`）；一般 SR 先配置（它在 `nvngx_dlss.dll` 加载时就装好了）。
事件：`capture_sr_requested` / `_unsupported` / `capture_sr_only`（loader），`capture_sr_hooks` / `_install` /
`_options` / `_feature` / `_prewarm` / `_waiting_for_key` / `_start` / `_burst` / `_frame` / `_skipped` / `_done`（后端）。

落盘之后还有三步离线处理（开发侧，都只用 Python 标准库）：无损压缩并逐个校验 SHA-256、把一堆会话列成表（按内核族、
分辨率、预设、DLSS 版本）、转成给探针重放的输入包（可以每段一个包）。抓取格式见 `docs/CAPTURE_SR.md`。

### [Compatibility]

| 配置项 | 值 | 默认 | 说明 |
|---|---|---|---|
| `OptimizedKernels` | — | — | `[DLSSG] Optimized` 最老的**旧名**，2026-10-22 起不再读取：与 `[DLSSG] Optimized` 同写、或写的是 `0` 时记一条 `configuration_warning`（`(no effect)`）；单独写了别的值（例如老 INI 的 `OptimizedKernels=1`）是 `configuration_invalid`，帧生成不启用——改写成 `[DLSSG] Optimized=1`。 |
| `KernelImage` | `Auto/PTX/Cubin/Original` | `Auto` | 内核加载格式，见下表。**驱动拒绝选中的镜像是错误、不回退 PTX**（2026-10-26 起；见“驱动版本要求：R580”）。显式 `Cubin` 而这次安装根本没有 cubin 可用——RTX 40 / 50（本项目不发 sm_89 / sm_120 cubin）、或内核包里没有 cubin——记一条错误级 `kernel_image_unavailable{reason}`，帧生成被拒绝（2026-10-20 起；此前悄悄改跑 PTX）；`Auto` 是"有 cubin 用 cubin，没有用 PTX"，安装时一次定死。`Original` 只装钩子不替换任何内核（本机取原厂数值参考；**Ampere 上用不了这一项**——310.9.1 的原厂图像内核是 sm_89，Ampere 上创建失败，本机参考请用 `Optimized=0`+`KernelImage=PTX`）。RTX 40 / 50 上它是让运行库自己的内核跑的唯一写法；RTX 40 上运行库自己的内核只能 2X，所以 `MaxGeneratedFrames` 大于 1 时帧生成被拒绝（`max_generated_frames_unsupported{reason:"runtime_kernels"}`）。 |
| `Router` | `Auto/SM86/SM75` | `Auto` | 内核族。`Auto` 按物理 GPU 选（SM86 及以上走 SM86，Turing 走 SM75）。**SM75 是实验项**。`Optimized=1` 时创建 37 个变体行（34 个网络变体 + 3 个按 310.9.1 RVA 重新推导的图像补丁）、跨内核合并与图像补丁全开，日志记一条 `sm75_route_limits`（`variant_rows=37`、`image_patches_partial` 为空）。3070 前向 JIT 上与 SM86 原厂路线逐位一致（含 6×）；**真 Turing（RTX 2080 Ti）上**：sm_75 cubin 与 PTX、档 0 与档 1 都逐位一致，且同一份输入下的输出与 RTX 3080 Ti 的输出逐位一致（rotate 合成场景 32/32；两者与 RTX 5070 上官方输出的差异也逐字节相同，是 Blackwell 与 Turing/Ampere 之间的硬件累加差）。Turing 上的性能尚未测量。SM75 原厂内核族来自 Coldwood1026（见 `THIRD_PARTY_NOTICES.txt`）。 |
| `Preset` | — | — | `[DLSSG] Preset` 的**旧位置**，2026-10-22 起不再读取：与 `[DLSSG] Preset` 同写、或写的是 `A` / `Auto` 时记一条 `configuration_warning`（`(no effect)`）；单独写了 `B` / `C` 是 `configuration_invalid`，帧生成不启用——改写成 `[DLSSG] Preset=B` / `C`。 |
| `SpoofArchToGame` | 缺省 / `0/1` | **缺省 = 自动** | 只管**游戏（以及代它做判断的 Streamline）那道架构闸门**。**这是个三态键**：**不写**（出厂 INI 就没有这一行）= 自动，**写 `1`** = 显式，两者装重定向的时机完全一样——取下面五个时机里最早的那个，与显卡无关；**写 `0`** = 永不装（这一档在完整解析 INI 之前就要知道，所以 loader 在启动时先用 kernel32 的 `GetPrivateProfileStringW` 单独读一次 `[General] Enabled` 与本键，值非法或为空都当自动），Turing 上记一条 `arch_spoof_disabled{redirect_installed}`。**这次早读决定整个进程**（2026-10-19）：完整解析 INI 时读到不同的 `Enabled` / `SpoofArchToGame` / `SpoofArchValue` 只记一条 Level 1 的 `spoof_policy_mismatch{peek,parsed}`，什么都不改；别的键写错让帧生成不启用（`configuration_invalid`）时，改写照早读的决定保持。自动与显式 `1` 的唯一区别在**日志**：自动档只在确认这是 Turing 主机之后才把 `arch_spoof_*` 记录放出来（第一次真正改写，或后端的 Turing 判决 + `turing_host_defaults`），所以**非 Turing 显卡上出厂 INI 一条 `arch_spoof_*` 都不会有**。为什么要装这么早：**Streamline 2.x 在 `slInit` 里就决定帧生成插件"本硬件不支持"** —— `sl.dlss_g` 默认最低架构 AD100，`sl.common` 拿每个适配器的 `NvAPI_GPU_GetArchInfo` 架构去比它，然后把插件卸掉（日志里是 `Ignoring plugin 'sl.dlss_g' since it is not supported on this platform`），这发生在 D3D 设备存在之前、远早于 NGX 核心向我们索要 DLSS-G snippet，所以任何"等运行库加载之后"的武装都来不及（RTX 2080 Ti + FF7 Rebirth / Streamline 2.8.0.0 实测：写 `1` 与不写这一行的结果一模一样，插件都不在进程里）。机制：把 `nvapi64.dll` 导出表里 `nvapi_QueryInterface` 那一项重定向到该模块自己节尾填充里的 14 字节跳板，使 `NvAPI_GPU_GetArchInfo` 对外报告一个更新的架构（报哪个由下一行的 `SpoofArchValue` 决定：默认对 Turing 报 Blackwell `0x1b0`、对 Ampere 报 Ada `0x190`；不改任何 NVIDIA 代码字节；**NVIDIA 自己的组件一律看真实架构**：`_nvngx.dll`、`nvngx_*`、`nvapi*`、我们自己的 `sm86_backend.dll`、`\DriverStore\FileRepository\` 下的驱动组件，以及 **NGX 模型库里的 snippet**——NVIDIA App 的「DLSS override」或 NGX OTA 之后，核心是从 `C:\ProgramData\NVIDIA\NGX\models\dlss\versions\<n>\files\160_E658700.bin` 这样的路径加载超分 / 光线重建 / 帧生成 snippet 的，名字里没有 `nvngx_`，0.3.3 把它们一起骗了，超分 snippet 于是走 `arch>=0x180` 的内核路径把 GPU 挂死（创建 SuperSampling 特性之后立刻 `DXGI_ERROR_DEVICE_HUNG`，帧生成都还没被创建；issue #535，以及启动崩溃 #538 / #540 / #542），所以 0.3.4 起 `\NVIDIA\NGX\models\` 下除 `sl_*` 以外的每个组件、以及任何 `.bin` 模块都看真实架构；`models\sl_*`（Streamline 自己的 OTA 插件）正是要改写的对象，照常改写）。日志里每个**不同的**调用方模块一条 `arch_spoof_applied`（新增 `path` 字段给出完整路径），被排除的调用方每个模块一条 `arch_spoof_excluded`，所以能看出是谁问的、又给了它哪个答复。**只有真实答案是 Turing（`0x160`）或 Ampere（`0x170`）才改写**（Ampere 是 0.3.3 加入的：RTX 3060 Laptop 在 FF7 Rebirth 上与 2080 Ti 现象相同，issue #509）：别的架构上重定向照装但什么都不做，并记一条 `arch_spoof_inert{real_arch,rewritten_archs}`。完整解析 INI 后读到 `0`（启动时那次读不到 INI 的情况）不再让它停止改写（2026-10-19 之前会，那是会话中途改变游戏看到的架构），只记 `spoof_policy_mismatch`。`arch_spoof_installed` 带 `trigger` 与 `default`：**早期三档** `dllmain`（我们被加载时 `nvapi64.dll` 已在进程里）/ `nvapi_load`（我们的 LoadLibrary 钩子接住了 `nvapi64.dll` 自己的加载）/ `dependency_load`（`nvapi64.dll` 作为别的模块的静态导入进来，某次加载之后才发现它在）——只有这三档赶在 `slInit` 之前；**兜底两档** `settings`（读完 INI，即游戏第一次索要 `nvngx_dlssg.dll`）/ `backend`（后端装完）。`default=true` 表示是自动档做的决定，与早晚无关。**它与 NGX 核心的 `0xbad0000b` 无关**：那一条是核心拿显卡架构去比运行库导出的最低架构（见下面 `fg_gate_create_feature` 一行），任何 NVAPI spoof 都改不了。 |
| `SpoofArchValue` | `Auto/Ada/Blackwell` | `Auto` | 改写时对外上报哪个架构。`Auto`（默认）对 **RTX 30（Ampere）报 Ada（`0x190`），对 RTX 20（Turing）报 Blackwell（`0x1b0`）**。RTX 30 上报 Ada 时，3070 上赛博朋克 2077、巫师 3、地平线 6 的菜单仍给 3X / 4X，选 4X 实跑 4X（倍数由后端决定，与这个答复无关）；Streamline 的在线更新组件改载 Ada 版，赛博朋克 2X 那种一阵一阵的卡顿明显减少。RTX 20 上报 Ada 能过帧生成那道闸门，但游戏里只剩 2X（真 20 系实测，issue #527 / #528），所以仍报 Blackwell。`Ada` / `Blackwell` 把两代都写死成这个值：`Blackwell` 就是 0.3.3–0.4.0 在 RTX 30 上的答复，需要对比时可以写回去（两者也接受 `0x190` / `0x1b0`）。**这个值从来不是 0.3.3 崩溃的原因**：崩的是 NVIDIA 自己的 DLSS 超分模型也收到了改写后的架构——NVIDIA App 的「DLSS override」或 NGX 在线更新生效时，它是从 NGX 模型目录里按 `160_E658700.bin` 这样的名字加载的，按叶名的排除规则拦不到；3070 上实测报 Ada 与报 Blackwell **都**会挂 GPU，而把这个 snippet 排除之后**两个都不会**，所以修的是「按路径排除 NVIDIA 自己的组件」（`src/common/spoof_policy.h`，见上一行），不是换一个更小的架构。和 `SpoofArchToGame` 一样在启动时用 kernel32 单独读一次，那次读到的值决定整个进程；完整解析时值非法是 `configuration_invalid`（帧生成不启用，改写仍照早读的决定）。`configuration.spoof_arch_value`、`arch_spoof_installed{spoof_arch_value,reported_arch_turing,reported_arch_ampere}`、`arch_spoof_applied.reported_arch`（432 / 400）说明实际用的是哪个。 |
| `ForceSM86Route` | `0/1` | `0` | `1` 允许在非 SM86 GPU 上强制走 SM86 后端做验证（低于 SM86 仍被拒）。这是验证配置：它连 RTX 30 那套架构桥接一起装上，与 RTX 40 / 50 默认的那条路线不是一回事。 |
| `SimulateAmpere` | `0/1` | `0` | `1` 为验证调整架构上报（运行库自己读到的架构改成 Ampere）；必须同时 `ForceSM86Route=1`。不改物理 GPU。RTX 40 / 50 默认的那条路线不需要它，也不会装它。只写它而没有 `ForceSM86Route=1` 是 `configuration_invalid`，帧生成不启用（C114）。 |
| `DisableFusions` | `0`–`63` | `0` | **诊断位掩码（打开会变慢）**，关闭优化集的某类跨内核合并：1 解码器 upscale+add+1×1，2 解码器 1×1 对，4 `k_initial_merge`+convPre，8 conv0+池化，16 残差链，32 `k_central_block`+block1 conv0。默认 0 = 全部合并开启。 |
| `ImagePatches` | `0/1` | **档位决定**（档位 ≥1 为 `1`） | `0` 关闭图像处理内核的**精确** PTX 级补丁（需要档位 ≥1）。补丁语义不变、逐位一致。有损行由 `ImageApprox` 管、不受它影响；**`CatmullRomWarp` 的行也是图像补丁行，`ImagePatches=0` 会一并关掉它**（那个键因此记一条 `configuration_warning`）。档位 0 不建任何行，在那里写 `1` 不起作用（同样记一条）。 |

### [SuperResolution]

**DLSS 超分（NGX feature 1）路线**，与帧生成互相独立。**2026-10-19 起它只做一件事：预设覆盖**（业主决定，`docs/COMPAT.md`
C96；机制与验证在 `docs/SR.md`）。玩家面只有一个键 `Mode`，五个值。**出厂 INI 写着 `Mode=Off`，与整节缺席完全相同：这个
进程与没有这段代码的版本逐字节相同**（不拦 `nvngx_dlss.dll`、不 hash 那份 74 MB 的 snippet、不装钩子、一条 `sr_*` 记录都
没有，`configuration` 连字段都不多一个）。

**游戏里开着光线重建（Ray Reconstruction）时，这一节不起作用。** 光线重建取代超分：游戏创建的是 NGX feature 13（DLSS-D，
降噪与放大由同一个网络完成），**根本不创建 feature 1**，所以 `Mode` 用不上；画质档仍然决定渲染分辨率，跑的是游戏自己的
光线重建（本项目不碰它，见下一节）。在游戏设置里关掉光线重建，超分路线就回来。赛博朋克 2077 实测：路径追踪 + 光线重建的
整局里没有一条 `sr_*` 记录、没有 feature 1。

| 配置项 | 值 | 默认 | 说明 |
|---|---|---|---|
| `Mode` | `Off` / `M-INT8` / `L-INT8` / `M-FP16` / `L-FP16` | **`Off`** | `Off` = 不介入。其余四个值都把游戏请求的超分 snippet 换成随包的 **310.9.16**（`dlssg_sm86\sr\nvngx_dlss.dll`），并在**全部六个画质档**（DLAA、超高画质、画质、平衡、性能、超级性能）上**钉住** M 或 L；用哪一档由玩家在游戏菜单里选。`X-INT8` 跑本项目的静态 INT8 内核（M 的 12 个网络内核；L 在 RTX 30 / RTX 20 上是 10 个——第一层与最后一层留在运行库自己的 FP16 内核上，那两层的 INT8 在这两代卡上反而更慢，`docs/SR.md` 第 3 节——加上 D3D12 上逐位等价的 post / downsample 重写）；`X-FP16` 一切相同、只是内核是 NVIDIA 自己的——**同字母 INT8 的 A/B 对照**（同运行库、同预设，只差内核）。**只有这五个值**：写别的（包括 2026-10-10 到 10-19 的 `On`）是非法值，记一条 L1 `configuration_invalid{section,key,value,allowed,effect:"super_resolution_not_enabled"}`，超分不启用（与 `Off` 相同）。 |

**INT8 的收益在性能档与超级性能档**：3070 实测一次 Evaluate 的 GPU 时间（INT8 对同运行库、同预设的 FP16，1080p–4K × 四档 × 两族）
M −9.4…−23.2 %（−0.10…−2.12 ms）、L −6.5…−16.3 %（−0.08…−1.89 ms）。**画质 / 平衡 / DLAA 档也被钉成 M / L**，而游戏在这几档
本来多半用 K：3070 上 K 比 M 快 1.24–3.66 倍、比 L 快 1.48–4.77 倍（Resonance 画质档 47.5 → 38.1 fps），所以在这几档开
`X-INT8` 比游戏自己的 K 慢——业主接受（2026-10-19），要帧率就在游戏里选性能 / 超级性能档。

**钉住的预设压过一切别的来源**：游戏写的预设提示（Streamline 的 `0x8000000B/C/D`，310.9.16 本来就丢掉）不读；NVIDIA App 的
预设覆盖生效时也被盖掉，每进程记一条 `sr_preset_drs_conflict`（C69）。2026-10-19 之前还有的 `Mode=On`（不钉预设、只在游戏
本来就用 L / M 的档换内核）、`Tier`（只钉某一档）、提示还原、跟随 NVIDIA App、一局里第二个预设被拒（`sr_route_preset_refused`）
都已删除：一个进程只会有一个预设。

**游戏内 A/B**：同一场景、游戏里同一档，`Mode=M-INT8` 对 `Mode=M-FP16`（或 `L-INT8` 对 `L-FP16`），两局只差内核；数字用
`[Logging] FrameTiming=1` 的日志（`scripts/perf/frame_timing_report.py`）。不要拿 `Mode=Off` 当基线：那是游戏自带的运行库和
它自己的预设，差的不只是内核。

**预设与画质档是两个维度**：画质档决定渲染/输出分辨率的比例，预设再选权重与网络。**钉预设不改变渲染分辨率**。

**只在 RTX 30 与 RTX 20 上生效**。第一次加载 `nvngx_dlss.dll` 时、换任何东西之前，代理先问一次显卡（CUDA 驱动 API：
计算能力与显卡名）并读随包内核包 `sr_kernels.dll` 的索引：只有某块 NVIDIA 显卡是 sm_86 或 sm_75、**名字里有 `RTX`**、且包里有它的集合，
才介入。**其它任何情况——RTX 50、RTX 40、GTX 16 与 Turing T 系列（也是 sm_75，但没有张量核心）、MX570、没有 CUDA、
包读不到——行为与 `Mode=Off` 完全一样**：不换 snippet、不装钩子，日志里多一条警告级 `sr_gate` 说明原因（GTX 16 一类是
`reason:"device_not_rtx"`；`configuration` 里照抄 INI 的 `super_resolution` 字段仍在，那是读 INI 时写的）。
这道闸只问“机器上有没有一块在名单上的卡”。游戏实际渲染的那块卡在第一次 CreateFeature 时按 LUID 再判一次（2026-10-19）：它不在名单上
（例如一台机器上有 RTX 30 也有 RTX 40 / GTX 16，游戏跑在后者上）时，INT8 的 Mode（`M-INT8`、`L-INT8`）下被判给 INT8 的特性
与内核包缺失同样被拒绝：一条 Level 1 的 `sr_kernels_unavailable{reason:"device_not_on_list"}`，这些 CreateFeature 返回 `0xbad00001`；
FP16 的 Mode 不受影响。另一个例外（C93）：
INT8 Mode 下 RTX 30 / 20 本身受支持、只是包里没有它能用的集合（缺失 / 损坏）时，snippet 照样换、路线照样装，要跑 INT8 的特性
在创建时被拒绝（`sr_kernels_unavailable` / `sr_feature_refused`），不悄悄跑 FP16。
RTX 40 / 50 排除是因为它们自己的 FP8 路径比我们的 INT8 快，介入没有收益（RTX 50 的 sm_120a 验证集合产品里没有任何
配置执行，2026-10-11 起不进发布包，见下面 `KernelSetSM` 一行）。RTX 20 在名单上，但它的宿主权重宽度还没在真卡上量过。

**已移除的键**：`Int8`、`Int8L`、`Int8M`、`ForcePreset`、`ForcePresetDLAA` / `ForcePresetQuality` / `ForcePresetBalanced` /
`ForcePresetPerformance` / `ForcePresetUltraPerformance` / `ForcePresetUltraQuality`（2026-10-03），以及 `Tier`（2026-10-19）。
这些键不读（2026-10-26 起；此前各记一条 `configuration_warning`）——**所以旧 INI 的 `Int8=1`
不再打开任何东西，没有 `Mode` 行就是 Off；`Tier=Performance` 不再把钉值限制在性能档**。

**诊断键**（不在出厂 INI 里，玩家不需要；没有一个能让介入在上面那道检查不放行的显卡上发生）：

| 配置项 | 值 | 默认 | 说明 |
|---|---|---|---|
| `Auxiliary` | 0/1 | 1 | 跑 INT8 的特性（`X-INT8`）连 post / downsample 的逐位等价重写一起替换。**只在 D3D12 上生效**：Vulkan 路线无条件拒绝它们（COMPAT C57 —— 在那条 API 上替换进去的不再是运行库自己的比特，而 post 内核写时域历史，差异每帧回灌到像素饱和），日志里是 `auxiliary_allowed:false` 与 `auxiliary_refused`。 |
| `KernelImage` | `Auto`/`Ptx`/`Cubin` | `Auto` | 先试哪一族镜像。集合里每个内核有两张：`.nvapi.fatbin`（PTX，驱动要 JIT）和 `.cubin.nvapi.fatbin`（本架构的原生 cubin，不用 JIT）。**`Auto` 与 `Cubin` 都优先 cubin**，`Ptx` 根本不看它。`Auto` 下集合里没有 cubin 镜像就用 PTX（不记录）；显式 `Cubin` 而为本设备选定的集合缺该预设某个替换内核的 cubin（或 `KernelSetSM` 选了别的架构的集合）：一条错误级 `sr_kernel_image_unavailable`，一个内核都不建，给了 INT8 的特性 CreateFeature 被拒（`sr_feature_refused`，`0xbad00001`），不会改用 PTX（2026-10-22）；驱动拒绝了选中的镜像（cubin 或 PTX）= `sr_kernel_fatal{kind:"kernel"}`，进程结束（C93；2026-10-26 起 cubin 被拒也是如此，不再改用 PTX）。**这就是这张镜像存在的理由**：只有 PTX 时，游戏第一次创建特性（以及每次改画质档重建）要让驱动编译 12 个约 600 KB 的内核。诊断开关 `DLSSG_FORCE_CUBIN_FAIL=1` 只弄坏 cubin 镜像，`DLSS_SM86_FORCE_IMAGE_FAIL=1` 弄坏**全部**镜像，两者都走到这条致命路径。 |
| `ObserveFrames` | 0–64 | 1 | 先观察几帧、把每个 launch 的合同逐条验过再开始替换。`1`（默认）观察第 1 帧、第 2 帧起替换。`0` 是测量用的：第一帧起就替换，每个 launch 仍先过自己的合同，但放弃了那一帧的"全有或全无"（3070 实测：8 帧里 8 帧都被替换，`1` 时是后 7 帧）。 |
| `KernelSet` | 路径 | `dlssg_sm86\sr` | 内核集**包**在哪里。默认目录里是发布的内核包 `sr_kernels.dll`（一个只有资源的 DLL，装着 RTX 30 / 20 的四套集合，每个 (架构, 输入族) 一套；后端读全部，按显卡的 compute capability 选）；它以文件随包、不内嵌进代理（约 106 MB，一个发布里有六个代理名）。目录里有 `sr_kernels.dll` 就读它，即使旁边还留着旧版的 `index.json`；没有就按目录包读（一个 `index.json` + 集合目录，研究用的集合就这么指，规则与以前相同）；值也可以直接是一个包 DLL 的路径。相对路径相对代理自己的目录。**设备闸也读这里**：没有本机集合 = Off。设备闸只读包的闸门摘要（`sr_kernels.dll` 的 INDEX 资源，或目录包 `index.json` 里的 `gate_summary`），所以指向单个集合的目录、2026-10-04 之前生成的目录包（没有摘要）、或缺了 `sr_kernels.dll` 的发布也 = Off，`sr_gate` 的 `sets` 里写着原因（`kernel_set_source` 说读的是包还是目录）。包本身损坏时由后端整包拒用（`sr_kernel_set{loaded:false}`，`source.reason` 写明哪一项），INT8 Mode 下被判给 INT8 的特性在创建时被拒（`sr_feature_refused`，C93）。 |
| `KernelSetSM` | `Auto`/`75`/`86`/`120` | `Auto` | **诊断/测量用，玩游戏请保持 `Auto`。** 换一个架构的内核集在本机上跑（前向 JIT）。`Auto` 用显卡自己的 compute capability，这是唯一正常的取值。**覆盖生效时镜像族被强制成 PTX**（cubin 是一个架构的机器码，驱动收不下），记一条 `sr_kernel_image{forced_ptx:true}`；**包里没有那个架构的集合时整条路线被拒绝**而不是悄悄换回本机的集合（`sr_kernel_set_override{applied:false}`、`route.disabled_reason="kernel_set_override_missing"`、`replaced` 恒 0）；外来架构上宿主权重缓冲宽度那道闸被豁免并逐 launch 计数（`route.binding_waivers`），其余合同（入口名、专用内核摘要、block、几何）照旧。设备闸判的是**显卡本身**，所以 3070 上的 `KernelSetSM=120` 能过闸；发布的内核包里没有 sm_120a 集合（2026-10-11 起），于是路线在建任何内核之前以 `kernel_set_override_missing` 拒绝，INT8 的特性在创建时被拒（C93）。把 `KernelSet` 指向带 sm_120a 集合的目录包时它会被选中——**但跑不起来**：PTX 只能往上 JIT，驱动不收 sm_120a 的 PTX，第一次创建就是 `sr_kernel_fatal`（C93 / C103；3070 实测 2026-10-10）。一张卡只能这样跑本机或**更老**架构的集合：3070 上是 `75`，2080 Ti 上只有它自己的 `75`（在它上面写 `86` 同样建不出内核），任何 RTX 30 / 20 都跑不了 `120`。**它存在的理由**：`set_sm_75` 与 `set_sm_86` 的权重镜像逐字节相同，只有机器码不同，所以一台 RTX 30 跑一遍 `Auto`、一遍 `75` 就能证明 sm_75 那 12 个内核算出来的东西一样——而 RTX 50 上做同一件事什么都证明不了（Blackwell 的累加与 Ampere/Turing 不同）。详见 `docs/SR.md` 第 8 节。 |

安装之前：显卡不在名单上 = 照原样转发并记一条 `sr_gate`；随包 snippet 缺失或哈希不符 = 这次加载被拒绝（`sr_runtime_unavailable`，C99）。INT8 Mode 选定之后
集合用不了（包缺失 / 损坏、显卡或输入族没有集合）或渲染设备不在名单上（上面一段），按 C93 拒绝那个特性（CreateFeature 返回
`0xbad00001`），之后的失败是 `sr_kernel_fatal`，不悄悄换回 FP16。

**LDR（色调映射后的颜色）输入的游戏按设计不加速**：这一族没有标定内容，路线在建任何内核之前就明确拒绝并记一条常规级
`sr_family{chosen:false, by_design:true, family:"mvlo_ldr"}`；每一档照样钉 Mode 的预设，跑的是 NVIDIA 自己的内核
（等同 X-FP16）——这是 NVIDIA 自己的代码，不是故障，日志里也不会有任何错误级记录
（详见 `docs/SR.md`「输入族」一节）。

**Vulkan 游戏**（荒野大镖客 2 一类）上这条超分路线**同样成立**：同一份运行库、同一套内核集、同一个 `Mode`。
换内核的机制不同（改 snippet 自己那张 Vulkan 入口点表里的五个函数指针，而不是打代码补丁），但对使用者只有三条区别：
① **Vulkan 游戏只能用四个工具类代理名**（根目录的 `version`，或 `alternatives\` 的 `winmm` / `dbghelp` / `dinput8`）——
`dxgi.dll` / `d3d12.dll` 在 Vulkan 后端下根本不会被加载；② 游戏建 `VkDevice` 时若没开 `VK_NVX_binary_import`
扩展，记一条错误级 `sr_vk_entrypoints{installed:false}`，INT8 的 Mode 下被判给 INT8 的 Vulkan 特性在创建时被拒（`sr_feature_refused`；这个决定发生在我们能
介入的一切之前，没有旋钮能改）；钩那张入口点表本身失败时（`sr_vk_install{installed:false}`），INT8 的 Mode 下被判给 INT8 的
Vulkan 特性被拒绝（`sr_feature_refused`，`route_hooks_unavailable`，`0xbad00001`），D3D12 游戏不受影响；③ **`[Debug] Capture` 与 `[Debug] CaptureSR` 只在 D3D12 上工作**，
`[FrameGeneration]` 整节在 Vulkan 上本轮也未支持。日志里认这条 API 的字段是 `sr_install` / `sr_exit` 的
`api:"vulkan"`、`sr_vk_install`、`sr_vk_entrypoints`、`sr_device{api:"vulkan"}`、`sr_resource_hook{api:"vulkan"}`，
以及第一次用到 Vulkan NGX 时才出现的 `fg_gate_hooks_vk`。**这条路线目前只有静态与无 GPU 的证据**，见
`docs/COMPAT.md` 缺口 G13。

### [RayReconstruction]（已删除）

**2026-10-09 起本项目不碰光线重建**（NGX feature 13，业主决定；为什么见 `docs/RR.md`）。原来的 `Mode=Mixed` 把游戏请求的
光线重建 snippet 换成随包的 310.9.0.0、钉预设 F、跑混合 INT8 内核集；赛博朋克 2077 实测它与游戏自己的光线重建帧率持平
（4K 超级性能档、帧生成 4X：26.86 对 26.89 fps），对玩家没有好处，所以整条删除。现在 `nvngx_dlssd.dll`（游戏目录里的，或
NVIDIA App 覆盖时模型库里的）**原样加载**：不拦截、不 hash、不装钩子、一条 `rr_*` 记录都没有——与原来出厂的
`Mode=Off` 完全相同（写了 `[Debug] CaptureSR` 时，它的加载照旧像 `nvngx_dlss.dll` 的一样武装超分抓取，好让日志说清
"这个游戏用的是 feature 13"）。发布包里不再有 `dlssg_sm86\rr\`，老安装留下的这个目录不会被读取，可以删。

**光线重建一开就取代超分**：游戏只创建 feature 13，`[SuperResolution]` 的设置在这期间全部不生效（上一节开头）；
要测超分 INT8 就在游戏里关掉光线重建。

**旧键**：这一节的全部键与 `[Runtime] RRMode` / `RRPath`、`[Debug] CaptureRR` / `CaptureFeature` 不读（2026-10-26 起，C113；此前只为告警而读）：旧 INI 里的 `Mode=Mixed` 什么都不做。

### [Runtime]

| 配置项 | 值 | 默认 | 说明 |
|---|---|---|---|
| `Path` | 路径 | `dlssg_sm86\runtime\nvngx_dlssg.dll` | `Mode=External` 与 `Mode=Pinned` 使用，指向 `nvngx_dlssg.dll`。**默认值就是 `Bundled` 用的那个文件**，所以 `Mode=External` 不写 `Path` 等价于 `Bundled`（与 `SRPath` 同构）。 |
| `SRMode` | `Bundled`/`External`/`Pinned` | 跟随 `Mode` | 超分 snippet 的同一套选择（`Auto` 与 `Mode=Auto` 一起在 0.4.0 删除，按 `Bundled` 运行并告警）。**默认跟随 `Mode`**：把一个 NVIDIA snippet 外置、钉住或关掉的人基本都是两个都想要。`Bundled` = **包里那个文件** `dlssg_sm86\sr\nvngx_dlss.dll`（74 MB，不内嵌进 DLL——一个发布有六个代理名），用之前校验 SHA-256；不在或不对就记 `sr_runtime_unavailable`，**这次 `nvngx_dlss.dll` 的加载被拒绝**、超分不启用（2026-10-20 起；此前把游戏交回它自己的 DLSS，INT8 Mode 下那就是原厂 FP16）。 |
| `SRPath` | 路径 | `dlssg_sm86\sr\nvngx_dlss.dll` | `SRMode=External` 与 `SRMode=Pinned` 使用。哈希不符会记 `external_runtime_rejected{feature:"super_resolution"}`（`Pinned` 是 `sr_pinned_runtime_rejected`），**这次加载被拒绝**、超分不启用（2026-10-20 起；此前回退 `Bundled` 并记 `configuration_warning`）。 |

`Mode` 值：`Bundled`（普通安装，随包的 `dlssg_sm86\runtime\nvngx_dlssg.dll` + 随包的后端）；**`External`（运行库放在别处，由 `Path` 指定——见下一段）**；（`Auto`——加载游戏请求的原运行库——在 0.4.0 删除，旧值按 `Bundled` 运行并记一条 `configuration_warning`，COMPAT C86）；`Pinned`（使用 `Path` 指定的运行库，需匹配后端）。加载 / 安装失败（`backend_unusable`、`install_failed`、后端的上限低于 `MaxGeneratedFrames` 等）时**不再把游戏自己的 `nvngx_dlssg.dll` 交给它**（2026-10-19；`External` / `Pinned` 同样）：加载器记 `runtime_selection_failed{action:"frame_generation_not_enabled: the nvngx_dlssg.dll load is refused"}`，这次加载失败，帧生成不启用——RTX 40 / 50 上这意味着没有帧生成，而不是运行库自己的 2X。
超分那一侧，游戏（或 NVIDIA App 的覆盖）已经映射了 310.9.16 时，任何模式下都不需要替换，只装钩子，记 `sr_runtime_adopted`。

#### `Mode=External`：运行库放在代理外面

```ini
[Runtime]
Mode=External
Path=dlssg_sm86\runtime\nvngx_dlssg.dll
SRMode=External
SRPath=dlssg_sm86\sr\nvngx_dlss.dll
```

- **路径写法**：绝对路径，或**相对于代理 DLL 自己所在的目录**——不是进程的工作目录。Steam、快捷方式的"起始位置"、
  以及自己 `chdir()` 到数据目录的引擎都会让工作目录变成别的东西，而代理是用户亲手放在游戏 EXE 旁边的，
  所以只有它能让同一份安装每次解析到同一个文件（`[Logging] Directory` 等所有相对路径本来就是这条规则）。
  写 `%LOCALAPPDATA%` 这类**环境变量会先展开**（仅 `External`；`Pinned` 的路径按字面用）。
- **必须是这个构建的后端服务的那个运行库**。不符就记一条错误级
  `external_runtime_rejected{feature,path,sha256,reason,action:"frame_generation_not_enabled: …"}`，**这次加载被拒绝**，
  随包的与游戏自己的运行库都不顶上（2026-10-20 起；此前回退 `Bundled` 并记 `configuration_warning`）。拿错的是本项目另一个发布包里的运行库时，记录会直接把它叫出名字
  （`runtime_version` / `expected_version`，比如"它是 310.1.0.0，而这个构建的后端服务 310.9.1.0"）。
- `Mode=External` 而 `Path` 没写**不再是需要纠正的配置**：`Path` 的默认值就是随包那个文件，所以它解析到和 `Bundled`
  完全相同的结果（`SRPath` 一直是这样）。以前那条 `configuration_warning` 已经删掉。
- 日志里认哪一档：`runtime_redirect` / `sr_runtime_redirect` 的 `selected_source` = `bundled` / `external` / `pinned` /
  `game_own`，外加 `sha256`（**答话那个文件本身**的摘要，四种模式都写）；`source` 仍然只说请求是**从哪儿来的**：
  游戏目录 / NGX 模型库 / 驱动库。
- **包里的运行库缺失或不是这个构建的那一份**时 `Bundled` 无处可退：每进程一条错误级
  `bundled_runtime_unavailable{path,sha256,runtime_sha256,found_version,reason,remedy}`，**这次加载被拒绝**、帧生成不启用
  ——不会悄悄变成 `Auto`，也不把游戏自己的 `nvngx_dlssg.dll` 交回去（2026-10-20 起；此前原样交回，RTX 40 / 50 上那就是运行库自带内核的 2X）。最常见的原因是只复制了代理 DLL 而没有复制 `dlssg_sm86\` 这个文件夹。
  判定在**加载之前**：哈希不符的模块根本不会被映射进进程。

### [Backends]（已删除）

2026-10-22 删除（C102）：后端只有代理旁边那一个，`Pinned` / `External` 只能指向这个构建的运行库。INI 里还有这一节时不读（2026-10-26 起，C113）。可选导出 `DlssgMod_SupportedRouteFlags` 与它的标志位协商（`kernel_selection_unsupported`）一起删除。

### KernelImage 行为

| KernelImage 值 | 路由启用后的行为 |
|---|---|
| `Auto` | 物理架构与所选内核族相同时用预编译 cubin，否则用 PTX。驱动拒绝它是错误（`fg_kernel_fatal`），不回退。 |
| `PTX` | 总是用所选内核族的 PTX，由驱动 JIT；3080 Ti 也适用。 |
| `Cubin` | **优先**用预编译 cubin，要求物理架构与内核族**完全一致**，否则安装失败（`install_failed`），`nvngx_dlssg.dll` 的加载被拒绝、帧生成不启用（2026-10-19 之前回退原始加载）。架构一致但驱动拒绝 cubin 时仍会回退 PTX（宁可 JIT 也不要没有帧生成）。**cubin 在随包的内核包 `dlssg_sm86\kernels\fg_kernels.dll` 里**（与 PTX 同在，后端 DLL 不带任何内核）：那个文件不在时帧生成被拒绝（后端记 `kernel_pack_missing` + `fg_kernels_unavailable`，2026-10-16 起，C93，与 `KernelImage` 写什么无关）；包在而里面没有 cubin 时，写 `Cubin` 记一条错误级 `kernel_image_unavailable`，帧生成同样被拒绝（2026-10-20 起，C99）。此前两种情况都被当作 `Auto`。 |
| `Original` | **诊断/参考用**：装钩子但**一个内核都不替换**，也不启用优化 / 合并 / 图像补丁。与 `PTX`/`Cubin` 互斥，与 `Router` 无关。**Ampere 上用不了。** |

`KernelImage` × `Router` 的实际选择：

| Router / 物理 GPU | `Auto` | `PTX` | `Cubin` |
|---|---|---|---|
| SM86 族，物理 SM86（3070/3080 Ti） | `cubin_sm86` | `ptx_sm86` | `cubin_sm86` |
| SM86 族，物理 SM89/SM120（0.4.0 起默认，2026-10-19 起不可关） | `ptx_sm86` | `ptx_sm86` | 拒绝（`kernel_image_unavailable`） |
| SM86 族，物理 SM89/SM120（`ForceSM86Route` 强制路由验证） | `ptx_sm86` | `ptx_sm86` | 拒绝 |
| SM75 族，物理 SM75（RTX 20 系） | `cubin_sm75` | `ptx_sm75` | `cubin_sm75` |
| SM75 族，物理 SM86 及以上（前向 JIT 验证） | `ptx_sm75` | `ptx_sm75` | 拒绝 |

SM75 族与 SM86 族在 RTX 3070 的前向 JIT 上**逐位一致**（原厂族与优化集都是，box + rotate 两个场景 7 个分辨率 240/240 张，**`--multi 5`（6×）另外 80/80 张**；MFG 相位（`i/(N+1)` 质心）两条路线完全相同；`m16n8k16` 拆成两条 `m16n8k8` 并没有多出一次与原厂不同的舍入）。310.9.1 的 70 个内核里，44 个用与 310.1 共用的那一族 sm_75 原厂内核，26 个新内核的 sm_75 镜像由本项目从自己的 sm_86 PTX 改写而来。优化集在 SM75 上**一行不少**：37 个变体行（34 个网络变体 + 3 行图像补丁）都有 sm_75 镜像。退役的 310.1 构建上同样逐位一致（240/240 张）。

数值在 RTX 3070 上以 `.target sm_75` 前向 JIT 验证为与 SM86 路线逐位一致；**真 Turing（20 系实机，2026-09-15/17）**上 `sm_75` cubin 全族创建并运行、0 回退，cubin 与 PTX 逐位一致，档 0 与档 1 逐位一致，游戏内可用。Turing 上的耗时尚未量化。详见 `docs/VALIDATION.md`。

#### 驱动版本要求：R580

| 路径 | 最低驱动 | 原因 |
|---|---|---|
| 全部：帧生成、XeSS-FG、超分 INT8，cubin 与 PTX | **R580**（CUDA 13.0；本轮在 617.14 上实测） | 全部 cubin 由 CUDA 13.0.2 的 `ptxas` 生成（ELF ABI 8，2026-10-26 起没有 12.x 的 cubin），PTX 是 ISA 8.5 / 8.7 |

后端安装时向 CUDA 驱动问一次它实现的 CUDA 版本（`cuDriverGetVersion`；R580 = 13.0）。**低于 R580 时什么都不装**：帧生成与
XeSS-FG 是一条错误级 `driver_below_floor{cuda_driver,floor:"R580 (CUDA 13.0)",driver}`，加载器随即拒绝 DLSS-G 运行库的加载（游戏里
没有帧生成）；超分 INT8 Mode 是一条 `sr_kernels_unavailable{reason:"driver_below_floor"}`，给了 INT8 的特性 CreateFeature 被拒
（`0xbad00001`），FP16 Mode 不受影响。没有"旧驱动改用 PTX"这条路（`docs/COMPAT.md` C127；2026-10-26 之前驱动拒绝 cubin
时改用同一内核的 PTX）。

**哪张卡用哪种镜像，帧生成、超分与 XeSS-FG 是同一条规则（2026-10-24 起，`docs/COMPAT.md` C109）**：按物理 GPU 的 SM 每进程定一次——SM 7.5（RTX 20）用 sm_75 cubin，SM 8.6（RTX 30）用 sm_86 cubin，其它任何 SM（RTX 40 的 8.9、RTX 50 的 12.x……）用 PTX、由驱动 JIT。不试探：不会把 cubin 交给不是为它选的卡。`[Compatibility] Router` / `KernelImage` 与超分的 `KernelImage` / `KernelSetSM` 照旧在这个结果上起作用；超分只服务 SM 7.5 / 8.6（设备闸），RTX 40 / 50 上不介入。XeSS-FG 在 2026-10-24 之前按 `.75`、`.86`、`.ptx` 的顺序试第一个内核、取驱动先接受的那一族：RTX 40 跑的是 sm_86 cubin（现在是 PTX），cubin 被拒时悄悄换下一族、不留记录。

R580 以上，被选中的镜像（cubin 或 PTX）被驱动拒绝就是该模块的错误：帧生成 / 超分 `fg_kernel_fatal` / `sr_kernel_fatal{kind:"kernel"}`、进程结束（C93）；XeSS-FG `xefg_feature_refused`，该特性在 CreateFeature 时被拒绝（C94）。不换另一种镜像重试。诊断变量 `DLSSG_FORCE_CUBIN_FAIL=1` 弄坏全部 cubin、强制走这条路（仅用于验证，三者都适用）。

**游戏里不再编译内核**（2026-10-26，`docs/COMPAT.md` C128）：
- **同一进程**：运行库每建一次特性（菜单 ↔ 游戏、改分辨率 / 设置、开关帧生成）都把它的约 65 个 CU 模块全部释放再建一遍。由本项目镜像建出的模块与函数现在留到进程结束，按（设备、镜像）交回给之后的每次创建，运行库的释放对它们只答成功（`feature_create` 的 `modules_reused` / `functions_reused` / `releases_kept`）。RTX 3070 实测：重建一次特性的模块创建从 4.8 ms（cubin）/ 1229 ms（PTX、驱动缓存关闭）降到 0.02–0.03 ms。
- **跨进程，只在 RTX 40 上（PTX 一族；RTX 50 不用，见本条末尾）**：每个 PTX 镜像在这台机器、这个驱动上只编译一次——后端安装时在后台线程用驱动自己的 CUDA 链接器（`nvcuda.dll` 的 cuLink）编译，结果存在 `%LOCALAPPDATA%\DlssgSm86\jit`（文件名 = PTX 的 SHA-256 + 物理 SM + 驱动版本），之后的进程把这份 cubin 直接交给 NvAPI。第一次 CreateFeature 时还没编完就等它编完。每批一条 `jit_cache{batch,images,disk_hits,compiled,wall_ms}`。缓存文件校验不过（截断、摘要不符、不是 ELF）或被驱动拒绝 = 致命错误并点名那个文件（删掉它，下次启动重新编译），不会悄悄重编、也不会改交 PTX。换驱动 = 新的键，旧文件留着不用。**RTX 50（Blackwell，物理 SM ≥ 10.0）上不用这个缓存**（2026-10-27，`docs/COMPAT.md` C131）：在那里 cuLink 编出的 cubin 对 D3D12 纹理的取样与 NvAPI 自己编译同一份 PTX 不同，帧生成和 XeSS-FG 的生成帧都会错。后端记一条 `jit_cache_off`，PTX 照旧交给 NvAPI，由驱动在进程第一次 CreateFeature 时编译（帧生成约 2 秒、XeSS-FG 约 9 秒，驱动自己的计算缓存命中时更短），之后模块留到进程结束。

#### `SkipRepeatedRealCopy`（已删除）

这个键曾让后端在 `MultiFrameIndex > 1` 时吞掉运行库的全分辨率 `CopyResource(Backbuffer → DLSSG.OutputReal)`。2026-10-08 删除：运行库只在调用方交出 `DLSSG.OutputReal` 时才发这次拷贝，而检查过的全部 Streamline `sl.dlss_g` 构建（2.6.10–2.14.1，9 个）只在 index 1 交出它、之后传空，所以 Streamline 游戏里没有重复的拷贝可跳（5 个游戏的抓取里 index ≥ 2 的 `OutputReal` 300/300 为空；巫师 3 实机 3486 次 Evaluate 跳过 0 次）。每真实帧省几十到几百微秒的数字只出现在本项目自己的 `sm86_scene` / `sm86_bench` 里，它们在 2026-10-08 之前每个 index 都传 `OutputReal`（之后与 Streamline 一样只在 index 1 传）。2026-10-26 起这个键不读。

#### 档位 2 / 3：有损图像行，以及“看不出差别”是怎么回事

档位 2 / 3 武装的是本项目里**唯一一批刻意不与官方运行库逐位一致**的内核镜像（内核包清单里的 `approx` 行）。

**档位 2（2026-10-04 到 2026-10-21 的出厂默认，之后出厂回到档位 1）只武装一行**：A6，`Kernel_OutputPull` 的两次 warp 取样从手写双线性降为最近点（掩码 `0x20`）。实测：

- **对官方数值**：RTX 3070，三份 4K 4X 真实游戏抓取，档位 2 的重放对档位 0 的重放（档位 0 就是官方运行库的数值），生成帧逐帧 PSNR 最差 / 中位：Resonance 44.06 / 48.06 dB（运动最快的一份，每真实帧位移 250–500 px，最低值在第 46–90 次 Evaluate 那一段），地平线 6 49.59 / 51.34 dB，黑神话悟空 49.87 / 52.09 dB；RTX 5070 上对官方 DLSS-G 输出：赛博朋克 2077 49.8 / 53.5 dB，黑神话悟空 53.2 / 56.3 dB。偏差随画面运动变大，不随历史累积：同一份抓取从中途新建特性重放，逐帧差在 ±0.5 dB 以内。
- **对真值**：五份抓取的跳帧真值上，全帧比档位 1 低 0.02–0.13 dB（五份都是“劣”）；94 648 个区块里比档位 1 差 1 dB 以上的 229 个、好 1 dB 以上的 36 个。把两者差得最多的区块（两个方向都挑）并排看，包括 Resonance 最快的那一段与地平线 6，**看不出差别**：档位 1 碎的地方档位 2 一样碎，位置也一样。
- **速度**：RTX 3070，4X，每生成帧比档位 1 快 1.8 %（1080p）/ 1.8 %（1440p）/ 2.9 %（4K，2207 → 2142 µs；`sm86_bench --hooks 0`，0.4.0）。0.4.0 之前是 2.6 / 2.9 / 4.5 %（2379 → 2272 µs，旧测法带逐 launch 打点与每个 index 一次真实帧拷贝），差距变小是因为 0.4.0 的精确补丁让档位 1 的 `Kernel_OutputPull` 快得更多：P8/P9 快了 58 µs，A6 本来就是 40 寄存器，只多拿到 P8 那一半；P10 在 bench 上让档位 1 再快 20 µs，而 A6 只剩两次取样可以提前，bench 上 1080p / 1440p 持平到略慢（每生成帧 +0.3…+2 µs）、4K 快 3 µs——真实 4K 游戏抓取上两档从 P10 拿到的差不多（这个内核 −4…−9 µs）。这个内核自己快约 16%（5070），但它只是生成帧里的一部分，帧率变化很小是预期。

**这一档曾经出厂（2026-10-04 到 10-21）的理由是“并排看不出差别”，不是“离官方近”；2026-10-21 起业主把出厂改回档位 1。** 档位的判据仍是“离官方输出多远”，按这个判据档位 2 在最快的运动上离官方 44 dB，所以它不逐位一致、也不再用“约 50 dB 以上”来描述；业主按并排目视判断它与档位 1 没有可见差别，换来每生成帧 3–5 %。要和官方画面逐位对照，用档位 1。

档位 3 再加两行实验性的：A1（`Kernel_WarpBlendingWeights`，40.5 / 46.8 dB 与 53.0 / 58.9 dB；4K 上这个内核快 12%、1080p 只快 0.9%，而它占一个生成帧的 2%）和 A5（`Kernel_EstimateIntermMvecsScatter`，scatter 半径钳到 3，52.9 / 62.7 dB 与 56.0 / 67.5 dB；合成场景上收益为 0，收益完全取决于画面内容，**从未在游戏内容上量过**；它同时是 MFG 相位内核，开它之后插值相位要重新核对）。用 `ImageApproxMask` 可以逐行点名。

**RTX 20（SM75）上这两档一样生效**：每一行都有 sm_75 镜像，后端武装它们时不看内核族。

怎么确认它到底武装了没有（把 `[Logging] Level` 临时改成 `2`，看日志里后端的记录）：

- 一条 `image_approx`：`enabled`、`optimized_tier`、`mask` 与 `mask_from`（INI 指定还是档位默认）、`rows`、`armed[]`（每行的入口名、标签 `A1`/`A5`/`A6`、槽位与资源号）、`exact_patches_replaced`。`enabled` 只对真正会运行的行为真：要求了却不会运行（档位 0、`KernelImage=Original`、路由没启用）时 `enabled=false`，`disabled_reason` 写明原因。
- 每武装一行各一条 `optimized_create`，带 `row="image_approx"`、`approx="A6"`、`entry`、`image`（`cubin_sm86` / `ptx_sm86` / `cubin_sm75` / `ptx_sm75`）和 `resource`。
- `install` 的 `optimized_tier` / `optimized_tier_name` 与 `optimized_knobs.image_approx` 说明档位最后解析成了什么。

这些记录都在、而画面与帧率都没有明显变化，就是这一档做到了它该做的事，不是它没生效。

#### Preset：A / B / C 预设（UI 重组、FG Lite）

`[DLSSG] Preset`（2026-10-14 之前是 `[Compatibility] Preset`）是**本 mod 自己的 INI 键**，不是 NVIDIA Inspector / NVIDIA App 里的那个设置——在那些工具里改预设不经过本键；本键在每个特性上钉住 `A` / `B` / `C`，盖掉驱动 / NVIDIA App 的预设覆盖与游戏自己的参数；2026-10-22 起没有“让驱动说话”的值（`Auto` 已删除，按 `A`）。

驱动 / NVIDIA App 里的 DLSS 帧生成“渲染预设 A / B”在 310.9.1 内部只对应一个布尔量：UI 重组（`DLSSG.UserInterfaceRecompositionEnabled`）。`A` = UI 被烘进颜色历史一起插值（HUD 会被运动矢量拖影）；`B` = HUD-less 颜色与“UI 颜色+Alpha”分开插值（静止 HUD 不被拖动）。`B` 需要游戏同时提供 `DLSSG.HUDLess` 和 `DLSSG.UI`，否则画面与 A 一致但白占两张全分辨率 FP16 表面（1080p 约 33 MB、4K 约 133 MB）。每次创建特性记一条 `preset_pinned`。

**预设 C（FG Lite，310.9.16 起，可选）。** 310.9.16 多了一个开关“轻量候选混合”（游戏参数 `DLSSG.LiteCandidateBlendingEnabled`，驱动预设 `0x10E41DF1 = 3`）：每个生成帧不跑 DL2 网络和它的后处理，改跑 `Kernel_BlendCandidatesFGLite` + 两个小内核，UI 重组同时关掉。`Preset=C` 把 UI 重组与 Lite 两组覆盖值一起钉住（`preset_pinned` 多出 `lite_*` 字段）。3070 实测（`sm86_bench`，档位 2）每生成帧 2× −32…−36 %、4× −46…−54 %（4× 下 1080p 0.946 → 0.437 ms、4K 2.167 → 1.181 ms）；画质对跳帧真值全帧 −0.20 / −0.27 dB，HUD 区块 −1.84 dB（赛博朋克，强制关 UI 重组），细节能量更高（更锐、遮挡处更块状）。**没有官方参照**：官方运行库在 RTX 30 上不跑，预设 C 对官方的逐位比较要一张 40 / 50 卡，目前只证明了本项目各条路线之间自洽（cubin / PTX / SM75 / 档位 0 逐位相同）。出厂是 `A`，C 只在写了才生效：310.9.16 自己会在游戏设了 `DLSSG.LiteCandidateBlendingEnabled` 或驱动预设是 `0x10E41DF1 = 3` 时打开 Lite，后端每次创建特性都把 Lite 这一组钉成 INI 的值（`A` / `B` 下是 0），`preset_pinned` 的 `lite_game_value` / `lite_drs_value` 写明游戏 / 驱动要的是什么，`lite_effective` 是实际跑的。运行库的这几组覆盖值布局与后端认识的不符时记 `preset_layout_unexpected`，帧生成被拒绝。

**两张平面交不交由游戏决定，不由这个键决定。** 本机的两份 4K 抓取正好是两种情况：黑神话悟空两张都交，`Preset=B` 真的改变画面；赛博朋克 2077 只交 `DLSSG.HUDLess`、不交 `DLSSG.UI`，在它上面写 `Preset=B` 是空操作。想知道自己的游戏属于哪一种，用 `scripts\game\dlssg_diag_capture.cmd` 录一份抓取，报告 `== 抓取 ==` 一节里的 `HUD 平面:` 一行直接写明游戏交出了哪几张。**这一行需要抓取才能给出**：普通运行的日志里没有任何记录能回答它。

**它不是细结构闪烁的解法。** `Preset` 只决定 UI / HUD 走哪条路，场景内容（植被、栏杆、细线条）本身的插值不受它影响。它对应的是 HUD / 浮空 UI 抖动那一类现象，见下一节。

#### 已知的画面瑕疵与它们的归属

**细结构闪烁（植被、草、栏杆、文字）、遮挡边缘的重影与画面左右边框伪影、转视角时的拖影**：这些是运行库自己的插值算法在做的事，不是本项目引入的。判据是逐位比较，**在档位 1 上做**——档位 1 是逐位一致的参照档：`Optimized=1` 下，两份 4K 真实游戏抓取（共 120 次 Evaluate）离线重放的输出与**官方 NVIDIA 运行库在同一批输入上的输出逐位相同**：`differing_pixels 0`、`max_abs_diff 0`，320 张平面全中。同样的输入产出同样的像素，所以这一类现象在官方路径上同样存在。档位 2（2026-10-04 到 10-21 的出厂档位）在档位 1 上只多换了一行图像内核（A6），输出不再逐位一致，但与档位 1 的差别并排看不出来，这一类现象在两档上以同样的样子、在同样的位置出现（上一节）。怀疑某个现象是本项目造成的，先确认 `Optimized` 是 `1`（出厂值）再看。多位报告者一致说**倍率越高越明显**；但“基础帧率低”不是唯一因素——也有人在基础帧 90 以上、以及 4X 下 155 fps 时照样看到（公开 issue #61 / #493）。用户侧唯一有过闭环报告的缓解手段是降低 `MaxGeneratedFrames`（issue #82，2X），而 issue #171 报告 2X 下仍然有。

**整屏闪烁 / 频闪是另一回事**，它与画面内容无关。公开仓库里已经闭环的几例分别来自：RTSS / MSI Afterburner 的覆盖层（issue #99）、另一个改画面的 MOD（体积雾，issue #523）、交换链模式（切一次窗口化再切回、或打开垂直同步就消失，issue #60 / #68 等多条）。这几例都是报告者自己确认的闭环，不是本机实测。issue #532（绝区零，0.3.2）**没有闭环**：公开帖子里报告者没有确认过任何改动的效果。当时怀疑的 `SkipRepeatedRealCopy` 在 Streamline 游戏里不起作用：绝区零带的 `sl.dlss_g` 是 2.11.1（版本号来自同帖另一位用户的日志），本机验过的 2.11.1 与其余 8 个构建一样只在 index 1 交出 `OutputReal`。在绝区零那份是 NVIDIA 签名原版的前提下，闪屏不是这个键造成的；原因至今没有定位。遇到整屏闪烁按这个顺序排除：关掉覆盖层 → 只留本 mod、移除其它改画面的 MOD → 切一次窗口化 / 打开垂直同步。

**HUD / 浮空 UI 抖动**走的是另一条路径，见上面 `Preset` 一节。

**要把某一条现象真正定性，只能靠抓取。** 按 `docs/CAPTURE.md` 在现象正在发生时录一份（`scripts\game\dlssg_diag_capture.cmd`，按 Insert），同一机位再录一份 `Optimized=0` 的做对照，就能在一张受支持的显卡上离线重放并与官方运行库在同一批输入上的输出逐像素比较：官方输出同样有 = 运行库算法的性质；官方输出没有 = 本项目的缺陷，而且有了逐像素的定位起点。至今没有收到过任何一份画质抓取（`docs/COMPAT.md` 缺口 G8）。

**第三方分支里的画质开关**（例如 `QualityValidWarp`）改的是 NVIDIA 自己的候选混合内核，输出因此不再与官方运行库一致，上游三个项目对它的画质测量次数是 0。本项目**实现过并量过，然后撤回了**：在 11 份真实游戏抓取、14 组配置上，非 HUD 区域 13/14 组测不出改善，HUD 区块变坏，而在唯一一份真正会闪的抓取上 4X 的交替量也变坏。所以本项目**没有**这类键。走同一个内核但**不引入任何判据**的重采样改动是另一回事，见 `CatmullRomWarp`。

## 如何确认配置生效

在 Level 2 或 3 的日志中检查（出厂 `Level=1` 只记错误，验证配置时临时改成 `2` 或 `3`）：

| 事件 / 字段 | 含义 |
|---|---|
| loader：`configuration` | 本次读取的 INI、运行库模式、内核格式请求，以及 `optimized_tier`/`optimized_tier_name`（解析后的一致性档位）、该档位解出的每个旋钮（`optimized_kernels`/`image_patches`/`image_approx`/`image_approx_mask`）、`enabled`（帧生成这一组最终开没开）、`warnings`（`configuration_warning` 的条数）、`invalid`（`configuration_invalid` 的条数，有才写）、`proxies`（`{active, standby[]}`，哪个代理在干活）。 |
| loader：`configuration_invalid`（Level 1） | 某个键的值非法或越界（2026-10-19 起，C97）：`section`/`key`/`value`/`allowed`/`reason`/`effect`。不换成默认值、不钳位，这个键所属的功能整组不启用：帧生成的节 → 帧生成不启用（同 `Enabled=0`，请求原样转发）；`[SuperResolution]` 与 `[Runtime] SRMode` / `SRPath` → 超分不启用（同 `Mode=Off`）；`[Logging]` / `[Debug]` → 只关那一项诊断（`Level` / `File` / `Directory` / `EvaluateEvery` 保留默认）。`effect` 写明是哪一种。每个写错的键一条。 |
| loader：`configuration_warning`（Level 2；2026-10-23 之前是 Level 1，C106） | 不关任何东西的告警：`section`/`key`/`value`/`default`/`reason`/`effect`。几种：`[Runtime] Mode=Auto`（按 `Bundled`）、`[DLSSG] Preset=Auto`（按 `A`，`default` 为 `a`）、另一个引擎的键（`ignored under Engine=…`）、旧键名 `[Compatibility] ImageApprox`（`default` 为 `(honoured)` / `(ignored)`，`reason` 写出新键）与 `[DLSSG] Optimized` / `Preset` 不再读取的旧位置（与新键同写或写的是默认值时 `(no effect)`）与在当前配置下不起作用的键（`default` 为 `(no effect)`：档位 0 或 `ImagePatches=0` 下的 `CatmullRomWarp`、档位 0 下的 `ImagePatches=1` / `ImageApprox=1` / `DisableFusions`、没有有损行可选时的 `ImageApproxMask`）；`effect` 写明是哪一种。已删除的键不读，不在这里出现（2026-10-26，C113）。 |
| loader：`configuration_error`（Level 1） | 整份配置根本读不了（路径无法构成、内存不足），mod 关闭并保留游戏原始加载。**值写错不会走到这里**，那是 `configuration_invalid`。 |
| loader：`diagnostic_environment`（Level 1） | 紧跟在 `configuration` 之后，只在设了改变行为的诊断环境变量时出现：`names`、`values`。这些变量发布版照读（`DLSSG_FAKE_HOST_SM`、`DLSSG_FAKE_TURING_HOST`、`DLSSG_FORCE_CUBIN_FAIL`、`DLSSG_FORCE_KERNEL_FAIL`、`DLSS_SM86_FORCE_IMAGE_FAIL`、`DLSSG_CAPTURE_WRITER_DELAY_MS`），所以有这一条的日志不是正常运行。`DLSSG_SM75_SLOTS`、`DLSSG_MFG_ARCH_ANY`、`DLSSG_FUSION_BARRIERS` 与只加日志的 `DLSSG_GPU_CALLERS`、`DLSSG_CU_CHAIN_LOG`、`DLSSG_MFG_DIAG` 2026-10-21 删除，`DLSSG_SR_SKIP_CU_HOOKS`、`DLSSG_XEFG_CUDA_IMAGE`、`DLSSG_XEFG15_DXIL` 2026-10-22 删除（C102），不再读取。 |
| loader：`spoof_policy_mismatch`（Level 1） | 启动时那次 kernel32 早读与完整解析读到的 `[General] Enabled` / `SpoofArchToGame` / `SpoofArchValue` 不同：`peek`、`parsed`。早读的决定对整个进程有效，这条只记录、什么都不改。 |
| loader / backend：`max_generated_frames_unsupported`（Level 1） | `MaxGeneratedFrames` 做不到，帧生成不启用或被拒绝，从不降：`engine:"xess"`（超过 XeSS-FG 的上限 3）；`reason:"runtime_kernels"`（RTX 40 跑运行库自己的内核，即 `KernelImage=Original`）；`reason:"driver_profile_limit"`（钩点未知的运行库构建上驱动 profile 的限制更低）。带 `max_generated_frames`、`supported`、`effect`、`remedy`。 |
| backend：`mfg_drs_override` | 每进程一条：后端钩住运行库的 DRS 读取，去掉驱动 profile 的帧数限制（私有键 `0x104C9A99`），`MaxGeneratedFrames` 优先。`present`、`value`、`max_generated_frames`。这个键真的被设过（`present:true`）时是 Level 1，否则 Level 2。 |
| loader：`proxy_forward_incomplete`（Level 1） | 本机 System32 里的那个系统 DLL 没有导出代理转发表里的全部名字（Windows 版本比打包机器旧，或 Wine/Proton）：`library`/`resolved`/`missing`/`missing_names`。这几个转发器会答 `ERROR_PROC_NOT_FOUND`，**其余转发与 mod 本身不受影响**。以前缺一个名字就让 DllMain 失败、游戏直接 `0xc0000142` 起不来且不产生任何日志。 |
| loader：`runtime_redirect` | 这一次 `nvngx_dlssg.dll` 的请求是由谁来答的：`mode`（`bundled`/`external`/`pinned`）、**`selected_source`（同一件事的四值写法：`bundled`/`external`/`pinned`/`game_own`，回退之后它说的是真正跑的那一档）**、`requested`、**`source`（请求是从哪儿来的：`game_dir` 游戏自己的安装目录 / `ngx_model_store` NGX 模型库 = NVIDIA App 的 DLSS 覆盖开着 / `driver_store` 驱动库里的那份 / `other`，裸文件名也算 `other`，真正解析到的路径在 `game_runtime.path`/`resolved_by`）**、`selected`、`game_runtime_used`，以及 **`game_runtime`——被替换掉的那个文件（游戏自带的那份）的身份**：`path`/`resolved_by`/`bytes`/`sha256`、`file_version`（该信的那一个）与它的两个来源 `file_version_fixed`（版本资源固定块）/`file_version_string`+`product_version_string`（`StringFileInfo` 字符串）、`snippet_version`（`NVSDK_NGX_GetSnippetVersion` 的立即数，`0x01360901` = 310.9.1）、`generation`（`A`/`B`/`unknown`）与 `generation_marker`（哪个判据定的）。读不了就只是 `available=false` + `reason`，重定向照常。 |
| loader：`external_runtime_rejected`（Level 1） | `[Runtime] Mode=External` / `SRMode=External` 指定的文件用不了：`feature`（`frame_generation` / `super_resolution`）、`key`、`configured`（INI 里怎么写的）、`path`（展开并解析之后的全路径）、`sha256`、`expected_sha256`/`expected_version`、`runtime_version`（认识它时把它叫出名字，比如"它是 310.1.0.0"）、`reason`、`action`（`frame_generation_not_enabled: …` / `super_resolution_not_enabled: …`）：这次加载被拒绝，不回退（2026-10-20 起；此前 `bundled_runtime_used` / `bundled_snippet_used` 并另有一条 `configuration_warning`）。 |
| loader：`bundled_runtime_unavailable`（Level 1） | 每进程一条：`[Runtime] Mode=Bundled`，但 `dlssg_sm86\runtime\nvngx_dlssg.dll` **不在**（最常见：只复制了代理 DLL）或**不是这个构建的那一份**。带 `path`（看的是哪个文件）、`sha256`（读到的摘要，文件不在时为 `null`）、`runtime_sha256` / `runtime_version`（该有的那一个）、`found_version`（拿错的是本项目另一个发布包的运行库时会直接叫出名字）与 `remedy`。此时这次 `nvngx_dlssg.dll` 的加载被拒绝，不会改用别的文件，也不交回游戏自己的那份（2026-10-20 起）。 |
| loader：`proxy_multiple`（Level 1） | 进程里有不止一个本项目的代理（2026-10-20 起）：`active`、`standby[]`。行为不变：active 是 mod，standby 只转发。删掉多余的那几个。 |
| backend：`preset_ineffective`（Level 1） | `Preset=B` 钉住了，但游戏的 Evaluate 没交 `DLSSG.HUDLess` 和 / 或 `DLSSG.UI`：`hudless`、`ui`、`missing[]`、`handle`、`evaluate`、`effect`（运行库照自己的行为，生成帧与不写 `Preset=B` 相同）。每个特性句柄只在第一次 Evaluate 查一次、记一次（2026-10-19 起不再每 64 次复查，`preset_effective` 删除）。 |
| loader：`preset_unavailable`（Level 1） | 只在 `-RuntimeModel 310.9` 的开发构建上：它的运行库 310.9.1 没有 FG Lite，`Preset=C` 无法生效，每进程一条，这次加载被拒绝、帧生成不启用（2026-10-20 起；此前跑运行库自己的预设）。发布包是 310.9.16，不会出现。 |
| loader：`backend_loaded` | 每进程一条，成败都写：`path`（`dlssg_sm86\backend\sm86_backend.dll`，相对代理自己的目录解析）、`sha256`（这份后端的实际摘要，**这是"答话的是哪一份后端"唯一的答案**）、`loaded`/`load_error`、后端自报的 `abi`/`id`/`runtime_sha256`，以及这个包期望的 `expected_runtime_sha256`/`expected_runtime_version`。 |
| backend：`kernel_pack` | 每进程一条（只在需要替换内核的安装里）：用的是哪个内核包、它与这个运行库是否配套。`path`、`manifest`（包的 id）、`model`/`schema`/`fusion_abi`、`images`/`image_bytes`、`cubins`（包里有没有 cubin）、`kernels`/`variants`/`patches`/`approx_rows`/`detail_rows`、`runtime_blobs`/`kernels_matched`/`kernels_unused`（已加载运行库里的内核按身份对上了几个，`docs/PACK_FORMAT.md`）、`scan_ms`、`selected_image` → `image`（包里没有 cubin 时 `cubin_sm86` 会解析成 `ptx_sm86`；**这不是拒绝**：没有错误记录）。2026-10-04 起取代 `kernel_cubin_pack`。 |
| backend：`kernel_pack_missing` / `kernel_pack_rejected`（Level 1） | 每进程一条：`dlssg_sm86\kernels\fg_kernels.dll` 不在（`missing`），或者不能用（`rejected`；`reason` 是 `manifest_missing`/`manifest_parse`/`tool`/`schema`/`fusion_abi`/`model`/`structure`/`detail_constants`/`missing_resource`/`resource_size`/`target`/`runtime_coverage` 之一，`detail` 说明具体哪一项）。带 `path`、`open_error`、`manifest`、`target_sm`/`actual_sm`、`effect:"route_inactive"` 与 `remedy`。此时本次安装不启用路由（`install.active=false`），游戏照常运行，**帧生成被拒绝**（2026-10-16 起：同时一条 `fg_kernels_unavailable`，每次创建帧生成特性失败并记 `fg_feature_refused`），任何显卡上都不改用运行库自己的内核。最常见的原因：只复制了代理和后端、没复制 `dlssg_sm86\kernels\`，或者拷进来的是另一个发布包（另一个运行库模型）的内核包。 |
| loader：`sr_gate.kernel_set_source`；backend：`sr_kernel_set.source`（拒用时 Level 1） | 超分的内核集从哪里来（2026-10-11 起）：`pack` = 发布的内核包 `dlssg_sm86\sr\sr_kernels.dll`，`directory` = `KernelSet` 指向的目录包。后端的 `source` 还带 `path`、`outcome`（`accepted` / `missing` / `rejected`）与拒用时的 `reason` / `detail`（`unmappable`/`manifest_missing`/`manifest_parse`/`tool`/`schema`/`runtime`/`structure`/`missing_resource`/`resource_size`/`index`/`set` 之一，`docs/PACK_FORMAT.md` 第三部分）。包不在时设备闸就不放行（`sr_gate` 的那一行写明找的是 `sr_kernels.dll`），等同 `Mode=Off`；包坏了而索引还在时 snippet 照换、路线整包拒用，INT8 Mode 下被判给 INT8 的特性在创建时被拒（`sr_feature_refused`，C93）。最常见的原因：只复制了 `nvngx_dlss.dll`、没复制 `sr_kernels.dll`。 |
| backend：`sr_resource_hook{installed:false}`（Level 1） | 超分读权重缓冲宽度的钩子没装上：别的模块（常见是覆盖层）先钩了 `ID3D12Device` 的 `CreateCommittedResource` / `CreatePlacedResource`。宽度核对做不了，只剩显卡架构这道闸，超分照常运行；不是本项目或游戏的故障。 |
| loader：`game_runtime_generation` | 每进程一条，说明游戏自带的运行库属于哪一代契约、这次替换因此损失了什么：`generation`、`sl_dlss_g_version`（进程里那个 Streamline 插件的版本，**按 `StringFileInfo` 读**，所以 SL 1.x 显示 `1.5.6.0` 而不是固定块里的 `1.0.0.0`）、`effect`、`lost[]`，以及 **`sl_dlss_g_path` + `plugin_match`（`leaf`/`ota`/`ota_override`/`none`）——那份插件是游戏自带的，还是 Streamline ≥ 2.11 从 NGX 模型库加载的 NVIDIA OTA 副本**（倍率上限由它自己的常量决定，本机地平线 6 / Resonance 跑的都是 OTA 那份 2.14.0）。**代 A 是 Level 1（错误级）**：2026-10-19 起代 A 的游戏帧生成不启用，`action` 写明 `frame_generation_not_enabled`、请求原样转发；代 B / unknown 是 Level 2。 |
| loader：`ngx_override_active`（Level 1） | 每进程一条，只在 `runtime_redirect.source=ngx_model_store` 时出现：**用户在 NVIDIA App 里打开了「DLSS 优设 - 帧生成」**（或收到过 NGX OTA），于是 NGX 核心从 `C:\ProgramData\NVIDIA\NGX\models\dlssg\versions\<n>\files\<arch>_<appId>.bin` 要 snippet。字段 `feature`、`requested`、`store_version`（模型库的版本目录，等于 snippet 自己的版本常量，`20318464` = 310.9.0）、`snippet_sha256`。**那份 snippet 我们照常接管**，但同一个开关通常还会把 Streamline 的帧生成插件换成 NVIDIA 的 OTA 版（`\ngx\models\sl_dlss_g_0\`），而倍率上限是那份插件自己的常量——本 mod 和 INI 都改不动（见 `plugin_match`）。 |
| backend：`install` | `actual_sm` 物理架构；`active` 本次是否启用路由；`image` 选定格式；`target_sm`/`router` 解析后的内核族；`optimized_tier`/`optimized_tier_name` 与 `optimized_knobs`（该档位真正落地的整组旋钮）；`kernel_pack` 是内核包的结果（`accepted`/`missing`/`rejected`，不需要替换内核时 `not_needed`）。 |
| backend：`image_approx` | 有损图像行的解析结果——`enabled`、`disabled_reason`、`optimized_tier`、`mask`/`mask_from`（INI 指定还是档位默认）、`armed[]`（每一行的入口名、槽位、资源号）、`rows_shadowed`、`exact_patches_replaced`。`catmull_rom_warp`、`image_patches` 两条记录同样带 `enabled`（只对会运行的行为真）与 `disabled_reason`（要求了却不会运行时的原因）。 |
| backend：`fg_kernel_fatal` / `sr_kernel_fatal`（Level 1） | 2026-10-16 起（`docs/COMPAT.md` C93）：本项目已选定的内核在特性建好之后出错（帧生成：驱动拒绝镜像、运行库建了内核包不认识的内核、优化行被拒等；超分 INT8：合同不符、权重镜像建不出来等）。这条之后游戏**按设计结束**，最后一行是 `fg_gate_crash`（异常码 `0xE0D55001`，`dlssg_kernel_fatal`）。不会悄悄换回运行库自己的内核。请把日志发回。（此前 CU 路径的入口名不符记 `cu_function_unrouted`，RTX 40 / 50 的镜像被拒记 `newer_gpu_route_fallback`，两者都已删除。） |
| backend：`fg_kernels_unavailable` / `fg_feature_refused`、`sr_kernels_unavailable` / `sr_feature_refused`（Level 1） | 选中了本项目的内核但内核包缺失、损坏或不是这个发布的：帧生成（或该超分特性的 INT8）**打不开**，CreateFeature 返回 `0xbad00001`，不改用运行库自己的内核。补回发布包里的 `dlssg_sm86\kernels\` / `dlssg_sm86\sr\`，或者显式选运行库自己的内核（`KernelImage=Original`；超分 `Mode=Off` 或 FP16 Mode）。超分还有一种原因：`reason:"device_not_on_list"` = 游戏渲染的那块卡不在名单上（`[SuperResolution]` 一节）。 |
| backend：`variants_unavailable_sm75`（Level 1） | SM75 路线在所选档位上缺内核包里的行（发布的包行数齐全）：帧生成被拒绝（CreateFeature 返回 `0xbad00001`）。 |
| backend：`optimized_create` | 每个替换进去的变体 / 图像行各一条：`row` 是 `variant`（重写变体）、`image_patch`（精确补丁）还是 `image_approx`（有损行），`approx` 给出 `A1`/`A5`/`A6` 这样的标签，另有 `entry`、`image`、`resource`、`status`。判断某一档到底武装了哪几行就看这条。 |
| loader：`backend_install` | `status=0` 表示后端安装成功；结合 `install.active` 判断是否启用了内核替换。`-100` = 后端不可用，原因见下一行。`kernel_pack`（2026-10-04 起）是后端在这次安装里对内核包的结论，经 `DlssgMod_GetInfo` 报回 loader：`outcome`（`accepted`/`missing`/`rejected`/`not_needed`）、`found`（文件映射上了没有）、包清单自报的 `model`/`schema`/`manifest`（被拒的包也照填——另一个运行库模型的包会在这里写出它是哪个模型）、`reason`（拒用理由，同 `kernel_pack_rejected`）；后端被拒（`-100`）时为 `null`。 |
| loader：`backend_unusable`（Level 1） | `backend_install` 报 `-100` 时紧挨着的那条，说清是哪一种：`loaded=false` + `load_error`（**最常见的一种：`dlssg_sm86\backend\sm86_backend.dll` 没被复制过来，或者杀毒软件 / Windows「智能应用控制」拦了它**，游戏侧表现为 `Bad Image 0xc0e90002`；上一行的 `backend_loaded` 会告诉你它去哪儿找的）、缺本发布的导出、ABI 不符（一个安装里混了两个发布包的代理和后端），或后端与被安装模块的运行库 SHA 不匹配。`remedy` 字段给出处理办法。 |
| `image=ptx_sm86` / `cubin_sm86` / `original` | 已选择的 SM86 内核格式，或未替换。 |
| backend：`driver_below_floor`（Level 1） | 2026-10-26 起：驱动低于 R580（CUDA 13.0），帧生成与 XeSS-FG 没有安装（`cuda_driver`、`floor`；`driver` 是 NvAPI 的驱动版本，有就写）。升级驱动。 |
| backend：`jit_cache` | 只在 RTX 40（PTX 一族，RTX 50 不用缓存）：一批 PTX 镜像的 cubin 缓存结果（`batch`、`images`、`disk_hits`、`compiled`、`wall_ms`、`directory`、`key`）。 |
| backend：`jit_cache_off` | 每进程一条，只在 RTX 50（物理 SM ≥ 10.0）：PTX 的 cubin 缓存不启用（`sm`、`reason`、`effect`），PTX 交给 NvAPI 由驱动编译（C131）。是预期状态，不是错误。 |
| backend：`preset_pinned` | `[DLSSG] Preset` 生效，每个特性一条：`requested`（`A`/`B`/`C`）、游戏与驱动给的值（`game_value` / `drs_value`，310.9.16 另有 `lite_game_value` / `lite_drs_value`）、实际生效值（`effective`、`lite_effective`）与 `overrode_runtime`。 |
| backend：`drs_dlssg` | 每进程一条（第一次 `PopulateParameters`）：**NVIDIA 驱动 per-game profile 里的七个 DLSS-G 键**，只读。`profile`/`profile_source`（`application` = 这个 EXE 自己的 profile，`current_global` = 全局那一份）/`profile_settings`，`keys{<id>:{name,consumer,value,location,predefined}}`，以及单独拎出来的 `plugin_limit`（`0x104D6667`，Streamline 插件读的"请求帧数"）与 `runtime_limit`（私有键 `0x104C9A99`，运行库每次 Evaluate 自己读）。**单位是生成帧数**（NVIDIA App 的 `4x` 写成 `3`）。没被写过的键是 `value:null` + `status:-160`（`NVAPI_SETTING_NOT_FOUND`），**不是 0**——一台没动过的机器七个键全是这样。`plugin_limit` 是插件自己读的请求，游戏照它选更低的倍率是允许的；`runtime_limit` 2026-10-19 起压不低 `MaxGeneratedFrames`（见 `mfg_drs_override`；钩点未知的运行库构建上是 `max_generated_frames_unsupported`）。 |
| backend：`mfg_probe` / `mfg_summary` | `[Debug] MfgProbe` 的记录（`ForceGeneratedFrames` 已删除），判定见 `docs/MFG_PROBE.md`。 |
| Level 3：`kernel_create` / `evaluate` | 每次内核创建实际用的格式与状态；实际生成数量、结果。 |

确认路由安装成功时同时检查 `install.active=true` 和对应的 `backend_install.status=0`。仅出现 `image=ptx_sm86` 不代表内核已创建或执行，还要看后续 `kernel_create` / `evaluate`。

### 游戏自带的运行库是哪一代：代 A 的游戏不启用帧生成

DLSS-G 在 `3.8.1 → 310.1` 之间换过一整套契约，而**游戏自带的 `nvngx_dlssg.dll` 是哪一代，决定了这一款最多能生成几帧**：

- **代 B（310.x）**：有 `DLSSG.MultiFrameCount` / `MultiFrameCountMax` 这一对参数，也就是多帧生成（MFG）。`MaxGeneratedFrames`
  在这些游戏上是真正起作用的（最大 5 = 6X）。
- **代 A（1.0.1 – 3.8.x，本机 Steam 库里 13 份自带运行库中有 5 份是这一代：巫师 3、光明记忆、蜘蛛侠重制、原子之心、三角洲）**：
  那一代**根本没有 `DLSSG.MultiFrameCount` 这个参数名**，所以游戏自带的 Streamline 插件不会写它，运行库就一直用默认值 1。

**2026-10-19 起代 A 的游戏帧生成不启用**（业主决定：这些旧版插件反复带来查不清的兼容问题），与 `MaxGeneratedFrames` 写几无关：
游戏对 `nvngx_dlssg.dll` 的请求原样转发（与 `[General] Enabled=0` 相同），每进程一条错误级 `game_runtime_generation`，
`action` 是 `frame_generation_not_enabled`。在 RTX 30 / 20 上这意味着这些游戏没有帧生成。此前我们照常接管它们、跑得起来
（巫师 3 DX12，自带 Streamline 1.5.6 + `nvngx_dlssg.dll` 3.1.13.0：三次会话 6009 / 7352 / 28757 次 Evaluate、96/96 个平面与官方
运行库逐位一致），但只能 2X，`MaxGeneratedFrames` 改不了，还少了 UI 重组（`Preset=B`）、Reflex Warp、动态 MFG 与"这一帧别插值"
这个逐帧开关（代 A 用 `DLSSG.EnableInterp` 表达，310.x 不读它）——这与"`MaxGeneratedFrames` 是要求"矛盾。

日志里怎么看：`game_runtime_generation` 的 `generation` = `A` / `B` / `unknown`，被检查的那个文件在 `game_runtime`（判据在
`generation_marker`）；代 A 是错误级，出厂 `Level=1` 也能看到。认不出代的文件（`unknown`）照常接管。
`scripts\game\dlssg_diag.cmd` 与 `python scripts\analysis\sm75_report.py` 都会把这一条讲成一句中文结论。

### 帧生成为什么打不开：`fg_gate_*`（始终开启，无 INI 键）

装上之后 `install` 一切正常、却一条 `kernel_create` / `evaluate` 都没有，说明游戏根本没有创建 DLSS-G 特性，或者创建在运行库建第一个内核之前就失败了 —— 这两件事我们原来的日志分不出来，更说不出原因。决定权全在运行库之上：Streamline 自己的闸门（**硬件加速 GPU 计划（HAGS）关闭** → `eFeatureNotSupportedHWSchedulingDisabled`、驱动低于插件要求）、NGX 核心对 `GetFeatureRequirements` 的最终答复（核心会在运行库的答复之上再加自己的显卡 / 驱动 / HAGS 判断）、插件读的 NGX 能力参数，以及游戏自己的 UI 开关。`src/backend/ngx/fg_gate.cpp` 只观察这四处，五条记录都是 Level 2 可见（只有失败的 CreateFeature 是错误级，Level 1 也会记）：

| 事件（backend） | 内容 / 怎么读 |
|---|---|
| `fg_gate_environment` / `mfg_modules` 里的模块版本 | 每个模块的 `file_version` 取的是**该信的那一个**：版本资源固定块与 `StringFileInfo` 不一致时（Streamline 1.x 的固定块一律停在 `1.0.0.0`，真版本只在字符串里），记的是字符串里那个，并额外带 `file_version_fixed`/`file_version_string` 两个字段说明分歧。两者一致时只有 `file_version`，与旧版本日志完全相同。 |
| `fg_gate_environment` | 安装时一条。`hags`（`D3DKMTQueryAdapterInfo` 的 `KMTQAITYPE_WDDM_2_7_CAPS`：`supported`/`enabled`/`enabled_by_default`，`state=on\|off\|unsupported`）、`driver`（`NvAPI_SYS_GetDriverAndBranchVersion`，如 `610.74` / `r610_00`，外加已加载 `nvapi64.dll` 的文件版本）、`windows`（`RtlGetVersion`）、`ngx_core`（`_nvngx.dll` 路径+版本）、`streamline`（已加载的 `sl.interposer.dll` / `sl.common.dll` / `sl.dlss_g.dll` / `sl.dlss.dll`，只用 `GetModuleHandle` 看，绝不加载）。**`hags.state=off` 基本就是答案**。每一项都可选，取不到就写 `available=false` + 原因，永远不会让安装失败。 |
| `fg_gate_hags_disabled`（Level 1） | 上一条里唯一**既是用户可改的 Windows 设置、又能单独让帧生成在所有游戏里都打不开**的那一项：HAGS 本机支持但被关掉了。因为出厂 `Level=1` 根本不写 `fg_gate_environment`，这条单独按错误级记录，任何日志等级都能看到；`remedy` 字段写着怎么开。（公开 issue #513：有人为此排查了三个小时。） |
| `fg_gate_hooks` / `fg_gate_hooks_deferred` | 钩住的是 **NGX 核心 `_nvngx.dll` 的公开导出**（`GetFeatureRequirements` / `GetCapabilityParameters` / `GetParameters` / `CreateFeature`），四个导出各自报 `attached`。核心还没加载时先监视 `LoadLibrary`（`fg_gate_hooks_deferred`），等它出现再装；记录里还带一次 `ngx_core` 版本。 |
| `fg_gate_requirements` | 每次核心的 `GetFeatureRequirements`（最多 8 条）。`feature_supported` 是**调用方真正拿到的**那个值，`decoded` 按位解开（0 支持 / 1 显卡不支持 / 2 驱动版本不支持 / 4 系统版本不支持 / 8 硬件计划关闭），另有 `min_hw_architecture`、`min_os_version`、`caller`（调用模块+模块内偏移）与 `adapter_description`。注意与 backend 自己的 `feature_requirements` 区分：那条只是运行库对我们桥接的答复，核心还会在上面再加自己的判断。**`status_decoded` / `status_reason`** 说明这个状态是运行库自己那个函数里的哪一条路：310.x 的 `GetFeatureRequirements` 一共只能返回三个值——`1`（`success`）、`0xbad00002`（`nvapi_enum_physical_gpus_failed`，它五项检查里**唯一**会让整个调用失败的那一项）、`0xbad00005`（`input_is_nullptr`）。**`success` 不等于"支持"**：系统版本过低、`NvAPI_GPU_GetArchInfo` 失败、`getDeviceID` 失败这几条路都是**返回成功、把结论写进 flags 里**，所以要看的是 `decoded` 而不是状态。别的值（例如 `0xbad0000b`）说明答这个调用的不是 310.x 的运行库。 |
| `fg_gate_capability` | 每次能力查询后（最多 4 条）读回 `FrameGeneration.Available` / `.NeedsUpdatedDriver` / `.MinDriverVersionMajor` / `.MinDriverVersionMinor` / `.FeatureInitResult`、`DLSSG.MultiFrameCountMax`、`DLSSG.ReflexWarp.Available`，以及作为对照的 `SuperSampling.Available`。`Available=0` 或 `NeedsUpdatedDriver=1` = 插件在我们这边任何东西跑起来之前就拒了。`.FeatureInitResult` 是核心为这个特性记下的 NGX 状态：`3134193675` = `0xBAD0000B`（`NVSDK_NGX_Result_FAIL_OutOfDate`），含义与下一行的 `CreateFeature` 失败完全相同，只是核心早在能力查询阶段就记下了，插件看到 `Available=0` 就再也不会调 `CreateFeature`（RTX 2080 Ti / 巫师 3 实测）。 |
| `fg_gate_create_feature` | 每次核心的 `CreateFeature`（最多 16 条）：`feature_id`（DLSS-G = 11）、`status`、`succeeded`、`handle`、`caller`，DLSS-G 还带 `parameters`（`multi_frame_count` / `multi_frame_count_max` / `width` / `height`）。**失败是错误级**，`0xbad0000b`（`NVSDK_NGX_Result_FAIL_OutOfDate`）= 核心拿它在自己初始化时从 NVAPI 读到的显卡架构，去比 DLSS-G 运行库（snippet）导出的 `NVSDK_NGX_GetGPUArchitecture`（= 运行库声明的最低架构）后拒绝。后端把那个导出报成**物理架构**（`turing_host.gpu_architecture_export`，Turing 上 `0x160`；之前报的是 Ada `0x190`，那正是真 Turing 上帧生成打不开的原因）。`SpoofArchToGame` 管的是另一道闸门（游戏 / Streamline 自己的），对这一条没有帮助：重定向现在在游戏启动时就装好了（`trigger=dllmain`/`nvapi_load`/`dependency_load`），但核心这次比的不是 NVAPI 的答复，而是 snippet 导出的那个最低架构。 |
| `fg_gate_summary` | 一条，第一次 Evaluate 时（`reason=first_evaluate`）或进程退出时（`process_exit`）：requirements / capability / create 的次数与最后一次创建的状态。`create_calls=0` 而 `install` 一切正常 = 游戏从来没要过这个特性。 |
| `fg_gate_exit` | 一条，只在 `DllMain(DLL_PROCESS_DETACH)` 写：`evaluates`（总共跑了多少次 Evaluate）、`features`（创建了几次帧生成特性）、`last_evaluate_age_ms`（最后一次 Evaluate 距进程退出多久）、`clean:true`。**被杀掉或崩溃的进程走不到这里，所以日志末尾没有这条就等于"不是正常退出"**——上一条 summary 是第一次 Evaluate 就写掉的，单看它分不出"玩完退了"和"玩着玩着挂了"。老版本构建一条 `fg_gate` 记录都没有，不要误读成崩溃。 |
| `fg_gate_crash`（Level 1） | **进程崩溃时的最后一条**（`[Logging] CrashRecord`，缺省开）：`code` / `name`（如 `0xc0000005` `access_violation`）、`address`、`module`（出事模块的完整路径）+ `offset`、`role`（`backend` / `proxy` = 我们的代码，`ours:true`；`runtime` = 我们钩住的 NVIDIA 运行库；`other` = 游戏或别的模块；`unknown` = 不在任何镜像里）、`thread`、访问违例时的 `av{access,target}`、`chained`（之后还调用了原来的处理器）、`rearmed`。每进程最多一条。**有 `fg_gate_summary`、没有 `fg_gate_exit`、也没有这一条**：被杀掉，或崩在它覆盖不到的地方（游戏自己的崩溃报告器先处理掉了、`__fastfail`、调试器下，`docs/COMPAT.md` C89）。 |
| `foreign_fg_mod`（Level 1） | **进程里还有别的插帧 mod**（`docs/COMPAT.md` C88）：每个模块一条，第一次帧生成 `CreateFeature` 之后扫，之后有新模块加载时再看。`module`、`kind`（`rtx_encore` / `dlssg_transfusion` / `mfg_unlock` / `optiscaler` / `dlssg_to_fsr3` / `dlssg_sm86_other_copy`（我们自己的旧版或伴随插件）/ `reshade_fg_addon` / `second_backend`（第二份 `sm86_backend.dll`）/ `second_dlssg_runtime`（不是我们加载的那份 DLSS-G 运行库）/ `ngx_wrapper`（导出 NGX 入口的非 NVIDIA 模块））、`matched`（凭什么认出来：文件名 / 版本字符串 / 旁边的配置文件 / 导出名）、`version`、`when`。只记录，什么都不改。两个插帧引擎同时在一个游戏里，常见的结果是帧生成"开着"却一帧都不生成：只留一个。没有别的 mod 时一条都不写。 |
| `kernel_handle_recycled` | 每进程最多一条：驱动把某个内核句柄从 `old_rva` 换给了 `new_rva`（`had_variants` = 它当时还绑着上一个内核的变体集）。这是驱动的正常行为，每次创建都会重建该句柄的含义，所以它本身**不是问题**；累计数在 `evaluate.optimized.handles_recycled` / `stale_bindings_cleared` 与 `feature_create` 里。 |
| `stale_variant_rejected`（Level 1） | **本来应该永远不出现**：发射前的最后一道护栏拦下了一个属于别的内核的变体行，说明还有某条创建路径没有重置句柄。见到它请把日志发给开发者。 |

`python scripts\analysis\sm75_report.py <日志目录>` 的 `== fg gate ==` 一节把上面几条拼成一页，VERDICT 里多一行 `feature created`：创建成功 PASS、失败 FAIL（带状态）、从未调用 NOT RUN 并按证据给出最可能的闸门。

## 高级配置示例

以下示例仅在诊断 / 验证时用，正常安装不需要。把对应节和键补进出厂 INI 即可（`[Compatibility]` / `[Debug]` 节不存在就自己加）。

**3080 Ti 强制 PTX JIT（自动识别用的是 cubin；只在诊断时写）：**

```ini
[Compatibility]
KernelImage=PTX
```

**RTX 5070 验证 SM86 PTX 路径（开发用的验证配置）：**

```ini
[Compatibility]
KernelImage=PTX
ForceSM86Route=1
SimulateAmpere=1

[Logging]
Level=3
```

**取本机原厂数值参考（stock）：** 直接用出厂 INI 把 `Optimized` 改成 `0` 即可（Ampere 上不要用 `KernelImage=Original`）。

**开启生成帧标记（区分毛刺在生成帧还是真实帧）：**

```ini
[Debug]
MarkGeneratedFrames=1
```

**允许 6×：**

```ini
[FrameGeneration]
MaxGeneratedFrames=5
```

## 安装注意与卸载

游戏目录里只放一个代理：根目录的 `version.dll`（首选）；游戏不加载它时按 `alternatives/winmm.dll` → `alternatives/dbghelp.dll` → `alternatives/dinput8.dll` 的顺序换，**一次只放一个，换之前先删掉上一个**（判断方法见上面“代理只放一个”）。它们都从同一个 `dlssg_sm86\backend\` 加载同一个后端，只有文件名和转发目标不同。**`dlssg_sm86\` 这个文件夹必须跟着一起复制**——代理本身只是加载器。

渲染路径代理 `alternatives/dxgi.dll`、`alternatives/d3d12.dll`（可用，但风险更高）：它们在 D3D12 热路径上、加载顺序敏感，只在上面四个都没被游戏加载时才用，最后手段，两者只放一个。挑选说明随包放在 `alternatives/README.md`。

每个代理都把自己的全部导出转发给 `System32` 里的同名真实 DLL，只拦截 `nvngx_dlssg.dll` 的加载。

发布 DLL 的代码签名是可选的：`build.ps1 -SignCert <pfx> -SignPass <pw>`（或 `DLSSG_SIGN_CERT`/`DLSSG_SIGN_PASS`）会对本项目自己的每一个 DLL（根目录的 `version.dll`、`alternatives\` 的五个代理、`dlssg_sm86\` 下的后端与三个资源包）做 SHA-256 Authenticode 签名，NVIDIA 的运行库与 snippet 不碰；不提供证书则产物未签名。步骤、如何造一次性测试证书、以及自签名不提供默认信任等见 `docs/SIGNING.md`。

目前针对 Windows x64 / D3D12。本包的构建检查和可选 GPU 验证状态见 `manifest.json` / `validation.json`，架构见 `docs/ARCHITECTURE.md`，抓取与重放见 `docs/CAPTURE.md`。

卸载时退出游戏，移走放进去的那一个代理（`version.dll`，或换用的那个备用代理）、`dlssg_sm86.ini` 和 `dlssg_sm86\` 文件夹（日志也在里面）。原 NVIDIA DLL 及内核资源的归属见 `THIRD_PARTY_NOTICES.txt`；SM75 GPU 资源来自 Coldwood1026 的 RTX 20 系移植工作，在此致谢。
