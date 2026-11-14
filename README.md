# DLSSG for SM86（代理版）- 0.4.2 版本

**中文** · [English](README.en.md)

在 RTX 30 系列（SM86）和 RTX 20 系列（SM75）上启用 NVIDIA DLSS 帧生成（DLSS-G），并提供 RTX 30 / 20 上的 DLSS 超分 INT8 加速。Windows x64 / D3D12，运行文件为 `version.dll`、`dlssg_sm86.ini` 和 `dlssg_sm86\` 目录。

> **从 0.3.x 升级先看这里：0.4.0 是大版本升级，文件结构和安装方式都变了，不能直接覆盖。**
>
> 1. 先删干净旧版：退出游戏，删掉游戏目录里本项目的全部代理 DLL（0.3.1–0.3.6 让复制了 `version.dll`、`winmm.dll`、`dbghelp.dll`、`dinput8.dll` 四个，四个都要删）、`dlssg_sm86.ini` 和整个 `dlssg_sm86\` 目录。
> 2. 再复制新版的三样：`version.dll`、`dlssg_sm86.ini`、整个 `dlssg_sm86\` 目录。运行库、后端、内核包、XeSS 模型包和超分文件都在 `dlssg_sm86\` 里，只复制 DLL 什么都不会发生。
> 3. 代理任何时候只放一个：根目录只有 `version.dll`；游戏不加载它时，从 `alternatives\` 换一个（换，不是加）。
> 4. 用新的 `dlssg_sm86.ini`：键的位置变了（`Optimized`、`Preset` 移到 `[DLSSG]`，新增 `[XeSSFG]`），`[Runtime] Mode=Auto` 已删除。旧位置的 `Optimized` / `Preset` 不再读取：只写在旧位置且不是默认值（例如旧 INI 的 `[FrameGeneration] Optimized=1`）时帧生成不启用并记一条 `configuration_invalid`；改过的设置请在新 INI 里重新改。
>
> 详细步骤见下文「安装与升级」和安装手册 `MANUAL.html`。

## 本次更新说明

### 0.4.2

- **XeSS-FG 在 RTX 50（Blackwell）上的性能修复**：0.4.1 给 RTX 40 / 50 的是 compute_75 PTX，ptxas 在 sm_120 上把每条 `m8n8k16` 编成一条完整 `m16n8k16`、一半结果丢弃（主干卷积 144 条 vs 72 条、寄存器 79 → 124），Blackwell 的张量核内核因此比 Ampere 形状慢约一倍。0.4.2 给 26 个内核单独编 `.bw` 镜像（compute_86 PTX + 3×3 抽头配对成 `m16n8k32`），并把 RTX 50 上跑 DXIL 更快的 dispatch 收缩到 D25 与 model 15 的 D02 / D30。赛博朋克 2077 3440×1440（RTX 5070）：4X 每真实帧 6.54 → 4.68 ms、真实帧率 46.5 → 52.0 fps；2X 2.43 → 1.69 ms（57.8 → 59.8 fps）；同卡 DLSS 4X 为 3.98 ms。
- **XeSS-FG 支持抖动运动矢量（`DLSSG.MvecJittered=1`）**：3X / 4X 不再以 `xefg_kernel_fatal{unsupported_input}` 结束——D31 改跑 4 个 `jt_` 内核，携带 model 18 的抖动项 `D = (jp − jc) · (1 − t)`；不带抖动时与普通路径逐位相同。代价约每生成帧 +0.08 ms（1440p RGBA8）、4K +0.17 ms。
- **4K 优化**：model 18 的 D24 / model 15 的 D29 的 MV splat 换成本项目自己的 CUDA 内核；散射缓冲只清「本次或上次散射可能写到的前缀」。4K 每真实帧再快 3.0–5.6 %，输出逐位不变。
- **新增 `dlssg_sm86ini_Configurator.html`**：单文件、无外部依赖的 INI 配置器——按分区编辑常用键、实时预览生成的 INI、载入现有 INI（选择或拖放）、下载 / 复制、中英切换，并内置校验（`MaxGeneratedFrames` 1–5、`Engine=XeSS` 只能 1–3、旧 `Model=` 键提示等）。随发行包一起发布。
- **`[XeSSFG] UseHudless=Auto` 改为与 `0` 相同**：无论游戏是否提供 HUD-less 画面，后缓冲都作为网络输入，整幅画面（含 UI）一起插值（不再产生 `xefg_hudless_unused`）；`1` 仍是严格档（缺可用的 HUD-less 画面即 `xefg_kernel_fatal{hudless}`）。键的取值与出厂值不变（`Auto`）。
- 版本号 0.4.2；`README` / `MANUAL` / `INSTALL` / `COMPAT` 同步更新（`COMPAT` 新增 C139）。

### 0.4.1

- XeSS-FG 引擎（可选）：
  - 多帧生成（MFG）：`[FrameGeneration] Engine=XeSS` 做到 2X / 3X / 4X（`MaxGeneratedFrames` 1–3）。模型是 MFG-Unlocker 1.6.0 移植的 Intel model 15：2X 仍是 model 18 的图（比原来多 4–5 %），3X / 4X 走 model 15 自己的多帧路径，每个真实帧的第 i 张生成帧用相位 t = i / (N + 1)。`[XeSSFG] Model` 键删除（只有一个模型），旧 INI 里写了只记一条告警。
  - 张量核 CUDA 后端：移植版的主干层（3×3 INT8 卷积）是手写的 INT8 IMMA 内核（`mma.sync.m16n8k16`），权重用 `cp.async` 双缓冲预取；整套图由本项目自己的 D3D12 + NVAPI CUDA 运行器在游戏的命令列表上执行，不加载 Intel 的 `libxess_fg.dll`。
  - `[XeSSFG] FlowResolution=CoarseAt4K`：输出像素数 ≥ 3840 × 2160 时自动 `Coarse`，1440p 仍 `Middle`。
  - `[XeSSFG] UseHudless` 改为三态：`Auto`（出厂，逐帧判断）/ `1`（严格：缺 HUD-less 即拒绝）/ `0`（不用）。
  - 在 RTX 30 上：1440p 及以下比 DLSS-G 快 8–15 %，4K 起慢 27–30 %，见「性能」。
- `[SuperResolution] Mode` 改为显式预设：`Off`（出厂）/ `M-INT8` / `L-INT8` / `M-FP16` / `L-FP16`；0.4.0 的 `Mode=On` 删除，非法值是 `configuration_invalid`、超分不启用。
- `[DLSSG] Preset` 只剩 A / B / C：`Auto` 删除（按 `A` 跑），出厂 `A`。
- 一批键删除、不再读取：旧位置的 `[FrameGeneration] Optimized` / `[Compatibility] OptimizedKernels` / `[Compatibility] Preset`、`[Compatibility] AccelerateNewerGpus` / `ForceGeneratedFrames` / `SM75Family` / `HardwareBilinear` / `LaunchChains` / `PlainVariant`、`[Backends]` 等；写了只记一条告警，行为按出厂值。RTX 40 / 50 一律跑本项目的内核。
- 驱动要求变成硬性 R580：CUDA 13 构建的 cubin 被驱动拒绝时不再改跑 PTX，直接报错并拒绝帧生成（`driver_below_floor` / `kernel_image_unavailable`）。
- 超分 INT8 首帧卡顿修复：12 张权重镜像的上传移出第一个观察帧（此前那一次 Evaluate 的 CPU 时间比同预设 FP16 多 5–6 ms）。
- 显式开关做不到就拒绝，不再静默回退：`[Runtime] Path` / `SRPath` 指错文件、随包文件缺失、RTX 40 / 50 上 `KernelImage=Cubin` 等，都拒绝并写清原因。
- 日志：只有一个文本日志 `dlssg_sm86\logs\dlssg_sm86.log`（0.4.0 的每进程 `frames_<pid>.jsonl` 取消）；`[Logging] FrameTiming=1` 另写 `frame_timing.txt`。


### 0.4.0

- 超分 INT8（只限 RTX 30 / 20）：`[SuperResolution] Mode=On` 后，游戏用预设 L / M 的超分档位（多数游戏是「性能」和「超级性能」）换成本项目的 INT8 内核，预设不变。RTX 3070 上一次超分的 GPU 时间对同预设 FP16：L −6..−16 %，M −9..−22 %。出厂 `Mode=Off`。（`Mode=On` 2026-10-19 删除，现为 `M-INT8` / `L-INT8` / `M-FP16` / `L-FP16`。）
- DLSS-G 预设 C（FG Lite，可选）：`[DLSSG] Preset=C`。每个生成帧不跑第二个网络与后处理，RTX 3070 真实抓取上每生成帧 GPU 时间 2X −29..−34 %、4X 约 −45 %，画质略低（主要在 HUD 与遮挡边缘）。出厂仍是 `Preset=Auto`（`Auto` 2026-10-22 已删除，出厂改为 `A`）。
- XeSS-FG 引擎（可选）：`[FrameGeneration] Engine=XeSS`，用 Intel XeSS-FG 的模型生成插帧：Intel model 15（MFG-Unlocker 1.6.0 的移植）到 3X / 4X，2X 是 Intel model 18 的图；2026-10-26 起只有这一个模型，`[XeSSFG] Model` 键删除（不再读取）；`[XeSSFG] FlowResolution`（`Coarse` / `Middle` / `Fine`）、`UseHudless`。不带、也不加载 Intel 的 `libxess_fg.dll`，模型由本项目自己的 CUDA 内核执行。在 RTX 30 上比 DLSS-G 慢，见「已知问题」。出厂仍是 `Engine=DLSS`。
- 随包运行库换成 `nvngx_dlssg.dll` 310.9.16（默认预设下与 310.9.1 的输出逐位相同）。游戏自带哪个版本都不影响。
- INI 新布局：DLSS-G 专用的 `Optimized`、`Preset` 搬到 `[DLSSG]`，XeSS 专用的键在 `[XeSSFG]`；旧位置照旧生效，记一条 `configuration_warning`（2026-10-22 起旧位置不再读取，见上面第 4 条）。`[Runtime] Mode=Auto` 删除：写了按 `Bundled` 运行并记一条告警。
- 发布根目录只放一个代理 `version.dll`，其余五个在 `alternatives\`，一次只放一个（0.3.1–0.3.6 是四个都复制，升级时要删掉多余的，见「安装与升级」）。
- RTX 40 / 50：帧生成也换成本项目的内核（PTX，由驱动编译），`[Compatibility] AccelerateNewerGpus=0` 可换回 NVIDIA 官方内核（这个键 2026-10-19 删除，不再读取：RTX 40 / 50 一律跑本项目的内核）。实机未测。
- `[Logging] FrameTiming=1`：每个真实帧在日志目录写一行（帧间隔、超分与帧生成的 GPU 时间），用于反馈性能。
- 新增安装手册 `MANUAL.html`（英文 `MANUAL.en.html`）。

### 0.3.6

- 新增可选项 `[FrameGeneration] CatmullRomWarp=1`（默认 0）：把生成帧里的两次运动补偿取样从双线性换成 Catmull-Rom，细结构（栏杆、文字、远处窗格）在生成帧里更清楚一些，代价是每生成帧约 +3%（1080p / 1440p）到 +4%（4K），开启后画面不再与官方逐位一致。在 6 款游戏 11 份真实抓取上离线测得细结构能量提高约 5 个百分点、闪烁指标同步改善，肉眼改善轻微；不进任何 `Optimized` 档位，310.1 构建不支持。另有实验键 `DetailRestore`（C / D 两项），实测收益小或有副作用，不建议开启，说明见 `docs/INSTALL.md`。
- 日志记录游戏自带的 DLSS-G 运行库版本、哈希和代际（`runtime_redirect` / `game_runtime_generation`）。使用早期 Streamline（1.x–2.4，自带 DLSS-G 3.x）的游戏只能 2X：那一代插件不会向运行库写生成帧数，不是本项目的设置能改的；日志现在会直接写明。Streamline 1.x 的版本号此前一律记成 1.0.0.0，已修正。
- NVIDIA App 的「DLSS 优设」打开后，帧生成组件由 NGX 模型库提供，本项目照常接管；日志新增 `ngx_override_active` 标明这种情况。模型库里的 Streamline 在线插件现在也能被识别（此前 `ForcePluginFrames` 在这类进程里静默无效）。
- 发布包根目录放齐四个代理（`version` / `winmm` / `dbghelp` / `dinput8`）和 INI，与 `alternatives/README.md` 的说明一致；`310.1\` 也带 INI。
- 删除一个从未被调用的 NVAPI 桩（`SetRawScgPriority`，在全部运行库、NGX 核心与 Streamline 里都不存在）。
- `docs/INSTALL.md` 补充：`Preset` 的含义与生效条件、`Optimized=2/3` 为什么"看不出差别"、画面瑕疵的归属与整屏闪烁的排除顺序。

### 0.3.5

- 修复自 0.3.0 起就存在的一个缺陷：游戏重建帧生成特性（切菜单、改分辨率/设置、开关帧生成，部分游戏运行中会频繁重建）时，驱动会回收内核句柄，被回收的句柄可能仍绑着上一个内核的优化版本，之后有概率用错内核——表现为生成帧花屏，或玩一段时间后随机崩溃 / 显卡驱动重置（`Optimized=0` 不受影响）。3070 上 15 次重建的离线测试里，旧版 4 次有 3 次出错帧，新版每次逐位一致。
- 日志新增 `fg_gate_exit`（进程正常退出时写），日志里没有它 = 游戏是被杀掉或崩溃的。

### 0.3.4

- 修复 0.3.3 在 RTX 30 上的崩溃（进游戏或开 DLSS 后显卡驱动重置，《极限竞速 地平线 6》表现为 FHC01）。原因：NVIDIA App 的 DLSS 覆盖或 NGX 在线更新生效时，DLSS 超分模型从 NGX 模型目录加载，它也收到了本项目给 Streamline 的显卡架构改写，在 RTX 30 上走了不属于这张卡的内核路径导致 GPU 挂起。现在 NVIDIA 自己的组件（NGX 模型目录、驱动模块）一律得到真实架构，改写只对 Streamline 与游戏生效。
- 日志：`arch_spoof_applied` 按调用模块各记一条并带完整路径，新增 `arch_spoof_excluded`；三处小修复（缺失系统导出不再让代理加载失败 #10 / #65，`backend_install -100` 给出具体原因，HAGS 关闭单独报错 #513）。

### 0.3.3（预览）

- `[FrameGeneration] Optimized=1` 不再打开 `SkipRepeatedRealCopy`，任何档位都不再打开它；需要时手动写 `SkipRepeatedRealCopy=1`。它在合成场景与离线重放里逐位一致，但那是内核以外的假设（取决于游戏侧插件如何提交同一组的多次 Evaluate），从未在实际游戏中验证过，而 0.3.2 上出现了一例画面闪烁报告（《绝区零》，公开仓库 issue #532）。档位 `1` 仍与官方运行库逐位一致。
- RTX 20 系（Turing）：向游戏改写 NVAPI 显卡架构改为在游戏启动时就装好（本 mod 的 `DllMain`，或 `nvapi64.dll` 被加载时）。Streamline 2.x 在 `slInit` 里就判定「本显卡不支持 DLSS-G」，那时 DLSS-G 运行库还没被加载、此前版本尚未装好这道改写，2080 Ti +《最终幻想 7 重生》（Streamline 2.8）因此丢掉了帧生成插件。RTX 30 同样需要：一台 RTX 3060 Laptop 在同一款游戏上有完全相同的现象（公开仓库 issue #509，0.2.4 的 native 版对所有显卡都报 Ada，所以那一版正常），因此真实答案是 Turing 或 Ampere 时都改写成 Blackwell（RTX 50，两个版本都做多帧生成，游戏和 Streamline 的 3X / 4X / 6X 选项按这个架构放开），Ada / Blackwell 不动。
- 诊断：日志字段 `arch_spoof_installed.trigger` 新增 `dllmain` / `nvapi_load` / `dependency_load` 三个取值。

### 0.3.2

- 重写了部分推理内核（310.9 版）：生成画面与官方 DLSS-G 完全一致（RTX 3080 Ti 与 RTX 5070 实测逐位相同；RTX 2080 Ti 的输出与 3080 Ti 逐位相同），不再有损，并有些许加速（3080 Ti 上 0~8%）。
- 优化内核等级调整：`[FrameGeneration] Optimized` 现在是 `0`–`3` 四级。`0` 原厂内核，不加速；`1` 全部加速，画面与官方逐位一致（出厂默认）；`2` 再开有损图像内核，更快，对官方画面 PSNR 约 50 dB 以上（仅 310.9 版）；`3` 全部有损加速，最快。说明见 [`docs/INSTALL.md`](docs/INSTALL.md)。

### 0.3.1

- 安装简化：发布包根目录现在直接带四个代理（`version.dll`、`winmm.dll`、`dbghelp.dll`、`dinput8.dll`），**全部复制到渲染 EXE 旁即可，不用挑**。进程里第一个被游戏加载的那个成为本 mod，其余自动待机、只把导出转发给同名系统 DLL，不会重复安装帧生成。只有这四个都没被加载时，才从 `alternatives\` 里手动拿 `dxgi.dll` 或 `d3d12.dll`（二选一）。
- 出厂 `MaxGeneratedFrames` 由 `5` 改为 `3`（4×）。自带 Dynamic MFG 的游戏会默认跑到这里写的上限，`5` 对多数人偏高（公开仓库 issue #497 / #499）。要 6× 请用 310.9 版并把这一行手动改成 `5`；游戏自带旧版 4X 插件时无论写几都只有 4×。
- INI 写错某个键不再关掉整个 mod：该键回退到自己的默认值并记一条 `configuration_warning`（2026-10-23 起 `Level=2` 才看得到，C106；此前 `Level=1`），其余键照常工作（2026-10-19 起改了：写错的值是 `configuration_invalid`，该键所属的功能不启用，见下面出厂 INI 一段）。后端不支持某个开关时（例如 310.1 版上写 `Preset=B`）也改为**去掉那个开关继续安装**并记 `kernel_selection_unsupported{stripped_flags}`，而不是整个拒绝。
- RTX 20 系（Turing）恢复可用，用出厂 `dlssg_sm86.ini` 即可，不需要改任何键：内核族按物理显卡自动选到 SM75 一族。RTX 2080 Ti + 巫师 3 次世代版 DX12 实测可开启 2X（该游戏自带的 Streamline 插件上限就是 2X）；4X / 6X 与 30 系一样由游戏自带的插件决定。驱动需 R580 以上（2026-10-26 起是硬性要求，见 `docs/INSTALL.md`“驱动版本要求：R580”）。
- 0.3.0 在任何 RTX 20 上都开不了帧生成：它向 NGX 核心报告的显卡架构不是这张卡的真实架构，核心因此拒绝了帧生成特性（对应公开仓库 issue #491 / #492）。本次修改不影响 RTX 30 的数值与性能。
- 与 20 系有关的两个键 `SM75Family` 和 `SpoofArchToGame` 都不在出厂 INI 里，缺省值即为正确取值；需要时见 [`docs/INSTALL.md`](docs/INSTALL.md)。
- 帧生成打不开时的诊断：`[Logging] Level=2` 现在会记录是哪一道闸门拒绝的——硬件加速 GPU 计划（HAGS）、驱动版本、NGX 核心对能力查询的答复、以及 `CreateFeature` 的返回值。这一组记录名都以 `fg_gate_` 开头。
- RTX 20 上的画质与性能尚未测量：本版只证明帧生成能在 20 系上跑起来（数千次 Evaluate 零失败、零回退），没有 FPS 数据，也没有与 RTX 30 的画质对比。

### 0.3.0

- 由 native 模式回退到代理模式。native 化（自建 NGX host）在部分游戏上存在难以修复的兼容问题；本版改用代理 DLL 内嵌未修改的原厂运行库，游戏对 NGX 的调用不变，兼容性更好。
- 310.9 运行库新增 6X（`MaxGeneratedFrames` 上限由 3 提到 5）。在自身支持 Dynamic MFG 的游戏上，帧生成选「动态 / 自动」时会跑到 6X。
- 内核最优集（离线基准快约 19~32%）、两档出厂 INI、录制回放与诊断日志等见下文与 `docs/`。

## 声明与后续计划

- 插帧（DLSS-G）的内核级工作已基本达到现阶段的最优水平，暂不继续做内核优化；后续版本以兼容性与 bug 修复为主。
- 超分 L / M 的 INT8 已在本版提供（RTX 30 / 20）。
- XeSS-FG 引擎：model 15，2X–4X，未做性能优化；性能与 HDR（RGBA16F）后缓冲见「已知问题」。
- Vulkan 支持在项目当前状态下难以继续：封装层本身的游戏兼容性还不够好、覆盖测试也不多，再加上 Vulkan 只会带来更多兼容性问题。有需要的用户可以选用社区的 patch 版本。

## 运行要求

- 系统与游戏：Windows 10/11 x64、D3D12。
- GPU：RTX 30 系列（SM86）或 RTX 20 系列（Turing / SM75）。RTX 3070 上做开发验证与全部游戏内测试，RTX 3080 Ti 上跑过 0.4.0 测试包的完整测试；RTX 20 在 2080 Ti 上实测过帧生成（0.3.x）与一次超分 INT8 冒烟。RTX 40 / 50 也可用，帧生成走本项目内核的 PTX 路线，实机未测。
- 驱动：**R580 或更新**的 NVIDIA 驱动（CUDA 13.0），591.86、610.74、610.88、617.14 实测可用。更旧的驱动上帧生成不启用，日志里一条 `driver_below_floor`。
- 不需要 CUDA Toolkit，不需要 Python。

| 发布包 | 随包运行库 | 帧生成上限 | `version.dll` 大小 | SHA-256 |
|---|---|---|---|---|
| `dlssg-release-0.4.0-x64.zip` | 310.9.16.0 | 6X | 853,280 | `7b3f90435be4f94348bf8d128c84871c7f904c704f7690601a263277c327a7b2` |

代理 DLL 约 0.85 MB；运行库、后端与各资源包都是 `dlssg_sm86\` 下的文件：后端 `backend\sm86_backend.dll` 2.1 MB（不含任何内核），帧生成内核包 `kernels\fg_kernels.dll` 18.3 MB（必需），运行库 `runtime\nvngx_dlssg.dll` 7.5 MB，XeSS 模型包 `xefg\xefg_kernels.dll` 16.9 MB（model 18 与 model 15 的程序、权重和 CUDA 镜像），超分 `sr\nvngx_dlss.dll` 74.2 MB 与 `sr\sr_kernels.dll` 106.6 MB（两者成对，只在 `[SuperResolution] Mode` 不是 `Off` 时用到）。

## 不同配置的插帧额外显存参考

帧生成自身的本地显存增量（预热后减去创建特性前，向上取整到 10 MiB，310.9 版、最优内核，0.3.x 时测得）。该增量只随输出分辨率变化，与倍率无关：2X 与 6X 占用相同，倍率不额外占用显存。

| 画幅 | 输出分辨率 | 帧生成额外显存 |
|---|---|---|
| 16:9 | 720p | 约 230 MiB |
| 16:9 | 1080p | 约 350 MiB |
| 16:9 | 1440p | 约 540 MiB |
| 16:9 | 4K / 3840×2160 | 约 810 MiB |
| 21:9 | 2560×1080 | 约 440 MiB |
| 21:9 | 3440×1440 | 约 700 MiB |
| 21:9 | 5120×2160 | 约 1050 MiB |
| 32:9 | 3840×1080 | 约 610 MiB |
| 32:9 | 5120×1440 | 约 990 MiB |
| 4:3 | 1920×1440 | 约 430 MiB |

同高度不同宽度的占用接近（按输出像素数计）。该值为帧生成自身增量，不含游戏本体与合成输入纹理。`Engine=XeSS` 时整机专用显存比 DLSS-G 多约 410–530 MiB（RTX 3070，4K 输出，五款游戏实测）。

## 安装与升级

第一次装 mod 的，照发布包根目录的安装手册 `MANUAL.html`（英文版 `MANUAL.en.html`，双击用浏览器打开）一步一步做。简要步骤：

1. 完全退出游戏。
2. 进入游戏实际渲染的 EXE 所在目录（如《黑神话：悟空》为 `...\b1\Binaries\Win64\`）。不确定是哪个目录时，在游戏运行时打开任务管理器的「详细信息」，右键游戏进程 →「打开文件所在的位置」。
3. 把发布包根目录的 `version.dll`、`dlssg_sm86.ini` 和 `dlssg_sm86\` 这个文件夹复制进去；该目录已有同名 DLL 时先备份原文件。运行库、后端、内核包、XeSS 模型包和超分 snippet 都在 `dlssg_sm86\` 里，代理本身不到 1 MB，**只复制 DLL 不复制文件夹会什么都不发生**。`alternatives\` 平时不用复制。
4. 启动游戏，在图形设置中开启 DLSS 帧生成，选 2X / 3X / 4X（要 6× 见下面「6X 说明」，需要手动把 `MaxGeneratedFrames` 改成 `5`）。RTX 20 / 30 上原来灰掉的帧生成选项现在能打开，就说明装上了。
5. 要确认代理确实被加载（RTX 40 / 50 的帧生成本来就能开，只能这样确认）：把 `[Logging] Level` 改成 `2`，开着帧生成玩一会儿再退出，`dlssg_sm86\logs\dlssg_sm86.log` 里有这个游戏 EXE 名的行就是加载了（这个文件也保留以前各次运行的行，看这一次的）。出厂 `Level=1` 只在出错时写日志，`logs` 是空的或者没有这个目录都是正常的。
6. 游戏不加载 `version.dll`（`Level=2` 下开了帧生成，`dlssg_sm86.log` 里也没有这个 EXE 名的行）：退出游戏，删掉它，从 `alternatives\` 按 `winmm.dll` → `dbghelp.dll` → `dinput8.dll` → `dxgi.dll` → `d3d12.dll` 的顺序一次换一个放到 EXE 旁再试。任何时候只放一个；`dxgi.dll` 和 `d3d12.dll` 在渲染路径上，最后才用，两个不要同时放。说明见 `alternatives\README.md`。
7. 升级：退出游戏，先删掉旧的代理 DLL、`dlssg_sm86.ini` 和 `dlssg_sm86\` 目录，再复制新版这三样；改过的设置在新 INI 里重新改。0.3.1–0.3.6 让把 `version.dll`、`winmm.dll`、`dbghelp.dll`、`dinput8.dll` 四个代理都复制过去，升级时四个都要删：同一目录里有几个本项目的代理时，进程里第一个被加载的那个运行本 mod，抢到的若是旧版本，跑的就是旧版本。从 0.3.x 升级的人还可以删掉 `%LOCALAPPDATA%\DlssgSm86`——新版不再往那里写任何东西。
8. 卸载：删掉 `version.dll`（换过的就删换上的那个）、`dlssg_sm86.ini` 和 `dlssg_sm86\` 目录，备份过的原 DLL 放回去。

出厂 `dlssg_sm86.ini` 里玩家可能要改的开关：

- `[DLSSG] Optimized`：一致性档位 `0`–`3`。出厂 `1` = 全部逐位一致的加速（2026-10-21 起；0.4.0 出厂是 `2`）；`2` = 再加有损图像内核：不再与官方逐位一致，对着真值并排看与 `1` 没有可见差别，每生成帧快 1–3 %；`0` 为原厂数值；`3` 为全部有损加速。2026-10-14 之前在 `[FrameGeneration]`，旧位置不再读取（见上面第 4 条）。
- `[DLSSG] Preset`：出厂 `A`；这里写什么就跑什么，游戏或驱动配置要 B / C 也不变（`Auto` 已删除，写了按 `A` 并记一条告警）；`B` = 强制 UI 重组（游戏同时交出无 HUD 画面与 UI 平面时才有效）；`C` = FG Lite（更快，画质略低），同样用本项目的加速内核（随 `Optimized` 档位），游戏自带哪个版本的 DLSS-G 都能用。
- `[FrameGeneration] MaxGeneratedFrames`：出厂 `3` 对应 4X；改成 `5` 对应 6X。这是要求而不是上限：原样告诉游戏（驱动 profile 压不低它），游戏菜单可以选更低的倍率；做不到时（超过 5、`Engine=XeSS` 时超过 3）帧生成不启用并在日志里说明（`max_generated_frames_unsupported`），不会悄悄降。
- `[FrameGeneration] Engine`：出厂 `DLSS`；`XeSS` = Intel XeSS-FG 模型（model 15，2X 走 model 18 的图）：`MaxGeneratedFrames` 可以是 `1`–`3`，出厂的 `3` 直接可用；`[XeSSFG] Model` 键 2026-10-26 删除，写了不读。`[XeSSFG] FlowResolution`（出厂 `Middle`，越细越慢）与 `UseHudless`（出厂 `Auto` = 与 `0` 相同：整幅画面含 UI 一起插值；`1` = 一律要求，游戏没给就报错退出）只对 XeSS 生效。
- `[SuperResolution] Mode`：出厂 `Off`。RTX 30 / 20 上 `M-INT8` / `L-INT8` = 所有超分画质档（含 DLAA）都用预设 M / L，跑本项目的 INT8 内核，档位在游戏里选；`M-FP16` / `L-FP16` = 同一预设、NVIDIA 自己的内核（对照用）。画质 / 平衡 / DLAA 档用 M / L 比游戏自己的 K 慢，加速在性能 / 超级性能档。只有这五个值，写别的（包括 0.4.0 的 `On`）超分不启用并记一条 `configuration_invalid`；旧键 `Tier` 不读。RTX 40 / 50 上不起作用。

出厂文件里其余几项（`Enabled`、日志、运行库来源 `[Runtime] Mode=Bundled`）一般不用动；更多诊断/兼容项不在出厂文件中、取安全默认，完整清单见 [`docs/INSTALL.md`](docs/INSTALL.md)。某个键的值写错时不会换成默认值：日志记一条 `configuration_invalid`，该键所属的功能不启用（帧生成的键 → 帧生成不启用，同 `Enabled=0`；超分的键 → 超分不启用；日志 / 调试的键 → 只关那一项诊断）。

## 杀软误报与签名

发布包里本项目自己的 DLL 都做了代码签名：根目录的 `version.dll`、`alternatives\` 下五个备用代理，以及 `dlssg_sm86\` 下的后端和三个资源包（帧生成内核 `fg_kernels.dll`、超分内核 `sr_kernels.dll`、XeSS 模型 `xefg_kernels.dll`），共 10 个（`dlssg_sm86\runtime\` 与 `dlssg_sm86\sr\` 里 NVIDIA 的运行库和超分 snippet 保留 NVIDIA 自己的签名，不动）。自签名证书仅验证签名者身份与文件完整性，不提供 Windows 默认信任；首次运行 Windows SmartScreen 仍可能提示「未知发布者」——这是基于信誉的提示，与杀软报毒是两回事。本次使用自签证书 `CN=DLSSG for SM86 (self-signed)`，SHA-1 指纹 `85BA66762F851E49148D706915D09026281418E6`；可在文件属性→数字签名或用 `signtool verify /pa` 核对签名者与指纹。

## 性能

下面的数都来自本项目自己的计时（每次 Evaluate 前后的 D3D12 时间戳，或 `[Logging] FrameTiming` 日志）。离线 GPU 毫秒数衡量的是帧生成 / 超分自身的开销，不等于游戏内帧率增幅。

### 帧生成：真实游戏抓取（RTX 3070，离线回放）

RTX 3070，驱动 617.14，0.4.0 测试包完整运行（run_all）。游戏里录下的输入（赛博朋克 2077、黑神话：悟空）经完整路线回放（代理 → 后端 → NVIDIA 运行库 310.9.16 或 XeSS 引擎）；每次运行去掉前 4 个真实帧，4 轮所有帧取中位数。单位为每个生成帧的 GPU 毫秒数，括号为对 DLSS 默认的变化（负数 = 更快）。DLSS 默认 = 0.4.0 的出厂 INI（`Optimized=2`、`Preset=Auto`；2026-10-21 起出厂档位是 `1`，每生成帧慢 1–3 %，见下面 3080 Ti 表）；原厂内核 = `Optimized=0`。

**2X**

| 抓取 | DLSS 默认 | 预设 C | XeSS Middle | XeSS Coarse | XeSS Fine | 原厂内核 |
|---|---:|---:|---:|---:|---:|---:|
| 赛博朋克 2077 1440p | 2.554 | 1.694（−33.7 %） | 2.974（+16.4 %） | 2.344（−8.2 %） | 3.828（+49.9 %） | 3.639（+42.4 %） |
| 黑神话：悟空 1440p | 2.581 | 1.745（−32.4 %） | 3.266（+26.5 %） | 2.634（+2.0 %） | 4.164（+61.3 %） | 3.657（+41.7 %） |
| 赛博朋克 2077 4K | 3.557 | 2.499（−29.7 %） | 6.379（+79.3 %） | 5.059（+42.2 %） | 8.296（+133.2 %） | 4.407（+23.9 %） |
| 黑神话：悟空 4K | 3.756 | 2.679（−28.7 %） | 6.974（+85.7 %） | 5.682（+51.3 %） | 9.025（+140.3 %） | 4.560（+21.4 %） |

**4X**（每个生成帧 = 一个真实帧的 3 次 Evaluate 之和 ÷ 3）：赛博朋克 2077 4K 默认 2.532 ms，预设 C 1.392 ms（−45.0 %）；黑神话：悟空 4K 默认 2.733 ms，预设 C 1.517 ms（−44.5 %）。

### 帧生成：真实游戏抓取（RTX 3080 Ti，离线回放）

RTX 3080 Ti，驱动 610.88，同一测试包完整运行（测试者机器）；方法与上表相同。这两段计时期间 GPU 稳定在 1665 MHz（低于合成场景计时时的 1950 MHz），绝对值偏保守、对比有效。

**2X**

| 抓取 | DLSS 默认 | 预设 C | XeSS Middle | XeSS Coarse | XeSS Fine | 原厂内核 |
|---|---:|---:|---:|---:|---:|---:|
| 赛博朋克 2077 1440p | 1.651 | 1.032（−37.5 %） | 2.235（+35.4 %） | 1.786（+8.2 %） | 2.791（+69.1 %） | 2.381（+44.2 %） |
| 黑神话：悟空 1440p | 1.663 | 1.062（−36.1 %） | 2.359（+41.9 %） | 1.911（+14.9 %） | 2.936（+76.6 %） | 2.391（+43.8 %） |
| 赛博朋克 2077 4K | 2.171 | 1.473（−32.2 %） | 4.640（+113.7 %） | 3.668（+68.9 %） | 5.908（+172.1 %） | 2.860（+31.7 %） |
| 黑神话：悟空 4K | 2.275 | 1.577（−30.7 %） | 4.763（+109.3 %） | 3.822（+68.0 %） | 6.060（+166.3 %） | 2.935（+29.0 %） |

**4X**：赛博朋克 2077 4K 默认 1.567 ms，预设 C 0.822 ms（−47.5 %）；黑神话：悟空 4K 默认 1.666 ms，预设 C 0.897 ms（−46.2 %）。

### 超分 INT8 对同预设 FP16（RTX 3070，离线）

同一测试包。一次超分 Evaluate 的 GPU 时间，单位 ms：每次运行 64 个测量帧取中位数，3 轮再取中位。同一运行库（310.7.129）、同一预设，只比 INT8 与 FP16。两族内核分别对应巫师 3 / Resonance / 地平线 6 / 蜘蛛侠（mvlo）与黑神话：悟空 / 霍格沃茨（mvhi）。

| 族 | 输出 | 档 | L FP16 | L INT8 | L 变化 | M FP16 | M INT8 | M 变化 |
|---|---|---|---:|---:|---:|---:|---:|---:|
| mvlo | 3840×2160 | 性能 | 7.443 | 6.459 | −13.2 % | 6.065 | 4.905 | −19.1 % |
| mvlo | 2560×1440 | 性能 | 3.415 | 2.993 | −12.4 % | 2.787 | 2.294 | −17.7 % |
| mvlo | 1920×1080 | 性能 | 1.965 | 1.736 | −11.7 % | 1.615 | 1.353 | −16.2 % |
| mvlo | 1920×1080 | 超级性能 | 1.258 | 1.179 | −6.3 % | 1.066 | 0.967 | −9.3 % |
| mvhi | 3840×2160 | 性能 | 7.305 | 6.127 | −16.1 % | 5.914 | 4.623 | −21.8 % |
| mvhi | 2560×1440 | 性能 | 3.346 | 2.845 | −15.0 % | 2.719 | 2.171 | −20.2 % |
| mvhi | 1920×1080 | 性能 | 1.934 | 1.656 | −14.4 % | 1.581 | 1.278 | −19.2 % |
| mvhi | 1920×1080 | 超级性能 | 1.235 | 1.106 | −10.4 % | 1.057 | 0.914 | −13.5 % |

### 超分 INT8 对同预设 FP16（RTX 3080 Ti，离线）

同一测试包、同一方法，GPU 1665 MHz。

| 族 | 输出 | 档 | L FP16 | L INT8 | L 变化 | M FP16 | M INT8 | M 变化 |
|---|---|---|---:|---:|---:|---:|---:|---:|
| mvlo | 3840×2160 | 性能 | 5.338 | 4.578 | −14.2 % | 4.341 | 3.455 | −20.4 % |
| mvlo | 2560×1440 | 性能 | 2.484 | 2.151 | −13.4 % | 2.028 | 1.644 | −18.9 % |
| mvlo | 1920×1080 | 性能 | 1.465 | 1.333 | −9.0 % | 1.208 | 1.047 | −13.4 % |
| mvlo | 1920×1080 | 超级性能 | 1.000 | 0.948 | −5.2 % | 0.849 | 0.791 | −6.9 % |
| mvhi | 3840×2160 | 性能 | 5.262 | 4.355 | −17.2 % | 4.252 | 3.239 | −23.8 % |
| mvhi | 2560×1440 | 性能 | 2.437 | 2.045 | −16.1 % | 1.991 | 1.545 | −22.4 % |
| mvhi | 1920×1080 | 性能 | 1.441 | 1.272 | −11.8 % | 1.186 | 0.994 | −16.1 % |
| mvhi | 1920×1080 | 超级性能 | 0.979 | 0.892 | −8.9 % | 0.848 | 0.746 | −12.0 % |

### 游戏内（RTX 3070，4K 输出，2X）

RTX 3070，驱动 617.14，`[Logging] FrameTiming=1` 的日志，游戏进世界后静置窗口 60 s 的中位数，档位 2，XeSS 为 `Middle`。每生成帧 = 帧生成 GPU 时间；真实帧率 = 游戏渲染的帧（显示帧率为其 2 倍）。

| 游戏 | DLSS 默认 (ms) | XeSS (ms) | XeSS 真实帧率 | 预设 C (ms) | 预设 C 真实帧率 |
|---|---:|---:|---:|---:|---:|
| 赛博朋克 2077 | 3.755 | 6.733（+79 %） | −8.6..−9.7 % | 2.748（−27 %） | +3.6 % |
| 黑神话：悟空 | 4.035 | 7.308（+81 %） | −11.0 % | 3.085（−24 %） | +4.3 % |
| 巫师 3 | 3.973 | 7.166（+80 %） | −15.0 % | 2.974（−25 %） | −0.8 % |
| A Plague Tale: Resonance | 3.943 | 6.968（+77 %） | −28.1 % | 2.947（−25 %） | +9.5 % |
| 极限竞速 地平线 6 | 3.856 | 6.847（+78 %） | −8.0 % | 2.803（−27 %） | +7.4 % |

赛博朋克 2077：路径追踪 + 光线重建，DLSS 超级性能；黑神话：悟空：DLSS 50 %。真实帧率的变化取决于游戏的瓶颈，巫师 3 上预设 C 每生成帧快 1 ms 但帧率不变。

### 超分 INT8 游戏内（RTX 3070，巫师 3）

RTX 3070，驱动 617.14，帧生成关，进世界后静置窗口的帧率；对照是同一预设的 FP16（`Mode=M-FP16` / `L-FP16`），INT8 与 FP16 交替各跑 2–4 次，表中为均值，提升按中位数。预设 M 的几行是 0.4.0 的出货内核集（10-10 测），L 的几行 10-09 测（这一族的 L 集合之后没有变）。

| 输出 | 档（渲染） | 预设 | FP16 fps | INT8 fps | 提升 | 每帧 |
|---|---|---|---:|---:|---:|---|
| 3840×2160 | 性能（1920×1080） | M | 47.15 | 50.85 | +7.85 % | 21.21 → 19.67 ms（−1.54） |
| 2560×1440 | 性能（1280×720） | M | 85.48 | 91.34 | +6.86 % | 11.70 → 10.95 ms（−0.75） |
| 1920×1080 | 性能（960×540） | M | 122.14 | 128.35 | +5.08 % | 8.19 → 7.79 ms（−0.40） |
| 3840×2160 | 超级性能（1280×720） | L | 63.56 | 66.13 | +4.0 % | 15.73 → 15.12 ms（−0.61） |
| 2560×1440 | 超级性能（853×480） | L | 109.42 | 111.80 | +2.2 % | 9.14 → 8.94 ms（−0.19） |
| 1920×1080 | 超级性能（640×360） | L | 145.83 | 147.57 | +1.2 % | 6.86 → 6.78 ms（−0.08） |

除最后一行外，提升都超过同配置重复运行差的两倍；1080p 超级性能 L 在一次运行的波动之内，但 4 次 INT8 全部高于 4 次 FP16。画质：预设 M 静止画面的闪烁略多于 FP16（游戏内静止像素的帧间稳定度 61.8 对 62.5 dB），对真值的 PSNR 不低于 FP16；预设 L 与 FP16 相差 ≤ 0.6 dB。

### 帧生成整组 GPU 时间（RTX 3080 Ti，合成场景）

RTX 3080 Ti，驱动 610.88，0.4.0 测试包（测试者机器）。单位为每个真实帧对应整组帧生成的 GPU 毫秒数（一组 Evaluate 一次提交的跨度）：每次运行 256 组取中位数，4 轮再取中位；降低 = (tier0 − 当前) / tier0。tier0 = 原厂内核（`Optimized=0`），tier1 = 逐位一致的加速（`1`，出厂默认），tier2 = 有损档（`2`，0.4.0 的出厂默认）。同一包的一致性检查与 RTX 3070 参考逐位相同：帧生成 18/18、真实抓取 28/28、超分 32/32、超分 `Mode=On` 4/4。

| 分辨率 | 倍率 | tier0 (ms) | tier1 (ms) | 降低 | tier2 (ms) | 降低 |
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

越低分辨率、越低倍率收益越大（更接近启动/延迟受限）。

## 插帧后帧率怎么估算

先在相同场景、输出分辨率、DLSS 超分档位与画质设置下关闭帧生成，得到帧率 `F_off`。用 `1000 / F_off` 换算基础帧时间，再从上表按分辨率、倍率与内核配置选整组插帧耗时 `T_FG`（ms）。

```text
基础帧时间   T_base (ms) = 1000 / F_off
帧组时间     T_group (ms) ≈ T_base + T_FG
真实帧/帧组速率 G (组/s)  ≈ 1000 / T_group
插帧后总帧率 F_out (FPS)  ≈ G × M = 1000 × M / (1000 / F_off + T_FG)
```

`M` 为倍率（2X…6X 对应 2…6）。一组含一个真实帧与 `M − 1` 个生成帧；上表的 `T_FG` 已包含整组生成帧与共用预处理，不能再乘 `M − 1`。开启插帧后的真实帧/帧组速率 `G` 低于关闭插帧的 `F_off`。

例：关闭帧生成约 **50 FPS**（基础帧时间 20 ms），取上表 RTX 3080 Ti **4K 4X** 的 `T_FG`：

| 内核 | T_FG (ms) | 帧组时间 (ms) | 帧组速率 (组/s) | 估计总帧率 |
|---|---|---|---|---|
| 原厂内核（tier0） | 5.112 | 25.112 | 39.8 | 159.3 FPS |
| 出厂默认（tier1） | 4.106 | 24.106 | 41.5 | 165.9 FPS |
| 有损档（tier2） | 3.987 | 23.987 | 41.7 | 166.8 FPS |

其他分辨率/倍率换用对应行，`F_off` 填该设置下自己实测的值。这是把基础渲染时间与插帧组开销相加的粗略估计；GPU 资源争用、同步、CPU 开销、限帧与刷新率都会影响实际结果，估计值不保证等于计数器读数或实际显示帧率。

## 6X 说明

6X（每真实帧生成 5 帧）为 NVIDIA DLSS 4.5 的 Dynamic Multi Frame Generation。随包的运行库支持该能力，本项目使其能在 Ampere 上运行（只限 `Engine=DLSS`）。

**出厂 `MaxGeneratedFrames` 是 `3`（4X），6X 需要自己改。** 原因是自带 Dynamic MFG 的游戏会默认直接跑到这里写的数，`5` 对多数人偏高（公开仓库 issue #497 / #499）；把它改回 `5` 随时可以拿到 6X。是否真的得到 6X 还取决于游戏：

- 游戏自身支持 6X（自带较新的 Streamline 帧生成插件、菜单中可选 6X 或 Dynamic MFG）：把 `MaxGeneratedFrames` 改成 `5` 即可，实测可稳定运行 6X。
- 游戏仅支持 4X（自带较旧的 4X 插件，当前多数游戏如此）：上限由游戏侧插件决定，本项目无法将其抬到 6X（该插件在初始化时按 4X 铺设内部呈现队列，强行加帧会越界导致帧生成失效或崩溃）。这类游戏请使用 4X。

## 实测反馈

《黑神话：悟空》、《赛博朋克 2077》与《极限竞速 地平线 6》实测可正常开启 4X；《A Plague Tale: Resonance》在帧生成选「动态 / 自动」时默认跑 6X。0.4.0 在 RTX 3070 上于这四款与《巫师 3》共五款游戏中各跑过 DLSS 默认、预设 C 与 XeSS（2X、4K 输出），没有崩溃、TDR 或设备移除，截图中 HUD 完整。插帧观感依赖基础帧率：基础帧率偏低时可能出现画面破坏与边缘效应，倍率越高越明显，部分游戏即使 4X 也需适当降低画质设置、把基础帧率抬高后才有较好观感；这是帧生成在帧率余量不足时的固有表现，非本项目缺陷。

## 已知问题

- XeSS-FG（`Engine=XeSS`）是可选项，默认仍是 DLSS-G。在 RTX 30 上它每生成帧比 DLSS-G 慢得多（4K：RTX 3070 +77..+86 %、约 +3 ms，RTX 3080 Ti 离线 +109..+114 %），真实帧率低 8..15 %，《A Plague Tale: Resonance》低 28 %（多出的等待原因未查明）；显存多约 0.4–0.5 GB；`Engine=XeSS` 到 4X（`MaxGeneratedFrames` 1–3），写 4 以上帧生成不启用（`max_generated_frames_unsupported`）。HUD-less 开 / 关的画质差别还没有做逐帧对比。XeSS 下 `[Debug] Capture` 录不到东西。
- XeSS-FG（model 15）的开销（RTX 3070，离线完整路线回放，`Middle`、DIRECT，每个真实帧连同它的生成帧的 GPU 毫秒数）：4X 时 1440p 4.5–5.0 ms，DLSS 默认 5.3–5.4 ms（快 8–15 %）；4K 10.3–10.8 ms，DLSS 默认 7.9–8.5 ms（慢 27–30 %）；预设 C 在所有档上最快。2X 走 model 18 的图，比原来的 `Model=18` 多 4–5 %（5070 游戏内实测 +1.7 %，输出逐位相同）（多拷上一帧的深度，用无 HUD 画面时再加后缓冲）。这是离线 GPU 时间，不等于游戏帧率。黑神话：悟空 4K（R10G10B10A2、无 HUD 画面）游戏内真实帧率：3X 32.0 fps（DLSS 默认 34.0），4X 28.5 fps（DLSS 默认 31.5）。画质只在 4 段抓取上对跳帧真值比过：靠近上一真实帧的序号（t = 1/4、1/3）上重影比 model 18 + t 少，但不是每个场景都更好。
- XeSS 跑不了时不改跑 DLSS-G：游戏声明的后缓冲不是 R8G8B8A8 / B8G8R8A8 / R10G10B10A2（例如 HDR 常用的 RGBA16F；Intel 自己的 XeSS-FG 也不接受这种格式）或 XeSS 的对象建不出来时，插帧特性被拒绝（插帧关，错误级 `xefg_feature_refused`）；运行中才发现的问题（缺后缓冲、深度、运动矢量或插帧输出，尺寸不一致，录制失败）记一条错误级 `xefg_kernel_fatal` 写明原因，游戏以 `0xE0D55001` 退出。出厂 `UseHudless=Auto` 等于 `0`（2026-11-10 起，C139），整幅画面含 UI 一起插值，不再出现 `xefg_hudless_unused`；`UseHudless=1` 时这是 `xefg_kernel_fatal`，游戏以 `0xE0D55001` 退出。
- `[DLSSG] Preset=B`（强制 UI 重组）只在游戏同时提供 HUD-less 画面与 UI 平面时生效，多数游戏不提供；这时日志（出厂级别）记一条 `preset_ineffective` 写明缺哪张，画面与不写 `Preset=B` 相同。
- 游戏自带的是代 A 的 DLSS-G 运行库（≤ 3.8.x、旧版 Streamline 插件，只能驱动 2X；例如巫师 3 次世代版、光明记忆、蜘蛛侠重制、原子之心、三角洲）时，2026-10-19 起帧生成不启用，游戏的请求原样转发（日志错误级 `game_runtime_generation`）；此前这些游戏能开 2X。
- 预设 C 没有与 RTX 40 / 50 上官方预设 C 的对照；巫师 3 上预设 C 每生成帧快 1 ms 但帧率不变（瓶颈在别处）。
- `[FrameGeneration] CatmullRomWarp` 在 310.9.16 包上没有对应的内核行：写了就拒绝帧生成（日志错误级 `detail_restore_row_unknown`；2026-10-20 之前只是不生效）。实验性的 `DetailRestore*`、`[Compatibility] LaunchChains` / `PlainVariant` / `ChainBlock0` / `HardwareBilinear`、`[Runtime] CacheDirectory`、`[Debug] QueueTrace` 2026-10-21 删除，2026-10-26 起不读（写了与没写相同）。
- 显式开关做不到时一律拒绝而不是换路径（2026-10-20）：`[Runtime] Path` / `SRPath` 指错文件、随包的运行库或超分 snippet 缺失（此前分别回退随包 / 交回游戏自己的那份，RTX 40 / 50 上就是原生 2X）、RTX 40 / 50 上 `KernelImage=Cubin`（`kernel_image_unavailable`）；超分安装失败时这次 `nvngx_dlss.dll` 的加载被拒绝，游戏里 DLSS 不可用而不是跑原厂 FP16。拒绝之后游戏菜单怎么表现没有在真实游戏里看过。
- 超分：`Mode` 把 M / L 钉在所有档位，画质 / 平衡 / DLAA 档因此比游戏自己的 K 慢（业主接受）；0.4.0 的 `Mode=On`（只加速游戏本来就用 L / M 的档）已删除。
- RTX 20：310.9.16 的三个新内核与 XeSS 只在 RTX 3070 上做过 sm_75 前向 JIT 验证，在真 Turing 上没有跑过；超分 INT8 在 2080 Ti 上做过一次冒烟。RTX 40 / 50：帧生成的 PTX 路线没有实机运行过。
- 游戏被强关等不正常退出时，后端不写 XeSS 的汇总记录（`xefg_summary`，只影响诊断）。

## 诊断与边界

- 日志写入游戏目录 `dlssg_sm86\logs\dlssg_sm86.log`：所有模块、所有进程都追加写这一个文件，每行先是本地时间、`pid N`、EXE 名和级别字母（E = Level 1 / I = 2 / D = 3），再是 JSON 记录，例如 `2026-10-22 14:03:11.482  pid 12345  Cyberpunk2077.exe  E  {"event":...}`；只看一个游戏时按 EXE 名或 pid 过滤（`Select-String` / `findstr`）。`FrameTiming=1` 时另有普通文本 `frame_timing.txt`，每个进程追加一段。默认 `[Logging] Level=1` 只记错误，排查时改 `2` 或 `3`。
- 帧生成未生效时先看日志里是否有 `install.active=true` 且 `backend_install.status=0`；缺失多为驱动/运行库不匹配，会记录原因；安装失败时 `nvngx_dlssg.dll` 的加载被拒绝、帧生成不启用（`runtime_selection_failed`），不再改用游戏自己的运行库。
- 帧生成开着却一帧都不生成：看日志里有没有 `foreign_fg_mod`——有就是游戏里还装着别的插帧 mod（rtx-encore、OptiScaler、旧版的 ReShade 伴随插件等），只留一个。游戏崩溃时日志末尾的 `fg_gate_crash` 写着崩在哪个模块（`ours` 表示是否本项目的代码），反馈问题时请附上。
- 离线耗时下降不能当成实测游戏 FPS 增幅；实际帧率增益取决于游戏与瓶颈所在。
- INI 全部键、日志字段与录制回放见 [`docs/INSTALL.md`](docs/INSTALL.md)、[`docs/CAPTURE.md`](docs/CAPTURE.md)。

## SM75 来源与致谢

- Coldwood1026 的 RTX 20 系列 / SM75 适配（SM75 内核族的来源，见 `THIRD_PARTY_NOTICES.txt`）。
- NVIDIA 的 DLSS-G 运行库、DLSS 超分 snippet、模型与前后处理（随包、未经修改）。
- Intel 的 XeSS-FG 模型（着色器与权重，MIT 许可，见 `THIRD_PARTY_NOTICES.txt`）。
- MFG-Unlocker 1.6.0（Michael Robles / dashdogy，MIT）：XeSS-FG 用的 model 15 移植程序与帧数解锁的做法；其中 46 个程序由 Intel 的程序改来，归 Intel XeSS SDK 条款。"XessFG Model 15 Final" 资料包的作者找到了把 model 15 改标成 model 16 的办法。

## 源码与构建

- 代理、后端、测试与工具的 C++ 源码在源码仓库里，用 `build.ps1` 构建（默认运行库模型 310.9.16，`-Runtime310916` 指向运行库；`-RuntimeModel 310.9` 是开发用备选、配 `-Runtime3109`；
  `-KernelPack` 指向内核包），步骤、测试与提交规则见源码仓库的 `CONTRIBUTING.md`。
- 内核包 `dlssg_sm86\kernels\fg_kernels.dll`、超分内核包、XeSS 模型包与 NVIDIA 运行库不在源码里，从同版本的发布包取。没有内核包时 `Engine=DLSS` 的帧生成
  被拒绝（日志 `kernel_pack_missing`、`fg_kernels_unavailable`），不改用运行库自己的内核；没有 XeSS 模型包时 `Engine=XeSS` 拒绝插帧（`xefg_unavailable`，插帧关）。

## 许可与第三方

- 本项目的 C++ 源码以 MIT 许可发布（`LICENSE.md`，Copyright (c) 2026 sdli1995）。随包提供的内核镜像只以二进制形式提供，不在 MIT 许可范围内。
- 随包的 `nvngx_dlssg.dll`（310.9.16，SHA 前缀 `cadee1aa…`）以文件形式放在 `dlssg_sm86\runtime\`，超分 snippet `nvngx_dlss.dll`（310.9.16）放在 `dlssg_sm86\sr\`，均未经修改。内核包 `dlssg_sm86\kernels\fg_kernels.dll` 与 `dlssg_sm86\sr\sr_kernels.dll` 里的内核镜像由 NVIDIA 的内核提取、重编译或重写而来（sm_75 一族导入自 Coldwood1026），只以二进制形式提供，不在 MIT 许可范围内。这些都是 NVIDIA 及上游第三方材料，不随源码转授权，见 `THIRD_PARTY_NOTICES.txt`。
- `dlssg_sm86\xefg\xefg_kernels.dll` 里的着色器与权重来自 Intel XeSS-FG，按 Intel 的 MIT 许可随包，声明见 `THIRD_PARTY_NOTICES.txt`。同一个 DLL 里的 model 15 程序取自 MFG-Unlocker 1.6.0（MIT）：其中 46 个由 Intel 的 XeSS SDK 程序改来，归 Intel XeSS SDK 条款，不在 MFG-Unlocker 的 MIT 与本项目的 `LICENSE.md` 之内。不包含 Intel 的 `libxess_fg.dll`。
