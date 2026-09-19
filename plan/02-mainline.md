# 02 技术主线

**《KV 容量墙：从 HBM 到远端池，SLO 约束下的并发前沿与每一层的硬件天花板》**

## 1. 脊柱问题

给定 SLO（p99 TTFT ≤ X、p99 TPOT ≤ Y），一组内存吃紧的 PCIe 推理卡能撑多少个长上下文 agent 会话；每加一层缓存，这个数字移动多少、代价是什么、离该层硬件天花板差多远、到哪里失效。

每一章 = 一层 = 一个硬件层 = 一个闭环 = 一篇文章。

## 2. 设计原则

1. **最小代表性配置**：在最小的、能让现象以生产形态出现的配置上做。低于它是玩具，高于它是浪费。
2. **平台而非引擎**：长期维护的是实验平台（回放器、测量口径、微基准、容量模型、面板）；组件只在多个闭环指向同一个坑时才长出来，且以现有系统插件 / 后端形式交付。
3. **先打天花板，再看引擎达到多少，再解释差距**：每章的硬件层都按这个套路。
4. **机制研究与生产声明分开标注**。

## 3. 统一的模型与硬件选择

- **正式模型**：vLLM tiered offloading 与 GLM 系列博客所用的 30B-A3B 级 FP8 MoE 同族模型（具体版本见 `08` 开放问题）。同模型不同硬件 / regime，结果可与框架作者的直接对读。
- **开发模型**：7B 级 dense，仅用于在 1 卡上跑通脚本，不出正式数字。
- **单实例正式单位**：一节点 2–4×L20（48GB，PCIe，无 NVLink），TP=2 / 4。
- **多机**：两节点 × 4×L20，eRDMA；CPU-only 节点可选（远端池）。
- **可选一次性大点**：8×L20 或 4×H20 跑 235B-A22B 级模型一个点，验证趋势。
- **架构维度模型**：一个能在 2–4×L20 上跑的 MLA 模型；一个混合架构（滑动窗口 / 线性 / Mamba 层）模型。

## 4. 容量算术（粗算，正式以实际 config 为准）

```
bytes_per_token = 2 (K,V) × L_kv_layers × H_kv × d_head × bytes_per_elem
resident_tokens = (N_gpu × HBM × util − W_weights − R_runtime) / bytes_per_token
sessions_at_L   = resident_tokens / L
```

30B-A3B 级（公开配置：48 层、4 个 KV 头、head_dim 128）：BF16 ≈ 96 KB/token，FP8 ≈ 48 KB/token。权重 FP8 ≈ 30.5 GB。L20 按 util 0.92 ≈ 44 GB/卡；运行时预留（激活、CUDA graph、采样缓冲）粗估 3 GB 起，随 TP 上升。

| 配置 | KV 可用 | 驻留 token（FP8） | 64K 会话数 |
|---|---|---|---|
| 1×L20 | ≈ 10 GB | ≈ 210K | ≈ 3（太早，只能当开发环境） |
| 2×L20 TP2 | ≈ 52 GB | ≈ 1.1M | ≈ 17 |
| 4×L20 TP4 | ≈ 137 GB | ≈ 2.9M | ≈ 44 |

7B 级 dense（28 层、4 KV 头、128）：BF16 ≈ 56 KB/token；1×L20 ≈ 26 GB KV → ≈ 460K token → 64K 会话 ≈ 7。

host 层：例如 256 GB pinned 池 ≈ 5.5M token ≈ 85 个 64K 会话；池大小 vs 命中率是 Ch2 的曲线。

磁盘 / 远端层交叉点：

```
reload_time(L)    = L × bytes_per_token / BW_tier + lat_tier
recompute_time(L) = prefill_time(L)          # 实测
L*: reload_time(L*) = recompute_time(L*)     # 低于 L* 回载不划算
```

这段算术放在每章开头，消灭「你至于分层吗」的问题。

