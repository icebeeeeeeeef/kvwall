# 09 学习路线：前置、融合、项目外

本文件回答：项目启动前必须会什么；每章开始前学什么；哪些知识在项目里实践、哪些只在项目外理解或小幅实操。它服务于 `02` 的章节顺序，不改变主线。

初始化日期：2026-09-23。

## 1. 原则

1. **三类知识**：
   - **前置**：不会就读不懂引擎 KV 路径、做不了 Ch0 算术。项目启动前完成。
   - **项目内实践**：直接变成 `micro/`、`analysis/`、`bench/`、`writeups/` 或 PR 里的产出。
   - **项目外**：主线展开不了的，只做理解或小实操，产出进 `notes/`。
2. **just-in-time**：每章的知识只在开章前一周内学，不提前囤积。
3. **学习不算证据**（`01` §4.3）：学到的东西必须转成微基准代码、writeup 段落、PR 或源码拆解文章之一，否则只算准备。
4. **产出归位**：阅读笔记与练习脚本进 `notes/`；能当测量工具的进 `micro/` 或 `analysis/`；进 `src/` 仍受 D-008 约束。
5. **自研微基准按引擎访问模式构造**（D-016）：这是阶段 1 写 C++ / CUDA / io_uring / verbs 代码的正当通道。

## 2. 深度等级

| 等级 | 含义 | 自检标准 |
|---|---|---|
| L1 了解 | 知道是什么、解决什么问题 | 能用两三句话讲清 |
| L2 理解 | 懂原理与取舍，能估算 | 能画图、能做数量级估算、能比较方案 |
| L3 掌握 | 动手实现或改造过 | 有能跑的代码与带 manifest 的数据 |
| L4 精通 | 能做生产级设计与优化 | 功能级 PR；能讲清设计决策 |

## 3. 总览

```
预备期（W1–W3）    纯学习 + ★★★ 阅读；项目只产出 notes/ 与容量算术
阶段 0（W4–W9）    平台骨架 + micro/pcie + Ch0 + 源码拆解文章 + 小 PR
阶段 1（M3–M5）    Ch1 → Ch2 → Ch3 → Ch4
  └ 副线           RDMA verbs（Soft-RoCE）+ Transfer Engine 源码 → Ch5 预热
阶段 2（M6+）      Ch5 → Ch6 → Ch7 → Ch8
始终不展开          CXL / SPDK 深入 / TRT-LLM / FlashAttention 实现 / 投机解码实现
```

时间是相对刻度，只用于排序；推进以每步完成标准为准。

## 4. 预备期（W1–W3）：前置

只学「读懂 vLLM KV 路径」与「做 Ch0 算术」所必需的东西。

| 编号 | 前置知识 | 深度 | 实操 | 完成标准 |
|---|---|---|---|---|
| A | Transformer 推理与 KV cache | L2 | 需要（小） | 给一个 0.5B 级小模型手写带 KV cache 的 decode 循环，logits 与 HF 对齐；能说清 prefill 计算受限、decode 访存受限的原因 |
| B | attention 变体与 bytes/token | L3 | 手算 | 从 `config.json` 手算正式模型与开发模型的 bytes/token，与 `02` §4 一致；能说明 GQA / MLA / 滑动窗口 / 混合层各自如何改变它 |
| C | PyTorch 与 CUDA runtime 最小集 | L2 | 需要（小） | dtype / device、`pin_memory`、`non_blocking`、`torch.cuda.Stream` / `Event`；pinned vs pageable 的 H2D 计时脚本（`micro/pcie` 雏形） |
| D | vLLM V1 KV 路径 | L3 源码级 | 读源码 + 画图 | 画出请求路径：scheduler → KV cache manager 分配 → block hash 前缀命中 → connector 查询外部命中 → worker load / save → 释放与驱逐；知道 per-tier metrics 与 KV events 的发出点 |
| E | ★★★ 阅读 | L2 | 否 | PagedAttention、Mooncake、vLLM tiered offloading、GLM 5.3 Part 1；按 `notes/README.md` 模板各一页 |
| F | 测量统计 | L2 | 否 | 分位数与样本量（p99 需要多少样本才稳）、预热、方差、Little's law；`bench/slo/` 要用 |
| G | 工具入门 | L1→L2 | 需要（小） | 开发机上用 nsys 抓一次 vLLM decode 时间线、用 py-spy dump 一次调度线程并读懂 |

