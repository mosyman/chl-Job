
# 电商MaaS平台研发工程师-TikTok Shop
北京
正式
研发 - 后端
职位 ID：A45510A
职位描述
1、平台规划与架构设计：负责MaaS平台（LumiRouter）的整体技术规划、架构设计与演进路线图制定，确保平台在功能、性能、稳定性及成本控制上具备业界领先水平；
2、统一接入与智能调度体系建设：主导建立标准化的模型与算子接入流程，推动集团内（如ModelHub、火山方舟）及多云多源模型的统一纳管，持续优化平台易用性与开发者体验，实现新模型/业务的分钟级接入与自助化管理；设计并实现高效的智能路由与调度系统，基于业务SLA、负载状况、KV Cache亲和性等特征动态调度请求，最大化模型复用率，探索落地语义理解智能路由策略，构建自动化多层级Fallback与模型降级机制，系统性提升业务韧性；
3、成本管控与FinOps体系建设：建立精细化的成本度量与归因体系，实现Token粒度的计费与可视化，构建完善从预算、监控、分析到优化的FinOps闭环，为业务提供透明成本看板并驱动成本持续优化；
4、资源与稳定性保障：从全局视角规划和管理电商的GPU资源，建立多租户的Quota管理与弹性伸缩机制，负责平台核心SLA，建设全链路可观测性（监控、告警、Tracing）与自动化治理能力，保障平台99.99%可用性；
5、全生命周期管理、合规治理与跨团队协同：定义并落地模型/算子的全生命周期管理规范，覆盖从引入、评测、上线、监控到下线的完整流程，与安全、法务团队紧密合作，确保所有通过平台提供的模型与服务均满足数据安全与合规要求；与底层推理框架（如MixInfer）、模型中台（如ModelHub）、云平台及众多业务方紧密协作，确保端到端链路顺畅高效，积极布道平台能力，推动存量业务（如LLM-studio）向MaaS迁移，提升平台覆盖率。
职位要求
1、本科及以上学历，计算机、软件工程、人工智能等相关专业，5年以上分布式系统、云服务或大模型平台开发经验；
2、深入理解MaaS（模型即服务）平台架构，有大模型网关、推理服务平台、AI中台相关建设经验优先，熟悉LLM/VLM/AIGC等主流模型的部署与服务化流程；
3、扎实的分布式系统设计功底，精通至少一种主流后端语言（Go/Java/Python），有高并发、低延迟系统设计与性能优化经验，熟悉微服务架构、服务治理、可观测性体系建设；
4、熟悉大模型推理相关技术，了解vLLM/SGLang/TensorRT-LLM等主流推理引擎，有GPU集群调度、资源管理、FinOps成本优化相关经验优先；
5、具备优秀的业务理解力与跨团队协作能力，能够深入理解电商场景需求，推动复杂项目落地，有电商行业相关经验优先；
6、具有强烈的技术热情与自驱力，能够主动发现并解决问题，具备良好的团队管理能力。

加分项
1、有过亿级流量大模型服务平台建设经验，熟悉主流云厂商MaaS产品架构与实现原理；
2、参与过大模型训练/推理基础设施建设，有Kubernetes、GPU调度、分布式存储相关经验；
3、有电商行业AI应用落地经验，熟悉搜推、客服、内容生成等电商场景的大模型使用需求；
4、有开源社区贡献经历或相关技术领域论文发表经历。



# claude
正在联网检索相关开源项目（已查到 AIBrix、Gödel/Katalyst、Flux、InfiniStore、Eino 等字节项目），再补几轮搜索就整理成清单。

我按 JD 的 5 个模块把技术能力拆成清单，再用联网检索的结果对应到开源项目，字节跳动的项目单独列出。

> JD 里的 **LumiRouter、MixInfer、ModelHub** 都查不到公开资料，应该是内部系统。**火山方舟**是火山引擎的商业 MaaS 产品，没有开源。

---

## 一、岗位技术能力清单

### A. 后端与分布式基础（职位要求 1、3）
- **语言**：主力 Go（字节后端主栈，Kitex/Hertz 都是 Go），Python 用于接推理生态，Rust 加分（网关、路由器越来越多用 Rust 写）
- **高并发、低延迟**：网络模型（epoll/io_uring）、连接池、零拷贝、序列化优化、压测和 profiling（pprof、火焰图）
- **分布式系统**：一致性、分布式限流（Redis 令牌桶）、分布式缓存、消息队列（Kafka）、多活和容灾
- **微服务与服务治理**：RPC、服务发现、熔断、重试预算、超时传递、灰度发布、Service Mesh（Envoy/xDS）

### B. 统一接入与大模型网关（JD 第 2 条）
- **协议**：OpenAI 兼容 API（Chat/Completions/Embeddings/Batch）、SSE 流式、gRPC；各厂商协议之间的转换（OpenAI/Anthropic/方舟）
- **多源纳管**：自部署模型、集团内平台和外部云厂商的 API 统一注册；模型和算子（embedding/rerank/OCR/CV）通过元数据加 CRD 声明式接入，目标是分钟级接入
- **租户与鉴权**：API Key、AK/SK、租户和应用隔离、审计日志
- **网关扩展**：Envoy ext_proc、WASM 插件、Kubernetes Gateway API
- **开发者体验**：自助控制台、SDK、Playground、接入文档、配额申请流程