## 5. 章节

### Part I 单实例分层（一节点 2–4×L20，TP=2/4）

#### Ch0 负载与容量模型

- **问题**：agent 负载长什么样；这套硬件在没有任何卸载时理论上能撑多少；指标怎么定义。
- **输入**：SemiAnalysis AgentX（真实 agentic coding trace）、Mooncake trace、GLM 博客附录的 OpenHands padded 数据集构造脚本 + EvalScope 多轮配方、neuralmagic/fs-offload-experiments 复现脚本。
- **产出**：负载刻画（轮数、上下文增长、输出长度、轮间间隙、前缀复用结构）；容量算术按 TP2 / TP4 各一版；指标定义（见 `03`）；平台骨架（从 vLLM 复现脚本起步，加 SLO 分析与 `/metrics` 抓取）。
- **硬件层**：无，但模型把 HBM / PCIe / 块设备 / 网络的天花板作为参数编进去，后续各章验证。

#### Ch1 HBM 层：抢占悬崖

- **问题**：只开 prefix caching 时悬崖在哪；实测容量与 Ch0 预测差多少，差在哪。
- **最小配置**：2×L20 TP2（悬崖在十几个会话）；4×L20 TP4 做第二档。
- **硬件层与工具**：HBM 容量与带宽；nsys 时间线、抢占计数器、DCGM 显存、`vllm:num_requests_running`。
- **trade-off 旋钮**：KV dtype（FP8 臂：bytes/token 减半 → 前沿移动；decode 访存受限 → TPOT 变化）；块大小 vs 碎片 vs 命中粒度；`max_num_batched_tokens` / `max_num_seqs` 的 admission。
- **预期发现**：预测与实测的差距来源（块粒度、碎片、CUDA graph 预留、admission）。
- **JD**：量化落地；attention / KV 理解；命中率与有效吞吐。

#### Ch2 主机内存层：PCIe 天花板、分片汇聚、decode 干扰

- **问题**：开 host 层（`TieringOffloadingSpec`，只配主层）后前沿移动多少；池大小 vs 命中率；`blocks_per_chunk` 的粒度 trade-off。
- **最小配置**：2×L20 TP2 与 4×L20 TP4 两档（分片汇聚只在 TP≥2 存在；多卡争用只在 4 卡存在）。
- **硬件层与工具**：先微基准——pinned vs pageable、chunk 尺寸、多流、NUMA 亲和（numactl），测出 PCIe Gen4 x16 实际可达带宽；再从 `/metrics` 拿引擎实际回载带宽；`nvidia-smi topo -m` 看多卡与根复合体 / NUMA 拓扑；DCGM PCIe 计数器。
- **只有多卡才有的问题**：(a) TP2 vs TP4 下 chunk 尺寸与主机侧 IO 模式的变化及其对二级层吞吐的影响；(b) 4 卡同时卸载时主机内存带宽与 PCIe 上行的争用。
- **副作用检查**：聚合模式下回载流量与 decode 争 PCIe / 显存带宽——TPOT p99 随回载带宽的变化。vLLM 团队只测了 PD 模式下的 prefill，这个问题未被回答。
- **trade-off**：池大小 vs 主机内存其他用途；chunk 大 → DMA 少而大但共享粒度粗；LRU vs ARC。
- **预期发现**：引擎回载带宽与 PCIe 天花板的差距来源（每 chunk DMA 尺寸、同步点、线程池、调度器关键路径上的 Python 开销）。
- **JD**：多级缓存、零拷贝、IO 路径、换入换出、生命周期。

#### Ch3 存储层：云盘上的 fs tier 与「重算 vs 回载」交叉点

