# bench

压测与测量。目标：一条命令起引擎、一条命令跑完一个扫描矩阵并落盘、结果可回溯。

```
bench/
├── workloads/   trace 获取与派生：AgentX / Mooncake trace 的下载与校验、
│                OpenHands padded 数据集构造（沿用 vLLM GLM 博客附录脚本）、会话形状参数化
├── runner/      引擎启动（读 manifest 生成启动命令）、扫描矩阵驱动（并发阶梯 × 臂 × 重复）、
│                预热 / 稳态窗口控制、结果落盘到 data/runs/<run_id>/
├── slo/         从原始结果计算：SLO 约束下的最大并发、goodput、p99 TTFT / TPOT、interactivity、
│                逻辑 token 吞吐（按 GPU 归一化）
└── scrape/      /metrics 定时抓取（起步 15–30 秒），原始样本落盘，差分工具（抢占、命中、传输字节）
```

## 规则

- 压测客户端优先沿用 EvalScope 多轮配方（钉 commit）；不够用再评估 AIPerf / 自写回放器。
- 每个扫描点用新的会话，或在 manifest 里明确说明复用。
- 每次正式运行前先写出本次要出的数据表（哪些点、哪些臂）；不在表里的不跑。
- 指标定义以 `plan/03-methodology.md` 第 3 节为准；metric 名称以钉死的 vLLM 版本为准并记入 manifest。
