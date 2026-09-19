# analysis

数据处理与绘图。输入是 `data/runs/`，输出是图和表，落到 `writeups/` 引用。

```
analysis/
├── capacity/    容量模型：bytes/token、驻留 token、会话数、host 池容量、交叉长度 L*。
│                先出预测，测量后对比并解释差距（这是每章的第一张表）
├── plots/       固定样式的图：前沿曲线（SLO 容量 / interactivity–throughput）、
│                机制指标（命中率、抢占、回载字节、池占用）、副作用（TPOT p99）、天花板 vs 实际
└── notebooks/   探索性分析。结论必须落回 plots/ 的脚本或 writeup，notebook 本身不作为证据
```

## 规则

- 每张图有：标题、坐标轴定义、run_id 列表、错误棒或分布。
- 图的样式统一（同一脚本出图），跨章节可比。
- 容量模型的参数（HBM、util、权重大小、运行时预留、bytes/token）来自 manifest 或模型 config，不手填。
