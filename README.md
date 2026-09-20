# ClaudeContext

本地知识库根目录。每个子目录 = 一个方向 = claude.ai 上的一个 chat Project。
分类依据：2024-06 至 2026-09 共 135 条 chat 的主题聚类，只保留密集出现的方向，
一次性的杂问（语言翻译、影视、历史考据、LeetCode）不归档。

| 目录 | 方向 | 种子 chat 数 | 形态 |
|---|---|---|---|
| `ai-infra/` | 训练/推理系统、MoE、显存与通信、评测与数据 pipeline | ~40 | 蒸馏笔记 + 只读上游 repo |
| `sys-ops/`（原 `mac-ops/`） | 外接存储与文件系统、数据抢救、网盘、代理网络；Linux 桌面（输入法、字体、GNOME 配置） | ~6 | runbook + incident 记录 + notes |
| `strength-training/` | 力量训练、解剖、跑步生理、伤后负荷 | ~8 | 机制笔记 + 训练日志 |
| `humanities/` | 哲学、社会理论、结构主义/精神分析、文学 | ~16 | 论证结构笔记 |

## 共同约定

1. **原始材料只读**：`repos/`、`sources/` 下是上游代码与抓取的原文，不改。
2. **唯一上传物是 `notes/`**：chat Project 的知识库只放蒸馏笔记 + `00-index.md`，
   不灌原始代码/长文档——否则 project knowledge 退化成召回不准的语义搜索框。
3. **`00-index.md` 是唯一入口**：topic → note → 对应源码/原文路径 → 掌握度。
4. **`inbox.md` 是待蒸馏队列**：历史 chat 的链接与主题，蒸馏完一条就从 inbox 划掉并在 index 登记。
5. **状态只写在一处**：`ROADMAP.md`（或 `00-index.md` 的掌握度列），不要在多个文件里重复。

## 工作流闭环

采集（Claude Code 抓 repo/原文 → `sources/`、`repos/`）
→ 蒸馏（带具体问题读源码/原文 → 写 `notes/`）
→ 消费（`notes/` 进 chat Project → 提问、被出题、被反驳）
→ 验证（写 microbenchmark / 复核引文，对不上回到蒸馏）

## 对应的 claude.ai Project

每个目录下的 `project-instructions.md` 就是该 Project 的 **Instructions** 原文（不是 Description——
Description 只放一句话说明）。本地是真相源，改了本地就同步改云端。

## 根目录下的单篇

`selfrouting-problematic.md` —— SelfRouting 的 λ-单调性问题。属于**算法/研究侧**，不归任何现有方向；等算法侧的对话攒够一簇再单开目录。

## 手册

`handbook.html` 是用法手册（双击打开）：五个面板各写什么、新信息怎么进来、上传知识库的机械规则、定期任务、交接清单与反模式。同一份也发布成了账号内的私有页面，手机上可看。

## 定期任务（跑在 Cowork，绑定本机 + ClaudeContext 文件夹）

| 任务 | 时间（北京） | 产物 |
|---|---|---|
| 知识库体检 | 周一 09:00 | 覆盖写入 `STATUS.md`，并推送结论 |
| ai-infra 自测出题 | 周五 20:00 | 3 道题（推导 / 数字估算 / 源码定位），答案折叠 |
| chat 归档巡检 | 每月 1 日 10:00 | 往各方向 `inbox.md` 末尾追加「待确认」小节 |

`STATUS.md` 由体检任务生成，随时可覆盖，不要手写内容进去。

## 蒸馏状态（2026-09-13）

74 条历史 chat 已全部读完并分诊（A 可直接成 note / B 需回源码原文验证 / C 丢弃），产出 30 篇 notes。
每个方向的 `notes/_triage.md` 记录了逐条判断与丢弃理由，`inbox.md` 已清空为「蒸馏过程中新产生的待办」。
四个 claude.ai Project 的 Context 各上传了一份合并副本（`<方向>-notes.md`）；**本地 notes/ 是真相源，云端那份是副本**，本地改了要重新上传。