**本期不学**：RDMA、io_uring、SPDK、GDS、KV 量化理论、PD 分离、FlashAttention 内部、CXL。

**产出**：`notes/reading/` 四篇；`analysis/capacity/` 第一版容量算术；A、C 的练习脚本放 `notes/`。

## 5. 阶段 0（W4–W9）：平台骨架 + Ch0

| 类型 | 内容 |
|---|---|
| 项目内实践 | `04` §6 的三个「一条命令」；`src/kvwall/common/`（manifest、`/metrics` 解析、硬件信息采集）；`micro/pcie/` 第一版（PyTorch 版，阶段 1 换 CUDA C++）；Ch0 负载刻画与容量模型；源码拆解文章（`06` §5 列出的 KV 路径） |
| 边做边学 | trace 分析（pandas）；EvalScope 多轮配方；Prometheus 数据模型（counter / gauge / histogram、counter 差分）；vLLM 贡献流程 |
| 小 PR | 文档、测试、metrics 描述、复现脚本修复——目的是熟悉 review 流程 |
| 项目外 | C++ 复习：RAII、移动语义、`std::thread` / atomic、内存序基础。小实操：线程安全的固定大小内存池（pinned buffer 池的 CPU 原型） |

CUDA 目标收窄为「CUDA runtime + 能读懂一个 paged attention kernel 的接口（block table 如何传入）」，不要求读懂 kernel 实现（D-018）。

**出口**：开发机上一条命令跑完 7B 小扫描并出图；`08` 「模型与负载」「软件栈」两节大部分关闭。

## 6. 阶段 1（M3–M5）：Part I 按章融合

### Ch1 HBM 层

| 类型 | 内容 |
|---|---|
| 开章前学 | V1 抢占方式（swap / recompute）现状；块大小与碎片；CUDA graph 显存预留；FP8（E4M3 / E5M2）与 KV 量化的 scale 粒度 |
| 项目内实践 | FP8 KV 臂；方法论第 6 步的精度副作用检查（困惑度或下游任务）；nsys 解释预测与实测差距 |
| 项目外 | KIVI（K 按通道、V 按 token）、INT4 KV，L2。小实操：导出开发模型 KV，离线做 FP8 / INT8 / INT4 量化并看误差分布，结果进 `notes/` |

### Ch2 主机内存层（CUDA 系统层主战场）

| 类型 | 内容 |
|---|---|
| 开章前学 | PCIe（Gen4 x16 理论与可达、DMA、根复合体）；NUMA；stream / event 与拷贝计算重叠；pinned 分配成本；vLLM offloading worker 的拷贝实现 |
| 项目内实践 | `micro/pcie/` 升级为 CUDA C++：pinned vs pageable × chunk 尺寸（按实际 `blocks_per_chunk`）× 流数 × NUMA × 4 卡并发；nsys 确认与 decode 的重叠；`analysis/` 下 trace 驱动的缓存模拟器，预测池大小 vs 命中率并与实测对比，LRU vs ARC 在模拟器中比较 |
| 项目外 | 小实操：paged KV gather / scatter CUDA kernel（离散 block ↔ 连续 buffer），对比 `cudaMemcpy2DAsync` 与逐块 memcpy；若结论与差距分析相关则并入 writeup |
| 只需理解 | S3-FIFO；前缀树约束下的淘汰，L2 |

### Ch3 存储层（Linux IO 主战场）

| 类型 | 内容 |
|---|---|
| 开章前学 | Linux IO 栈（page cache、脏页回写、O_DIRECT 对齐、blk-mq）；io_uring（SQ / CQ、SQPOLL、registered buffers / files、IOPOLL）；NVMe 队列模型；云盘与物理 NVMe 的差别 |
| 项目内实践 | fio 天花板；`micro/blockdev/` 下 liburing + O_DIRECT 按 fs tier chunk 布局的读写器；perf / py-spy 每 IO CPU 成本；iostat。数据指向时再写自定义 `SecondaryTierManager`，复用 micro 代码 |
| GDS | 机型支持则做直通 vs host 中转对照；不支持则用 cuFile 兼容模式学 API，writeup 中分析框架为何走 host，并标注未实测 |
| 项目外 | SPDK（D-019）：malloc / aio bdev 无需 NVMe 硬件，跑通 hello_bdev，读架构文档（轮询、无锁、独占核、用户态驱动），L2 |
| 只需理解 | NVMe-oF；SSD GC 与稳态性能，L1–L2 |

