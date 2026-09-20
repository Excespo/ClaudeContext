# ai-infra ladder（唯一 state 文件）

## 平台说明

**概念基座是 CUDA。** memory hierarchy、warp/occupancy、coalescing、roofline、集合通信的 bandwidth 模型这些，先在 CUDA 的语境里建立——文献、教材与绝大多数开源实现都以它为默认坐标系，换坐标系学等于放弃了全部现成参照。

**昇腾 NPU 是实际运行场景**（见 notes/hardware/ascend-npu-env.md）。CANN 不单独立一条学习线，而是**通过与 CUDA counterpart 对照来学**：每学完一个 CUDA 概念，追问「昇腾上对应的是什么、差在哪、为什么差」。对照表见 notes/hardware/cuda-cann-counterparts.md。

## 练习场分工

| 机器 | 角色 | 覆盖 |
|---|---|---|
| **desktop 的 RTX 4070**（Ada，compute 8.9） | **日常 kernel 练习机** | L1 全部 + L2 的绝大部分：tiling / bank conflict / coalescing / occupancy / roofline / shared memory double buffering / warp primitives / online softmax 实现 / nsys + ncu 全套 |
| **云上按小时** | 只为三件事开机，用完就关 | ① 多卡通信（NCCL ring/tree 带宽曲线、TP/PP/EP 的真实通信）② Hopper 专属特性（TMA、wgmma、thread block cluster、FP8 + Transformer Engine，FlashAttention-3 依赖它们）③ 大模型的 MFU 实测 |
| **昇腾 NPU** | 真实生产场景 | `[CANN]` 验证项；counterpart 对照 |

标 `[云]` 的条目是**单卡做不了**的，攒在一起用一次云上机时集中做完，不要为它们单独开机。

**4070 的边界（决定哪些结论不能外推）**：单卡 12 GB、显存带宽约 504 GB/s（H100 SXM 约 3.35 TB/s，差约 6.6×）——**roofline 拐点的位置完全不同，方法可以迁移，数字不能**。Ada 没有 TMA / wgmma / thread block cluster，这些是 Hopper 起才有的，而现代 CUTLASS 与 FlashAttention-3 的核心实现建立在它们之上。

**第一件事是自己量，不是查表**：跑 `deviceQuery` 与 `bandwidthTest`，把 SM 数、每 SM 的 shared memory 上限、L2 大小、实测 HBM 带宽（通常是理论值的 80–90%）记进 L1 第一项。任何二手表格（包括我给的 504 GB/s）都只作对照。

ladder 里的 `[CANN]` 条目是**验证项**：用手上真能跑的 NPU 环境去检验刚学的 CUDA 概念迁移得过去与否——**迁移不过去的地方，恰恰是理解最深的地方**。

当前阶段：**L1 进行中**（自评，2026-09-13 初始化）

## L1 研究者够用
- [ ] memory hierarchy：SMEM/L2/HBM 容量与带宽，算术强度与 roofline 拐点
- [ ] 算子融合的收益来源（访存次数而非 FLOPs）
- [ ] DP/TP/PP 的通信量与显存账，能手算给定 config 的 activation + optimizer state
- [ ] 混合精度（fp16/bf16/fp8）的数值行为与 loss scaling 失效模式
- [ ] profiler 读图：nsys timeline 找 gap，ncu 看 occupancy 与 memory throughput
- [ ] `[CANN]` 用 msprof / MindStudio Insight 采一次真实 job，找出与 nsys timeline 对应的视图，并列出它**没有**的信息

## L2 校招水平
- [ ] 手写 CUDA kernel + tiling / bank conflict / coalescing 优化
- [ ] `[CANN]` 同一个 kernel 用 Ascend C 写一遍，比较两边的内存层级抽象差在哪（SMEM/L2 ↔ UB/L1，warp ↔ ？）
- [ ] FlashAttention 的 online softmax 推导与 IO 复杂度
- [ ] PagedAttention 与 continuous batching 的调度语义
- [ ] `[云]` NCCL ring / tree allreduce 的 bandwidth 模型（需 ≥2 卡实例，实测带宽-消息尺寸曲线并与模型比对）
- [ ] `[CANN]` HCCL 的对应算法与拓扑感知：同一条 bandwidth 模型是否成立，实测一次 allreduce 带宽-消息尺寸曲线
- [ ] 能估算给定 config 的 MFU 并解释差距来源（估算在 4070 上做，`[云]` 上做一次实测校准）

## L3 从业者
- [ ] Triton / CUTLASS 写 fused kernel（4070 可做；但 CUTLASS 的 Hopper 路径需 `[云]`）
- [ ] `[云]` 通信与计算 overlap（compute stream vs comm stream，单卡无法验证）
- [ ] `[云]` ZeRO / FSDP sharding 取舍与显存-通信权衡
- [ ] KV cache 与 prefix cache 调度、抢占与 swap
- [ ] 量化 kernel（W4A16 在 4070 上可做；**FP8 + Transformer Engine 需 `[云]` 的 Hopper**）的精度-吞吐曲线
- [ ] `[云]` 多机 NVLink / InfiniBand 拓扑与故障排查
- [ ] `[CANN]` 昇腾侧互联与故障排查：hccn_tool 的 TLS/链路检查、HCCL_CONNECT_TIMEOUT 的实际触发条件（已有一次真实事故记录可直接复用）

## 进度记录
- 2026-09-13：目录建立，历史 chat 导入 inbox.md 待蒸馏
- 2026-09-13：44 条 ai-infra chat 全部分诊完毕，产出 12 篇 notes；inbox 改为「需读源码/读原文/跑实验」三类待办
- 2026-09-14：确认本地有 RTX 4070 可用 + 云上按小时可开，练习场分工见上；确认**概念基座是 CUDA**；CANN 侧按「找 counterpart」的方式学，ladder 里以 `[CANN]` 验证项的形式出现（见上方平台说明）
