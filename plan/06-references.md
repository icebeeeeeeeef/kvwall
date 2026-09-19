# 06 参考资料

只列直接相关的。阅读优先级：★★★ 开工前必读；★★ 对应章节前读；★ 了解即可。

## 1. 直接相关的近期文章

| 优先级 | 文章 | 为什么读 | 链接 |
|---|---|---|---|
| ★★★ | Tiered KV Cache Offloading in vLLM（2026-09-10） | 主线的直接工作面：host 中心设计、fs / obj / p2p tier、`SecondaryTierManager` 接口、per-tier metrics、KV events、hybrid 支持；他们的性能测试 regime 是我们的补集 | https://vllm.ai/blog/2026-09-10-tiered-kv-offloading |
| ★★★ | GLM 5.3 Optimizations Part 1: Hybrid HiSparse Offloading（2026-09-08） | 闭环的标准形态：机制 + Pareto 扫描 + 完整复现附录（钉 commit、数据集构造脚本、EvalScope 配方、指标定义） | https://vllm.ai/blog/2026-09-08-glm53-part1-hybrid-sparse-offloading |
| ★★★ | vLLM x AgentX: Optimizing for Real-World Agentic Serving | 从负载刻画推出挑战的开头方式；数据面 / 执行面 / 控制面分层；MooncakeStore standalone、保留策略、P/D 比例 | https://x.com/vllm_project/status/2097427730983776758 ；vllm.ai/blog/2026-09-08-vllm-agentx |
| ★★ | jaga_prasanna: Hierarchical CPU KV Offloading with SGLang and Dynamo on 16×H100 | 与本人处境最接近的个人实践；学其实验设计（工作集超 HBM、并发阶梯、decode 隔离检查、遥测证机制），修其短板（见 `03` 反模式） | https://x.com/jaga_prasanna/status/2093217133841064233 |
| ★ | akshay_pachaar: Your KV Caching Is Broken | LMCache 内容营销样本；只作为「不要这样写」的参照 | https://x.com/akshay_pachaar/status/2074502882812952666 |
| ★★ | Decode Context Parallelism（vLLM） | MLA 模型下 TP 之外的并行方式；Ch8 相关 | https://vllm.ai/blog/2026-08-07-decode-context-parallelism |
| ★ | DSpark adaptive verification（vLLM） | 投机解码与热缓冲交互的背景 | https://vllm.ai/blog/2026-08-14-dspark-adaptive-verification |
| ★ | LMSYS GLM 5.2 optimization | OpenHands 多轮负载的来源 | https://www.lmsys.org/blog/2026-07-13-glm52-optimization |

## 2. 复现脚本与数据

| 资源 | 用途 |
|---|---|
| neuralmagic/fs-offload-experiments（GitHub） | vLLM tiered offloading 博客的复现脚本；平台骨架起点 |
| GLM 5.3 博客附录：`build_openhands_padded_dataset.py`、`install_evalscope_deps.sh`、EvalScope 钉死 commit | 多轮 agent 负载构造与压测配方 |
| SemiAnalysis AgentX | 真实 agentic coding trace benchmark |
| Mooncake 公开 trace | 请求 trace |
| vLLM 文档：tiered offloading usage guide；`vllm/v1/kv_offload/tiering/example/` | 接口与参考实现 |

## 3. 系统与代码库

| 系统 | 关系 |
|---|---|
| vllm-project/vllm | 主引擎；KV connector、OffloadingConnector、TieringOffloadingSpec、NixlConnector、hybrid memory allocator |
| kvcache-ai/Mooncake | MooncakeStore、Transfer Engine；远端池；Moonshot 生态 |
| ai-dynamo/nixl、ai-dynamo/dynamo | 传输库；KV-aware router |
| sgl-project/sglang | HiCache、HiRadixTree、PD 分离；阶段 2 对照臂 |
| LMCache/LMCache | 可选对照臂 |
| llm-d | KV events 驱动的路由与 P2P 编排（k8s 为主，谨慎） |
| deepseek-ai/3FS | 方案层面了解 |
| FlexKV | 方案层面了解（JD 点名） |

## 4. 论文

| 优先级 | 论文 | 相关章节 |
|---|---|---|
| ★★★ | Mooncake: A KVCache-centric Disaggregated Architecture for LLM Serving（FAST'25；arXiv 2407.00079） | Ch4–Ch6 |
| ★★★ | Efficient Memory Management for LLM Serving with PagedAttention（vLLM，SOSP'23；arXiv 2309.06180） | Ch0–Ch1 |
| ★★ | DistServe: Disaggregating Prefill and Decoding（OSDI'24；arXiv 2401.09670） | Ch6 |
| ★★ | SGLang / RadixAttention（arXiv 2312.07104） | 阶段 2 |
| ★★ | HiSparse（arXiv 2608.07009）；IndexShare（arXiv 2603.12201） | 稀疏 MLA 下的卸载；Ch8 背景 |
| ★★ | CacheGen（arXiv 2310.07240）、CacheBlend（arXiv 2405.16444） | KV 压缩传输与非前缀复用 |
| ★ | ServerlessLLM（arXiv 2401.14351） | 权重加载 / 冷启动；Ch5 热启动背景 |
| ★ | Marconi（hybrid 模型的前缀缓存） | Ch8；vLLM 保留策略的来源 |
| ★ | FlashAttention 系列、MLA（DeepSeek-V2）、DeepSeek 稀疏注意力 | Ch0 算术与 kernel 接口理解 |

## 5. 阅读产出规则

- 每篇 ★★★ 读完写一页笔记进 `notes/reading/`：它测了什么 regime、用了什么指标、没测什么。
- 源码拆解文章（阶段 0 产出）只写 KV 路径：block manager、prefix hash、connector 接口、tiering manager、reload 的 RETRY 路径。
