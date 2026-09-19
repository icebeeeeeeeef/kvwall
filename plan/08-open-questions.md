# 08 开放问题

没有答案、需要在做的时候回答的问题。每章开始前从这里挑出相关项；回答后记回 `07-decision-log.md` 或该章 writeup，并在此标注「已回答 → 位置」。

## 模型与负载（Ch0 前）

- [ ] 正式模型的具体名称与 revision：30B-A3B 级同族中哪一个；FP8 权重是否官方提供；在 2×L20 上能否稳定运行；实际 config 的层数 / KV 头数 / head_dim（更新 `02` 的算术）。
- [ ] MoE 的并行配置：TP2 / TP4 下是否启用 EP 或 DP-attention；固定哪一种并在 Ch1 声明。
- [ ] 开发模型的具体选择（7B 级 dense）。
- [ ] AgentX 数据的格式、获取方式、许可；能否直接回放或需派生分布。
- [ ] Mooncake trace 的格式与覆盖的负载类型。
- [ ] OpenHands padded 数据集构造脚本在非 GLM 模型 tokenizer 上的适配。
- [ ] SLO 取值：p99 TTFT ≤ X、p99 TPOT ≤ Y 的 X、Y 怎么从 trace 的真实性反推；是否需要两组 SLO（交互式 / 批处理式）。
- [ ] 会话形状的正式参数：轮数、首轮长度、后续轮长度、输出长度、轮间间隙分布。

## 软件栈（阶段 0）

- [ ] vLLM 钉哪个版本 / commit；`TieringOffloadingSpec` 在该版本的可用性与配置项；fs / obj / p2p tier 各自的成熟度。
- [ ] `SecondaryTierManager` out-of-tree 加载（`module_path`）在该版本是否可用；参考实现路径确认。
- [ ] MooncakeStore 在 vLLM 中的集成状态与 standalone 模式的配置方式。
- [ ] NIXL 在 eRDMA 上的后端支持（UCX？）；NixlConnector 在无 GPUDirect RDMA 时的行为。
- [ ] `/metrics` 中与 KV 相关的具体 metric 名称（抢占、prefix cache 查询 / 命中、per-tier 指标、transfer 字节）。
- [ ] EvalScope 多轮模式对自定义 trace 的支持程度；是否需要 AIPerf 或自写回放器。
- [ ] Prometheus + Grafana 的本地部署方式（docker compose）。

## 硬件与云资源（Ch2 / Ch3 / Part II 前）

- [ ] 多卡 L20 节点的 PCIe 拓扑：根复合体、switch、NUMA 节点划分（`nvidia-smi topo -m`、`lstopo`）。
- [ ] 主机内存规格：容量、通道数、可 pin 上限。
- [ ] 云块设备两种盘型的具体型号、标称 IOPS / 带宽、是否支持 O_DIRECT 与 io_uring 的全部操作码；内核版本。
- [ ] 本地物理 NVMe / 高性能网络机型的可购性。
- [ ] eRDMA：实例配置方式、是否支持 GPUDirect RDMA、perftest 实测带宽与延迟。
- [ ] GDS 可用性；若可用，驱动与文件系统要求。
- [ ] CPU-only 节点作为远端池的规格与网络。
- [ ] 235B-A22B 级验证点的可行配置（8×L20 FP8 是否装得下；或 4×H20）。

## 方法与流程

- [ ] 预热请求数、稳态窗口长度、重复次数的正式取值（起步：16 / 5–10 分钟 / 3）。
- [ ] `/metrics` 抓取间隔（起步 15–30 秒）。
- [ ] 每章的「完成」判定由谁复核（自查清单 vs 找人 review）。
- [x] 是否 `git init` → 已回答：2026-09-18 本地初始化，`main` 分支；大文件策略采用「外部存储 + manifest 记录位置与校验和」，见 `data/README.md` 与两级 `.gitignore`。远程仓库是否推送待定。
- [ ] 文章发布渠道与语言（中文平台 / 英文 / 双语）；是否同步到 vLLM 社区（Slack / discuss）。
- [ ] 阶段 1 的月度里程碑与实习投递时间点。

## 组件生长（仅当数据指向）

- [ ] Ch3 差距分析是否指向 fs tier 的 page cache / 小文件问题——若是，候选组件是 O_DIRECT / io_uring 块设备层还是打包大文件布局。
- [ ] 国内云对象存储的 obj tier 适配是否有真实需求（NIXL S3 插件的兼容性）。
- [ ] Ch2 差距分析是否指向调度器关键路径上的开销——若是，PR 目标在 vLLM 哪个模块。

## 长期

- [ ] 第二年往 RL 系统搭桥的具体切入点（传输引擎 → weight sync）。
- [ ] 阶段 2 是否把主引擎切到 SGLang（取决于目标团队）。
