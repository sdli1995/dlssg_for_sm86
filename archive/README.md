# DLSSG SM86 融合版

简体中文 | [English](README.en.md)

当前 **Release（2026-09-07）** 为首个 milestone，游戏实测、验证范围及已知项见 [测试记录](docs/VALIDATION.md)。

**将 `version.dll + dlssg_sm86.ini` 放到游戏实际渲染 EXE 旁，照常启动。** 运行时无需 Python 或 PowerShell 启动器。

代理 DLL 内嵌配套的原版 DLSSG 310.1（含模型和管线）及其 SM86 后端，默认 `Mode=Bundled`。游戏请求不同版本的 DLSSG 时，统一加载这套内置实现。首次运行将配套文件释放到 `%LOCALAPPDATA%\DlssgSm86\bundles\<bundle-id>`，校验后加载；以后复用缓存，损坏时自动恢复。

## INI 放在哪里、何时生效

配置文件固定命名为 `dlssg_sm86.ini`，放在当前使用的代理 `version.dll` 或 `winmm.dll` 旁。**修改后完全退出并重新启动游戏**，当前没有热重载。

- 布尔开关填写 `0` 或 `1`。以 `;` 开头的行为注释。
- `Mode`、`KernelImage` 的值不区分大小写。
- 日志目录、缓存目录和 DLL 路径可以填写绝对路径；相对路径以代理 DLL / INI 所在目录为基准。自定义路径不会展开 `%LOCALAPPDATA%`、`%TEMP%` 等环境变量，请填写实际路径；使用系统默认缓存目录时将 CacheDirectory 留空。
- 以下“默认值”指随包 INI 中的设置。常规安装保留 `Mode=Bundled` 和两个兼容性开关为 `0`。
- 使用新选项时同步更新代理 DLL 和 INI，确保程序支持相应设置。

## 完整默认配置

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

### [General]：总开关

| 配置项 | 默认值 | 说明 |
|---|---|---|
| `Enabled` | `1` | `1` 启用 DLSSG 重定向、适配、能力上报和可选标记；`0` 保留游戏原始 DLSSG 加载行为。代理仍转发系统 DLL 的原有导出。 |

### [FrameGeneration]：上报插帧数量

| 配置项 | 默认值 | 说明 |
|---|---|---|
| `MaxGeneratedFrames` | `3` | 上报的最大“额外生成帧”数量。`0` 保留运行库原有上报；`1` 最多 2×；`2` 最多 3×；`3` 最多 4×。 |

例如 `MaxGeneratedFrames=3` 表示每个真实帧间隔最多额外生成 3 帧。**实际生成数量由游戏请求决定**，该设置不会单独增加游戏菜单选项，也不会强制呈现 4×。

当前后端上限为 3；解析器接受的 4–16 会限制到 3，超过 16 或非法数字会导致配置读取失败。正常使用填写 0–3 即可。

### [Compatibility]：选择 PTX / cubin 与验证模式

| 配置项 | 默认值 | 说明 |
|---|---|---|
| `KernelImage` | `Auto` | 内核加载格式，支持 `Auto / PTX / Cubin`，具体行为见下表。缺少此项也使用 Auto。 |
| `ForceSM86Route` | `0` | `0` 自动识别，真实 SM86 启用适配路径；`1` 允许其他 GPU 强制使用 SM86 后端进行验证，低于 SM86 的 GPU 仍会被拒绝。 |
| `SimulateAmpere` | `0` | `1` 为验证调整架构报告；必须同时设置 `ForceSM86Route=1`。它不会改变物理 GPU。 |

| KernelImage 值 | 路由启用后的行为 |
|---|---|
| `Auto` | 真实 SM86 使用预编译的 SM86 cubin；其他 GPU 使用 SM86 PTX。 |
| `PTX` | 总是使用 SM86 PTX，由驱动 JIT 编译成本机机器码；3080 Ti 也适用。 |
| `Cubin` | 使用预编译的 SM86 cubin，要求真实 SM86。其他架构会拒绝安装该适配路径；默认 Bundled 模式下会尝试回退原始加载请求。 |

**KernelImage 只选择内核格式，不单独开启路由。** RTX 3080 Ti 会自动识别为 SM86，使用 Auto、PTX 或 Cubin 时两个兼容性开关均保留 0。未启用 SM86 路由的设备仍使用运行库原生内核。

