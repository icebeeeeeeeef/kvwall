# dashboards

可观测性。对应 JD「建立监控告警体系」与主线 Ch5。

```
dashboards/
├── grafana/   面板 JSON：按 tier 的命中率、传输带宽、层延迟 p99、host 池占用、抢占、
│              num_requests_running、TTFT / TPOT 分布；PD 与远端池的传输面板
└── alerts/    Prometheus 告警规则：命中率跌落、层延迟 p99 超阈、host 池饱和、
│              router 排队、传输错误
```

## 规则

- 面板的数据源是 vLLM `/metrics`（含 tiered offloading 的 per-tier 指标）与 KV events 的导出；Mooncake / NIXL 的指标按可得情况接入。
- 告警阈值必须能指回某一章的测量（例如「命中率低于构造上限的 X% 触发」）。
- Prometheus + Grafana 的本地部署方式待定（见 `plan/08-open-questions.md`）；确定后在此放 compose 文件。
