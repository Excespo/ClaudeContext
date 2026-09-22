# Notion Rules v2 — 「AI-Infra 工作台」§4–§8 替换稿

> 草稿，不应用。替换 Notion 页面里现有的 §4/§5/§6/§8（§7 是新增），§0–§3 不动。
> 由 yijie 或 Cowork 审改后贴进 Notion。

---

## §4 本地 skill

- **`study`**（新）—— 路径 `ai-infra/.claude/skills/study/SKILL.md`。ai-infra 专属，按当前模块所在阶段（①minimal/②源码对照/③worked example/④PR测验）生成当天材料，替代 `vllm-kata-pull`/`vllm-kata-push` 的日历强推部分。不限时、不打分，记录"最高提示级+卡在哪"。触发词："今天学什么""开始今天的学习""/study"。
- **`note`**（更新）—— 路径 `.claude/skills/note/SKILL.md`。跨全部四个方向通用。六段骨架收尾，写 Notion 掌握度 database；这次改动后同时也会更新「模块进度」database 的阶段状态字段。触发词："写 note""记一下今天学的""/note"。
- **`vllm-kata-pull` / `vllm-kata-push`**（**已废弃**，2026-09-22）—— 实际位置在全局目录 `~/.claude/skills/`，**不在 ClaudeContext 仓库里，不受 git 管理**。已做非破坏性标记（description 前缀 `[DEPRECATED]` + 正文指向 `study`），未物理删除。两者的机制性组件（worktree 隔离命名惯例、`run_verification.sh`、`concepts.md` 概念词典）被 `study` 复用，不是推倒重写。物理删除留给 yijie 自己确认新流程跑顺后手动执行。

---

## §5 课程与 PR 规则

### 5.1 模块表

| # | 模块 | vLLM 主路径 | minimal ① | 数字账主题 |
|---|---|---|---|---|
| M0 | 端到端骨架：offline generate 循环、naive 连续 KV | `vllm/v1/engine/llm_engine.py`, `vllm/v1/engine/core.py`, `vllm/v1/request.py` | ✅ | 单请求 prefill/decode 的 FLOPs 与字节数 |
| M1 | paged KV + block pool | `vllm/v1/core/kv_cache_manager.py`, `block_pool.py`, `kv_cache_utils.py` | 未开始 | 单 block 字节数、8GB 下可分配 block 数 |
| M2 | scheduler：continuous batching → chunked prefill → preemption | `vllm/v1/core/sched/scheduler.py` | 未开始 | token budget、max_num_seqs、chunked prefill 切分 |
| M3 | prefix caching + block hash | `kv_cache_utils.py`(hash), `tests/v1/core/` | 未开始 | hit rate 对 prefill FLOPs 的节省 |
| M4 | model runner：input batch、block table、attention metadata | `vllm/v1/worker/gpu_model_runner.py`, `gpu_input_batch.py`, `block_table.py` | 部分（input batch 已做） | persistent batch 的 H2D 字节数 |
| M5 | attention backend（FA2/FlashInfer，排除 FA3/MLA） | `vllm/v1/attention/backends/` | 阅读级 | attention arithmetic intensity、roofline 拐点 |
| M6 | sampler（+ structured output 只读） | `vllm/v1/sample/` | 未开始（需先重构） | logits 张量大小、top-p sort 代价 |
| M7 | engine core 进程边界 / async / IPC | `vllm/v1/engine/`, `vllm/v1/serial_utils.py`（**勘误**：非 `vllm/v1/engine/serial_utils.py`） | — | 每 step CPU 开销 vs GPU step 时间 |
| M8 | CUDA graph / torch.compile | `vllm/v1/cudagraph_dispatcher.py`（**勘误**：非 worker/ 或 compilation/ 子目录），`vllm/compilation/` | 部分 | capture 数量 × 显存 |
| M9 | spec decode（ngram → EAGLE → parallel drafting） | `vllm/v1/spec_decode/` | 部分（ngram） | $\frac{1-\alpha^{k+1}}{1-\alpha}$ 与 draft/verify 开销 |
| M10 | 量化（W4A16、AWQ/GPTQ；FP8 只读） | `vllm/model_executor/layers/quantization/` | — | 权重字节数、dequant 带宽 |
| M11 | kernel 周（FlashInfer / sgl-kernel） | 切 repo | 部分 | SMEM、bank conflict、coalescing |
| M12 | 分布式 TP/PP + kv-connector | `vllm/v1/executor/`, `vllm/distributed/` | — | all-reduce 通信量、HCCL 对照 |

路径核对基准：`~/InfraWorkspace/projects/vllm`，commit `601e1c7c51a10d6fd884d39f079f38bb942e436f`（2026-09-20）。其余路径均已核对存在，未列勘误的路径按原表即可。

### 5.2 阶段定义

