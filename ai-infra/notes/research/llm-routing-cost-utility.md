# LLM routing 里 cost 与 utility 的既有定义

来源：2025-12 的调研对话（带 web 检索）。证据等级：**二手，条目需回原文核对**。

| 工作 | cost 定义 | utility / 目标 |
|---|---|---|
| GraphRouter | 归一化固定成本（0–1），按模型规模给 | `Reward = α·Performance − β·Cost`；报告 Oracle 上界 |
| RouterDC | 隐式（选单模型即最小成本） | 双重对比学习损失，无显式 utility |
| RouteLLM | **真实 API 单价**（如 GPT-4 $24.7/M tokens vs Mixtral $0.24/M） | cost 阈值约束下最大化质量 |
| Oracle | 假设预知所有模型输出 | 两种：质量最优、cost-quality 最优；作性能上界 |

**通用模式**：`Utility = Performance − λ·Cost`，靠调 λ 切换性能优先 / 平衡 / 成本优先。

## 与本人工作的差异（这是选题空间）

RouteLLM 用真实价格，因此 perf 与 cost 有共同的货币锚点；本人的 virtual cost 是人为标度，没有锚点——`../../../selfrouting-problematic.md` 里的核心困难正来自这里。另外上表全是 **单跳 router**（一个外部分类器选模型），本人的是 **sequential self-routing**（模型自己决定是否弃答），成本结构是累加的而非单次的，因此 λ 与 utility 的关系不能直接套用它们的结论。

## 待验证

- [ ] 上表的数字（尤其 RouteLLM 的单价与节省比例 85%/45%）需回原论文核对，本条来自检索摘要
- [ ] 确认这四篇里有没有任何一篇报告过 utility 关于成本权重的**内部极值**；若全都是单调，说明单调本身不是异常，问题应重述为「如何设计出非单调」
