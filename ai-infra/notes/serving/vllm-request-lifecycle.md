# 进程边界和配置同步

> 上游 PR：#39102（`7c94ae16c6c265a095235ee90c87226e931ca409`），vLLM v1 engine。

## 结论

`max_model_len=-1`（auto-fit）时，EngineCore 子进程把 auto-fit 后的值改在自己那份 `vllm_config` 拷贝里，从不同步回 frontend 进程；frontend 的长度校验永远读着旧的、更大的占位值。**可证伪**：在 `7c94ae16c~1`（修复前）上，用极小 KV 预算跑 auto-fit，发一个介于「旧上限」和「新上限」之间长度的 prompt——如果 frontend 直接放行且请求最终卡在 `allocate_slots`/抢占循环，结论成立；如果 frontend 提前用新值拒绝，结论不成立。

## 机制

**启动路径（frontend 进程）**
`vllm/v1/engine/async_llm.py:AsyncLLM.__init__` → `vllm/renderers/__init__:renderer_from_config`（建 renderer，早于 client）→ `vllm/v1/engine/core_client.py:EngineCoreClient.make_async_mp_client` → `MPClient.__init__`（拉起 EngineCore，在 **input socket** 上等每个 engine 的 ready 帧）

**启动路径（EngineCore 子进程）**
`vllm/v1/engine/core.py:EngineCoreProc.run_engine_core` → `_perform_handshake`（**handshake socket** 发 `{"status":"READY","num_gpu_blocks":...}`，此时还没有 `max_model_len` 字段）→ `EngineCore._initialize_kv_caches` → `vllm/v1/core/kv_cache_utils.py:get_kv_cache_configs → _auto_fit_max_model_len`（二分搜索，原地改 `model_config.max_model_len`）→ 值变化时 `collective_rpc("update_max_model_len")` 同步给 worker → `process_input_sockets` 线程在每个 **input socket** 上发首帧（改动点①：原来是 `b""`，现在带 auto-fit 后的值）

**请求路径**
`vllm/renderers/base.py:BaseRenderer.default_cmpl_tok_params`（`@cached_property`，读 `model_config.max_model_len`）→ `TokenizeParams._token_len_check`（修复后应在此拦截）→ 修复前放行 → `vllm/v1/core/sched/scheduler.py:Scheduler.schedule`：running 循环 L413 `num_new_tokens = min(..., max_model_len-1-num_computed_tokens)` 用的是 engine 侧已改小的值 → L463 `allocate_slots` 失败 → 抢占 → break

**关键对照**：worker 方向的同步（`Worker.update_max_model_len`）在这个 PR 之前就已经存在且正确；frontend 方向是漏掉的那一半——同一个「配置改了要不要通知别人」的问题，一个方向修了、另一个方向没修。

## 数字

代入值：Qwen2.5-1.5B（`num_hidden_layers=28`, `num_key_value_heads=2`, `head_dim=128`），bf16，`block_size=16`，`--kv-cache-memory-bytes 536870912`（512 MiB）。

- 单层单 block：`2 × 16 × 2 × 128 × 2 = 16,384 B`
- 跨 28 层：`16,384 × 28 = 458,752 B/block`
- `num_gpu_block = 536,870,912 / 458,752 = 1170`
- auto-fit 后 `L = 18,720`（`cdiv(L,16) ≤ 1170` 的最大 L）
- 可用 slot 数（刨去 1 个 null_block）：`(1170-1) × 16 = 18,704`，比 `L-1=18,719` 小 → 20000-token 请求先撞 `allocate_slots`，不是撞 `max_model_len-1` 截断
- 触发 auto-fit 的显存临界值：`32,768 × (458,752/16) ≈ 896 MiB`；4070 在 0.9 显存预算下 KV 剩余空间远大于此，所以不加 `--kv-cache-memory-bytes` 复现不了——理论值，未在真实 4070 上跑 `vllm serve` 实测确认（brief §8 给了实测命令，今天没时间跑）。

## 我错在哪

*(待填 - 见上方提问)*

## 证伪观察 + 置信度

- **推测 A（置信度 0.55，未测试）**：auto-fit 二分没扣掉 null_block，长度恰好等于 L 的合法请求也可能放不下。证伪实验：prompt=18,700 + max_tokens=19，若正常结束无抢占则 A 错。
- **推测 B（置信度 0.7，未测试）**：PR 自带测试用的 llama-160m 配置算出来只有 1 个可用 block，全被 null_block 占掉。证伪实验：测试里打印 `num_gpu_blocks` 实际值。
- 两条都没做实验证伪，时间不够——下次复习时应该先补这两个实验，而不是从头重读代码。

## 复查日期

2026-09-28（掌握度「2 能复述」→ 7 天）