- **问题**：加 fs 二级层后哪些 regime 有收益、哪些变差；交叉长度 L* 的预测与验证。
- **最小配置**：Ch2 同配置 + 云块设备两种盘型（普通 + 高 IOPS），让 IOPS 受限与带宽受限两个 regime 都出现。
- **硬件层与工具**：fio 先打天花板（O_DIRECT、io_uring、队列深度、块大小）；iostat；perf / py-spy 看每 IO 的 CPU 成本；page cache 行为。
- **差距分析方向**：POSIX IO 走 page cache 的双缓冲与脏页回写抖动；一 chunk 一文件的元数据开销；线程池尺寸；级联到所有层的写放大。
- **组件生长点（仅当数据指向）**：框架支持 `module_path` 加载自定义 `SecondaryTierManager`（`lookup / submit_store / submit_load / get_finished_jobs`，直接拿到主机区域的 memoryview）。候选：O_DIRECT / io_uring 裸块设备层；打包大文件布局的层；针对国内云对象存储的 obj 层适配。GDS 若可用：直通 vs host 中转对照——框架是故意走 host 的，为什么、多付了什么。
- **可选臂**：fs tier 挂 JuiceFS（底下对象存储）vs 直接 obj tier，比较两条通往对象存储的路径。
- **预期发现**：低带宽云盘上 L* 可能高于典型前缀长度（负结果）；chunk 尺寸与队列深度到多少才翻转。
- **JD**：NVMe / 云盘、io_uring、O_DIRECT、GDS、全链路 IO 瓶颈分析、独立分层。

#### Ch4 同节点多实例共享池（Part I 收尾）

- **问题**：两个实例共享 fs 挂载点或本地 MooncakeStore，跨实例命中带来的收益；双写者的去重与驱逐竞争。
- **最小配置**：一节点两个 TP2 实例（4 卡）。不需要多机。
- **硬件层**：共享主机内存带宽与块设备队列。
- **trade-off**：共享带来的命中 vs 竞争带来的抖动；内容寻址 key 的一致性语义；RETRY 路径。
- **JD**：共享复用、KV 池、独立缓存服务的起步形态。

### Part II 多机（两节点 × 4×L20，eRDMA）

#### Ch5 跨节点 KV 池与独立缓存服务 + 可观测性

- **问题**：KV 走到共享远端池后，多实例的跨实例命中收益 vs 延迟代价；新实例从共享池**热启动**的命中率恢复曲线（扩容场景，把「启动延迟」以正当方式接入）。
- **两个臂**：MooncakeStore standalone 模式（CPU-only 节点或对端 DRAM）；vLLM p2p tier（ZMQ + NIXL RDMA）走 eRDMA。
- **硬件层与工具**：perftest（ib_write_bw 等）先打 eRDMA 实测 vs 标称；传输引擎达到的带宽；内存注册成本；元数据查询延迟。
- **工程化**：用框架 per-tier 指标与 KV events 搭 Grafana 面板；定义告警规则（命中率跌落、层延迟 p99、host 池饱和）。JD 第 6 条原文。
- **trade-off**：本地层 vs 共享远端（延迟 vs 共享）；一致性语义（被驱逐条目、RETRY、多写者）。
- **JD**：RDMA、独立分布式缓存服务、监控告警、Mooncake、计算存储分离、跨节点共享。

#### Ch6 PD 分离

- **问题**：vLLM 声称 host 中心的 P2P 层靳合并 IO 优于 GPU 直传的 PD 方案——用 NixlConnector（GPU 直传）对 p2p tier（host 中转），测传输时间随 prefill 长度的曲线、TTFT 分解、与 chunked prefill 的重叠。
- **最小配置**：同节点 2P+2D 起步 → 跨节点；8 卡时做 P/D 比例扫描（AgentX 博客列为三大挑战之一）。
- **硬件层**：L20 无 NVLink，`nvidia-smi topo -m` 看 P2P 路径；host bounce；eRDMA 是否支持 GPUDirect RDMA 待确认——若不支持，「这类云网络上 GPU 直传 PD 不可用、host 中心是唯一路径」本身就是硬件现实层面的发现。
- **边界**：机制研究，不做吞吐声明，除非达到 8 卡且负载真实。
- **JD**：PD 分离、KV 池化、跨节点一致性。

