# 06 参考资料

只列直接相关的。2026-09-25 起，星级表示与主线的相关性，不是开工前完成清单：★★★ 在对应问题出现时优先读；★★ 按章节/适配需求读；★ 了解即可。具体学习出口以 [09](09-learning-roadmap.md) 为准。

这里保留规划阶段收集的链接；文章标题、模型名称、接口与版本须在实际使用时复核并钉住代码。阅读上游报告不等于本人复跑，也不保证未覆盖范围就是无人研究的新机会。

## 1. 直接相关的文章

| 优先级 | 文章/链接 | 何时读、为什么读 |
|---|---|---|
| ★★★ | [Tiered KV Cache Offloading in vLLM（2026-09-10）](https://vllm.ai/blog/2026-09-10-tiered-kv-offloading) | 第一次 offload/次级层实验前：host 中心、fs/obj/p2p、接口与指标 |
| ★★★ | [GLM 5.3 Optimizations Part 1（2026-09-08）](https://vllm.ai/blog/2026-09-08-glm53-part1-hybrid-sparse-offloading) | 学实验设计或进入混合/稀疏模型适配时；不是小 dense 模型开工前置 |
| ★★★ | [vLLM x AgentX](https://vllm.ai/blog/2026-09-08-vllm-agentx) | 构造多轮负载/选择 trace 时，核验数据许可与可获取性 |
| ★★ | [jaga_prasanna：Hierarchical CPU KV Offloading](https://x.com/jaga_prasanna/status/2093217133841064233) | 学工作集、并发扫描、decode 隔离与机制测量；保持其测试范围，不照搬数字 |
| ★ | [Your KV Caching Is Broken](https://x.com/akshay_pachaar/status/2074502882812952666) | 对比文章表达与测量证据，不替代一手实现 |
| ★★ | [Decode Context Parallelism](https://vllm.ai/blog/2026-08-07-decode-context-parallelism) | DCP/MLA 分片相关需求出现时 |
| ★ | [DSpark adaptive verification](https://vllm.ai/blog/2026-08-14-dspark-adaptive-verification) | 选定推测模式及其 KV 有效范围需要核验时 |
| ★★ | [LMSYS GLM 5.2 optimization](https://www.lmsys.org/blog/2026-07-13-glm52-optimization) | 对应模型或多轮复现配方需要时 |

## 2. 一手接口与复现材料

- [vLLM prefix caching](https://docs.vllm.ai/en/latest/design/prefix_caching/)：第一版冷/热对照、hash 与 block 身份。
- [vLLM hybrid KV manager](https://docs.vllm.ai/en/latest/design/hybrid_kv_cache_manager/)：滑动窗口/混合状态与分配/命中。
- [vLLM secondary tier 接口](https://docs.vllm.ai/en/latest/api/vllm/v1/kv_offload/tiering/base/)：主机区、非阻塞任务、完成与清理；实际方法名按 pin。
- [vLLM NIXL 兼容矩阵](https://docs.vllm.ai/en/latest/features/nixl_connector_compatibility/)：模型、dtype、推测与并行配置不能假定任意组合。
- [SGLang HiCache](https://docs.sglang.io/advanced_features/hicache_design.html)：路径、布局、预取与存储。
- [CUDA 性能实践](https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/index.html)、[CUDA on WSL](https://docs.nvidia.com/cuda/wsl-user-guide/index.html)、[GDS overview](https://docs.nvidia.com/gpudirect-storage/overview-guide/)：按当前实验补知识，区分提交/完成、真实路径与兼容模式。
- [neuralmagic/fs-offload-experiments](https://github.com/neuralmagic/fs-offload-experiments)：候选脚本起点，使用前确认依赖与 commit。
- GLM 文章附录的 padded 数据构造、EvalScope 配方，AgentX/Mooncake trace：先核验格式、许可、tokenizer 和获取方式；没有外部 trace 时可先用明确标 synthetic 的负载。

latest 文档会变化；必须记录实际使用代码版本，不能仅凭文章配置判断路径已经支持。

## 3. 系统与代码库

| 系统 | 关系 |
|---|---|
| [vllm-project/vllm](https://github.com/vllm-project/vllm) | 主引擎，scheduler/allocator、offload、NIXL 与混合缓存 |
| [kvcache-ai/Mooncake](https://github.com/kvcache-ai/Mooncake) | Store、Transfer Engine；远端对象与高速数据路径 |
| [ai-dynamo/nixl](https://github.com/ai-dynamo/nixl)、[dynamo](https://github.com/ai-dynamo/dynamo) | 传输抽象与缓存感知路由 |
| [sgl-project/sglang](https://github.com/sgl-project/sglang) | HiCache、前缀树、混合状态与 PD；后续对照/适配 |
| [LMCache/LMCache](https://github.com/LMCache/LMCache) | 存储 backend、独立服务与 GPU 搬运，可按具体路径比较 |
| llm-d | KV events/P2P 编排的候选；不为用它提前引入 k8s |
| [deepseek-ai/3FS](https://github.com/deepseek-ai/3FS)、FlexKV | JD 点名系统，先理解边界，按实际需求再接入 |

## 4. 论文菜单

| 优先级 | 论文 | 对应问题 |
|---|---|---|
| ★★★ | [PagedAttention（SOSP'23）](https://arxiv.org/abs/2309.06180) | 第一版的分块/复用与容量；先读相关部分 |
| ★★★ | [Mooncake（FAST'25）](https://arxiv.org/abs/2407.00079) | Ch4–Ch6；远端缓存和 PD 时深读，不作第一步门槛 |
| ★★ | [DistServe（OSDI'24）](https://arxiv.org/abs/2401.09670) | P/D 干扰、资源与传输取舍 |
| ★★ | [SGLang/RadixAttention](https://arxiv.org/abs/2312.07104) | 前缀树与后续对照 |
| ★★ | [HiSparse](https://arxiv.org/abs/2608.07009)、[IndexShare](https://arxiv.org/abs/2603.12201) | 规划中收集的稀疏/混合架构背景，使用时核验对应实现 |
| ★★ | [CacheGen](https://arxiv.org/abs/2310.07240)、[CacheBlend](https://arxiv.org/abs/2405.16444) | 压缩传输与非前缀复用，按需了解 |
| ★ | [ServerlessLLM](https://arxiv.org/abs/2401.14351) | 区分权重冷启动与 KV 热启动 |
| ★ | Marconi、FlashAttention、DeepSeek-V2/MLA 等 | 对应状态/布局或 kernel 接口需求再展开 |

## 5. 阅读产出与停止规则

每次阅读围绕一个问题写短笔记：它的设置/假设是什么，如何验证，没证明什么，对当前实验有何影响。模板见 `notes/README.md`。

能解释当前路径或提出下一次有区分力的实验就先返回实践，不以「读完全部 ★★★」验收。源码拆解只沿当前任务的创建、buffer 所有权、提交、完成、回收展开；上游 PR/issue 可直接作为小范围阅读入口。
