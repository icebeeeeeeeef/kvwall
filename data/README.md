# data

实验数据。**大文件不入库**：入库的只有运行清单、汇总结果、启动命令；原始结果、metrics 样本、日志、trace 本体、模型权重一律不入库（见根目录 `.gitignore` 与本目录 `.gitignore`）。

```
data/
├── runs/<run_id>/
│   ├── manifest.yaml     运行清单（必需；模板见 manifest.template.yaml）
│   ├── launch.sh         逐字的启动命令
│   ├── summary.csv       汇总结果（每个扫描点一行）
│   ├── raw/              原始压测输出（不入库）
│   ├── metrics/          /metrics 抓取样本（不入库）
│   └── logs/             引擎 / 客户端日志（不入库）
└── traces/
    ├── README.md         每个外部 trace 的来源、版本、许可、校验和、派生方式
    └── *.sha256          校验和
```

## run_id

`YYYYMMDD-chNN-<arm>-<seq>`，例如 `20261101-ch01-fp8kv-02`。

## 规则

- 没有 `manifest.yaml` 的运行不算正式数据。
- writeup 里每个数字可回溯到一个 `run_id`。
- 原始数据在本地或对象存储保留副本，`manifest.yaml` 里记录其位置与校验和。
