# 04 基础设施与预算

## 1. 三个环境

| 环境 | 配置 | 用途 | 规则 |
|---|---|---|---|
| 开发 | 1×L20（或本地一张卡）+ 7B 级模型 + 云盘 | 跑通所有脚本、调试、平台开发 | 不出任何正式数字 |
| 单节点测量 | 一节点 2–4×L20 TP2 / TP4，两种盘型 | Part I 正式扫描 | 只用来出数据；无人值守跑完整矩阵 |
| 多机 | 两节点 × 4×L20，eRDMA；CPU-only 节点可选 | Part II | 压成几个连续整天窗口；开机即测 |

本地若购一张卡用于开发，不必是 L20。

## 2. 云资源调研（截至规划时）

| 要求 | 当前结论 | 对主线的影响 |
|---|---|---|
| 本地物理 NVMe | 普通 GPU 实例方案不满足；高性能网络机型有能力但未落实可购配置 | Ch3 不依赖它；可购时用同一平台重跑 |
| 裸块设备、O_DIRECT、io_uring | 云盘块设备路径可做；不等于物理 NVMe 透传；内核、操作码、对齐需验机 | Ch3 起步路径；标注「云盘」 |
| 跨节点高速网络 | eRDMA 有明确支持路径；标称带宽 ≠ 实测 RDMA 吞吐 | Ch5 / Ch6；先 perftest；GPUDirect RDMA 支持待确认 |
| GDS 直接路径 | 待确认；不能作为下单后的必有能力 | 当 bonus；没有本地 NVMe 意义也不大 |
| 显存、卡型 | L20 48GB 单卡 / 多卡规格明确 | PCIe Gen4、无 NVLink：offload 天花板是 PCIe，P2P 走 PCIe 或 host |
| 同机双引擎 | 可部署；单卡双进程验功能，双卡做性能对照 | Ch4 / Ch6 起步 |

## 3. 软件栈与版本钉死

| 组件 | 作用 | 钉死方式 | 状态 |
|---|---|---|---|
| vLLM | 主引擎；tiered KV offloading（`TieringOffloadingSpec`，fs / obj / p2p tier，`SecondaryTierManager` 接口，per-tier metrics，KV events） | commit hash，每章一钉 | 待钉 |
| MooncakeStore / Transfer Engine | 远端池；standalone 模式 | 版本 / commit | 待钉 |
| NIXL | obj / p2p tier 的传输；NixlConnector（GPU 直传 PD） | 版本 | 待钉 |
| SGLang + HiCache | 阶段 2 对照臂 | commit | 阶段 2 |
| 路由 | Dynamo frontend 或 SGLang router（cache-aware） | 版本 | 阶段 2 |
| EvalScope（`perf`，多轮） | 压测客户端；沿用 vLLM 博客配方 | commit | 待钉 |
| AIPerf / vllm bench | 备选压测客户端 | 版本 | 可选 |
| Prometheus + Grafana | 指标抓取与面板 | docker 镜像 tag | 待定 |
| nsys、DCGM、perf、py-spy、fio、iostat、numactl、perftest | 硬件层与差距分析工具 | 记录版本 | — |
| Python 环境 | 平台脚本 | lock 文件 | 待建 |

## 4. 数据与工具来源

- **SemiAnalysis AgentX**：真实 agentic coding trace 的公开 benchmark（中位 43 轮、142K 输入、444 输出、前缀命中率 >96%、44% 会话含子 agent）。负载刻画与回放的首选。格式与许可待确认。
- **Mooncake trace**：公开请求 trace。
- **OpenHands padded 数据集构造脚本 + EvalScope 多轮配方**：vLLM GLM 5.3 博客附录，自包含、可钉 commit。
- **neuralmagic/fs-offload-experiments**：vLLM tiered offloading 博客的复现脚本。平台骨架的起点。

## 5. 预算形态规则

形态比总额重要：

1. **开发时间与测量时间严格分开**。开发在 1 卡；测量在 4 卡或两节点无人值守。租来的多卡时间不调试。
2. **多机窗口集中**。环境镜像化、脚本化；开机即测；跑完即释放。
3. **粗估量级**：Part I 正式扫描在 4 卡上是低三位数 GPU 小时；Part II 两节点集中窗口是同一量级。每章 6–8 个扫描点 × 2–4 个臂 × 3 次重复 × 5–10 分钟稳态。
4. **大模型只跑一个点**。
5. 每次租用前写清本次要出的数据表格（哪些点、哪些臂）；租用后按表格跑；没在表格里的不跑。

## 6. 自动化要求

平台骨架在阶段 0 必须做到：

- 一条命令起引擎（读运行清单里的配置）
- 一条命令跑完一个扫描矩阵并落盘（原始结果 + `/metrics` 样本 + 运行清单）
- 一条命令出图（前沿曲线、机制指标、副作用）
- 微基准可独立运行并输出天花板数字
- 环境镜像 / 快照可复用；依赖 lock

## 7. 数据管理

- `data/<run_id>/` 存原始结果、metrics 样本、运行清单、启动命令。
- 大文件（trace、模型）不入库，记录来源与校验和。
- 每章 writeup 引用的每个数字都能回溯到一个 `run_id`。