### C. 智能路由与调度（JD 第 2 条，岗位核心）
- **LLM 专属指标**：TTFT、TPOT/ITL、队列长度、KV Cache 使用率、在途 token 数
- **路由策略**：least-request、power-of-two、**前缀缓存和 KV Cache 亲和**（前缀哈希、近似 radix tree、KV 事件同步）、session 粘性、LoRA 亲和、按 SLO 路由、公平性调度（按 token 计的虚拟计数器）
- **P/D 分离**：Prefill 和 Decode 分池、KV 传输（RDMA/NIXL）、按拓扑选 worker
- **语义路由**：请求分类（BERT 或小模型打分），把简单请求送到小模型、难请求送到大模型；语义缓存
- **韧性**：多级 Fallback（同模型换实例 → 换集群或地域 → 换供应商 → 降级到其他模型）；流式请求出首 token 之后就不能重试，要单独处理；对冲请求、熔断、按优先级降级和削峰

### D. 推理引擎与推理优化（职位要求 2、4）
- **引擎**：vLLM、SGLang、TensorRT-LLM，以及 Triton Inference Server（适合传统算子）
- **核心机制**：PagedAttention、Continuous Batching、Chunked Prefill、Prefix Caching、投机解码、量化（FP8/INT4/AWQ）、Multi-LoRA
- **大模型部署**：TP/PP/EP 并行、MoE 专家并行、多机推理、Attention/FFN 分离
- **KV Cache 分层**：GPU → CPU → SSD → 远端池，支持跨实例复用
- **多模态**：VLM 和 AIGC（图像、视频生成）的服务化，比如长耗时任务要走异步队列
- **冷启动**：权重预热和缓存、P2P 分发、流式加载

### E. GPU 资源与 K8s（JD 第 4 条）
- Kubernetes Operator 和 CRD 开发、调度器扩展、LeaderWorkerSet（多机推理）、DRA
- **GPU 调度**：拓扑感知（NVLink/NUMA）、异构 GPU 混布、在离线混部、抢占
- **多租户 Quota**：TPM/RPM/并发配额、ResourceQuota、队列式配额（Kueue）、优先级
- **弹性伸缩**：基于 KV 使用率和队列长度等 LLM 指标的 HPA/KEDA/自研 autoscaler；P/D 两个角色按比例协同伸缩；缩容到零
- **GPU 共享**：MIG、vGPU、时间片；GPU 监控（DCGM）
- **多集群和多云**：集群联邦、跨地域调度

### F. 成本管控与 FinOps（JD 第 3 条）
- **Token 级计量**：输入、输出、缓存命中、推理（reasoning）token 分开计；多模态按图片或时长折算；流式请求的 usage 采集
- **计量链路**：网关异步上报 → Kafka → Flink → ClickHouse/OLAP，要求幂等、不丢不重、可对账
- **成本归因**：GPU 小时摊到模型、租户、业务线（Showback/Chargeback），供应商账单和自建成本合在一起看
- **闭环**：预算 → 告警 → 分析 → 优化。优化手段包括提高缓存命中、Batch API 错峰、模型降级和蒸馏、闲时资源复用、提高资源利用率

### G. 可观测性与稳定性（JD 第 4 条）
- Prometheus、Grafana、OpenTelemetry（含 GenAI 语义约定）、分布式 Tracing（Jaeger）、日志平台
- SLI/SLO 和错误预算。99.99% 相当于全年停机不超过约 52 分钟，需要多活、无单点、灰度发布、自动回滚、混沌演练
- 自动化治理：异常实例自动摘除、慢节点检测、容量规划（大促压测）

### H. 模型全生命周期与合规（JD 第 5 条）
- **生命周期**：模型注册和版本管理 → 离线评测（benchmark、业务评测集）→ 灰度和 A/B → 线上监控（质量漂移）→ 下线和迁移
- **合规**：输入输出内容安全（Guardrails）、PII 脱敏、审计留存、跨境数据和数据驻留（TikTok Shop 业务跨多个国家和地区）、开源模型 License 合规、外部供应商的数据使用条款

### I. 业务与软能力（职位要求 5、6）
- **电商场景对推理的不同要求**：
  - 搜推：embedding/rerank，高 QPS，小模型
  - 客服：低延迟，流式输出
  - 商品内容生成和多语言翻译：大批量离线任务，适合走 Batch API
  - 审核：VLM
  - 直播和短视频：AIGC
- 存量平台迁移（LLM-studio 迁到 MaaS）、跨团队推动、平台布道、团队管理

---

## 二、字节跳动相关开源项目（按重要性排序）

