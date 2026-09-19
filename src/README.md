# src

可复用代码，Python 包 `kvwall`。只放会被 `bench/`、`micro/`、`analysis/` 多处调用的东西；一次性脚本留在各自目录。

```
src/kvwall/
├── __init__.py
├── common/     运行清单（manifest）读写与校验、/metrics 文本解析、run_id 生成、硬件信息采集
└── tiers/      自定义 vLLM SecondaryTierManager 实现（out-of-tree，经 module_path 加载）
```

## 规则

- `tiers/` 在数据指向之前保持为空。任何 tier 实现的出现必须对应 `plan/07-decision-log.md` 里一条「因为测到了 X」。
- 与 vLLM 接口耦合的代码必须记录所针对的 vLLM commit。
- 依赖管理方式待定（见 `plan/08-open-questions.md`）；确定后在仓库根添加 `pyproject.toml` 与 lock 文件。
