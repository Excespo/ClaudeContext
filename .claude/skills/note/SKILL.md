---
name: note
description: 把今天学到的东西写成一篇六段骨架的 note，并更新 Notion 掌握度与复查日期。用户说「写 note」「记一下今天学的」「/note」时使用。
---

# /note — 收尾写笔记

目标：60 分钟 kata 的最后 5 分钟，把今天真正学到的东西固化成可复习的形态。

## 前置

规则真相源：Notion「AI-Infra 工作台」https://app.notion.com/p/3e1854271cf781dfb804dd88945d6283
- 掌握度 database `collection://3d544e1c-76c9-4745-aa76-7d19cc4a5e9f`
- kata database `collection://05578c59-1627-4e7f-826f-d59d47f8ea00`

没接 Notion MCP 就先跑 `claude mcp add --transport http notion https://mcp.notion.com/mcp`，再 `/mcp` 登录。

## 步骤

1. `git pull --rebase --autostash`。
2. 从 kata database 取今天（或用户指定日期）的条目：brief 正文、盲写命中度、「我的作答」。没有条目就问用户今天学的是什么 topic。
3. 按六段骨架生成 note 草稿，写到 `<方向>/notes/<domain>/<topic>.md`：

   1. **结论** — 1–3 行，必须可证伪。
   2. **机制** — 调用链或推导，每跳标 `file:func`，顶部记上游 commit。
   3. **数字** — 代入值 + 来源 + 理论值 vs 实测值。
   4. **我错在哪** — 盲写偏离的点、之前的错误认知。**这一段必须问用户，不要替他编。** 复习时只读这一段。
   5. **证伪观察 + 置信度** — 什么现象会推翻结论。
   6. **复查日期** — 按掌握度推算。

   第 2、3 段可以从 brief 和 diff 里填；第 4 段只能由用户口述。

4. 给用户看草稿，改到他认可为止。
5. 更新 Notion 掌握度 database（有同 Topic 条目就更新，没有就新建）：
   - Topic、方向、note路径（仓库内相对路径）、掌握度、一句话结论、我错在哪、最近更新=今天
   - 复查日期 = 今天 + 间隔：`1 读过` → 3 天，`2 能复述` → 7 天，`3 能改` → 21 天，`4 能写` → 60 天
   - 关联PR、上游commit（如果这篇来自 kata）
6. 提醒用户在 kata 条目里填「盲写命中度」（这一项只能他自己判）。
7. `git add -A && git commit -m "note: <topic>" && git push`。

## 约束

- 不替用户写第 4 段。他答不出来就说明今天没学到东西，如实写「本次无明确偏离点」。
- 不在仓库里重建 `00-index.md` / `inbox.md` / `ROADMAP.md`，那些已迁到 Notion。
- note 里的每个数字都要带代入值与来源；没实测就写「理论值，未实测」。