该设置不调整模型权重、FP16 精度模式或上报帧数。PTX 需要驱动支持相应的 PTX 版本，首次加载可能发生 JIT 编译。

### [Logging]：日志输出

| 配置项 | 默认值 | 说明 |
|---|---|---|
| `Level` | `2` | `0` 关闭日志；`1` 错误；`2` 增加配置、加载和能力信息；`3` 再增加内核创建、Evaluate 和标记信息。 |
| `File` | `1` | `1` 写日志文件；`0` 关闭文件输出。仍受 Level 控制。 |
| `DebugOutput` | `0` | `1` 同时通过 Windows 调试输出发送日志，可由调试器接收；不在游戏画面上显示。 |
| `EvaluateEvery` | `120` | 正常 Evaluate / 标记日志的采样间隔，按 Evaluate 调用计数。`1` 记录每次；`0` 按 120 处理；最大 1,000,000。 |
| `Directory` | `dlssg_sm86\logs` | 日志目录，可修改为其他非空的相对或绝对目录。 |

日志文件名为 `loader_<PID>.jsonl` 和 `backend_<PID>.jsonl`。Level 3 下，前 12 次 Evaluate 会记录，后续正常调用按 `EvaluateEvery` 采样；失败事件仍按错误级别记录。多帧生成可能在一个真实帧间隔内调用多次 Evaluate，因此采样间隔不等于游戏帧数。

Level 2 通常足够确认配置和内核选择。需要分析逐次调用时再开启 Level 3，它会产生更多日志。

### [Debug]：生成帧标记

| 配置项 | 默认值 | 说明 |
|---|---|---|
| `MarkGeneratedFrames` | `0` | `1` 在生成输出上绘制 `FG 1/3` 等实际序号标记；`0` 关闭。真实帧和 Reset 输出跳过。 |
| `MarkerX` | `8` | 标记左上角的横坐标，单位为输出纹理像素，范围 0–65535。 |
| `MarkerY` | `8` | 标记左上角的纵坐标，单位为输出纹理像素，范围 0–65535。 |
| `MarkerScale` | `2` | 标记缩放倍率，范围 1–8。矩形大小为 `24×scale` 宽、`9×scale` 高，默认 48×18 像素。 |

标记必须完整位于输出纹理内部；超出范围会绘制失败，日志中记录 `marker_failed`。标记会实际修改生成帧像素，因此做全图数值比较时关闭标记，或单独比较标记矩形之外的区域。

### [Runtime]：运行库选择和缓存

| 配置项 | 默认值 | 说明 |
|---|---|---|
| `Mode` | `Bundled` | 运行库来源，支持 `Bundled / Auto / Pinned`。这里的 Auto 与 KernelImage=Auto 是两个独立选项。 |
| `CacheDirectory` | 空 | 空值使用 `%LOCALAPPDATA%\DlssgSm86\bundles`；非空时使用指定缓存根目录，程序在其下按 bundle ID 建立子目录。 |
| `Path` | 空 | 仅 Pinned 使用，应明确填写目标原版 `nvngx_dlssg.dll` 的路径。Bundled 和 Auto 不使用此项。 |

| Mode 值 | 行为 |
|---|---|
| `Bundled` | 使用代理内嵌的运行库和配套后端。普通安装使用这个模式，无须填写 Path 或 Backends。 |
| `Auto` | 加载游戏请求的原运行库。内置已知哈希可自动配套后端；其他版本需另有匹配后端，否则保留原库行为。 |
| `Pinned` | 尝试使用 Path 指定的原运行库，并要求有匹配后端。目标缺失或不满足选择条件时保留原始请求，详情见日志。 |

Bundled 固定使用本包的 310.1 模型，不会自动采用游戏自带新 DLL 的模型改进。默认模式下，缓存释放、运行库加载或后端安装失败会记录 `runtime_selection_failed` 并尝试原始请求；回退成功不表示 SM86 路由已经生效。

### [Backends]：高级外部后端映射

默认留空，仅 Auto / Pinned 模式的外部运行库适配需要填写。键是目标原运行库文件的完整 SHA256，值是与它匹配的后端 DLL 路径：

