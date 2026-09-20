---
name: vllm-kata
description: 执行每日 60 分钟的 vLLM PR kata 流程（读 brief → 盲写 → 对 diff → 写 note）。用户说「开始今天的 kata」「/vllm-kata」时使用。
---

# /vllm-kata — 每日 60 分钟

规则真相源：Notion「AI-Infra 工作台」https://app.notion.com/p/3e1854271cf781dfb804dd88945d6283
kata database：`collection://05578c59-1627-4e7f-826f-d59d47f8ea00`

## 时间盒

| 段 | 时长 | 做什么 |
|---|---|---|
| 读 brief | 15 min | 从 kata database 取今天的条目，读 1–6 节。**不要往下翻第 7 节之后的内容。** |
| 盲写 | 25 min | **硬限**。只看 issue 全文 + PR 描述首段 + 改动文件路径清单，写出函数签名与伪代码。不看 diff、不看函数名。 |
| 对 diff | 15 min | `cd ~/InfraWorkspace/projects/vllm && git checkout <merge_commit>`，`git show --stat` 再逐文件看；跑 brief 里指定的测试。 |
| 写 note | 5 min | 调用 `/note`。 |

## 本助手在每一段里的职责

- **读 brief**：只答用户问的机制问题，不要提前剧透改动内容；他问「这个函数在哪」可以答，问「这个 PR 怎么改的」要拒绝。
- **盲写**：计时，到点提醒。用户卡住时给的提示止于「去看哪个文件的哪个概念」，不给实现。
- **对 diff**：帮他逐条比对盲写与真实 diff 的差异，明确指出偏离点——这些点是 note 第 4 段的原料，记下来。
- **写 note**：转 `/note`。

## 收尾

提醒用户在 kata 条目里填「盲写命中度」：1 完全偏离 / 2 找到入口 / 3 结构对 / 4 细节差 / 5 基本一致。周日规划任务会用这个分数决定下周是推进还是留级。
