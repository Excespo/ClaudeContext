# 大规模语料处理：什么时候别用 Python

来源：2025-03/04 的五次对话。证据等级：**A，来自本人在 dolma 上的实际处理**。

## 场景

dolma-v1_6 → 按 1% 概率行采样出 `dolma_sampled_30B` → 按类别（books / cc_en_head / cc_en_middle / cc_en_tail / reddit / stack）合并成每类一个 jsonl。

## 最重要的一条结论

**只要不需要解析 JSON 内容，`cat` 比 Python 快一个量级。**

原 Python 版对每行做 `json.loads` → 存 list → 再 `json.dumps` 写出，等于为「把文件拼起来」这件事付了一次完整的序列化往返 + 几千万个 Python 对象的分配与 GC。bash 版 `find | grep <category> | xargs cat >> out.jsonl` 是纯字节流复制，没有解析、没有对象、没有 GIL。

判据：**任务是否需要理解每行的结构**。需要（过滤字段、改 schema）就用 Python；只是分类和拼接就用 shell。

## 多进程的粒度

以**文件**为单位分进程是对的（`gzip.open` 解压是 CPU 密集，文件之间天然独立，无需通信）。以行为单位需要协调同一文件的读写，通信开销盖过收益。`n_proc = cpu_count()/2` 是为了给解压之外的 IO 留余量。

判断瓶颈在解压还是 IO 的方法（这才是可复用的部分）：
1. 在 `random_sample_single_file` 里分段计时（解压 / 判定 / 写出）并把三个数汇总
2. `cProfile` 按 cumtime 排前 20
3. 跑的同时用 `psutil` 采 CPU% 与 `disk_io_counters`
判据：CPU 接近满载 → 解压为主，加进程数无用，应换 `pigz` 或预转格式；CPU 低而 iowait 高 → IO 为主，加并发有效。

## 零散事实

- `datasets.load_dataset(path="~/data/x")` 会失败：**Python 不展开 `~`**，必须 `os.path.expanduser`。报错信息里会把 `~` 拼成字面路径，很好认。
- parquet → jsonl：`Dataset.from_parquet(..., num_proc=N).to_json(..., num_proc=N)` 够用；分片大小按「抽样 1000 条算平均字节数 × 总条数 / 目标分片大小」估。
- 大文件 token 计数用 `mmap` 的收益来自零拷贝与页缓存按需加载；劣势是 32 位地址空间、随机写、NFS 上表现差。**但对 HF arrow/parquet 不要用 mmap+tiktoken**，直接用 datasets 的列式读取。
