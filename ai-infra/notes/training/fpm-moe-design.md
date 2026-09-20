# FPM：从多个 dense 模型拼 DeepSeek-V3 式 MoE

来源：2025-04 的四次对话。证据等级：**设计约束来自本人需求（可信）；transformers 实现细节需回源码核对**。

## 目标

N 个 Qwen2.5-1.5B dense 模型 → 一个 DeepSeek-V3 结构的 MoE：
- embedding 与 self_attn 取 N 个模型的**逐元素平均**（因此这些层的形状必须完全一致）
- expert = N1 个 shared expert + N2 个 function-specific group × 每组 G 个 expert
- group 名由 `func_group_names` 给出（如 base, base, math, code, code, code），`shared_group_name="base"`
- 类是 `FPMConfig(DeepseekConfig)`，`model_type = "fpm"`，入口 `build_from_dense_expert_configs(func_model_ckpts, func_group_names, shared_group_name, **kwargs)`

## 关键约束（整件事的要害）

既然要平均 self_attn，**DeepSeek 侧的维度必须继承 Qwen，不能用 DeepSeek 默认值**：

| 必须从 reference_config 继承 | 为什么 |
|---|---|
| `hidden_size`（1536，不是 7168） | 否则 attn 权重形状对不上 |
| `intermediate_size`、`num_hidden_layers`（28，不是 61） | 同上 |
| `num_attention_heads`(12)、`num_key_value_heads`(2) | GQA 结构 |
| `vocab_size`(151936)、`bos/eos_token_id`(151643) | tokenizer 一致性 |
| `max_position_embeddings`、`rope_theta`(1e6)、`rms_norm_eps`、`hidden_act`、`tie_word_embeddings` | 行为一致性 |

只有 MoE 专属参数才用 DeepSeek 默认：`n_shared_experts=N1`、`n_routed_experts=N2×G`、`n_group=N2`、`moe_intermediate_size`、`num_experts_per_tok`、`first_k_dense_replace`、`topk_method`、`scoring_func`、`norm_topk_prob`、`routed_scaling_factor`。

`moe_intermediate_size` 不能照抄 2048——它决定总参数量，应由 Qwen 的 intermediate_size 与专家数推算。

## DeepSeek-V3 的路由是两阶段

`n_group=8` → 先按组给分，选 `topk_group=4` 个组 → 只在这些组内选 `num_experts_per_tok=8` 个 expert。配合 `first_k_dense_replace=3`：前 3 层恒为 dense。层分布形如 `Embed → Dense×3 → MoE×58 → LM Head`。

**勘误**：当时的回答把 `kv_lora_rank=512` / `q_lora_rank=1536` 解释成「LoRA 微调的秩」，这是错的。它们是 **MLA（Multi-head Latent Attention）的低秩 KV 压缩维度**，属于架构本身，与参数高效微调无关。

## 训练路线（当时的决策）

需求：输入带 `function_tag` → 生成 `expert_mask`，逐层限定可用 expert 序号；每层走 1 个 shared expert + router 选出的 top-(k−1) 个。

三条路径的取舍：
1. `deepspeed.moe.layer.MoE` —— 直接拿到 expert parallel，但不支持 shared expert 与 expert_mask
2. **迁移 `transformers` 的 deepseek 实现，只改 MoE 层，自己实现 expert parallel** ← 选这条
3. 从头写 —— 成本最高

里程碑：读懂 DeepSeek MoE 层 → 加 shared expert → 把 expert_mask 接到 router 的 topk 之前 → DeepSpeed expert 分片与 all-to-all → 单测 mask 与 router → 小规模跑通 → 监控专家负载。

## 未解决

- [ ] `expert_mask` 作用在 router logits（置 −inf）还是 topk 之后？两者在负载均衡统计上不等价
- [ ] expert parallel 的 all-to-all 通信量推导（tokens × hidden × top-k / EP size）
- [ ] DeepSeek-V3 的 **aux-loss-free 负载均衡**（bias 项如何更新）——当时那次对话没有真实信息，必须回技术报告读
