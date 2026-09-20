# ai-infra 分诊表（2026-09-13）

44 条历史 chat 全部读过。评级：**A** 可直接成 note；**B** 结论方向可信但数字/代码定位需重查；**C** 丢弃（内容不足、已过时或模型编造）。

| 评级 | chat | 判断 |
|---|---|---|
| A | Qwen2.5→DeepseekV3 MoE 配置 | FPM 项目的真实设计约束，已蒸馏 |
| A | DeepSpeed MoE 路线选择 | expert_mask 需求与三方案取舍，已蒸馏 |
| A | 分布式后端重构 / 复用 | DistManager 抽象，已蒸馏 |
| A | lm_eval 环境变量（昇腾） | 暴露了真实运行环境，已蒸馏 |
| A | lm_eval harness JSON 报错 | 真实故障与根因，已蒸馏 |
| A | lm_eval 日志 HCCL 报错 | 昇腾多机通信排错，已蒸馏 |
| A | lm_eval 批量脚本 | 完整评测矩阵，已蒸馏 |
| A | pass@k 实现纠错 / repeats 关系 | 无偏估计量，已蒸馏 |
| A | 组合 HF 模型 / SegmentedAutoModel | device_map 与懒加载，已蒸馏 |
| A | jsonl 合并 / 压缩集采样 | bash vs python 的量级差异，已蒸馏 |
| A | Bayes 风险在 routing 中的应用 | 核心研究讨论，已蒸馏 → **产物移到 ClaudeContext 根目录**（算法侧，非 infra） |
| A | LLM routing cost/utility 调研 | 四篇工作的定义对照，已蒸馏 |
| A | 前沿研究博客调研 | 信息源清单，已蒸馏 |
| A | LLM 演进综述扩写 | 研究动机来源，并入 problematic |
| B | Accelerate/vLLM 显存管理 | 方向对，代码是示意的；KV cache 预分配逻辑需读 vllm 源码核对 |
| B | DeepseekV3 config 参数 | 多数正确，但把 kv_lora_rank/q_lora_rank 解释成 LoRA 微调秩是错的（实为 MLA 的低秩 KV 压缩）|
| B | 带宽公式 | GPU 显存带宽公式正确；WiFi 那段算法是编的 |
| B | S4 论文段落 | 来自原文摘要，可信；30×/400× 出自原文 |
| B | 分段加载 / 截断层 | 释放内存的点正确，但示例代码同时保留了旧层引用，自相矛盾 |
| B | ray.remote 资源 | `num_npus` 参数不存在，Ray 只有 num_gpus 与 resources |
| B | token 计数 / 超长序列 | mmap 分析可信；对 arrow/HF 数据用 tiktoken 不合适 |
| B | parquet→jsonl / KMedoids / glob | 常规工程，价值有限 |
| C | 高效推理框架论文出处 | **论文标题与会议基本是编的**（"LLM-in-Flash"、"DejaVu EuroSys 2024"、NeurIPS2023 emergent abilities 论文名）。待办：重查 PowerInfer / LLM in a flash / Deja Vu 的真实出处 |
| C | DeepSeek-MoE 负载均衡 | 模型自承不掌握，通篇示意代码。改为待办：读 DeepSeek-V3 技术报告的 aux-loss-free 负载均衡 |
| C | self-attention 矩阵 | 通识，且「下三角把 O(N²) 降到 O(N)」是错的（causal mask 只省常数）|
| C | Blackwell vs Hopper | 2024 年的猜测（"likely 3nm"），不可用 |
| C | patch embedding / in_proj / partial / namedtuple / layernorm vs rmsnorm / glob | 通识或跑题 |
| C | 论文精读模板那次 | 当时没取到论文，后半跑题到 LeetCode |
| ! | lm_eval 环境变量那条对话 | **明文贴了一个 API key（xi-ai.cn 中转站）**。没有写进任何笔记，建议去控制台吊销 |
