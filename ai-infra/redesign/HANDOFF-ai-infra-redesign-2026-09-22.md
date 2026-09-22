# HANDOFF：ai-infra 学习模式重设计（2026-09-22，来自 Cowork 会话）

> 给 Claude Code 的执行说明。执行环境：Ubuntu slaanesh，仓库 `~/ClaudeContext`，工作区 `~/InfraWorkspace/my-minimal-infra`。
> **先读完全文 → 给出计划 → 等 yijie 确认后再动手。** 全程中文，术语/路径/函数名保留英文。

---

## 0. 为什么要改（结论先行）

旧模式（vLLM PR Kata，W1 从 2026-09-21 开始）在 W1D2 就让 yijie 卡住了，而且精神压力很大。诊断如下：

1. **难度跨级**：掌握度还是 0，就要在 25 分钟内盲写一个真实 merged PR。这等于跳过「能读 → 能复述 → 能改」三级，直接考「能写」。
2. **选题不检查前置依赖**：W1 的子系统是 scheduler，但选题规则只看「改动是否落在主路径 + diff 行数」，结果选中了一个 parallel drafting（spec decode）相关、却碰到了 `scheduler.py` 的 PR。读懂它需要 scheduler 的 token budget、KV slot 预留与回滚、rejection sampler、multi-query verify 这四块，yijie 都还没学。数字账（acceptance rate α 与实际加速的关系）也因此卡住。
3. **按日历推进、每天记账**：一周 7 天都有推送，周检和体检还统计「未开始」「疏远」「空壳」并建议降级。系统每天都在生成欠账感。

已经做的（Cowork 侧，2026-09-22）：
- 已**手动暂停**三个任务：`trig_011YdiiqbXa57SB5EMLuP2JW`（每日 brief，含周六实验）、`trig_01TtFUzro5gbFspQ9dQqjyxu`（周五 capstone）、`trig_01Nn28LkoeVXWVewBbSB6AXe`（周日规划）。
- 看门狗日检 / 周检的 prompt 已改成：只恢复 `suspension_reason=device_absent` 的任务，手动暂停的一律不动。周检已删掉评判性措辞。
- 知识库体检 `trig_01UaLo7L3P4Brh8T8dGMSu2Z` **还没改**，其中「疏远 / 降级」判据待新方案定稿后由 Cowork 修改。

---

## 1. 新模式的已定决策（yijie 已同意，不要再推翻）

### 1.1 课程结构：按模块推进，每个模块内部逐步撤掉支架

每个模块按 ①→④ 顺序走。前一阶段达到出口条件才进入下一阶段，**不按日历强推**。

| 阶段 | 做什么 | 出口条件 |
|---|---|---|
| ① minimal | 在 `my-minimal-infra` 里实现该模块的最小版本（可以问、可以看提示） | 有能跑的代码 + 一个自测，commit 进 my-minimal-infra |
| ② 源码对照 | 读 vLLM 对应文件，写一篇「minimal vs real 设计差异」note（六段骨架） | note 入库，掌握度 ≥ 2（能复述） |
| ③ PR worked example | 读一个已合并 PR 的完整讲解，然后复述：为什么改在这里、去掉会挂哪个测试、数字账 | 复述通过，掌握度 ≥ 2 |
| ④ PR 综合测验 | 模块末尾的**综合小测**，见 1.2 | 做完即可（不以对错为出口） |

- 不适合写最小实现的模块（量化 kernel、分布式、spec decode 的部分内容）跳过 ①，从 ② 开始。
- 前几个模块只做到 ③。**④ 只在该模块掌握度 ≥ 2 之后开放。**

### 1.2 PR 的定位（yijie 原话的意图）

> 「把 PR 当成一次小测试性质的综合测试，允许自由时间，主要目的是学习，尝试根据指导来自己完成 PR，但是不能指望我一下自己就写明白。只有后面很熟练了才可能。」

据此设计为：
- **不限时**，取消 25 分钟硬限。
- **引导式**：brief 里给一条**分级提示梯**，每个要点配 3 级提示，全部放在 `<details>` 里：
  - L1：指出方向，说明涉及哪个数据结构 / 哪个调用阶段；
  - L2：给出函数名和约束；
  - L3：给出伪代码。

  yijie 按需展开。
- **记录方式改掉**：删除「盲写命中度」，改成「用到的最高提示级（0–3）」和「卡在哪」。这些是学习信号，不是分数。
- **硬规则**：PR 改动的所有非测试文件，都必须落在「已完成 ② 的模块」的路径集合内。这条机械规则就能修掉今天那个 bug。

### 1.3 时间预算

- 基线：**每周 5 次 × 2–2.5h**。yijie 自己的计划是每天 4h+，超出基线的部分算加练，不计进度、不产生欠账。
- 2 周后复盘：实际平均每天稳定 > 3.5h、且不拖论文，才考虑压缩模块周期。

### 1.4 状态与留痕分工

