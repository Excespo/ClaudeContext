# 交接说明（2026-09-22，Claude Code 执行 HANDOFF §3.2/§3.3）

**ClaudeContext**（commit 8d512c4→54fd669）：C5 三份草稿+本文件、C1 消解 CLAUDE.md 矛盾、C2 新增
`ai-infra/.claude/skills/study/SKILL.md`（+ 非 git 方式标记全局 `~/.claude/skills/vllm-kata-{pull,push}/`
为 deprecated）、C3 改 `note/SKILL.md`、C4 重写 `project-instructions.md`（**未同步云端**，等你审改）。

**my-minimal-infra**（commit 22f041b→ef4382c）：落地已完成的 v2 实现；修模型路径 bug（`model/`→`models/`）；
`PLAN.md` 按 M0–M6 重排。v0/v1/v2 已验证能跑；v2 decode 循环暴露一个既有 bug（`finished` 张量广播出错，
`(batch,)|(batch,1)`），与本次任务无关，未修，留给你自己处理。

**待决问题**：
1. `README.md`/`handbook.html` 整篇仍以已废弃文件为核心机制描述——跨全部四方向的文档，未动，需要你决定要不要单独起一次清理任务。
2. Notion「模块进度」database 建库后，collection id 需回填进 `study`/`note` 两个 SKILL.md（当前是 `TODO`）。
3. `~/.claude/skills/vllm-kata-{pull,push}/` 只做了标记，未物理删除，需要你自己手动清理。