| 项目 | 仓库 | 对应 JD | 看点 |
|---|---|---|---|
| **AIBrix** ⭐⭐⭐ | [vllm-project/aibrix](https://github.com/vllm-project/aibrix) | 网关、路由、伸缩、KV Cache、Batch、自助化 | **和这个岗位最贴近的开源项目。** 字节 2024 年启动，已在多个业务线生产使用。基于 Envoy Gateway 的 LLM 感知网关，路由策略包括 prefix-cache、least-request、公平性、session 亲和、按 P/D 拓扑选实例。v0.7.0（2026-06）新增自助 Console、OpenAI 兼容 Batch API、多引擎（vLLM/SGLang/TRT-LLM）、高可用网关（多副本状态同步）、多种路由策略可组合 |
| **Gateway API Inference Extension** | kubernetes-sigs | 路由标准 | 字节与 Google、Red Hat 共建的 K8s LLM 感知路由标准（InferencePool/InferenceModel），支持 LoRA |
| **InfiniStore** | [bytedance/InfiniStore](https://github.com/bytedance/InfiniStore) | KV Cache 亲和、P/D | 基于 RDMA 的分布式 KV Cache 池，支持 P/D 之间的 KV 传输和跨节点复用，已接入 LMCache |
| **Gödel Scheduler** | [kubewharf/godel-scheduler](https://github.com/kubewharf/godel-scheduler) | GPU 调度、在离线混部 | 字节统一调度器，采用乐观并发的分布式架构，调度吞吐约 5000 pods/s，支持拓扑感知和抢占（论文发表于 SoCC'23） |
| **Katalyst** | kubewharf/katalyst-core | 资源效率、QoS | 在离线混部、超卖、干扰检测，与 Gödel 配合使用 |
| **KubeAdmiral / KubeZoo / Kelemetry** | [kubewharf](https://github.com/kubewharf) | 多集群、多租户、可观测 | KubeAdmiral 做多集群联邦（在字节管理超过 10 万个微服务）；KubeZoo 是轻量多租户网关；Kelemetry 做 K8s 控制面全局 Tracing |
| **CloudWeGo：Kitex / Hertz / Netpoll / Sonic** | [cloudwego](https://github.com/cloudwego)、[bytedance/sonic](https://github.com/bytedance/sonic) | 高并发 Go 后端、服务治理 | 字节内部标准 RPC 和 HTTP 框架。**自研网关大概率用这套栈**；Sonic 是基于 SIMD 的高性能 JSON 库，适合网关大量解析请求体的场景 |
| **Eino** | [cloudwego/eino](https://github.com/cloudwego/eino) | 模型接入抽象 | Go 语言的 LLM 应用框架（用于豆包、TikTok），ChatModel 抽象可以参考做多供应商统一接入 |
| **Coze Studio / Coze Loop** | [coze-dev](https://github.com/coze-dev/coze-loop) | 生命周期、评测、观测 | Coze Loop 覆盖 Prompt 调试、自动评测、全链路观测和 Trace，Apache 2.0 许可 |
| **Flux (Comet) / Triton-distributed** | [bytedance/flux](https://github.com/bytedance/flux)、[ByteDance-Seed/Triton-distributed](https://github.com/ByteDance-Seed/Triton-distributed) | 推理底层 | MoE/TP 的计算与通信重叠，用于理解底层推理框架的性能边界 |
| **ShadowKV** | [ByteDance-Seed/ShadowKV](https://github.com/ByteDance-Seed/ShadowKV) | 长上下文 KV | ICML'25 Spotlight。低秩 Key 放 GPU、Value 卸载到 CPU，batch 最多扩大到 6 倍 |
| **veScale / verl** | [volcengine/veScale](https://github.com/volcengine/veScale)、verl-project/verl | 训推基础设施（加分项） | PyTorch 原生的分布式训练框架和 RL 后训练框架 |
| **g3** | [bytedance/g3](https://github.com/bytedance/g3) | Rust 代理和网关 | 企业级代理，Rust 实现。已进入维护模式，官方建议新开发转到 VEY |

**只有论文、没有代码的字节项目，值得读**：
- **MegaScale-Infer**（SIGCOMM'25）：MoE 模型的 Attention/FFN 分离部署，吞吐提升 3.3–5.8 倍
- **HeteroScale**：P/D 分离加异构 GPU 的协同伸缩，在豆包生产环境部署，GPU 利用率提升 26.6 个百分点，每天节省数十万 GPU 小时
- **AIBrix 论文**（arXiv 2504.03648）

---

## 三、业界其他开源项目（对照学习）

| 方向 | 项目 |
|---|---|
| **AI 网关** | Envoy AI Gateway（2026-06 发布 v1.0，已更名 Agent Router；支持 token 级限流和成本追踪）、LiteLLM、Higress（阿里）、Portkey Gateway、APISIX/Kong AI、new-api |
| **推理感知路由和编排** | llm-d、NVIDIA Dynamo（KV-aware Router 和 P/D 分离）、SGLang Model Gateway（原 sgl-router，Rust 实现，cache-aware 路由加熔断限流）、vLLM production-stack |
| **语义路由** | vLLM Semantic Router（Red Hat 主导，作为 Envoy ext_proc 运行）、RouteLLM |
| **KV Cache 层** | LMCache、Mooncake（月之暗面） |
| **推理引擎** | vLLM、SGLang、TensorRT-LLM、LMDeploy |
| **K8s 和 GPU** | KServe、LeaderWorkerSet、Kueue、Volcano、HAMi（GPU 共享和配额）、KAI Scheduler、KEDA、DCGM Exporter |
| **FinOps 和计量** | OpenCost（已支持 llm-d 推理成本追踪）、OpenMeter、Lago、Langfuse（Trace 级 token 成本） |
| **可观测** | OpenTelemetry GenAI 语义约定、OpenLLMetry、Prometheus/Grafana |
| **评测和安全** | lm-evaluation-harness、OpenCompass、Llama Guard、NeMo Guardrails、Presidio（PII） |

---

## 四、学习顺序建议
1. **精读 AIBrix 源码**：先看 gateway plugin 的路由算法和 autoscaler，再看 KV Cache 相关模块。它的架构和 JD 中的 LumiRouter 最接近
2. **对照 Gateway API Inference Extension、llm-d 和 SGLang Model Gateway**：看各自怎么做 KV Cache 亲和路由
3. **Envoy AI Gateway 加 OpenCost**：token 限流、计量、成本归因这条链路
4. **Gödel/Katalyst 加 HeteroScale 论文**：GPU 资源、多租户 Quota、弹性伸缩
5. **动手做一个 mini MaaS 网关**：用 Hertz 或 Kitex 写，OpenAI 兼容，包含多供应商 Fallback、前缀亲和路由、token 计量，作为面试项目

需要的话，我可以把这份清单整理成可分享的网页，或者做成面试准备清单。

**Sources:**
- [AIBrix GitHub](https://github.com/vllm-project/aibrix) · [AIBrix Releases](https://github.com/vllm-project/aibrix/releases) · [vLLM Blog: Introducing AIBrix](https://vllm.ai/blog/2025-02-21-aibrix-release) · [AIBrix v0.4.0 release](https://aibrix.github.io/posts/2025-08-04-v0.4.0-release/) · [AIBrix paper](https://export.arxiv.org/pdf/2504.03648)
- [Google, ByteDance and Red Hat improve AI on Kubernetes](https://cloud.google.com/blog/products/containers-kubernetes/google-bytedance-and-red-hat-improve-ai-on-kubernetes)
- [bytedance/InfiniStore](https://github.com/bytedance/InfiniStore)
- [godel-scheduler](https://github.com/kubewharf/godel-scheduler) · [Gödel paper](https://dl.acm.org/doi/pdf/10.1145/3620678.3624663) · [Katalyst](https://kubewharf.io/blog/2023/12/06/katalyst-a-qos-based-resource-management-system-for-workload-colocation-on-kubernetes/) · [KubeWharf](https://github.com/kubewharf) · [KubeAdmiral (CNCF)](https://www.cncf.io/blog/2023/11/24/kubeadmiral-next-generation-multi-cluster-orchestration-engine-based-on-kubernetes/)
- [CloudWeGo](https://github.com/cloudwego) · [Eino](https://github.com/cloudwego/eino) · [Hertz](https://github.com/cloudwego/hertz) · [Sonic](https://github.com/bytedance/sonic) · [g3](https://github.com/bytedance/g3)
- [Coze Loop](https://github.com/coze-dev/coze-loop)
- [Flux/Comet](https://github.com/bytedance/flux) · [Triton-distributed](https://github.com/ByteDance-Seed/Triton-distributed) · [ShadowKV](https://github.com/ByteDance-Seed/ShadowKV) · [veScale](https://github.com/volcengine/veScale) · [verl](https://github.com/volcengine/veRL)
- [MegaScale-Infer](https://arxiv.org/abs/2504.02263) · [HeteroScale](https://arxiv.org/html/2508.19559v1)
- [vLLM Semantic Router](https://github.com/vllm-project/semantic-router) · [SGLang Model Gateway](https://docs.sglang.io/advanced_features/sgl_model_gateway.html) · [NVIDIA Dynamo](https://github.com/ai-dynamo/dynamo)
- [Envoy AI Gateway v1.0](https://www.prnewswire.com/news-releases/envoy-ai-gateway-reaches-v1-0--establishing-the-open-source-standard-for-enterprise-ai-traffic-302808088.html) · [Envoy AI Gateway 限流文档](https://aigateway.envoyproxy.io/docs/0.1/capabilities/usage-based-ratelimiting/) · [LLM Gateways 对比 2026](https://wavect.io/blog/llm-gateway-router-comparison-2026/)
- [OpenCost 推理成本追踪](https://opencost.io/blog/opencost-llmd-inference-cost/) · [Langfuse Token & Cost Tracking](https://langfuse.com/docs/observability/features/token-and-cost-tracking)
- [HAMi](https://github.com/project-hami/hami) · [Kueue + HAMi](https://kueue.sigs.k8s.io/docs/tasks/run/using_hami/)





# gpt

这个岗位的核心是：**设计并运营一个企业级大模型服务平台，同时对接入效率、推理性能、稳定性和成本负责。** 按 JD 判断，需要具备平台技术负责人级别的系统设计能力。

我已联网核查相关项目，以下按“能力清单 → 字节开源项目 → 其他相关项目 → 学习重点”整理。检索日期为 **2026 年 9 月 28 日**；项目链接优先采用官方仓库。项目范围覆盖这份 JD 的主要技术面，不代表穷尽所有开源仓库。

先说明几个关键名称：

- **LumiRouter、MixInfer、JD 中的 ModelHub**：本次没有找到能够确认对应关系的官方开源仓库，不能把同名项目当成字节内部系统。
- **AIBrix**：确认由字节跳动开发，在 `vllm-project` 下开源，是与这份 JD 最贴近的项目之一。没有公开证据证明它就是 LumiRouter。[官方介绍](https://vllm-project.github.io/2025/02/21/aibrix-release.html)
- **火山方舟**：可以研究其开放接口和 SDK；SDK 开源不等于 MaaS 服务端实现开源。[官方 Python SDK](https://github.com/volcengine/volcengine-python-sdk)

以下能力中的“核心”对应 JD 主线，“深入”表示需要理解机制并具备实践能力，“加分”表示可以选择方向深挖。具体技术细项是我依据 JD 做的工程拆解，并非招聘方逐项列出的硬性要求。

| 技术能力 | 具体需要掌握什么 | 深度 |
|---|---|---|
| **1. MaaS 整体架构** | 控制面与数据面分离；模型目录、网关、调度、推理、计量、治理的边界；多地域、多云架构；容量规划；技术选型、演进路线与迁移方案 | 核心 |
| **2. 后端语言与工程能力** | Go/Java/Python 至少精通一种；并发、异步、内存管理、连接池、线程池、错误处理、性能分析；接口设计、测试、代码评审和模块扩展 | 核心 |
| **3. 分布式系统基础** | 一致性、事务、幂等、分片、缓存、服务发现、配置下发、分布式锁；数据库索引；消息队列的重试、去重、乱序和补偿 | 核心 |
| **4. 高并发网络服务** | HTTP、HTTP/2、gRPC、SSE 流式传输；长连接、背压、取消传播、超时预算、连接复用；Linux 网络与 CPU/内存瓶颈定位 | 核心 |
| **5. 多模型统一接入** | Provider Adapter；协议、鉴权、参数、错误码、usage 统一；模型能力声明；文本、图片、音视频、Embedding、Rerank、工具调用、结构化输出的差异处理 | 核心 |
| **6. 自助化与分钟级接入** | 模型注册、接入模板、配置校验、自动探测、凭证托管、API Key、SDK、Playground、文档和控制台；接入流程自动化与兼容性测试 | 核心 |
| **7. 模型选择与语义路由** | 根据任务、语言、模态、复杂度、质量门槛、成本、延迟和数据地域选择模型；规则路由、分类器/Embedding 路由、强弱模型分流、效果评估 | 核心 |
| **8. 实例路由与请求调度** | 根据队列、并发、剩余容量、输入长度、KV Cache 命中情况选择副本；负载均衡、公平性、热点治理、亲和性与尾延迟权衡 | 核心 |
| **9. LLM 推理原理** | Transformer、Attention、Tokenizer、KV Cache；Prefill/Decode；计算与显存带宽瓶颈；模型权重、激活和缓存的显存占用估算 | 深入 |
| **10. 推理引擎与优化** | vLLM/SGLang/TensorRT-LLM 的部署、配置和排障；Continuous Batching、PagedAttention、Prefix Cache、Chunked Prefill、量化、投机解码、LoRA 服务 | 深入 |
| **11. 分布式推理与缓存** | TP/PP/DP/EP 等并行方式；MoE；Prefill/Decode 分离；KV Cache 传输、分层存储、淘汰和一致性；理解 NVLink、PCIe、RDMA、NCCL 的影响 | 深入 |
| **12. Kubernetes 与 GPU 管理** | Docker、K8s、CRD、Operator、Controller；GPU Device Plugin、GPU Operator；节点健康、拓扑感知、Gang Scheduling、GPU 碎片和异构卡管理 | 深入 |
| **13. 多租户与 Quota** | 租户/项目/模型维度的权限与隔离；RPM、TPM、并发数、GPU 配额、预算配额；预留、借用、抢占、公平调度和 noisy neighbor 治理 | 核心 |
| **14. 弹性伸缩** | 根据队列、Token 负载、延迟和资源压力伸缩；模型预热、冷启动、权重加载、缩容排空；预测扩容、在线与离线资源协调 | 核心 |
| **15. Fallback 与高可用** | 超时、重试、退避、熔断、隔离、限流、过载保护；实例/地域/供应商/模型多层兜底；灰度、回滚、容灾、故障演练和错误预算 | 核心 |
| **16. 全链路可观测性** | Metrics、Logs、Tracing；请求跨网关/引擎/供应商关联；TTFT、TPOT、端到端延迟、吞吐、排队时间、缓存命中率、GPU 指标和告警 | 核心 |
| **17. 性能测试与容量规划** | 输入/输出长度分布、并发、突发流量、多租户混合负载；P95/P99；冷/热缓存对照；SLO 内有效吞吐；容量水位和瓶颈分析 | 核心 |
| **18. Token 计量与账务** | 输入/输出/缓存 Token 的计价口径；流式中断、失败、重试的计费；事件去重、账本、价格版本、预算预占与结算、供应商对账 | 核心 |
| **19. FinOps 成本优化** | 成本归因到业务、团队、模型和请求；GPU 空闲与共享成本分摊；预算、预警、预测、异常分析；缓存、批处理、量化、模型分流的收益评估 | 核心 |
| **20. 模型与算子生命周期** | 制品、版本、元数据、依赖和兼容性管理；引入、评测、审批、发布、监控、回滚和下线；效果、安全、性能、成本联合准入 | 核心 |
| **21. 安全与合规工程** | IAM/RBAC、密钥轮换、加密、审计、PII 脱敏、日志保留策略、数据地域限制；模型许可证管理、供应链安全；将治理规则落实到路由和发布流程 | 核心 |
| **22. 电商业务与迁移** | 搜推、客服、商品理解、内容生成、多语言多模态场景；RAG、Embedding、Rerank 的服务需求；效果指标、A/B 实验、存量接口兼容、影子流量、迁移回滚 | 深入 |

另外，JD 明确要求架构规划、跨团队推进和团队管理。因此还需要能产出设计文档、容量与成本测算、迁移方案和事故复盘，并推动业务、推理框架、云平台、安全团队共同落地。

**CUDA/Triton 算子开发、训练框架、RLHF/RL、分布式 Checkpoint 属于进一步加分方向。** JD 没有要求候选人同时精通三种后端语言，也没有明确要求成为底层 Kernel 专家。

有几个技术边界尤其值得理解：

| 容易混淆的概念 | 对这个岗位应有的理解 |
|---|---|
| **语义路由与 KV Cache 路由** | 前者决定“用哪个模型”，后者通常决定“请求发给哪个推理实例”。两层策略需要协同 |
| **KV Cache 与语义结果缓存** | 前者复用中间计算状态；后者复用已有答案。有效性条件、收益和数据隔离要求不同 |
| **请求调度与 GPU 调度** | 请求调度决定流量去哪；GPU 调度决定模型实例部署在哪、分配多少资源 |
| **Token 账单与资源成本** | API 用量计价、GPU 实际成本、内部业务成本分摊是不同口径，需要关联核算 |
| **分钟级接入与分钟级部署** | 已支持协议的服务接入可以模板化；新模型架构适配、权重下载和评测需要另算时间 |
| **“算子”** | JD 未明确粒度，可能包含平台业务能力单元；需要与 CUDA/Triton 计算算子区分 |

下面优先列出**字节跳动开发、发起或相关团队维护的开源项目**。相关性是针对这份岗位的判断，不意味着该团队实际采用了这些项目。

| 项目 | 定位与可以学习的内容 | 岗位相关性 |
|---|---|---|
| **[AIBrix](https://github.com/vllm-project/aibrix)** | 云原生推理基础设施：网关与路由、弹性伸缩、LoRA 管理、统一运行时、分布式推理和 KV Cache | **最高，建议首先研究** |
| **[InfiniStore](https://github.com/bytedance/InfiniStore)** | 分布式推理的 KV Cache 存储与传输；跨节点复用、缓存池、TCP/RDMA 接入 | **很高：缓存与推理效率** |
| **[Katalyst](https://github.com/kubewharf/katalyst-core)** | QoS 资源模型、水平/垂直弹性、拓扑感知、资源分配与隔离 | **很高：资源效率与成本** |
| **[Gödel Scheduler](https://github.com/kubewharf/godel-scheduler)** | 在线/离线统一调度；并发调度、Gang Scheduling、抢占、亲和性和资源预留 | **高：大规模调度架构** |
| **[KubeAdmiral](https://github.com/kubewharf/kubeadmiral)** | Kubernetes 多集群编排、调度策略、资源分发、差异化配置和状态聚合 | **高：多集群资源管理** |
| **[KubeZoo](https://github.com/kubewharf/kubezoo)** | 通过 API 网关提供 Kubernetes 多租户视图隔离；共享控制面与数据面 | **中高：租户抽象设计** |
| **[Kitex](https://github.com/cloudwego/kitex)** | Go RPC 框架；服务发现、负载均衡、超时、重试、熔断、限流、Tracing 等治理能力 | **很高：后端基本功** |
| **[Hertz](https://github.com/cloudwego/hertz)** | Go HTTP 框架；协议处理、中间件、扩展机制和高性能网络服务 | **高：网关与控制面开发** |
| **[Netpoll](https://github.com/cloudwego/netpoll)** | 非阻塞网络 I/O 框架，适合学习 RPC 网络层设计 | **中高：性能优化** |
| **[Sonic](https://github.com/bytedance/sonic)** | 高性能 JSON 序列化与反序列化 | **中：网关热点优化** |
| **[Eino](https://github.com/cloudwego/eino) / [Eino-ext](https://github.com/cloudwego/eino-ext)** | Go AI 应用框架与组件扩展；ChatModel、Tool、Retriever、Embedding 抽象，图编排和流处理 | **高：接入抽象与业务协同** |
| **[Coze Loop](https://github.com/coze-dev/coze-loop)** | Prompt 调试与版本、评测集、评测器、实验管理和调用链观测 | **很高：评测与 LLMOps** |
| **[Coze Studio](https://github.com/coze-dev/coze-studio)** | Agent 开发平台，包含工作流、模型与工具集成等能力 | **中高：理解平台使用方** |
| **[Flux / COMET](https://github.com/bytedance/flux)** | GPU 上计算与通信重叠，覆盖 Dense/MoE、张量/专家并行相关优化 | **加分：底层性能** |
| **[Triton-distributed](https://github.com/ByteDance-Seed/Triton-distributed)** | 分布式编译器与并行 Kernel，研究多 GPU 计算和通信协同 | **加分：底层性能** |
| **[verl](https://github.com/verl-project/verl)** | 字节 Seed 发起的强化学习后训练框架；学习 rollout、训练/推理资源协同和框架集成 | **加分：训练推理基础设施** |
| **[veScale](https://github.com/volcengine/veScale)** | 大规模分布式训练、分布式 Tensor 和 FSDP 相关设计 | **加分：分布式训练** |
| **[ByteCheckpoint](https://github.com/ByteDance-Seed/ByteCheckpoint)** | 统一 Checkpoint、加载时重分片、异步/并行 I/O 和制品转换 | **加分：存储与恢复** |
| **[Monolith](https://github.com/bytedance/monolith)** | 推荐系统开源实现，可了解推荐业务及其基础设施需求 | **场景补充：搜推业务** |
| **[火山引擎 Python SDK](https://github.com/volcengine/volcengine-python-sdk)** | 研究方舟等服务的客户端接口、请求模型和接入方式 | **接入参考：不是 MaaS 服务端** |

其中，CloudWeGo 的字节背景可以通过其[官网说明](https://www.cloudwego.io/)核实；Eino 的[官方开源介绍](https://www.cloudwego.io/docs/eino/overview/eino_open_source/)也说明了其在字节内部的应用。verl 当前已迁移至 `verl-project/verl`，官方 README 明确说明由字节 Seed 发起。[verl 官方说明](https://github.com/verl-project/verl/blob/main/README.md)

研究这些项目时，有三处具体边界需要留意：

- **AIBrix** 的部分控制台、Batch 和资源管理能力在 v0.7 发布说明中仍被标为 Preview，不能把路线图当成已成熟的生产能力。[发布说明](https://github.com/vllm-project/aibrix/releases)
- **Katalyst** 依赖 KubeWharf 增强集群；Gödel、KubeAdmiral、KubeZoo 的 README 还列有特定 K8s 版本约束，学习设计与直接部署需要分别判断。[Katalyst](https://github.com/kubewharf/katalyst-core)、[Gödel](https://github.com/kubewharf/godel-scheduler)
- **veScale** 官方说明公开的是内部系统的一部分，旧实现位于 `legacy/`。[仓库说明](https://github.com/volcengine/veScale)

除字节项目外，以下开源实现可以补齐整个 MaaS 技术体系。

**统一接入、智能路由与推理服务**，建议对照这些项目：

| 项目 | 对应能力与阅读重点 |
|---|---|
| **[LiteLLM](https://github.com/BerriAI/litellm)** | 多供应商协议适配、统一接口、虚拟密钥、用量成本、负载均衡；特别适合研究统一模型网关 |
| **[Higress](https://github.com/higress-group/higress)** | 基于 Envoy/Istio 的 AI 网关、插件扩展、模型接口接入和流量治理 |
| **[Agent Router](https://github.com/theagentrouter/agent-router)** | 原 Envoy AI Gateway；统一接入、凭证、路由、Quota、故障切换与用量归因 |
| **[vLLM Semantic Router](https://github.com/vllm-project/semantic-router)** | 根据请求信号和策略选择或组合模型，直接对应语义路由方向 |
| **[RouteLLM](https://github.com/lm-sys/RouteLLM)** | 模型路由训练与评测，研究强弱模型选择以及质量—成本权衡 |
| **[Gateway API Inference Extension](https://github.com/kubernetes-sigs/gateway-api-inference-extension)** | Kubernetes 推理流量调度接口与扩展机制 |
| **[vLLM](https://github.com/vllm-project/vllm)** | PagedAttention、连续批处理、Prefix Cache、量化、分布式并行与模型服务化 |
| **[SGLang](https://github.com/sgl-project/sglang)** | RadixAttention、缓存复用、请求调度、P/D 分离、结构化输出和多模态服务 |
| **[TensorRT-LLM](https://github.com/NVIDIA/TensorRT-LLM)** | NVIDIA GPU 推理优化、运行时、量化与并行执行 |
| **[LMCache](https://github.com/LMCache/LMCache)** | KV Cache 存储、卸载和复用，适合与推理引擎结合研究 |
| **[Mooncake](https://github.com/kvcache-ai/Mooncake)** | KV Cache 为中心的分布式推理、缓存传输与存储；来自 Moonshot AI |
| **[NVIDIA Dynamo](https://github.com/ai-dynamo/dynamo)** | 推理引擎之上的分布式编排层，覆盖路由、P/D 分离、分层 KV Cache 和伸缩 |
| **[llm-d](https://github.com/llm-d/llm-d)** | Kubernetes 上的分布式推理，涵盖缓存/负载感知路由、KV 管理与运维 |
| **[Llumnix](https://github.com/llumnix-project/llumnix)** | 分布式推理调度与重调度、引擎状态感知、KV 传输；具有阿里云背景，不是 LumiRouter |
| **[KServe](https://github.com/kserve/kserve)** | Kubernetes 模型服务标准化、部署抽象和控制器设计 |
| **[Ray / Ray Serve](https://github.com/ray-project/ray)** | 分布式执行、服务组合、模型服务与资源编排 |
| **[Xinference](https://github.com/xorbitsai/inference)** | 多种模型的统一部署与推理接口，适合观察模型管理和服务化设计 |
| **[Triton Inference Server](https://github.com/triton-inference-server/server)** | 多框架推理服务、批处理和模型服务管理；注意与 Triton Kernel 编程语言区分 |

**GPU 资源、多租户和弹性**，建议看：

| 项目 | 对应能力与阅读重点 |
|---|---|
| **[Volcano](https://github.com/volcano-sh/volcano)** | Kubernetes 批任务与 AI 工作负载调度、队列、资源协调 |
| **[Kueue](https://github.com/kubernetes-sigs/kueue)** | 作业排队、准入、配额与资源公平共享 |
| **[HAMi](https://github.com/Project-HAMi/HAMi)** | Kubernetes 异构 GPU 共享和资源管理 |
| **[NVIDIA GPU Operator](https://github.com/NVIDIA/gpu-operator)** | GPU 驱动及相关组件的自动部署和生命周期管理 |
| **[DCGM Exporter](https://github.com/NVIDIA/dcgm-exporter)** | GPU 指标采集，连接 GPU 健康、利用率与监控体系 |
| **[KEDA](https://github.com/kedacore/keda)** | 事件与指标驱动的弹性伸缩，可研究如何接入队列或业务负载信号 |

**FinOps、可观测性、评测与治理**，建议看：

| 项目 | 对应能力与阅读重点 |
|---|---|
| **[OpenCost](https://github.com/opencost/opencost)** | Kubernetes 和云资源成本监控、资源成本分配，补齐 GPU/基础设施成本视角 |
| **[OpenMeter](https://github.com/openmeterio/openmeter)** | 用量事件采集、聚合与按量计费，适合研究 Token 计量事件链路 |
| **[Langfuse](https://github.com/langfuse/langfuse)** | LLM 调用链、评测与应用观测，可与 Coze Loop 对照 |
| **[MLflow](https://github.com/mlflow/mlflow)** | 模型与实验管理、评测和生命周期管理 |
| **[EvalScope](https://github.com/modelscope/evalscope)** | LLM/VLM/AIGC 效果评测和性能基准测试 |
| **[OpenCompass](https://github.com/open-compass/opencompass)** | 多模型、多数据集评测，适合构建模型准入与回归评测体系 |
| **[OpenTelemetry Collector](https://github.com/open-telemetry/opentelemetry-collector)** | 遥测数据接收、处理与导出，统一链路和指标采集 |
| **[Prometheus](https://github.com/prometheus/prometheus) / [Grafana](https://github.com/grafana/grafana)** | 指标、查询与看板，承载 SLA、推理性能、资源和成本视图 |
| **[OPA](https://github.com/open-policy-agent/opa)** | 策略即代码，可用于表达访问、发布或路由准入规则 |
| **[Presidio](https://github.com/data-privacy-stack/presidio)** | 敏感信息识别、脱敏和匿名化 |
| **[NeMo Guardrails](https://github.com/NVIDIA-NeMo/Guardrails)** | 为 LLM 应用增加可编程安全约束 |
| **[Chaos Mesh](https://github.com/chaos-mesh/chaos-mesh)** | Kubernetes 故障注入，验证超时、降级、容灾和恢复行为 |

这些项目覆盖的层次不同。例如，OpenCost 解决资源成本视角，OpenMeter 解决用量计量视角；两者仍需要业务归因和账务逻辑才能组成完整 FinOps 闭环。LiteLLM、Langfuse 等项目还区分开源与企业功能，选型时应核对具体能力所在版本。

如果是为了应聘，**我建议优先读透下面八条主线**，每条选择一个主项目做实践，其他项目用于对照：

| 顺序 | 主项目 | 应该能够讲清楚的问题 |
|---|---|---|
| 1 | **AIBrix** | 一个请求如何经过网关、路由和推理实例？控制面故障会怎样影响已有流量？ |
| 2 | **LiteLLM 或 Higress** | 如何新增供应商，并统一流式协议、鉴权、错误码、限流和 usage？ |
| 3 | **vLLM 或 SGLang** | 为什么吞吐提高后尾延迟可能恶化？如何调整批处理、缓存和并行配置？ |
| 4 | **InfiniStore + LMCache** | 缓存命中节省多少计算？传输成本何时抵消收益？如何隔离租户及模型版本？ |
| 5 | **Semantic Router 或 RouteLLM** | 如何证明模型分流降低了成本，同时满足业务质量门槛？ |
| 6 | **Katalyst + Kueue/Volcano** | 如何设计配额、公平性、抢占和资源借用，同时保护在线业务？ |
| 7 | **Coze Loop + OpenTelemetry** | 如何区分模型效果退化、网关故障、排队拥堵和 GPU 异常？ |
| 8 | **OpenMeter + OpenCost** | 如何将请求 Token、供应商账单、GPU 成本和业务预算关联起来？ |

最有针对性的练习，是搭建一个包含“两种模型、两个租户、多个推理副本”的 MaaS 原型，并完成以下验收：

1. **统一接入**：新增一个供应商主要通过 Adapter 和配置完成，协议兼容性有自动验证。
2. **路由实验**：比较轮询、负载感知、缓存感知路由的 P95/P99、TTFT 和有效吞吐。
3. **故障演练**：注入超时、429、实例退出和流式断连，验证重试预算与 Fallback；已经输出内容的流式请求不能随意切换模型拼接结果。
4. **成本核算**：覆盖正常请求、重试、取消和失败，展示用量去重、结算与对账。
5. **资源治理**：一个租户突发流量时，另一个租户仍能满足约定的服务目标。
6. **业务评测**：分别用客服、商品理解、内容生成任务衡量质量、延迟和成本，让优化结果能对应电商业务价值。