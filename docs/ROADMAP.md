# 下一版本计划 / Next release plan

以下两项为计划，当前 0.2.4 仍使用 D3D12 和 310.1 模型。

1. **Vulkan 支持**：增加 Vulkan 接入和资源互操作，处理图像布局、队列同步及逐次生成调用，复用现有 SM75/SM86 推理管线。分别验证启停、Reset、2X/3X/4X、输出正确性和 GPU 耗时。
2. **模型更新到最新 DLSSG**：实施时确认并固定最新可用 DLSSG 的版本与哈希，整理模型、推理图和所需算子，继续采用自包 DLL。对 SM75/SM86 分别评估画质、显存、稳定性和性能；保留旧模型对照及统一 Release 性能基线。

The next release targets Vulkan integration and migration to the latest available DLSSG model. The model version and source hashes will be pinned when implementation starts. Both SM75 and SM86 need correctness, quality, memory and GPU-time validation. These are planned features; version 0.2.4 remains D3D12 with the 310.1 model.
