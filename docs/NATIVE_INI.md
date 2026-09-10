# Native 0.2.4 配置

日常配置仅保留 5 项。修改与所选代理 DLL 同目录的 `dlssg_sm86.ini` 后重启游戏。

```ini
[Compatibility]
Router=SM86
KernelImage=PTX
HardwareBilinear=0

[FrameGeneration]
MaxGeneratedFrames=3

[Logging]
Level=1
```

| 配置 | 默认值 | 用法 |
|---|---|---|
| `Router` | `SM86` | RTX 3080 Ti 等 SM86 用 `SM86`；Turing 或 SM75 路由测试用 `SM75` |
| `KernelImage` | `PTX` | `PTX` 由驱动 JIT；`Auto` 在物理 SM 等于 Router 时用 Cubin，否则用 PTX；`Cubin` 只允许精确匹配 |
| `HardwareBilinear` | `0` | `0` 精确输出；`1` 开放近似硬件双线性采样，仅作用于 SM86 |
| `MaxGeneratedFrames` | `3` | 每个真实帧最多生成 1/2/3 帧，对应 2X/3X/4X；实际倍率由游戏请求决定 |
| `Level` | `1` | `0` 关闭日志，`1` 仅错误，`2` 运行诊断，`3` 详细日志 |

## 两档预设

- `config/presets/sm86-default.ini`：精确默认档，`HardwareBilinear=0`。
- `config/presets/sm86-performance.ini`：性能档，`HardwareBilinear=1`。

两档只有采样开关不同。选一份复制为 `dlssg_sm86.ini`；覆盖前保留自己需要的 Router 与倍率上限。性能档会改变生成帧像素；误差和收益依赖输入，不保证所有游戏更快。默认精确模式已包含本版的 CUDA 整数清理优化，不需要性能档才能获得这部分收益。性能表和帧率估算见[随包 README](../README.md)。

## 固定启用的精确路径

SM86 固定使用默认变体、优化卷积、decoder/pair/merge、block 1 残差链、精确图像补丁和 CUDA uint32 缓冲清理。整数清理复用已有 kernel 58，保留原始哨兵数值及每步 UAV 屏障，不改变插值精度。

移除了 `ChainBlock0`、`PlainVariant`、`DependencyBarriers` 的运行选择和实验调度代码。`OptimizedKernels`、`DisableFusions`、`ImagePatches`、`CudaBufferClear` 也不再作为开关；它们对应的有效精确优化固定启用。旧 INI 中这些键会被忽略，包括无效旧值，不会恢复慢路径或关掉新默认。旧消融记录中的配置只适用于对应历史 DLL。

## SM75

将 `Router=SM75`，保留 `KernelImage=PTX`。3080 Ti 测 SM75 时也这样填写；`Auto` 在该组合中会选择 PTX。物理 SM86 搭配 `Router=SM75, KernelImage=Cubin` 会明确返回架构不匹配错误，应改为 PTX 或 Auto。

SM75 使用自己的整条内核路径、原有 D3D12 清理和共同包装层，不启用 SM86 融合、CUDA 清理或近似采样；因此 `HardwareBilinear=1` 在 SM75 上无效。已在 3080 Ti 上完成 SM75 前向 PTX 检查，物理 Turing/Cubin 仍待验证。

## 按需添加的诊断配置

这些选项仍用于排查，不放进日常 INI：

```ini
[Diagnostics]
Performance=1
PipelineSteps=0

[Logging]
Level=2
EvaluateEvery=120
```

将现有 `[Logging]` 修改为上述值，不要重复创建同名段。日志默认在 `dlssg_sm86/logs/native_<PID>.jsonl`。`Performance` 记录异步 GPU 计时；`PipelineSteps=1` 另开逐步骤时间戳，会影响性能，只用于剖析。诊断完成后删除 `[Diagnostics]` 并恢复 `Level=1`。这里测量 GPU pipeline，不是 Reflex 或显示帧节奏。

高级日志项 `File`、`DebugOutput`、`Directory`，`[Debug] MarkGeneratedFrames/MarkerX/MarkerY/MarkerScale` 与 `[General] Enabled` 仍支持。它们只在相应排查或停用场景需要，无须填写。旧 `Runtime.Mode`、缓存、显卡伪装配置不参与 Native 路径。
