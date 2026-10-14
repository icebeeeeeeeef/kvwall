# KV cache 方向 PR / issue 阅读索引

核对日期：2026-09-25。以下日期采用 GitHub API 的 UTC 日期。
主检索窗口：2026-08-01—2026-09-25；少数较早跟踪 issue 因近期关联 PR 而纳入。
这是相关工作样本，不是各仓库全部 PR 的统计。Merged 指合并到目标分支，不代表已包含在正式发行版本中。
性能与测试结果来自作者报告；本次调研没有独立运行这些实验。Open / Draft 不代表无人负责。

使用方法见 [如何从 KV 路径实验找到上游贡献](2026-09-25-kv-upstream-contributions.md)。本表是历史快照，不是认领清单；实际参与前重新核对当前 head、讨论、负责人和合并状态。

## 一、局部性能与数据路径

| 项目 / 编号 | 状态（核对日） | 创建 / 合并日期 | 内容与证据边界 |
|---|---|---|---|
| [SGLang #40960](https://github.com/sgl-project/sglang/pull/40960) | Merged | 09-23 / 09-24 | 在一次 flush 内合并 buffer-only KV 备份提交。保留 GPU 源锁和 host staging 生命周期。单 GPU CPU 准备/入队微基准，不是端到端吞吐提升声明。 |
| [LMCache #5036](https://github.com/LMCache/LMCache/pull/5036) | Merged | 09-09 / 09-12 | 不让一个未对齐/带 padding 的请求导致整批 io_uring 读取退化为串行；真实 Rust IO/O_DIRECT 回归，验证数据和边界保护。 |
| [LMCache #5308](https://github.com/LMCache/LMCache/pull/5308) | Merged（dev） | 09-22 / 09-24 | 对符合布局与运行时条件的 KV 对象使用 cudaMemcpyBatchAsync，避免临时 GPU buffer 与 scatter kernel 排队；不支持的布局保留旧路径。 |
| [LMCache #4413](https://github.com/LMCache/LMCache/pull/4413) | Open | 08-04 / — | MP L1 稳定区域接入 io_uring fixed buffers；lazy 增长和跨注册区域拆分仍为后续范围。正文量化了稳态读取收益与写入退化边界，未证明端到端收益。 |

## 二、模型 / 并行 / 硬件适配

| 项目 / 编号 | 状态（核对日） | 创建 / 合并日期 | 内容与证据边界 |
|---|---|---|---|
| [SGLang #40907](https://github.com/sgl-project/sglang/pull/40907) | Merged | 09-23 / 09-25 | 修复 non-DCP 下 Mamba checkpoint 保存规则造成的 AgentX 高并发前缀复用退化；保留 DCP 原有对齐保护。 |
| [vLLM #56810](https://github.com/vllm-project/vllm/pull/56810) | Merged | 09-14 / 09-20 | SimpleCPUOffload 跳过不允许前缀复用的 scratch group；GLM-5.3-Flash 存储/恢复/KV events 的组合适配。 |
| [vLLM #57160](https://github.com/vllm-project/vllm/pull/57160) | Merged | 09-16 / 09-18 | ROCm CPU KV offload 改走既有 per-rank pinned 分配路径，避免巨型 shared mmap 注册失败；代价是该路径不再获得共享区的复制 KV 去重。 |
| [SGLang #41187](https://github.com/sgl-project/sglang/pull/41187) | Draft / Open | 09-24 / — | MLA DCP + file L3，同拓扑实例间复用；每分片写者、逻辑 token 与局部行映射、key 隔离。已有部分 H200 验证，未跑性能基准，非任意拓扑转换。 |
| [vLLM #58676](https://github.com/vllm-project/vllm/pull/58676) | Open | 09-25 / — | TP2 Mamba2 prefix-cache regression，仅测试代码；冷、热、绕过缓存三组，要求真实命中并比较输出与 logprobs。作者在 2×L20 上验证。 |

## 三、后端与传输接入

| 项目 / 编号 | 状态（核对日） | 创建 / 合并日期 | 内容与证据边界 |
|---|---|---|---|
| [Mooncake #3714](https://github.com/kvcache-ai/Mooncake/pull/3714) | Merged | 08-27 / 09-15 | 将 OSS 接到 ObjectStorageAdapter，区分对象语义与文件 offset 语义。正文记录 mock 测试；其真实 OSS 测试因凭证缺失跳过，不能仅据合并状态断言验证覆盖。 |
| [Mooncake #4231](https://github.com/kvcache-ai/Mooncake/pull/4231) | Open | 09-19 / — | SSD bucket offload 多磁盘化；每盘配额、预留、回收、路径锁与恢复。已有关联 RFC #4230，非无人负责的新方向；未报告真实多盘扩展性能。 |
| [Mooncake #3491](https://github.com/kvcache-ai/Mooncake/pull/3491) | Draft / Open | 08-18 / — | 复用 TENT GDS transport，将 GDS 接入 Store 副本与完成/撤销状态。单节点单 GPU 单 SSD 初步验证；容量满时淘汰等尚未完成。 |
| [LMCache #4661](https://github.com/LMCache/LMCache/pull/4661) | Open | 08-20 / — | RawBlock SPDK IO engine，内存注册、worker、无锁环、批量提交；MP 模式，超时/重入队仍待补，NVMe-oF 性能验证进行中。 |
| [Mooncake #4267](https://github.com/kvcache-ai/Mooncake/pull/4267) | Merged | 09-21 / 09-22 | classic RDMA transport 的可选 NVIDIA Data Direct 注册；动态符号、旧路径兼容和跨节点 GPU payload 验证。未改变 TENT/IBGDA，不等同于 GDS。 |

## 四、值得学习其调查过程的 issue

### SGLang #40232：特定设备属性下的 HiCache 批量 D2H

- [Issue](https://github.com/sgl-project/sglang/issues/40232)，09-18 创建，Open。
- 原报告 RTX 3060 Laptop 6GB + WSL2；需先测 CanUseHostPointerForRegisteredMem，而非假定任意 WSL/RTX4060 相同。
- 已有 [#40233](https://github.com/sgl-project/sglang/pull/40233) 和 [#40571](https://github.com/sgl-project/sglang/pull/40571)，两者在核对时均 Open。
- [09-21 的硬件交叉验证和方案对照](https://github.com/sgl-project/sglang/issues/40232#issuecomment-5759345117)：fallback 与 alias 两种方向均在报告机器验证通过；取舍仍需维护者判断。
- 可考虑的协作：新设备/驱动上的独立验证、批量路径确认、边界测试。不是再写第三份重复修复。

### SGLang #39444：远端有数据，但恢复为何只命中一部分

- [Issue](https://github.com/sgl-project/sglang/issues/39444)，09-14 创建，Open。
- 不要只读标题。原报告怀疑 write-through 未充分持久化。
- [09-16 后续定位](https://github.com/sgl-project/sglang/issues/39444#issuecomment-5691678696)：失败批次是一个问题，补重试后症状仍在；远端存在全部 16010 token，L2 分配只支持 2551 token，恢复因此被截短。
- 批次重试已有 [#39701](https://github.com/sgl-project/sglang/pull/39701)；作者表示会另修恢复路径。先协调并复测当前版本。
- 可学习/验证的内容：逐层命中证据、分配失败、驱逐、写策略与恢复策略的交互。

### vLLM #58653：配置显示 offload 开启，实际上只写不读

- [Issue](https://github.com/vllm-project/vllm/issues/58653)，09-25 创建，Open。
- 混合模型显式 block-size 与自动对齐路径的差异；报告仅验证指定 dev build，稳定版行为未确认。
- [已有参与者表示正在处理](https://github.com/vllm-project/vllm/issues/58653#issuecomment-5826196686)。适合补复现/配置矩阵，不应视为待认领空位。

### LMCache #4100：MP allocator 与 fixed-buffer 注册

- [Issue](https://github.com/LMCache/LMCache/issues/4100)，07-13 创建，Open；因关联近期 #4413 纳入。
- 原作者已有 PR #4413，先读已实现和明确延期的边界。
- 可商议的后续：lazy 扩容注册顺序、跨区域请求分拆、多个物理完成合成逻辑完成、失败与 shutdown 并发。

### SGLang #38866：混合模型 + PP 的恢复完成语义

- [Issue](https://github.com/sgl-project/sglang/issues/38866)，09-10 创建，Open，已有 assignee。
- 非首个 pipeline stage 的层编号与恢复完成编号不一致，可能提前读取。
- 这是既有 PP1 修复 #37870 之外的后续范围。适合向负责人提出窄测试或协作验证，不宜直接重复主实现。

## 五、选择一个参与点前

1. 先确认它确实经过你的 KV 路径，区分权重 offload 与 KV offload。
2. 固定 base/head、依赖、模型与运行参数，判断问题是否仍存在。
3. 阅读 issue 的最新讨论和关联 PR；Open 不表示没人做。
4. 先提出能够独立完成的验证范围，与作者/维护者协调，不另造重复实现。
5. 同时保存路径证据、数据正确性、资源回收和性能口径；不要把 API 成功或模型输出正常当作真正命中。
6. GPU 大模型上发现的问题可以尝试缩成独立 CUDA/IO/状态机测试，但是否保留触发机制要验证。
7. io_uring / SPDK 写测试应使用专门测试文件或无重要数据的专用设备，不对系统盘或共享数据盘做破坏性测试。
