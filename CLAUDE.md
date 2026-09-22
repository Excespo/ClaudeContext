<!-- BEGIN notion-pointer -->
> **状态只在 Notion 改，本地不要建同名文件。**
> 规则真相源：[AI-Infra 工作台](https://app.notion.com/p/3e1854271cf781dfb804dd88945d6283)
> 掌握度 / 能力阶梯 / inbox / kata 四个 database 都在该页下。
> 本仓库只放笔记正文与 skill；`00-index.md`、`inbox.md`、`ROADMAP.md`、`STATUS.md` 已废弃，不要重建。
> 写完 note 用 `/note` 更新掌握度与复查日期。
<!-- END notion-pointer -->

# ClaudeContext 根目录

四个方向的知识库。每个子目录有自己的 CLAUDE.md 与 .claude/rules/，约定见 @README.md，用法见 handbook.html。

## 这个目录的边界
- 这里只放**蒸馏产物**：notes、runbooks、index、roadmap。散活的日志、报告、一次性脚本不写进来，写到 `~/codes/ClaudeDesktopContext`。
- `repos/`、`sources/` 只读；实验代码写 `experiments/`。

## 跨方向的硬规则
- 新增理解 = 一篇 `notes/<domain>/<topic>.md`，写完跑 `/note` 更新 Notion 掌握度与复查日期。
- 进度只写在 Notion（掌握度 / 模块进度等 database），不在仓库里建同名文件。
- 四份 `project-instructions.md` 是云端 Project 的 Instructions 原文，本地是真相源；改了要同步到云端——由 yijie 或 Cowork 负责同步，Claude Code 改完本地文件后不要自己同步。
- 机器与卷的事实归 `sys-ops/inventory.md`，不要散落到别的方向或全局 CLAUDE.md。
