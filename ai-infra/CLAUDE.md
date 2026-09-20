<!-- BEGIN notion-pointer -->
> **状态只在 Notion 改，本地不要建同名文件。**
> 规则真相源：[AI-Infra 工作台](https://app.notion.com/p/3e1854271cf781dfb804dd88945d6283)
> 掌握度 / 能力阶梯 / inbox / kata 四个 database 都在该页下。
> 本仓库只放笔记正文与 skill；`00-index.md`、`inbox.md`、`ROADMAP.md`、`STATUS.md` 已废弃，不要重建。
> 写完 note 用 `/note` 更新掌握度与复查日期。
<!-- END notion-pointer -->

# ai-infra 个人知识库

## 目标
把 AI-Infra 从「研究者够用」推进到「专业从业者」。三阶段 ladder 与当前状态见 @ROADMAP.md

## 硬约束
- `repos/` 下是上游代码，**只读**。任何实验代码写到 `experiments/`
- `sources/` 下是归档原文，**只读**；抓取时保留原始 URL 与抓取日期
- 新增理解一律写成 `notes/<domain>/<topic>.md`，并同步更新 `notes/00-index.md`
- 待蒸馏的历史 chat 在 @inbox.md，蒸馏完划掉并在 index 登记

## notes 格式
机制 → 代码定位（`file:line` + commit hash）→ 数字 → 未解决问题
禁止复述文档，只写「读了源码才知道」的部分

## domain 划分
`kernel/` CUDA 与算子 · `comm/` 集合通信与并行策略 · `serving/` 推理与调度
`training/` 训练框架与分布式 · `hardware/` GPU 微架构与互联 · `eval-data/` 评测与数据 pipeline
`research/` 论文与前沿追踪（routing、架构演进）

## 常用
- profile 在 `experiments/` 下用 nsys/ncu，输出留档并在 note 里引用路径
- 引用源码必须带版本/commit；跨版本结论要标注失效条件
