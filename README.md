# kvwall

**KV 容量墙：从 HBM 到远端池，SLO 约束下的并发前沿与每一层的硬件天花板。**

以「LLM 推理的内存层级与数据搬运」为主线的个人工程实践项目。目标不是做一个新的 KV cache 引擎，而是：

- 一个长期维护的**实验平台**（负载回放、测量口径、微基准、容量模型、可观测性）
- 一系列在平台上完成的**闭环**（每层缓存一章，每章对着硬件天花板算账）
- 从闭环数据里**长出来的组件**（以现有引擎插件 / 后端的形式 upstream）

脊柱问题：给定 SLO，一组内存吃紧的 PCIe 推理卡能撑多少个长上下文 agent 会话；每加一层缓存，这个数字移动多少、代价是什么、离硬件天花板差多远、到哪里失效。

主引擎 vLLM（tiered KV offloading 框架），远端池 MooncakeStore，阶段 2 以 SGLang HiCache 为对照臂。详见 `plan/`。

## 目录树

```
kvwall/
├── plan/            总体方案规划（策略、主线、方法论、基础设施、JD 映射、参考、决策日志、开放问题）
├── notes/           过程记录
│   ├── reading/       阅读笔记（每篇 ★★★ 参考一页）
│   ├── discussions/   讨论记录
│   └── retro/         阶段复盘
├── src/kvwall/      可复用代码（Python 包）
│   ├── tiers/         自定义 SecondaryTierManager（仅当数据指向时才出现）
│   └── common/        运行清单读写、metrics 解析等公共工具
├── bench/           压测与测量
│   ├── workloads/     trace 获取、派生分布、数据集构造
│   ├── runner/        起引擎 / 跑扫描矩阵的驱动
│   ├── slo/           SLO 约束下的容量计算、goodput
│   └── scrape/        /metrics 定时抓取
├── micro/           微基准：先打硬件天花板
│   ├── pcie/          H2D / D2H、pinned vs pageable、chunk 尺寸、多流、NUMA
│   ├── blockdev/      fio 配方：O_DIRECT / io_uring / QD / bs
│   └── rdma/          perftest 配方与解析
├── analysis/        数据处理与绘图
│   ├── capacity/      容量模型（算术 → 预测；后与测量对比）
│   ├── plots/         固定样式的前沿曲线、机制指标、副作用图
│   └── notebooks/     探索性分析（结论落回 plots/ 与 writeups/）
├── dashboards/      可观测性
│   ├── grafana/       面板 JSON
│   └── alerts/        Prometheus 告警规则
├── data/            实验数据（大文件不入库）
│   ├── runs/          data/runs/<run_id>/ ：manifest + summary 入库，raw 不入库
│   └── traces/        外部 trace 的来源、校验和、派生方式（trace 本体不入库）
└── writeups/        每章文章草稿
    └── templates/     章节模板
```

## 一个闭环在仓库里的流动

```
plan/02-mainline.md（章的问题与最小配置）
  → plan/08-open-questions.md（挑出该章待答问题）
  → analysis/capacity/（算术：预测悬崖 / 交叉点）
  → micro/（打该硬件层的可达天花板）
  → bench/workloads/ + bench/runner/（构造负载、跑扫描矩阵）
  → data/runs/<run_id>/（原始结果 + manifest）
  → bench/slo/ + analysis/plots/（SLO 容量、前沿曲线、机制指标、副作用）
  → 差距分析（天花板 − 实际，逐项归因，证据来自 nsys / perf / py-spy / iostat）
  → writeups/chNN-*.md（按模板写）
  → plan/07-decision-log.md（若指向功能 / 组件 / PR，记「因为测到了 X」）
  → src/kvwall/tiers/ 或 upstream PR（仅当数据指向）
```

## 从哪里开始读

1. `plan/README.md` — 规划总览与阅读顺序
2. `plan/02-mainline.md` — 技术主线（章节、规模阶梯、模型选择）
3. `plan/03-methodology.md` — 闭环怎么做、怎么算完、怎么写
4. `plan/07-decision-log.md` — 已做的决定和理由

## 约定

- 规划类文档只写方向、边界、规则、约束；实施细节进 `writeups/` 与 `notes/`。
- 改变方向的决定先写进 `plan/07-decision-log.md`；新增功能必须有「因为测到了 X」。
- 每次正式测量必须有 `data/runs/<run_id>/manifest.yaml`，否则数据不采用；writeup 里每个数字可回溯到一个 `run_id`。
- 估算数字标「粗算」；正式数字只来自带 manifest 的测量。
- 开发环境（1 卡 + 7B 级模型）不出正式数字。
- 文风对齐 vLLM 博客：平、准、每个数字带定义。

## 状态

- 2026-09-18：规划初始化（`plan/`），目录骨架建立。阶段 0 尚未开始。
