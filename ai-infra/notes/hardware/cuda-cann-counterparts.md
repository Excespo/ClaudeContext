# CUDA ↔ CANN 对照表（**逐项待核实**）

学习方式：**CUDA 是概念基座，这张表是把概念搬到昇腾上时的翻译层。** 每一行只有在你亲手在两边各验证过一次之后，才把「状态」改成「已验证」。

> 证据等级：下表除标注「已验证」的行外，全部是**待核实的先验**，不要当事实引用。错误的对照比没有对照更危险——它会让你以为自己懂了。

## 编程模型与执行单元

| CUDA | 昇腾 / CANN | 关键差异（待填） | 状态 |
|---|---|---|---|
| kernel | Ascend C kernel | 编程模型是否也是「一份代码多实例」 | 待核实 |
| SM | AI Core | AI Core 内含 Cube（矩阵）与 Vector 单元，与 SM 的 SIMT 抽象不是一回事 | 待核实 |
| warp / SIMT | — | **可能没有直接对应**；这一格若真的空着，是理解差异的关键入口 | 待核实 |
| occupancy | — | 若无 warp 概念，occupancy 的类比对象是什么 | 待核实 |

## 存储层级

| CUDA | 昇腾 / CANN | 关键差异（待填） | 状态 |
|---|---|---|---|
| 全局内存 / HBM | HBM | 带宽数值与访问粒度 | 待核实 |
| shared memory (SMEM) | Unified Buffer / L1 Buffer | 是否由程序员显式搬运（ping-pong / double buffer） | 待核实 |
| register file | — | | 待核实 |
| coalescing | — | 昇腾侧的对齐与搬运约束形式 | 待核实 |

## 通信

| CUDA | 昇腾 / CANN | 关键差异 | 状态 |
|---|---|---|---|
| NCCL | **HCCL** | ring/tree 是否都有；拓扑感知逻辑 | 待核实 |
| NVLink / NVSwitch | HCCS / RoCE | 单向带宽与跨机拓扑 | 待核实 |
| `NCCL_*` 环境变量 | `HCCL_*`（如 `HCCL_CONNECT_TIMEOUT`） | **已验证**：超时与建链失败的排错路径见 ascend-npu-env.md | 已验证 |

## 工具链

| CUDA | 昇腾 / CANN | 关键差异 | 状态 |
|---|---|---|---|
| nsys（timeline） | msprof / MindStudio Insight | 本机已装 MindStudio Insight | 待核实 |
| ncu（kernel 级 roofline、occupancy） | ? | **最可能缺失或形态不同的一格** | 待核实 |
| cuBLAS / CUTLASS | aclnn 算子库 / ATB | 手写 fused kernel 的入口在哪 | 待核实 |
| `CUDA_LAUNCH_BLOCKING=1` | `ASCEND_LAUNCH_BLOCKING=1` | **已验证**，语义一致（同样显著降速） | 已验证 |
| `CUDA_VISIBLE_DEVICES` | `ASCEND_RT_VISIBLE_DEVICES` | 待核实拼写 | 待核实 |

## 怎么用这张表

1. 学完一个 CUDA 概念 → 回来填对应那一行的「关键差异」
2. 填不出来就去 CANN 文档查，查不到就记成「文档未覆盖」——**这本身是有价值的观察**
3. 每填完一行，在 ROADMAP 的对应 `[CANN]` 验证项上打勾
