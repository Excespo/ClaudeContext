# lm-evaluation-harness 使用与踩坑

来源：2025-03/04 的四次对话。证据等级：**A，来自本人的实际脚本与报错**。

## 评测矩阵（本人在用的）

| 类别 | task |
|---|---|
| 通用 | hellaswag（10-shot）、agieval（3-shot） |
| 知识 | mmlu、mmlu_pro（5-shot） |
| 代码 | humaneval、humaneval_plus、mbpp、mbpp_plus（0-shot，需 `--confirm_run_unsafe_code` + `HF_ALLOW_CODE_EVAL=1`） |
| 数学 | gsm8k、gsm8k_cot（8-shot，`max_gen_toks=512`）、hendrycks_math（4-shot）、gpqa_diamond_cot_n_shot（4-shot） |

vLLM 后端参数：`pretrained=$ckpt,dtype=auto,max_model_len=4096,data_parallel_size=8,gpu_memory_utilization=0.8`，`--batch_size auto`。
HF 后端等价写法：`accelerate launch -m lm_eval --model hf --model_args pretrained=$ckpt,dtype=auto`——注意 `data_parallel_size` / `max_model_len` / `gpu_memory_utilization` 是 vLLM 专属，换 hf 时要删掉，改用 accelerate 控制并行。

**断点续跑**：靠探测 `${output_path}/${task}/**/results_*.json` 是否存在来跳过已完成的 task（`compgen -G`）。

## 故障：结果序列化时 `TypeError: keys must be str, int, float, bool or None, not function`

发生在 `lm_eval/__main__.py` 的 `cli_evaluate()` 里 `json.dumps(...)` 那一步，即**所有评测都跑完之后**——最坑的一点是白跑一遍才炸。

根因：task yaml 里写了
```yaml
metric_list:
  - metric: !function utils.pass_at_k
```
`!function` 把函数对象本身放进了结果字典的键位，而 json 不接受函数作键。

处置：给自定义 metric 一个字符串名（`metric: pass_at_k` + 在 utils 里注册），或在序列化前把非法键转成字符串。同时检查自定义 metric 函数有没有返回带函数键的 dict。

**顺带的真实 bug**：当时的 `pass_at_k(references, predictions, k)` 里写了 `k_list = k; for k in k_list:`，参数与循环变量同名，后续再用 `k` 已被覆盖。

## 参考 config（math_500）

```yaml
task: math_500
output_type: generate_until
doc_to_text: "Problem: {{problem}}\nSolution:"
process_results: !function utils.process_results
generation_kwargs: {until: ["Problem:"], do_sample: true, temperature: 0.6, top_p: 0.95, max_gen_toks: 128}
repeats: 16
```
`max_gen_toks: 128` 对数学推理偏小，容易在写完推理前被截断——见 [[pass-at-k]] 的讨论。

相关：[[pass-at-k]]、[[../hardware/ascend-npu-env]]
