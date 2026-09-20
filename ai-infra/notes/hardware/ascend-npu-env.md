# 昇腾 NPU 上跑评测与训练的环境事实

来源：2025-03/04 的两次真实排错。证据等级：**A，来自实际日志**。

## 环境

SJTU HPC，**华为昇腾 NPU**（不是 NVIDIA）：`torch_npu`、`vllm-ascend` 插件、`ascend-toolkit 8.1.RC1`、aarch64-linux、Python 3.10.15。vLLM 启动时会打印
`plugin ascend loaded` / `Platform plugin ascend is activated`；若看到 `No platform detected, vLLM is running on UnspecifiedPlatform` 与 `Failed to import from vllm._C`，说明插件已接管，不是报错。

**这条事实怎么用**：概念仍以 CUDA 为基座（见 ../../ROADMAP.md 的平台说明），本篇提供的是**实际运行侧的事实**——当 CUDA 概念要落到这台机器上验证时，从这里查对应物与已知坑。逐项对照见 cuda-cann-counterparts.md。

## 故障一：`NUMEXPR_MAX_THREADS`

`Error. nthreads cannot be larger than environment variable "NUMEXPR_MAX_THREADS" (64)`

NumExpr 的线程上限受**逻辑核心数**约束。`nproc` 给的是逻辑核（开了超线程则是物理核两倍）；物理核要算
`lscpu` 的 `Socket(s) × Core(s) per socket`。在 eval 脚本开头显式导出即可，实测对结果无影响，只是消除噪声。

## 故障二：多机 HCCL 通信超时（真正会卡死作业的那个）

日志关键串：
- `Getting socket times out. Reason: Remote Rank did not send the data in time`
- `Transport init error ... [Create][DestLink]Create Dest error! createLink para:rank[2]... dst_rank[1]`
- 卡在 `torch.distributed.barrier()`
- 收尾出现 `There appear to be 30 leaked semaphore objects`

处置顺序：
1. `hccn_tool -i $devid -tls -g` 逐卡检查 TLS 状态（各节点 TLS 开关不一致会直接建链失败）
2. 调 `HCCL_CONNECT_TIMEOUT`（默认偏小，大模型 warmup 慢时不够）
3. 复现时加 `ASCEND_LAUNCH_BLOCKING=1` 拿准确堆栈——**会显著降速，定位完必须取消**
4. 确认进程数与可见 NPU 数一致（accelerate 配置与实际卡数不匹配是高频原因）

## 附：Ray 上的 NPU 资源

`@ray.remote(resources={"npu": 1})` 需要在 `ray start` 时用 `--resources='{"npu": 8}'` 声明自定义资源，否则任务永久 pending。
**勘误**：对话里给的 `@ray.remote(num_npus=1)` 不存在——Ray 只有 `num_cpus` / `num_gpus` 与通用 `resources`。