### Ch4 同节点共享池

| 类型 | 内容 |
|---|---|
| 开章前学 | MooncakeStore 架构（master、client、副本、租约）；内容寻址 key 语义；多写者、RETRY |
| 项目内实践 | MooncakeStore 本地部署；读 Mooncake Store C++ 源码（驱逐、租约、put / get）；双写者竞争测量；第一个 Mooncake PR 的机会点 |
| 项目外 | 一致性哈希、租约、「缓存可丢」下的弱一致设计；Raft 读到 L2 |

### 阶段 1 副线：RDMA 预热（Ch3 开始后，每周固定少量时间）

目的：缩小 D-006 的代价；阶段 2 开机即测，不在租来的多机上边学边调。

| 步骤 | 内容 | 深度 |
|---|---|---|
| 1 | Linux 开发机配置 Soft-RoCE（rxe） | — |
| 2 | verbs 程序：注册 MR → 建 RC QP → 单边 WRITE 大块数据 → 轮询 CQ | L3 |
| 3 | MR 注册耗时随大小的变化（Soft-RoCE 上只看趋势，不出正式数字） | L3 |
| 4 | 读 Mooncake Transfer Engine 源码：拓扑感知选网卡、批量传输、多网卡聚合 | L2→L3 |
| 5 | 读 NIXL 抽象层，与 Transfer Engine 对比 | L2 |

**产出**：`micro/rdma/` 自研基准代码（Ch5 在 eRDMA 上直接跑）；Transfer Engine 源码拆解文章。

## 7. 阶段 2（M6+）：Part II / III

| 章 | 开章前学 | 项目内实践 | 项目外 / 只需理解 |
|---|---|---|---|
| Ch5 | eRDMA 特性；GPUDirect RDMA（nvidia-peermem / dma-buf）；UCX 基础 | perftest 与副线自研基准在 eRDMA 实测；GPUDirect RDMA 可用性探测；Grafana 面板与告警；热启动曲线 | RoCE 拥塞控制（PFC / ECN / DCQCN），L1–L2 |
| Ch6 | DistServe；NixlConnector 源码；TTFT 分解方法 | GPU 直传 vs host 中转；P/D 比例扫描 | P/D 不同 TP 下的 KV 重排，L2；传输失败回退重算、取消请求的资源回收，读源码到 L2 |
| Ch7 | SGLang router / Dynamo KV router 源码；KV events | 命中率 vs 负载均衡曲线；router 瓶颈 | 带负载上限的一致性哈希，L2 |
| Ch8 | MLA（DeepSeek-V2）；混合架构 state cache；Marconi；HiSparse | 重跑 Ch1–3；hybrid allocator 边界 | 读 FlashAttention / FlashInfer paged 接口中 block table 的传法与 MLA kernel 的 KV 布局，只读不写，L2 |

## 8. 始终不在项目里展开

| 知识点 | 处理 | 深度 |
|---|---|---|
| CXL | 讲清在层级中的位置（DRAM 与 RDMA 远端内存之间）、延迟特性、现状 | L1–L2 |
| FlashAttention | tiling 思想与 paged 接口，不实现 | L1–L2 |
| 投机解码 | 原理；被拒 token 的 KV 回滚对缓存管理的影响 | L1–L2 |
| TensorRT-LLM | 架构与 KV 管理方式 | L1 |
| CacheGen / CacheBlend / H2O 等 | 压缩与非前缀复用思路 | L2 |
| 3FS / FlexKV / LMCache | 设计文档；能与 Mooncake、vLLM tiering 做方案对比 | L2 |
| Rust / Go | 阶段 2 视目标团队技术栈决定 | — |

## 9. 维护

- 每章结束在 `notes/retro/` 核对本章知识项：完成、降级（改为项目外）、推迟。
- 调整本路线的方向性内容，先写 `07`。
- 项目外小实操若产出了与某章差距分析相关的结论，移入该章 writeup，并按 `03` 补 manifest。
