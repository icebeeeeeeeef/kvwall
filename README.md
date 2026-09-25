# kvwall

**KV 容量墙：从 GPU 显存到远端池，SLO 约束下的并发前沿与数据搬运。**

以 LLM 推理的内存层级与数据搬运为主线。不是从零实现推理引擎，也不是把所有推理优化技术堆在一个平台中，而是：

- 用可复现的多轮长上下文负载、容量模型与测量工具解释资源边界；
- 在真实引擎中逐步研究 KV 保留、卸载、恢复、复用与调度；
- 从需求/证据中接管一段真实执行路径，通过修改、适配和回归形成工程成果。

主引擎 vLLM，远端池优先 MooncakeStore；SGLang HiCache 等按对照或上游需求引入，不作为第一步前置。

## 第一版最小项目

```text
固定小模型 + 多轮 agent-like workload
                ↓
stock vLLM：GPU-resident KV + prefix caching
                ↓
逐请求结果/原始 metrics + 容量估算 + 重复扫描与 SLO 边界
```

先在本地小配置搭这一版，无外部 offload、无 PD、无远端服务要求。自己先维护负载适配、runner、统计与数据；后续优化都与明确的原生基线比较。

学习和搭建交错推进，不先纯学习三周；最终工程责任不止 benchmark，但也不预先强制自研某种后端或全部算法。

## 目录

```text
plan/                 方向、主线、方法、环境、JD、决策与学习路线
notes/reading/        上游/资料阅读与来源
notes/discussions/    学习目标、讨论与执行备忘
notes/retro/          阶段复盘
src/kvwall/tiers/     有实际需求的 tier/适配实现
src/kvwall/common/    manifest、指标解析等公共工具
bench/workloads/     负载定义、派生与回放适配
bench/runner/        实验驱动
bench/slo/           SLO 与 goodput 统计
bench/scrape/        原始指标采集
micro/pcie/          GPU/主机搬运微基准
micro/blockdev/      存储访问模式与 IO 微基准
micro/rdma/          协议/设备传输验证
analysis/capacity/   容量模型
analysis/plots/      固定绘图
analysis/notebooks/  探索分析
 dashboards/         后续 Grafana/告警（首版不要求）
data/runs/<run_id>/  manifest、汇总与原始结果位置
data/traces/         外部数据来源/许可/校验和
writeups/            正式闭环报告与复现说明
```

## 一个闭环

问题/适配需求 → 最小验证 → 按需学习 → 容量与路径证据 → 干预/负结果 → 正确性与副作用回归 → writeup/上游贡献。

先确认操作实际发生，再分析性能；卡数、模型大小和 PR 数量不能替代证据。

## 从哪里开始读

1. [学习路线](plan/09-learning-roadmap.md)：前置出口与第一版项目。
2. [技术主线](plan/02-mainline.md)：Ch0/Ch1 起步，后续按问题扩展。
3. [测量方法](plan/03-methodology.md)：路径、正确性、SLO 与复现。
4. [环境和预算](plan/04-infra-and-budget.md)：本地验证与按需租用。
5. [决策日志](plan/07-decision-log.md)：历史门槛和最新修订。

更多过程记录在 [notes](notes/README.md)，规划导航在 [plan](plan/README.md)。

## 约定与状态

估算注明假设；实际数字来自带 manifest 的运行，统一使用 `data/runs/<run_id>/`。允许小模型/单卡/WSL 的范围内证据，不外推未运行的设备、配置或生产容量。大数据与凭证不入库。

- 2026-09-18：规划与目录骨架初始化。
- 2026-09-23：增加学习路线与 D-016～D-019。
- 2026-09-25：调整为最小前置与实验并行，区分 baseline、工程责任和代表性适配，更新 D-020～D-024。

**当前仍是规划/文档阶段：上述 runner、最小服务与性能结果尚未在本仓库实现或验证。下一步是完成前置小产出并搭首个闭环，不是等待全部章节知识完成。**
