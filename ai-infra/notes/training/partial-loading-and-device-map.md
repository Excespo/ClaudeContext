# 部分加载模型权重与 device_map

来源：2025-03 的三次对话，其中两次本人纠正了模型的说法。证据等级：**B，方向正确，需回 accelerate 源码确认**。

## 场景

自定义 `nn.Module`（`SegmentedAutoModelForCausalLM`：front/back segment 各持有一个 AutoModelForCausalLM 的一部分层），权重要从**多个 checkpoint 按 pattern 部分读取**再绑定。

## 加载慢的根因

```python
model = AutoModelForCausalLM.from_config(cfg)                         # 空模型
state_dict = AutoModelForCausalLM.from_pretrained(path).state_dict()  # 又完整加载一次
```
内存里加载了两份模型，且绕开了 transformers 的分片加载与 mmap。

方向：
1. `init_empty_weights()` 在 meta 设备建骨架（不占内存）
2. 预先算 `{ckpt: [需要的 key]}`，用 `safetensors.safe_open` 只读需要的张量（mmap + 惰性，按 key 取不加载整文件）
3. `load_state_dict(..., assign=True)` 或 `accelerate.load_checkpoint_in_model` 直接落到目标设备

## infer_auto_device_map 的两个事实

- **它不需要把权重加载到 CPU**：`init_empty_weights()` 在 meta 设备建结构，靠 `numel × element_size` 算模块大小，再按 `max_memory` 贪心分配；`no_split_module_classes` 指定不可跨设备切分的模块（通常是 DecoderLayer）。模型最初说「要先加载到 CPU」，被本人质疑后更正——**采信更正后的版本**。
- **但对本场景仍不好用**：自定义模型持有「结构上存在、forward 不走」的层，自动推断按完整结构分配，结果偏大。三条出路：传只含实际使用层的精简副本；按 `named_modules()` 自己算；层数固定时直接硬编码 device_map。

## 截断层时内存在哪一步释放

```python
new_layers = nn.ModuleList(list(layers)[:depth])
setattr(transformer, "layers", new_layers)   # ← 此后旧层失去引用才会被 GC
```
**原对话的代码自相矛盾**：它同时 `self.truncated_layers = layers[:depth]`，又持有一份引用，内存并没释放。要么不保存，要么显式 `del`。必要时 `gc.collect()` + `torch.cuda.empty_cache()`。
