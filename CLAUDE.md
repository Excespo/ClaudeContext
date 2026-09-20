# ClaudeContext 根目录

四个方向的知识库。每个子目录有自己的 CLAUDE.md 与 .claude/rules/，约定见 @README.md，用法见 handbook.html。

## 这个目录的边界
- 这里只放**蒸馏产物**：notes、runbooks、index、roadmap。散活的日志、报告、一次性脚本不写进来，写到 `~/codes/ClaudeDesktopContext`。
- `repos/`、`sources/` 只读；实验代码写 `experiments/`。
- `STATUS.md` 由每周体检任务覆盖生成，不要手写内容进去。

## 跨方向的硬规则
- 新增理解 = 一篇 `notes/<domain>/<topic>.md` + `00-index.md` 里一行登记 + inbox 对应条目划掉。三件事缺一件就算没做完。
- 进度只写在该方向的 `ROADMAP.md`，不要在别处重复。
- 四份 `project-instructions.md` 是云端 Project 的 Instructions 原文，本地是真相源；改了要同步到云端。
- 机器与卷的事实归 `sys-ops/inventory.md`，不要散落到别的方向或全局 CLAUDE.md。