沿用现有三层原则：**不双写**。
- **git 仓库（留痕）**：note 正文在 `~/ClaudeContext/ai-infra/notes/`，minimal 代码在 `~/InfraWorkspace/my-minimal-infra`。
- **Notion（状态，定时任务读写）**：新增一个「模块进度」database（schema 见 §3.3），记录每个模块当前的阶段 ①–④、每阶段的状态、关联 note、关联 my-minimal commit。
- 旧 kata database **不删**，改名或加一个「类型 = PR 测验 / worked example」继续用。W1 已建的条目由 Cowork 归档。

---

## 2. 模块顺序（草案，你要对照 my-minimal-infra 的现状校准）

| # | 模块 | vLLM 主路径 | minimal ① | 数字账主题 |
|---|---|---|---|---|
| M0 | 端到端骨架：offline generate 循环、naive 连续 KV | `vllm/v1/engine/llm_engine.py`, `core.py`, `request.py` | ✅ | 单请求 prefill/decode 的 FLOPs 与字节数 |
| M1 | paged KV + block pool | `vllm/v1/core/kv_cache_manager.py`, `block_pool.py`, `kv_cache_utils.py` | ✅ | 单 block 字节数、8GB 下可分配 block 数 |
| M2 | scheduler：continuous batching → chunked prefill → preemption | `vllm/v1/core/sched/scheduler.py` | ✅ | token budget、max_num_seqs、chunked prefill 切分 |
| M3 | prefix caching + block hash | `kv_cache_utils.py`(hash), `tests/v1/core/` | ✅ | hit rate 对 prefill FLOPs 的节省 |
| M4 | model runner：input batch、block table、attention metadata | `vllm/v1/worker/gpu_model_runner.py`, `gpu_input_batch.py`, `block_table.py` | ✅（简化） | persistent batch 的 H2D 字节数 |
| M5 | attention backend（FA2/FlashInfer，排除 FA3/MLA） | `vllm/v1/attention/backends/` | 部分 | attention arithmetic intensity、roofline 拐点 |
| M6 | sampler（+ structured output 只读） | `vllm/v1/sample/` | ✅ | logits 张量大小、top-p sort 代价 |
| M7 | engine core 进程边界 / async / IPC | `vllm/v1/engine/`, `serial_utils.py` | — | 每 step CPU 开销 vs GPU step 时间 |
| M8 | CUDA graph / torch.compile | `cudagraph_dispatcher.py`, `vllm/compilation/` | 部分 | capture 数量 × 显存 |
| M9 | spec decode（ngram → EAGLE → parallel drafting） | `vllm/v1/spec_decode/` | 部分（ngram） | $\frac{1-\alpha^{k+1}}{1-\alpha}$ 与 draft/verify 开销 |
| M10 | 量化（W4A16、AWQ/GPTQ；FP8 只读） | `model_executor/layers/quantization/` | — | 权重字节数、dequant 带宽 |
| M11 | kernel 周（FlashInfer / sgl-kernel） | 切 repo | 部分 | SMEM、bank conflict、coalescing |
| M12 | 分布式 TP/PP + kv-connector | `vllm/v1/executor/`, `vllm/distributed/` | — | all-reduce 通信量、HCCL 对照 |

与旧 12 周表的差异：
- 新增 M0；
- 把 KV（M1）提到 scheduler（M2）前面，因为 scheduler 的 `allocate_slots` 和 preemption 依赖 KV 语义；
- engine core 的 IPC 后移到 M7，这部分不做 minimal；
- spec decode 保持在后段，今天那道 parallel drafting 放进 M9 的 ④。

已有 note `vllm-request-lifecycle.md`（#39102，max_model_len 同步）归入 M7，`vllm-memory-and-batching.md` 归入 M1/M2。

---

## 3. 你（Claude Code）要做的事

### 3.1 先调研，不改文件

1. 读 `~/InfraWorkspace/my-minimal-infra` 全部：目录结构、`PLAN.md`（或 `ROADMAP.md`）、已实现到哪一步、能不能跑。输出一张表：M0–M6 各自在 minimal 里的现状（未开始 / 部分 / 完成），以及 minimal 现有设计和 §2 顺序冲突的地方。
2. 读 `~/ClaudeContext` 中与 ai-infra 相关的：根 `CLAUDE.md`、`ai-infra/CLAUDE.md`、`ai-infra/project-instructions.md`、`.claude/skills/note/SKILL.md`、`.claude/skills/vllm-kata/`（若存在）、`README.md`、`handbook.html` 里 ai-infra 部分。列出**自相矛盾之处**。已知一处：根和 `ai-infra/CLAUDE.md` 顶部说 `00-index.md` / `ROADMAP.md` / `inbox.md` 已废弃，但正文和 `project-instructions.md` 仍要求读写它们。
3. 若本机有 vLLM checkout，确认 §2 表里的路径在当前 commit 下存在；不存在的标出来。

### 3.2 直接改仓库（确认后执行，每类一个 commit）

