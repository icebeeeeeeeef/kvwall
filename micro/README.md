# micro

微基准。用途只有一个：**先打出该硬件层的可达天花板**，供差距分析用。每个微基准输出一个带硬件信息的数字表，存入对应章节的 `data/runs/<run_id>/`。

```
micro/
├── pcie/       H2D / D2H 带宽：pinned vs pageable × chunk 尺寸 × 流数 × NUMA 绑定；
│               多卡同时拷贝时的争用；对照 nvidia-smi topo -m 与 DCGM PCIe 计数器
├── blockdev/   fio 配方：O_DIRECT / io_uring × 队列深度 × 块大小 × 读写混合；
│               区分 IOPS 受限与带宽受限；记录盘型、内核版本、调度器
└── rdma/       perftest（ib_write_bw / ib_read_lat 等）配方与解析：消息大小 × QP 数；
│               eRDMA 实测 vs 标称；GPUDirect RDMA 可用性探测
```

## 规则

- 天花板指实测可达，不是标称。
- 每个微基准跑 3 次以上，报告中位数与离散度。
- 记录测量时的硬件状态（时钟、链路宽度 / 代数、NUMA、盘队列参数、MTU）。
- 微基准结论进入对应章节 writeup 的「差距分析」一节。

## 自研微基准（D-016）

除通用工具（fio / perftest / 通用 memcpy 基准）外，允许按引擎的实际访问模式自研微基准，打出「该访问模式下的天花板」，把差距分析拆成两段：硬件天花板 → 访问模式天花板 → 引擎实际。

| 目录 | 语言 / 接口 | 按什么构造 | 对应章节 |
|---|---|---|---|
| `pcie/` | CUDA C++ | vLLM offloading 的实际 `blocks_per_chunk`、流数、多卡并发 | Ch2 |
| `blockdev/` | C / C++ + liburing + O_DIRECT | fs tier 的 chunk 文件布局与 IO 尺寸 | Ch3 |
| `rdma/` | C / C++ + libibverbs | KV chunk 的消息大小；MR 注册成本 | 阶段 1 副线（Soft-RoCE 开发）→ Ch5（eRDMA 实测） |

规则：

- 自研基准必须先与通用工具交叉验证；同条件下偏差须解释，解释不了不采用。
- 访问模式参数来源（vLLM commit、配置项）写入 manifest。
- Soft-RoCE 与开发机上的结果只用于验证正确性，不出正式数字。
