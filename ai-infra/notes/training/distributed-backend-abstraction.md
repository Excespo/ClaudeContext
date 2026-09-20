# 训练框架里的分布式后端抽象

来源：2024-11 的两次重构对话。证据等级：**本人代码 + 设计讨论**。

## 问题

原实现用一堆 `xxx_torch` / `xxx_accelerate` 函数对 + `dist_type` 参数分发，每个入口都重复 `assert dist_type == "torch" or accelerator is not None`。加第三种后端要改十几处。

## 结论

抽象成 `DistributedBackend` ABC + 两个实现（Torch / Accelerate），`DistManager.build_from_type()` 直接做工厂（只有一种构造方式时不必单设 Factory 类）。

需要覆盖的接口比最初四个多：
`is_available_and_initialized` / `world_size` / `rank` / `local_rank` / `node_rank` / `local_world_size` / `master_addr` / `local_ip` / `is_main_process` / `barrier()` / `synchronize(SyncData)`

## 跨进程同步的等价写法

| | torch.distributed | accelerate |
|---|---|---|
| 栅栏 | `dist.barrier()` | `accelerate.utils.wait_for_everyone()` |
| 求和归约 | `dist.all_reduce(t)` | `accelerate.utils.reduce(t, reduction='sum')` |

MetricLogger 的 `count/total` 同步应收敛到一个 `backend.synchronize(SyncData(values, dtype, device, reduction))`。

**一个坑（蒸馏时发现，原对话没指出）**：原代码用 `dtype=torch.bfloat16` 承载计数。bf16 尾数 8 位，整数超过 256 就开始丢精度，`count` 会算错。计数必须用 int64 或 float64。

## 多机 GPU 调度（同一对话后半）

把 4 层嵌套循环的评测脚本拆成子脚本分发到 `d8-hpc-gpu-N`：ssh 各节点跑
`nvidia-smi --query-gpu=index,memory.total,memory.free --format=csv,noheader,nounits`，按空闲显存阈值筛卡再投任务。可复用，应落到 experiments/。