| 阶段 | 做什么 | 出口条件 |
|---|---|---|
| ① minimal | 在 `my-minimal-infra` 里实现该模块的最小版本（可以问、可以看提示） | 有能跑的代码 + 一个自测，commit 进 my-minimal-infra |
| ② 源码对照 | 读 vLLM 对应文件，写一篇「minimal vs real 设计差异」note（六段骨架） | note 入库，掌握度 ≥ 2（能复述） |
| ③ PR worked example | 读一个已合并 PR 的完整讲解，然后复述：为什么改在这里、去掉会挂哪个测试、数字账 | 复述通过，掌握度 ≥ 2 |
| ④ PR 综合测验 | 模块末尾的综合小测，见 5.3 | 做完即可（不以对错为出口） |

不适合写最小实现的模块（量化 kernel、分布式、spec decode 的部分内容）跳过①，从②开始。前几个模块只做到③，④只在该模块掌握度 ≥ 2 之后开放。

### 5.3 PR 选题规则

- **不限时**，取消 25 分钟硬限。
- **依赖硬规则**：PR 改动的所有非测试文件，都必须落在「已完成②的模块」对应的 vLLM 路径集合内（见 5.1 表）。`study` skill 选 PR 时用 `git show --stat <candidate_commit>` 核对，不满足就换下一个候选。
- **引导式**：brief 里给一条分级提示梯，每个要点配 3 级提示，全部放在 `<details>` 折叠里（见 5.4 模板）。
- **记录方式**：删除「盲写命中度」，改成「用到的最高提示级（0–3）」和「卡在哪」——这些是学习信号，不是分数。

### 5.4 brief 模板

```markdown
# PR 综合测验：<PR 标题>

模块：<Mx> ｜ PR：#<number> ｜ commit：<sha>

## 背景
<一段话说明这个 PR 解决什么问题，不超过 3 句>

## 任务
<要求做什么：读 diff 前先做什么，读完之后要交付什么>

## 提示梯

### 要点 1：<一句话概括>
<details><summary>L1 — 方向提示</summary>
涉及哪个数据结构 / 哪个调用阶段。
</details>
<details><summary>L2 — 函数与约束</summary>
给出函数名和约束条件。
</details>
<details><summary>L3 — 伪代码</summary>
给出伪代码级别的实现思路。
</details>

### 要点 2：...
（同上结构，每个要点一份）

## 数字账
<这个 PR 涉及的关键数字：改动前后的复杂度/开销对比>

## 完成后
- 在 kata 条目里记录：最高用到的提示级（0-3）、卡在哪
- 跑 `/note` 收尾
```

---

## §6 定时任务表

| 任务 | trig_id | 状态 | 说明 |
|---|---|---|---|
| 每日 brief | `trig_011YdiiqbXa57SB5EMLuP2JW` | 见 `trigger-prompts-v2.md`，两个方案待选 | 原含周六实验，已手动暂停 |
| 周五 capstone | `trig_01TtFUzro5gbFspQ9dQqjyxu` | **取消**，并入④ | 已手动暂停 |
| 周日规划 | `trig_01Nn28LkoeVXWVewBbSB6AXe` | 改为新版本，见 `trigger-prompts-v2.md` | 已手动暂停 |
| 知识库体检 | `trig_01UaLo7L3P4Brh8T8dGMSu2Z` | 改为新版本，删除疏远/降级判据 | 尚未改，本次改 |

状态与留痕分工三层原则（沿用，不改）：
- **git 仓库（留痕）**：note 正文在 `~/ClaudeContext/ai-infra/notes/`，minimal 代码在 `~/InfraWorkspace/my-minimal-infra`。
- **Notion（状态，定时任务读写）**：「模块进度」database（新增，见 `notion-schema-v2.md`）+ 掌握度 database + kata database。
- **不双写**：仓库不重建 `00-index.md`/`inbox.md`/`ROADMAP.md`/`STATUS.md`。

---

## §7 hook：my-minimal 与模块进度的挂钩方式

这是**人工/半自动**惯例，当前没有 CI/webhook，不是自动化触发：

- `/study` 结束一次①阶段 session 时，把当天 my-minimal 的 commit hash 手动写进「模块进度」database 对应模块的 "minimal commit" 字段。
- `/note` 收尾时，同步更新「模块进度」database 对应阶段的状态字段（`进行中`→`完成`）、note 路径。
- 模块从①推进到②，或从②推进到③/④，都由 `study` 在下一次调用时读「模块进度」的阶段字段决定生成什么材料——阶段推进的判断（出口条件是否达成）目前由 yijie 自己在 Notion 里手动切换「当前阶段」字段，`study` 不自动判定「能复述」「能改」是否达标。

---

## §8 定时任务 Prompt 全文

见 `trigger-prompts-v2.md`（本文件不重复，避免两处维护同一份 prompt 正文）。