1. **`my-minimal-infra/PLAN.md`**：按 §2 的 M0–M6 重排，每个模块写清 minimal 的范围边界（做什么、明确不做什么）和自测标准。保留原计划里有价值的部分，并注明来源。
2. **`ai-infra/CLAUDE.md` 与根 `CLAUDE.md`**：消除 3.1-2 找到的矛盾；ai-infra 的「目标 / 硬约束」改为指向新课程。
3. **新 skill `.claude/skills/study/SKILL.md`**：替代 `vllm-kata`，旧的删除或标 deprecated。行为要求：
   - 读 Notion「模块进度」，确定当前模块和阶段；
   - 按阶段生成今天的材料：
     - ① 给 minimal 任务和分级提示；
     - ② 给带问题的源码阅读清单；
     - ③ 给 PR worked example 讲解和复述题；
     - ④ 给 PR 综合测验 brief（含提示梯）。
   - 选 PR 时执行 §1.2 的硬规则。
   - 结束时调用 `/note` 并更新阶段状态。
   - 不限时，不打分。
4. **改 `/note` skill**：
   - 「盲写命中度」换成「最高提示级 + 卡在哪」；
   - 更新的对象从 kata database 扩展到「模块进度」；
   - 第 4 段「我错在哪」仍然必须问本人。
5. **`ai-infra/project-instructions.md` 重写**（claude.ai Project 的 Instructions 原文）：
   - 去掉对 `00-index.md` / `ROADMAP.md` 的依赖，改为「当前模块与阶段见 Notion 模块进度；笔记见 project knowledge」；
   - 加入「按 yijie 当前阶段调节讲解深度：① ② 阶段允许先讲机制再提问，④ 阶段才用苏格拉底式追问」；
   - 回答协议里其余条款保留。

   写完后 yijie 会把这份拿回 Cowork 审改，**你不要自己同步到云端**。
6. `git add -A && git commit && git push`。my-minimal-infra 若有自己的 remote 也 push。

### 3.3 只产出草稿，不要自己应用（由 Cowork 应用到 Notion 与定时任务）

写到 `~/ClaudeContext/ai-infra/redesign/`，这个目录入库：

1. `notion-rules-v2.md`：Notion「AI-Infra 工作台」要替换的章节全文，包括 §4 本地 skill、§5（改名为「课程与 PR 规则」：模块表、阶段定义、PR 选题规则含依赖硬规则、brief 模板改成含提示梯的版本）、§6 和 §8 定时任务表、§7 hook（my-minimal 与模块进度的挂钩方式）。§0–§3 不动。
2. `notion-schema-v2.md`：「模块进度」database 的 schema。建议字段：
   - 模块（title）、序号；
   - 当前阶段（① minimal / ② 源码对照 / ③ worked example / ④ PR 测验 / 完成）；
   - ①–④ 每阶段的状态（未开始 / 进行中 / 完成 / 跳过）；
   - 掌握度；
   - note 路径；
   - minimal commit；
   - 开始日期、完成日期；
   - 下一步（一句话，定时任务和 `/study` 读这个字段）。

   另写 kata database 的字段变更：删「盲写命中度」，加「最高提示级」「卡在哪」「阶段」。
3. `trigger-prompts-v2.md`：新的定时任务 prompt 全文，每个都要能独立执行，格式参照现有任务。建议的最小集合：
   - **周日规划（周日 20:00）**：读模块进度，给下周 3–5 个 session 的建议清单，每条标阶段和预计时长。**不预建条目、不按日排死**，只发一封邮件，并把「下一步」字段写进 Notion。
   - **每日 brief**：改成**可选**。只在「模块进度」某模块的「下一步」非空、且当天没有进行中的 session 时生成，内容按阶段走 §3.2-3 的逻辑。或者干脆取消，完全交给本地 `/study`。两种方案都写出来，并说明取舍。
   - **周五 capstone**：取消，功能并入 ④。
   - **知识库体检**：删掉「21 天疏远」「连续 3 周建议降级」判据，只陈述数字。
   - 推送文案不出现「落后 / 未完成 / 疏远」一类词。

---

## 4. 约束

- 本 handoff 里标「已定」的决策不要推翻。你有异议可以写进计划，但默认照做。
- 不修改 Notion、不修改定时任务（这两处只由 Cowork 写，避免双写）。
- 仓库是公开的：不写 token、集群路径、未发表研究细节。
- 引用 vLLM 源码要带 commit；没读过的不要写成事实，推测标置信度。
- 做完后在 `ai-infra/redesign/README.md` 写一段 ≤ 15 行的交接：改了哪些文件、哪些草稿等 Cowork 应用、有哪些未决问题要 yijie 决定。

## 5. 回到 Cowork 后的动作（给 yijie 看，CC 不用做）

把 `ai-infra/redesign/` 的三份草稿和新的 `project-instructions.md` 交给 Cowork。Cowork 会：
- 审改；
- 更新 Notion 工作台，新建「模块进度」database 并迁移 kata schema；
- 按草稿改或建定时任务；
- 修改知识库体检；
- 归档 W1 已建的 kata 条目；
- 把 `project-instructions.md` 同步到 claude.ai Project 的 Instructions。
