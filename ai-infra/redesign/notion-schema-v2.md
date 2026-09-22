# Notion Schema v2 — 模块进度 database + kata database 变更

> 草稿，不应用。由 Cowork 在「AI-Infra 工作台」下建库/改库。字段名在此定稿后，
> `study`/`note` 两个本地 skill 里引用的 collection id 需要回填（当前占位 `TODO: collection id`）。

## 1. 新建「模块进度」database

| 字段 | 类型 | 说明 |
|---|---|---|
| 模块 | title | 例如 "M2 scheduler" |
| 序号 | number | 0–12，对应 §2 模块表顺序，用于排序/校验依赖方向 |
| 当前阶段 | select | `① minimal` / `② 源码对照` / `③ worked example` / `④ PR 测验` / `完成` |
| ①状态 | select | `未开始` / `进行中` / `完成` / `跳过` |
| ②状态 | select | 同上 |
| ③状态 | select | 同上 |
| ④状态 | select | 同上 |
| 掌握度 | number（复用现有掌握度量表 0–4） | 同掌握度 database 的量表定义，避免两套标准 |
| note 路径 | text/url | 仓库内相对路径，如 `ai-infra/notes/serving/scheduler-token-budget.md` |
| minimal commit | text | `my-minimal-infra` 里对应该模块的 commit hash（短 hash 即可） |
| 开始日期 | date | 进入①（或跳过①时进入②）的日期 |
| 完成日期 | date | 该模块四阶段全部完成/跳过的日期 |
| 下一步 | text | 一句话，`/study` 和定时任务都读这个字段决定今天生成什么材料 |
| PR 关联 | relation → kata database | 该模块④阶段用到的 kata 条目，双向关联 |

**排序/展示建议**：默认视图按"序号"排序，看板视图按"当前阶段"分组，方便一眼看出各模块进度分布。

## 2. kata database 字段变更

| 操作 | 字段 | 说明 |
|---|---|---|
| 删除 | 盲写命中度 | HANDOFF §1.2 已定决策：删除这个记分制字段 |
| 新增 | 最高提示级 | select，`0` / `1` / `2` / `3`，对应 brief 里 L1/L2/L3 提示梯，`0` = 未展开任何提示 |
| 新增 | 卡在哪 | text，自由文本，学习信号不是分数 |
| 新增 | 阶段 | select，`③ worked example` / `④ PR 测验`，用于区分该条 kata 属于哪个阶段（③只读讲解+复述，④才是自己做的综合测验） |
| 新增 | 关联模块 | relation → 模块进度 database，与上面「模块进度」的 PR 关联字段互为双向关系 |
| 保留 | brief 正文、我的作答、PR 链接、上游 commit 等既有字段 | 不变 |

## 3. 迁移注意事项

- W1 已建的旧 kata 条目（按星期强推产生的）**归档**，不删除、不套用新 schema 反填——它们记录的是旧流程下的真实历史，反填"最高提示级"没有意义（当时记录的是"盲写命中度"，两者不是同一件事，不能换算）。具体归档方式（加一个"已归档"视图 filter，还是移到单独的 archived 数据源）由 Cowork 执行，这里不展开。
- 「模块进度」database 初始化时，M0（已完成）应直接标记为 `完成`，①-④全部 `完成`（或②③④按 `跳过`处理——因为 minimal 实现和源码对照都还没有正式走过①②③④流程，实际发生的只是"代码写完了"，这一点需要 yijie 决定怎么反映在①-④状态上，建议初始化时先都标 `完成` 但在 note 路径留空，等回头真的读源码写 note 时再补）。
- collection id 占位符：本文件不建库，不产生真实 id。建库后，请把两个 id 分别回填进：
  - `ai-infra/.claude/skills/study/SKILL.md`
  - `.claude/skills/note/SKILL.md`