#### Ch7 cache-aware 路由

- **问题**：≥3 实例下，命中率 vs 负载均衡的 trade-off；router 自身成为瓶颈的点（jaga 文章埋在警告框里的发现）。
- **配置**：现成 router（Dynamo frontend 或 SGLang router 的 cache-aware 模式，消费 KV events）。不上 k8s 除非被迫。
- **JD**：全局调度、KV events、跨实例路由。

### Part III 架构维度（平台红利）

#### Ch8 换模型重跑

- MLA 模型与混合架构模型重跑 Ch1–3。bytes/token 与层间异构改变每层经济学；验证框架 canonical layout 与 hybrid allocator 的透明处理并找边界（Mamba 状态 chunk 覆盖、保留策略）。
- **JD**：attention 变体、前缀 / 滑动窗口复用。

#### Ch9 可选：大模型一个点

- 8×L20 或 4×H20 跑 235B-A22B 级模型**一个点**，验证 Part I 核心趋势。不在大模型上扫描。

## 6. 规模阶梯

| 现象 | 最小配置 | 一张卡上存在吗 |
|---|---|---|
| 抢占悬崖、host 层、磁盘层收支 | 2–4 卡 TP，一节点 | 存在但悬崖太早 |
| TP 分片汇聚（合并 IO） | TP≥2 | 不存在 |
| 多卡同时卸载的 PCIe / NUMA 争用 | 一节点 4 卡 | 不存在 |
| 多实例共享池、跨实例命中 | 一节点两实例 | 不存在 |
| PD 传输路径、TTFT 分解 | 同节点 2+2 起步，跨节点 + RDMA 为生产形态 | 不存在 |
| P/D 比例 vs 并发 | 两节点，≥6–8 卡 | 不存在 |
| cache-aware 路由、router 瓶颈 | ≥2–3 实例 | 不存在 |
| 共享池一致性、RETRY、多写者 | 多实例 + 独立存储进程 | 不存在 |

## 7. 阶段映射

- **阶段 1（实习证据）= Part I（Ch0–Ch4）**。正式数字全部来自 2–4 卡。Ch3 收窄到「交叉点 + 差距分析」，组件生长留到有数据之后。
- **阶段 2 = Part II + III**。
- 顺序不变：先单实例，再多实例，再多机。Part I 做完整比 Part II 做一半值钱。

## 8. 与 vLLM 团队已发布工作的交叉

| 他们测了 | 我们补 |
|---|---|
| 2×H100 TP2，HBM 充裕 | 内存吃紧的 PCIe 卡（L20 / H20 / L40S 是国内推理主体） |
| 本地 NVMe fs tier | 云块设备两种盘型（IOPS 受限 regime） |
| PD 模式，只测 prefill 吞吐 | 聚合模式，回载对 decode 的干扰 |
| 吞吐 vs 会话数 | SLO 约束下的容量 |
| 12K + 4K×8 合成多轮 | agentic trace 派生负载 |
| — | TP2 / TP4 分片汇聚对比；多卡主机争用 |
| — | 同节点 / 跨节点多实例共享；热启动 |
| 声称 host 中心 P2P 优于 GPU 直传 | 在 eRDMA 上验证 |

每个发现对框架维护者都有直接价值；issue 与 PR 的目标现成。

## 9. 风险

- 框架快速变化（v0.22 → v0.30）：每章钉 commit；跑坏了是摩擦发现与 PR 机会。
- 云盘让磁盘层看起来差：是发现不是失败；保证两种盘型。
- 蔓延：Ch8 与跨引擎对照严格留在阶段 2；阶段 1 只做 Part I。
- 多机调试吞时间：开发与测量分离；多机窗口集中；镜像化。
