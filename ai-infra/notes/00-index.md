# ai-infra notes 索引

掌握度：`未读` → `读过`（看懂别人的解释）→ `能复现`（能跑通并解释数字）→ `能改`

| domain | topic | note | 源 | 掌握度 |
|---|---|---|---|---|
| research | routing 的 cost/utility 定义 | research/llm-routing-cost-utility.md | GraphRouter/RouterDC/RouteLLM/Oracle | 读过（条目待回原文核对） |
| research | 前沿信息源 | research/info-sources.md | 检索 | — |
| training | FPM：dense → MoE 配置 | training/fpm-moe-design.md | 本人需求 + DeepSeek-V3 config | 读过 |
| training | 分布式后端抽象 | training/distributed-backend-abstraction.md | 本人代码 | 能改 |
| training | 部分加载与 device_map | training/partial-loading-and-device-map.md | accelerate / safetensors | 读过（需回源码） |
| serving | vLLM 显存与批处理 | serving/vllm-memory-and-batching.md | **B 级，待验证** | 未读 |
| hardware | 昇腾 NPU 环境与 HCCL 排错 | hardware/ascend-npu-env.md | 实际日志 | 能复现 |
| comm | 带宽账与 TP/EP/PP | comm/bandwidth-and-parallelism.md | 公式可信，硬件段过时 | 读过 |
| eval-data | lm-evaluation-harness | eval-data/lm-eval-harness.md | 本人脚本与报错 | 能改 |
| eval-data | pass@k 与 repeats | eval-data/pass-at-k.md | Chen et al. 2021 / CodeEval | 能复现 |
| eval-data | 大规模语料处理 | eval-data/data-pipeline.md | dolma 实际处理 | 能改 |

> SelfRouting 的 λ-单调性问题已移出本目录 → `../../selfrouting-problematic.md`（算法侧）

分诊表见 _triage.md（含被丢弃的条目与理由）；待办见 ../inbox.md
