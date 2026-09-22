---
name: study
description: 按当前模块所在阶段（①minimal/②源码对照/③worked example/④PR综合测验）生成今天的 ai-infra 学习材料，替代 vllm-kata-pull/vllm-kata-push 的日历强推流程。用户说「今天学什么」「开始今天的学习」「/study」时使用，或提到当前模块名（M0-M12）时主动触发。不按星期几分配，不限时，不打分。
---

# /study — 按模块阶段推进

替代 `vllm-kata-pull`（已标 deprecated，见 `~/.claude/skills/vllm-kata-pull/SKILL.md`）。核心区别：不按日历强推，
按 Notion「模块进度」database 里记录的**当前模块 + 当前阶段**决定今天生成什么，不限时、不打分。

## 前置

规则真相源：Notion「AI-Infra 工作台」https://app.notion.com/p/3e1854271cf781dfb804dd88945d6283
- 模块进度 database：`TODO: collection id`（Cowork 按 `ai-infra/redesign/notion-schema-v2.md` 建库后回填）
- 掌握度 database：`collection://3d544e1c-76c9-4745-aa76-7d19cc4a5e9f`（同 `note` skill）
- kata database：`collection://05578c59-1627-4e7f-826f-d59d47f8ea00`

模块表、阶段定义、PR 选题硬规则、brief 模板的权威版本在 Notion「AI-Infra 工作台」§5（草稿见
`ai-infra/redesign/notion-rules-v2.md` §5，尚未应用时以草稿为准）。

## 第 0 步：确定当前模块和阶段

读「模块进度」database，找到"当前阶段"不是"完成"的、序号最小的模块（即最早还没走完①-④的那个）。
**不跑任何按星期分配的脚本**——`vllm-kata-pull` 的 `todays_slot.sh` 那套逻辑（周一到周四固定 PR、周五
capstone、周日规划）在这里不适用，这正是旧模式"按日历强推"的部分，本 skill 刻意不复用它。

如果 `~/InfraWorkspace/vllm-kata/W{n}/D{d}_*` 或对应模块已有本地未完成的产出，先报告当前进度，不要覆盖。

## 第 1 步：按阶段生成材料

### ① minimal
- 读 `my-minimal-infra/PLAN.md` 里该模块的范围边界描述（做什么、明确不做什么、自测标准）。
- 给出今天要写的最小实现任务，分级提示（L1 方向 / L2 函数与约束 / L3 伪代码），放 `<details>` 折叠。
- 出口条件：有能跑的代码 + 一个自测，commit 进 `my-minimal-infra`。

### ② 源码对照
- 给一份带问题的源码阅读清单：读 vLLM 对应文件（模块表见前置），问题按 `ai-infra/CLAUDE.md`「notes 格式」
  的顺序设计（机制 → 代码定位 → 数字 → 未解决问题）。
- 出口条件：写一篇「minimal vs real 设计差异」note（六段骨架，转 `/note`），掌握度 ≥ 2。

### ③ PR worked example
- 选一个该模块相关、已合并的 PR，给完整讲解（不是盲写——这一步是"读懂+复述"，不是"自己先写"）。
- 复述题：为什么改在这里、去掉会挂哪个测试、数字账怎么算。
- 出口条件：复述通过，掌握度 ≥ 2。

### ④ PR 综合测验
- **选 PR 硬规则**：候选 PR 的所有非测试改动文件，必须落在「已完成②的模块」对应的 vLLM 路径集合内。
  用 `git -C ~/InfraWorkspace/projects/vllm show --stat <candidate_commit>` 核对，不满足就换下一个候选。
- **不限时**（对比 `vllm-kata-pull` 的 25 分钟硬限，这里取消）。
- 按 `ai-infra/redesign/notion-rules-v2.md` §5.4 的 brief 模板生成，三级提示梯全部放 `<details>`。
- 复用 `vllm-kata-pull` 的机制组件，不重新实现：
  - worktree 隔离：`git -C ~/InfraWorkspace/projects/vllm worktree add ~/InfraWorkspace/vllm-kata/worktrees/study-<module>-<pr>-blindwrite -b study/<module>-<pr> <parent_commit>`
  - `~/InfraWorkspace/vllm-kata/scripts/run_verification.sh`：对 diff 阶段的验证脚本，参数顺序见其头部注释
  - `~/InfraWorkspace/vllm-kata/concepts.md`：跨天共享的概念词典，新概念第一次出现时写清楚机制再写进去，之后只链接
- **记录方式**：不记"盲写命中度"。记「用到的最高提示级（0–3）」和「卡在哪」——这些是学习信号，不是分数。
  做完即可，不以对错为出口条件。

## 第 2 步：结束

1. 调用 `/note` 收尾。
2. 更新「模块进度」database：对应阶段状态（`进行中`→`完成`）、note 路径、若来自①还要写 minimal commit hash、
   「下一步」字段更新成下一次该做什么的一句话。

## 约束

- 不重建 `00-index.md`/`inbox.md`/`ROADMAP.md`/`STATUS.md`，状态一律读写 Notion。
- 不擅自判断用户的「最高提示级」或「卡在哪」——这两项只能用户自己写，不要替他编。
- ①②阶段允许先讲机制再提问；④阶段（PR 综合测验）才用苏格拉底式追问，不要在①②阶段就开始盘问。
