# 05 JD 映射

## 1. 目标岗位 JD（去重归纳版，收集于规划前）

### 核心职责

1. 设计实现 KV Cache 多级缓存系统（GPU 显存 → 本地内存 → 本地 NVMe SSD → 分布式远端存储），推高缓存命中率和有效吞吐
2. 分析全链路 IO 瓶颈，基于 RDMA / CXL / GPU Direct Storage 等高速互联技术设计极致 IO 路径
3. 实现 KV Cache 生命周期管理：换入换出、跨节点迁移、共享复用、前缀 / 滑动窗口复用
4. PD（Prefill-Decode）分离架构下的 KV Cache 池化、全局调度与跨节点一致性保障
5. KV Cache 量化 / 压缩算法的设计与落地
6. 将 KV Cache 封装为独立分布式缓存服务，与推理引擎解耦集成，建立监控告警体系
7. （部分岗位）推理框架本身的性能优化：算子融合、投机解码、并行策略、kernel 开发

### 核心技术要求

- **存储与高速 IO**：RDMA、CXL、GPU Direct Storage (GDS)、io_uring、NVMe / NVMe-oF、SPDK、零拷贝、异构存储统一接入
- **分布式系统基本功**：分布式一致性协议（如 Raft）、分布式缓存系统经验（Redis / Alluxio / JuiceFS / Ceph）、计算存储分离架构
- **KV Cache / 推理专项**：主流推理引擎（vLLM、SGLang、TensorRT-LLM）至少一个有深入理解甚至源码级经验；了解 Mooncake、3FS、FlexKV、LMCache 等相关系统的方案或实践经验；理解 attention 机制、prefill / decode、投机解码、FlashAttention、量化等推理优化原理
- **工程基本功**：C / C++ / Go / Rust 至少一门，扎实的数据结构算法功底；Linux 系统开发、调试、性能分析（profiling）；操作系统 / 网络 / 并发编程基础

## 2. JD 的三类拆分

| 类别 | 条目 | 现状 | 伪造难度 | 证据重心 |
|---|---|---|---|---|
| 存储 / 分布式服务 | 多级缓存、生命周期、独立服务 + 监控、Raft / Redis / Alluxio | 有基础，能讲 | 低 | 迁移资本，过筛选 |
| 推理引擎内部 | 引擎源码级、Mooncake / LMCache / FlexKV、attention / PD / 投机解码 | 空白 | 高 | **重心** |
| 高速 IO | RDMA / GDS / io_uring / NVMe / 零拷贝 | 空白 | 高（需硬件） | **重心**，诚实标注实测范围 |

## 3. 章节 → JD 映射

| JD 条目 / 关键词 | 章节 |
|---|---|
| 多级缓存 GPU → DRAM → SSD → 远端 | Ch1 → Ch2 → Ch3 → Ch5，每层一章 |
| RDMA | Ch5、Ch6（eRDMA 实测） |
| GDS | Ch3（若可用：直通 vs host 中转对照；否则只讨论框架为何走 host） |
| io_uring / O_DIRECT / NVMe / 云盘 | Ch3 |
| 零拷贝、IO 路径、全链路瓶颈分析 | Ch2（PCIe、pinned）、Ch3、Ch5；各章差距分析 |
| 换入换出、生命周期 | Ch2 |
| 跨节点迁移、共享复用 | Ch4、Ch5 |
| 前缀 / 滑动窗口复用 | Ch1、Ch8 |
| PD 分离下的池化、一致性 | Ch6、Ch5 |
| 全局调度 | Ch7（KV events → router） |
| 量化 / 压缩落地 | Ch1 的 FP8 KV 臂 |
| 独立分布式缓存服务、与引擎解耦、监控告警 | Ch4（起步）、Ch5（MooncakeStore standalone + Grafana + 告警规则） |
| 推理框架性能优化 | 各章 nsys 差距分析；投机解码与卸载交互为可选臂 |
| vLLM 源码级 | 主引擎，阶段 0 起 |
| SGLang | 阶段 2 对照臂 |
| Mooncake | Ch4 / Ch5 |
| LMCache / FlexKV / 3FS | 方案层面了解；LMCache 可作对照臂 |
| attention / prefill-decode / 投机解码 / FlashAttention | Ch0 算术、Ch1 机制解释、Ch6 TTFT 分解 |
| 计算存储分离 | Ch5 |
| C / C++ | Mooncake Transfer Engine；自定义 tier 若需要性能 |
| Linux profiling / 并发 | 各章 nsys / perf / py-spy / fio / perftest |
| JuiceFS | Ch3 / Ch5 之间的可选臂（fs tier 挂 JuiceFS vs obj tier） |

## 4. 诚实地不覆盖

- **Raft**：不硬凑。面试中作为已知概念讨论。
- **Redis / Alluxio / Ceph 经验**：无；用对象存储实习与本项目的 tier 工作替代叙事。
- **CXL**：个人碰不到。
- **SPDK**：没有本地 NVMe 没意义。
- **TensorRT-LLM**：不覆盖。

## 5. 面试中怎么用这张图

- 每个 JD 条目对应一章，一章对应一篇有图的文章和一组 `run_id`，回答时给数字与边界。
- RDMA / GDS 说清实测范围与未实测部分的准备程度。
- 反问：KV cache 组与调度器的边界；驱逐策略谁决定；PD 下的 KV 传输走什么路径；有没有 cache-aware 路由。
