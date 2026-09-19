# 03 方法论：闭环怎么做、怎么算完、怎么写

## 1. 闭环的八步骨架

每一章都按这八步走。来源：vLLM GLM 5.3 Part 1、vLLM x AgentX、jaga 的 16×H100 HiCache 文章的共同结构。

1. **从 regime 出发，不从技术出发**。负载形状（数字）+ 硬件约束 → 资源冲突 → 现有选项及其 trade-off → 技术才登场。
2. **用构造让约束绑定，把算术放在开头**。工作集故意超过某一层容量，算清超多少。
3. **baseline 是真实的替代方案**。同资源不同策略（如同样主机内存预算下的两种分配），不是「什么都不开」。
4. **扫一个控制变量，报告曲线**。并发阶梯、池大小阶梯；发现藏在拐点里，一个点只是 demo。
5. **测机制，不只测结果**。抢占次数、分层命中率、回载字节、池占用平台期——把相关变成解释。
6. **检查有没有弄坏别的东西**。TPOT / ITL、精度、主机内存、CPU 占用。
7. **钉死并附复现**。两臂各自的 commit、压测工具 commit、数据集构造脚本、完整启动参数、指标定义。
8. **声明边界**。在什么模型 / 负载 / 硬件下成立，超出后预计如何变化。

## 2. 闭环「完成」的判定

四条全满足即停，不追求「再优化一轮」：

1. 有 baseline
2. 瓶颈有证据（profiler 数据，不是推断）
3. 至少一个干预有测量结果——**负结果 + 解释同样算完成**
4. 有明确的有效边界声明

## 3. 指标定义

| 指标 | 定义 |
|---|---|
| **SLO 约束下的最大并发（主指标）** | 满足 p99 TTFT ≤ X 且 p99 TPOT ≤ Y 的最大并发会话数。X、Y 从 trace 的真实性反推，见 `08` |
| goodput | SLO 内完成的请求 / 单位时间 |
| interactivity | 1000 / mean TPOT（沿用 vLLM 定义） |
| 逻辑 token 吞吐 | 含 prefix-cached 的 prompt token，按 GPU 数归一化（沿用 vLLM 定义） |
| 分层命中率 | 每层的 hit / query；与构造上限对比并解释差距 |
| 回载 / 卸载字节与带宽 | 从 `/metrics` 差分；与微基准天花板对比 |
| 抢占次数 | 引擎计数器 |
| 并发占用 | `num_requests_running` 非零样本均值（沿用 vLLM 做法） |
| 交叉长度 L* | reload_time(L) = recompute_time(L) 的解 |

具体 metric 名称以钉死的 vLLM 版本为准，记录在运行清单里。

## 4. 测量卫生

- 预热：每个点固定预热请求数；首轮会话排除。
- 稳态窗口：每点固定时长（起步 5–10 分钟），只统计窗口内数据。
- 重复：每点至少 3 次，报告均值与方差（或中位数与 p5/p95）。
- 尾延迟：始终报告 p99，不只报均值。
- 变量隔离：一次只变一个旋钮；同一章内其余配置完全相同。
- 每点用新的会话（避免跨点 cache 污染），或明确说明复用。
- `/metrics` 固定间隔抓取（如 15–30 秒）并存原始样本。
- 硬件状态记录：GPU 时钟、温度、PCIe 链路宽度 / 代数、NUMA 绑定、盘型与队列参数、网络 MTU / RDMA 设备。
- 数字离天花板多远必须解释；解释不了写进开放问题。

## 5. 差距分析的固定套路

```
1) 微基准打出该硬件层的可达天花板（不是标称）
   PCIe: cudaMemcpy pinned/pageable × chunk 尺寸 × 流数 × NUMA
   块设备: fio × O_DIRECT/io_uring × QD × bs
   RDMA: perftest × 消息大小 × QP 数
2) 从引擎 /metrics 取实际达到的数字
3) 差距 = 天花板 − 实际；逐项归因（DMA 尺寸、同步点、线程池、Python 开销、page cache、元数据）
4) 归因需要证据（nsys / perf / py-spy / iostat），不是猜
5) 若归因指向一个可改的点 → 才考虑 PR 或组件
```

## 6. 文章模板与文风

固定章节：

1. TL;DR（三句：regime、干预、结果与边界）
2. regime 与算术
3. 设置（钉死的版本、硬件、负载、SLO）
4. baseline 与对照臂
5. 扫描结果（曲线 + 拐点）
6. 机制指标
7. 副作用
8. 差距分析
9. 边界声明与开放问题
10. 复现附录

文风：平、准、每个数字带定义。不出现 massive / rock-solid 之类形容词；不解释参数含义（链接文档）；不写「什么是 KV cache」。

## 7. 反模式

来自 jaga 文章的短板：

- 合成固定长度负载却不声明它不是真实分布
- 报告饱和点吞吐（p99 TTFT 50 秒的 regime 没人会跑）
- 不解释命中率与构造上限的差距
- 单次运行无方差
- 最有价值的摩擦发现（router 先崩）埋在警告框里
- 一半篇幅在配置 YAML

来自内容营销类文章的体裁：引用别人的数字、商业吓人数字、类比代替测量。不产生工程可信度。

## 8. 运行清单（run manifest）字段

每次正式测量必须附带，否则数据不采用：

```
run_id, date
hardware: node type, GPU model/count, PCIe gen/width, NUMA topo, disk type/size/QD, NIC/RDMA
software: vllm commit, connector/tier config, mooncake/nixl versions, driver/CUDA, OS/kernel
model: name, revision, dtype, kv dtype, TP/DP/EP
workload: trace source + revision, dataset builder args, SLO (X, Y)
sweep: variable, points, warmup, steady window, repeats
launch command (verbatim)
metrics scrape interval; raw metrics path
notes: anomalies, restarts, anything unusual
```

## 9. 决策日志规则

- 每个新增功能、每个配置选择、每次方向调整，一条记录。
- 格式见 `07-decision-log.md`。
- 「因为测到了 X」是必填项；没有 X 的功能不加。