```ini
[Backends]
; 将占位符换成原运行库的完整 SHA256；此行为格式示例。
; <runtime-sha256>=backends\matching_backend.dll
```

映射项不会自动适配未知 DLL，后端必须确实支持目标文件。显式选择 PTX / Cubin 时，外部后端还需支持内核选择扩展；旧后端不支持时会记录 `kernel_selection_unsupported` 并拒绝安装。Auto 内核选择仍兼容原 ABI 1 后端。

## 常用配置示例

下面是要修改的片段，在已有同名分节中替换对应键即可；其他参数保留完整默认配置。

### 3080 Ti：使用 PTX JIT

```ini
[Compatibility]
KernelImage=PTX
ForceSM86Route=0
SimulateAmpere=0
```

恢复预编译 cubin 可将 KernelImage 改回 Auto，或明确设为 Cubin。无需开启架构模拟。

### 开启生成帧标记

```ini
[Debug]
MarkGeneratedFrames=1
MarkerX=8
MarkerY=8
MarkerScale=2
```

标记分母来自游戏实际请求数量。例如上限设为 3、游戏实际只请求 1 张时，标记为 `FG 1/1`。

## 如何确认配置生效

在 Level 2 或 3 的日志中检查：

| 事件 / 字段 | 含义 |
|---|---|
| loader：`configuration` | 本次读取的 INI、运行库模式与内核格式请求。`requested_max` 是 INI 请求值，尚未应用后端上限；实际上报值看 `mfg_capability`。 |
| backend：`install` | 安装准备阶段记录。`actual_sm` 是物理架构；`active` 是本次计划启用的 SM86 路由状态；`kernel_image_requested` 是请求值，`image` 是选定格式。该事件早于钩子安装完成。 |
| loader：`backend_install` | `status=0` 表示后端安装调用成功；还需结合 `install.active` 判断是否启用了 SM86 内核替换。非零状态表示安装失败。 |
| `image=ptx_sm86` | 已选择 SM86 PTX。 |
| `image=cubin_sm86` | 已选择 SM86 cubin。 |
| `image=original` | 本次没有启用内核替换，使用运行库原生内核。 |
| `mfg_capability` | 运行库原上限与本次上报的最大生成帧数。 |
| Level 3：`kernel_create` | 每次内核创建实际使用的格式与返回状态。 |
| Level 3：`evaluate` / `frame_marker` | 实际生成数量、调用结果或标记序号。 |

确认路由安装成功时，同时检查 `install.active=true` 和对应的 `backend_install.status=0`。仅出现 `image=ptx_sm86` / `cubin_sm86` 不能证明内核已经创建或执行；实际推理还需检查后续 `kernel_create`、`evaluate` 及其结果。

非法数字、枚举值或组合会导致配置读取失败；日志开启时可见 `configuration_error`。SM86 cubin 与物理 GPU 不匹配等后端错误可在 `install_failed` 中查看。恢复默认 INI 后重启可重新验证。

## 安装注意与卸载

游戏不导入 VERSION.dll 时，可改用 `alternatives/winmm.dll`，它同样内嵌完整运行库。只启用一种代理；已有同名 mod 时需要处理入口冲突，当前没有实现任意代理链。

目前针对 Windows x64 / D3D12。M1 已在 RTX 3080 Ti / 驱动 591.86 上完成 Auto、PTX、Cubin 离线执行和同机输出一致性检查，同一代理在悟空和赛博朋克中实际执行 Auto/cubin 的 2X、4X；悟空还覆盖关闭后重新开启。

用户手动实测关闭 / 2X / 4X：悟空约 50 / 80 / 150 FPS，赛博朋克路径追踪约 35 / 60 / 100 FPS，并反馈相对稳定。详见 [验证记录](docs/VALIDATION.md)。结果绑定 M1 的 DLL 哈希，其他产物以自身 `manifest.json` 的状态为准。规范化帧时间、延迟和长期稳定性尚未测量；固定参考的数值差异作为 M1 已知项保留。

卸载时退出游戏，移走本包添加的代理和 INI。缓存可保留供其他安装使用。原 NVIDIA DLL 及内核资源的归属见 `THIRD_PARTY_NOTICES.txt`。
