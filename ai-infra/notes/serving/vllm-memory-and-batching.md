# vLLM / Accelerate 的显存管理（待验证）

来源：2024-11 的对话。证据等级：**B — 方向可信，代码全是示意，未经源码核对。不要把本篇当实现依据。**

## 可信的部分

- Accelerate：`device_map="auto"` + `low_cpu_mem_usage`，支持 CPU offload；分配来自 `infer_auto_device_map`（见 [[partial-loading-and-device-map]]）
- vLLM：初始化时按 `gpu_memory_utilization` **预先吃掉**该比例显存，扣除权重与激活后剩余全部划给 KV cache block pool；核心是 PagedAttention（KV cache 分页，消除外部碎片）+ continuous batching（迭代级调度，完成的序列立即让位，新请求随时插入，而非等整批做完）

## 明确不可信的部分

对话里的 `CUDAMemoryPool`、`ContinuousBatcher` 类全是编的。「按 size 索引空闲块字典」这种朴素池化与 PagedAttention 的固定 block 设计不是一回事。

## 待验证

- [ ] 读 `vllm/worker/cache_engine.py`，确认 KV cache block 数的计算式，并用本人的 config 手算显存账（layers × 2 × kv_heads × head_dim × block_size × dtype_bytes × num_blocks）
- [ ] 读 `vllm/core/scheduler.py`，看 preemption（recompute vs swap）的触发条件
- [ ] 确认 `max_num_batched_tokens` 与 `max_model_len` 对 block 分配的约束
