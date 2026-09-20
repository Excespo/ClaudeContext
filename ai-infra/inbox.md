# inbox：待蒸馏队列

**2026-09-13：44 条历史 chat 已全部读完并分诊，逐条归宿见 notes/_triage.md。**
下面只保留**蒸馏过程中新产生的、需要回源码或原文才能关闭的问题**——它们才是真正的待办。

## 需要读源码

- [ ] `vllm/worker/cache_engine.py`：KV cache block 数的计算式，用本人 config 手算一次显存账 → 关闭 notes/serving/vllm-memory-and-batching.md 的待验证清单
- [ ] `vllm/core/scheduler.py`：preemption（recompute vs swap）的触发条件
- [ ] `accelerate` 的 `infer_auto_device_map`：确认 meta 设备路径与 `no_split_module_classes` 的实际行为
- [ ] transformers 的 deepseek 实现：`expert_mask` 应接在 router logits 还是 topk 之后

## 需要读原文

- [ ] DeepSeek-V3 技术报告的 **aux-loss-free 负载均衡**（bias 项如何更新）——旧对话在这里完全没有真实信息
- [ ] 重查 PowerInfer / LLM in a flash / Deja Vu 的真实发表出处（旧对话给的会议与标题基本是编的）
- [ ] RouteLLM 的单价与成本节省数字（85% / 45%）回原论文核对
- [ ] 确认现有 routing 工作里有没有任何一篇报告过 utility 关于成本权重的**内部极值**

## 需要跑实验

- [ ] SelfRouting 解剖实验：按 λ 输出各模型使用率 / final accuracy / 平均 cost 三条曲线
- [ ] 固定单模型的 λ 扫描线性度检验（R²、斜率是否 ≈ −c）
- [ ] math_500 用 `max_gen_toks=128` 时因截断而判错的比例

## 运维

- [ ] 吊销 2025-03 那次对话里明文出现的 xi-ai.cn API key
