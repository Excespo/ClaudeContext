# 带宽账与并行策略缩写

来源：2024-06 的两次对话。证据等级：**B，公式可信，硬件细节已过时**。

## 显存带宽

```
带宽 = 内存时钟频率 × 内存总线宽度 × 2(DDR) / 8
例：14 Gbps × 384 bit × 2 / 8 = 1344 GB/s
```
带宽是**理论上限**，吞吐是**实测值**；两者的比值（达成率）才是 kernel 优化的指标。

**勘误**：同一对话里给的 WiFi 带宽算法（`8 信道 × 8 bit/符号 × 5/6 × 160 MHz = 8.53 Gbps`）是编的，量纲都不对，不要引用。

## TP / EP / PP

NVIDIA 训练 GPT 的图上 `TP2 EP16 PP2` 指：
- **TP**（Tensor Parallel）：单个张量切到多卡，每层前向都要 all-reduce，通信最频繁，只放同机 NVLink 内
- **EP**（Expert Parallel）：MoE 的专家切到多卡，通信是 all-to-all，量取决于 top-k 与 token 数
- **PP**（Pipeline Parallel）：按层切，通信只在 stage 边界（点对点），但有 bubble

总卡数 = TP × EP × PP × DP。排布原则：TP 在机内，PP 跨机，EP 视 all-to-all 量定。

## 待办

- [ ] 本人的实际平台是昇腾（见 [[../hardware/ascend-npu-env]]），HCCL 的 ring/tree allreduce 带宽模型与 NCCL 不完全一致，需要单独查 CANN 文档后重写本篇的第二节
