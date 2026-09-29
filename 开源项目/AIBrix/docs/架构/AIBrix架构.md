

# gpt

**AIBrix 负责组织、管理和优化模型推理服务；实际的模型计算由 vLLM、SGLang 等推理引擎完成。** 控制平面决定部署什么、部署多少、加载在哪里；数据平面处理实际请求，选择实例，并协助复用计算结果。

以下依据你当前仓库提交 **`c6d7b694`** 解释。代码引用均标明文件、类或函数及行号；附图按架构示意解读，不把每个框都视为当前独立部署的服务。这个定位也与[官方概念说明](https://aibrix.readthedocs.io/latest/getting_started/overview.html)一致，其仓库原文见 [overview.rst，第 14–27 行](/Users/chl/chl-code/Job/aibrix/docs/source/getting_started/overview.rst:14)。

---

**先按“管理资源”和“处理请求”理解两个平面。**

| 对比项 | 控制平面 | 数据平面 |
|---|---|---|
| 核心问题 | 系统应该处于什么状态？ | 当前请求应该如何执行？ |
| 典型决策 | 副本数量、适配器放置、模型生命周期 | 请求准入、目标实例选择、缓存读取 |
| 处理对象 | Kubernetes 资源、模型配置、指标、状态 | HTTP 请求、响应、token、KV 张量 |
| 主要工作方式 | 观察状态，持续纠正实际状态与期望状态的差异 | 随请求执行路由、转发、推理与缓存操作 |
| 关系 | 根据运行指标调整资源和策略 | 使用这些资源与策略，并产生新的运行指标 |

依据：[architecture.rst，第 16–37 行](/Users/chl/chl-code/Job/aibrix/docs/source/designs/architecture.rst:16)；[overview.rst，第 55–65 行](/Users/chl/chl-code/Job/aibrix/docs/source/getting_started/overview.rst:55)。

下面是根据当前实现简化的 ASCII 图，省略了可选功能和异常分支：

```text
                         CONTROL PLANE

  Kubernetes API / desired state
                  |
                  v
  +---------------------------------------------+
  | ModelAdapter | RayClusterFleet | Autoscaler  |
  | ModelRouter  | GPU Optimizer   | KVCache CR  |
  +---------------------------------------------+
          | management / configuration
          v
  +-------------------+             metrics / events
  | AI Runtime        |-------------------------------+
  | inside engine Pod |                               |
  +-------------------+                               |
          | manage engine                             |
          v                                           |
                         DATA PLANE                   |
                                                      |
  Client ---> Envoy --------------------> Engine Pod -+
                |                            |
                | ext_proc                   | KV blocks
                v                            v
          AIBrix Router                L1 / L2 cache
                |
                +-- model / pod / load / prefix state
                |
                +-- return target-pod to Envoy

  Client <--- Envoy <--------------------- Engine Pod
                  response / token stream
```

这里有三个关键点：

- **网关插件参与请求决策，Envoy 承担通常的上游转发。** 插件通过 `ext_proc` 接收请求处理消息，返回目标地址等修改结果。不能把它理解为插件完全看不到请求内容。
- **AI Runtime 位于推理 Pod 内，但主要承担控制平面的管理职责。** 物理部署位置和逻辑职责是两回事。
- **控制器不需要在每个 token 生成时参与决策。** 路由与控制通常使用持续更新的状态、指标和缓存。

代码依据：`Server.Process`，见 [gateway.go，第 329 行](/Users/chl/chl-code/Job/aibrix/pkg/plugins/gateway/gateway.go:329)；`Server.HandleRequestBody` 写入 `target-pod`，见 [gateway_req_body.go，第 180–205 行](/Users/chl/chl-code/Job/aibrix/pkg/plugins/gateway/gateway_req_body.go:180)。Runtime 的职责边界见 [aibrix-engine-runtime.rst，第 16–23 行](/Users/chl/chl-code/Job/aibrix/docs/source/designs/aibrix-engine-runtime.rst:16)。

**图右上角的模型元数据注册，解决的是“模型名称对应哪些可用服务实例”。**

客户端指定一个模型名称，基础设施需要知道：

- 这是基础模型还是某个 LoRA 适配器；
- 哪些 Pod 能提供它；
- 对应的服务端口是什么；
- 哪些实例已经可用；
- 适配器依赖哪个基础模型。

这些是“关于模型和服务的信息”，与模型权重文件、推理产生的 KV 张量不同。

当前代码中，这些职责分布在资源元数据、发现机制和缓存中。例如：

- `initCacheInformers` 监听 Pod、ModelAdapter 的新增、更新与删除，见 [informers.go，第 54–99 行](/Users/chl/chl-code/Job/aibrix/pkg/cache/informers.go:54)。
- `Store.addModelAdapter` 根据 `status.instances` 建立模型与 Pod 的映射，见 [informers.go，第 287–315 行](/Users/chl/chl-code/Job/aibrix/pkg/cache/informers.go:287)。
- `Store.ListPodsByModel` 从缓存获取模型对应的 Pod，见 [cache_impl.go，第 73–79 行](/Users/chl/chl-code/Job/aibrix/pkg/cache/cache_impl.go:73)。
- `ModelRouter.createHTTPRoute` 创建模型匹配规则和后端 Service 引用，见 [modelrouter_controller.go，第 243–305 行](/Users/chl/chl-code/Job/aibrix/pkg/controller/modelrouter/modelrouter_controller.go:243)。

因此，图里的 `Model Metadata Controller / Store` 更适合作为职责概括，不能直接推导出“每个请求都必须同步访问一个独立元数据数据库”。

**模型适配器控制器管理 LoRA 的放置、加载、卸载和服务发现。**

LoRA 可以理解为附着在基础模型上的轻量适配参数。同一基础模型可以搭配不同适配器，为不同任务提供服务，从而减少重复部署完整基础模型的开销。

你图中的三个 Pod 表示：

| Pod | GPU 示例 | 加载内容 |
|---|---|---|
| Pod 1 | A100 | Base、LoRA1、LoRA2 |
| Pod 2 | A100 | Base、LoRA2、LoRA3 |
| Pod 3 | L40 | Base、LoRA1 |

由此可以得到：

- 请求 LoRA1，可从 Pod 1、Pod 3 中选择。
- 请求 LoRA2，可从 Pod 1、Pod 2 中选择。
- 请求 LoRA3，图示中只有 Pod 2 能承接。
- 多个适配器共享一个 Pod 中的基础模型，但各 Pod 仍有各自的基础模型实例。

这解释了“每个 Pod 多个 LoRA”的价值：资源密度提高了，但这些适配器仍共同竞争所在 Pod 的算力、显存和请求处理能力。

仓库文档对此的说明见 [lora-dynamic-loading.rst，第 7–20 行](/Users/chl/chl-code/Job/aibrix/docs/source/features/lora-dynamic-loading.rst:7)。

控制器的工作过程是：

```text
ModelAdapter CR
       |
       v
Select matching Pods
       |
       v
Load adapter through runtime / engine API
       |
       v
Publish Service + EndpointSlice
       |
       v
Update status and keep reconciling
```

这张图中的“调度”是**选择把适配器加载到哪个已有 Pod**；Kubernetes 将 Pod 放到哪个节点，是另一个层次的调度。

对应代码是 `ModelAdapterReconciler.DoReconcile`：依次协调实例、加载、Service 和 EndpointSlice，见 [modeladapter_controller.go，第 447–503 行](/Users/chl/chl-code/Job/aibrix/pkg/controller/modeladapter/modeladapter_controller.go:447)。

图中的 `Custom Endpoint Slice` 尤其重要：一个适配器可能只在部分 Pod 上加载成功，服务发现必须反映这个集合。当前 `reconcileEndpointSlice` 根据适配器的 `status.instances` 收集 Pod，并排除正在删除的 Pod，见 [modeladapter_controller.go，第 1029–1039 行](/Users/chl/chl-code/Job/aibrix/pkg/controller/modeladapter/modeladapter_controller.go:1029)。

还有一个当前版本的具体限制：`ModelAdapterSpec.replicas` **省略时加载到全部匹配 Pod，设置为 `1` 时选择单个 Pod；并不支持任意整数副本数**。见 [modeladapter_types.go，`ModelAdapterSpec`，第 50–56 行](/Users/chl/chl-code/Job/aibrix/api/model/v1alpha1/modeladapter_types.go:50)。

**RayClusterFleet 管理“由多个节点共同组成的推理副本”。**

普通单机服务的一个副本通常对应一个 Pod。大模型如果需要跨节点分布式执行，一个完整副本可能需要一组 Pod：

```text
RayClusterFleet
       |
       v
RayClusterReplicaSet
       |
       +--> RayCluster A --> Head Pod + Worker Pod(s)
       |
       +--> RayCluster B --> Head Pod + Worker Pod(s)
```

各层职责是：

| 层次 | 职责 |
|---|---|
| `RayClusterFleet` | 版本更新、滚动发布、暂停、历史版本管理 |
| `RayClusterReplicaSet` | 维持指定数量的 RayCluster |
| KubeRay `RayCluster` | 管理一个 Ray 集群对应的 Head、Worker Pod |
| Ray 与推理引擎 | 在该集群内部组织分布式推理执行 |

因此，Fleet 的副本数应理解为**完整 RayCluster 的数量**。假设一个集群包含一个 Head 和一个 Worker，三个集群副本通常就涉及六个 Pod，而不是三个。

标准配置下，网关把推理请求送到 Head Pod，Worker 提供分布式计算资源。依据：[multi-node-inference.rst，第 87–111 行](/Users/chl/chl-code/Job/aibrix/docs/source/features/multi-node-inference.rst:87)。控制器在 `RayClusterFleetReconciler.Reconcile` 中区分扩缩事件和发布策略，见 [rayclusterfleet_controller.go，第 166–179 行](/Users/chl/chl-code/Job/aibrix/pkg/controller/rayclusterfleet/rayclusterfleet_controller.go:166)。

“确保最佳性能”是架构目标，不能理解为 Fleet 自动决定最优张量并行、流水线并行或网络配置。Fleet 主要提供生命周期管理；实际性能仍受引擎配置、GPU 和节点通信条件影响。

**LLM 自动扩缩决定“需要多少服务容量”。**

LLM 请求的成本差异很大：短输入、短输出与长上下文、长输出，对 GPU 的占用可能完全不同。因此，只看请求数量或 CPU 利用率往往不足以判断推理负载。

AIBrix 的 `PodAutoscaler` 可以配置指标来源与目标值，例如使用 KV cache 占用率作为压力信号。当前支持：

| 策略 | 主要特点 |
|---|---|
| HPA | 使用 Kubernetes 原生 HPA |
| KPA | 使用稳定窗口与较短的 panic 窗口，应对突发负载 |
| APA | 按指标与目标值的比例计算，并通过容忍区间减少抖动 |

依据：[metric-based-autoscaling.rst，第 8–24 行](/Users/chl/chl-code/Job/aibrix/docs/source/features/autoscaling/metric-based-autoscaling.rst:8)；KV cache 指标配置示例见同文件 [第 96–116 行](/Users/chl/chl-code/Job/aibrix/docs/source/features/autoscaling/metric-based-autoscaling.rst:96)。

以 APA 为例，忽略容忍区间与扩缩速率限制时，其比例计算可以简化为：

`期望副本数 = ceil(当前副本数 × 当前每 Pod 指标值 / 目标值)`

例如，当前四个副本、平均指标为 `0.8`、目标为 `0.5`，比例计算得到七个副本。这只是算法示例，实际结果还会受约束影响。对应 `APAAlgorithm.computeTargetReplicas`，见 [apa.go，第 70–109 行](/Users/chl/chl-code/Job/aibrix/pkg/controller/podautoscaler/algorithm/apa.go:70)。

整个控制过程是：

```text
Engine metrics
      |
      v
Collect / aggregate
      |
      v
Compute desired replicas
      |
      v
Apply bounds / stabilization
      |
      v
Update workload replica count
      |
      v
New instances become ready
```

`PodAutoscalerReconciler.reconcileCustomPA` 获取当前副本数、计算决策并调用 `SetDesiredReplicas`，见 [podautoscaler_controller.go，第 857–908 行](/Users/chl/chl-code/Job/aibrix/pkg/controller/podautoscaler/podautoscaler_controller.go:857)。

**“秒级扩缩”描述的是控制响应能力，不等于模型在几秒内一定完成扩容。** 当前控制器默认同步周期为十秒，见同文件 [第 104–106 行](/Users/chl/chl-code/Job/aibrix/pkg/controller/podautoscaler/podautoscaler_controller.go:104)。真正可接请求之前，还可能需要等待 GPU、拉取镜像、下载权重、初始化引擎和通过就绪检查。

**GPU Optimizer 决定“用哪些 GPU、各用多少，成本更合适”。**

自动扩缩器主要执行容量调整；GPU Optimizer 进一步考虑异构硬件的性能和成本差异。

它需要三类输入：

- **性能画像**：某个模型在不同 GPU、不同输入输出长度下的服务能力。
- **工作负载分布**：当前请求更偏向短请求、长上下文还是长输出。
- **成本与服务目标**：GPU 成本，以及允许的延迟、吞吐等约束。

输出可以是类似“某类 GPU 对应两个副本，另一类 GPU 对应四个副本”的建议。它并不是把正在运行的 A100 Pod 直接改造成 L40 Pod，而是协调不同硬件对应的工作负载容量。

仓库要求先针对具体模型和 GPU 做离线性能测试，优化结果再通过 HTTP 指标接口提供给 `PodAutoscaler`，见 [heterogeneous-gpu.rst，第 101–127 行](/Users/chl/chl-code/Job/aibrix/docs/source/features/heterogeneous-gpu.rst:101)。

代码中：

- `Optimizer.set_profile` 注册 GPU 性能画像，见 [optimizer.py，第 42–64 行](/Users/chl/chl-code/Job/aibrix/python/aibrix/aibrix/gpu_optimizer/optimizer/optimizer.py:42)。
- `Optimizer.run` 调用求解器，返回 GPU 副本数与成本结果，见同文件 [第 114–132 行](/Users/chl/chl-code/Job/aibrix/python/aibrix/aibrix/gpu_optimizer/optimizer/optimizer.py:114)。
- `get_deployment_metrics` 暴露建议副本数指标，见 [gpu_optimizer/app.py，第 272–290 行](/Users/chl/chl-code/Job/aibrix/python/aibrix/aibrix/gpu_optimizer/app.py:272)。

这里的“服务保证”依赖画像准确性和负载假设。离线测试条件与线上差异很大时，优化建议也可能偏离实际需求。

**AI Engine Runtime 是控制器与推理引擎之间的管理接口。**

你图中的 Sidecar Container 包含 Model Loader、Runtime Agent、Watch Dog，表达的是一组管理职责，不意味着当前一定有三个独立进程。

Runtime 主要负责模型下载、适配器管理、指标标准化，以及引擎状态查询。当前可以直接定位到这些接口：

| 能力 | 当前代码位置 |
|---|---|
| 抓取并标准化引擎指标 | `mount_metrics`：[app.py，第 88–116 行](/Users/chl/chl-code/Job/aibrix/python/aibrix/aibrix/app.py:88) |
| 加载 LoRA | `load_lora_adapter`：[app.py，第 140–174 行](/Users/chl/chl-code/Job/aibrix/python/aibrix/aibrix/app.py:140) |
| 卸载 LoRA | `unload_lora_adapter`：[app.py，第 184–213 行](/Users/chl/chl-code/Job/aibrix/python/aibrix/aibrix/app.py:184) |
| 查询引擎已提供的模型 | `list_engine_models`：[app.py，第 223–231 行](/Users/chl/chl-code/Job/aibrix/python/aibrix/aibrix/app.py:223) |
| 下载模型、查询本地模型文件 | `download_model`、`list_model`：[app.py，第 234–249 行](/Users/chl/chl-code/Job/aibrix/python/aibrix/aibrix/app.py:234) |

这种抽象的作用是：控制器调用相对统一的管理接口，再由具体引擎适配层转换。例如，`VLLMInferenceEngine.load_lora_adapter` 调用 vLLM 的 `/v1/load_lora_adapter`，见 [vllm.py，第 107–110 行](/Users/chl/chl-code/Job/aibrix/python/aibrix/aibrix/openapi/engine/vllm.py:107)。

图左侧的 S3、Artifact Location、credentials，表示**模型或适配器文件的来源与访问凭据**。文件可以下载到 Pod 可访问的存储，再由引擎加载。图中的 `Mount` 是挂载存储，`Mem` 表示内存层次；下载完成与模型已经能接请求，是两个不同状态。

另外，模型仓库访问凭据与客户端调用推理 API 的 `api_key`，承担不同用途。

**加速器诊断工具的目标是发现故障、辅助定位并验证系统的故障应对能力，但需要区分架构目标和现有实现。**

可以把相关能力分成三层：

1. 进程和服务检查：Runtime 是否存活、引擎是否就绪。
2. 测试与诊断：模拟推理行为、收集 Pod 状态和故障日志。
3. 硬件诊断与恢复：识别 GPU、显存、设备通信等故障，并触发隔离或替换。

架构文档确实列出了 `Accelerator Diagnose Tools`，见 [architecture.rst，第 28 行](/Users/chl/chl-code/Job/aibrix/docs/source/designs/architecture.rst:28)。但在当前检查到的控制器注册和相关实现中，没有找到与该名称对应的完整独立诊断控制器；控制器注册列表见 `Initialize`：[controller.go，第 51–99 行](/Users/chl/chl-code/Job/aibrix/pkg/controller/controller.go:51)。

尤其不能把 `/healthz` 当成 GPU 健康证明：当前 `liveness_check` 直接返回正常，而 `readiness_check` 检查引擎就绪状态，见 [app.py，第 257–268 行](/Users/chl/chl-code/Job/aibrix/python/aibrix/aibrix/app.py:257)。Ray E2E 测试里的 `logDiagnostics` 则收集 Fleet、ReplicaSet、集群、Pod 和事件信息，见 [diagnostics_test.go，第 72–99 行](/Users/chl/chl-code/Job/aibrix/test/e2e/controller/raycluster/diagnostics_test.go:72)。

因此，现有这些功能可以辅助故障发现与排查，但不足以证明图中的完整 GPU 自动诊断、自愈能力已经全部实现。

**请求路由器决定“这个请求现在交给谁”。**

路由器面对的不是一组完全等价的 HTTP 后端。不同实例可能：

- 加载不同的 LoRA；
- 拥有不同的 GPU 性能；
- 有不同的排队数量和运行中请求；
- 有不同的 KV cache 占用；
- 已经计算过不同的 prompt 前缀。

因此，路由器需要先确定**哪些实例有资格处理请求**，再选择**哪个实例更合适**。

当前 `Server.HandleRequestBody` 会解析模型、查询可用实例、解析路由配置，并选择目标 Pod，见 [gateway_req_body.go，第 77–123 行](/Users/chl/chl-code/Job/aibrix/pkg/plugins/gateway/gateway_req_body.go:77)及 [第 180–205 行](/Users/chl/chl-code/Job/aibrix/pkg/plugins/gateway/gateway_req_body.go:180)。

图中的几类能力应分别理解：

| 能力 | 解决的问题 | 不能直接等同于 |
|---|---|---|
| RPM | 一分钟允许多少次请求 | 实际计算量限制 |
| TPM | 一分钟允许多少 token 用量 | 请求次数限制 |
| 公平策略 | 防止少数用户长期占据过多服务能力 | 每个用户完全相同的响应时间 |
| 负载感知 | 避免请求持续堆积在繁忙实例上 | 简单平均分配请求 |
| 前缀感知 | 尽量利用已计算过的 prompt 前缀 | 返回缓存的完整回答 |
| 工作负载隔离 | 按策略约束候选实例、配额和并发 | 自动提供硬件级隔离 |

RPM/TPM 检查由 `Server.checkLimits`、`checkRPM`、`checkTPM` 实现，超限可返回 HTTP 429，见 [gateway_ratelimit.go，第 32–113 行](/Users/chl/chl-code/Job/aibrix/pkg/plugins/gateway/gateway_ratelimit.go:32)。

TPM 还有一个实现细节：响应处理会根据 `usage.TotalTokens` 回记 token 用量，见 [gateway_rsp_body.go，第 300–305 行](/Users/chl/chl-code/Job/aibrix/pkg/plugins/gateway/gateway_rsp_body.go:300)。所以不能将其理解成“对所有并发请求未来可能生成的 token，都已经严格预扣”。

公平性方面，`BasicVTCRouter.Route` 使用用户 token 记录、输入输出 token 估计和利用率相关参数参与决策，见 [vtc_basic.go，第 128–174 行](/Users/chl/chl-code/Job/aibrix/pkg/plugins/gateway/algorithms/vtc/vtc_basic.go:128)。默认构造使用内存中的滑动窗口 tracker，见 [vtc_router.go，第 63–68 行](/Users/chl/chl-code/Job/aibrix/pkg/plugins/gateway/algorithms/vtc/vtc_router.go:63)。这是一种具体的公平策略，不能推导为所有路由算法天然具有严格的全局公平保证。

前缀感知路由可以用这个例子理解：两个请求都带有同一段很长的系统提示词，某个 Pod 已经算过这段前缀，把后续请求送过去就可能节省计算。但如果该 Pod 太忙，等待时间可能抵消缓存收益。

当前 `prefixCacheRouter.routeOriginal` 同时查看前缀匹配与请求数；没有合适的匹配 Pod 时，会回退到请求数较少的 Pod。见 [prefix_cache.go，第 399–454 行](/Users/chl/chl-code/Job/aibrix/pkg/plugins/gateway/algorithms/prefix_cache.go:399)。

图里的 `resource oversell` 可以理解为通过共享资源提高整体利用率的设计目标；它不会增加 GPU 的物理容量，也不能仅凭该标签认定当前具有完整的资源超卖保证机制。

**分布式 KV Cache Runtime 复用的是推理中间结果。**

LLM 推理可以粗略分成：

- **Prefill**：处理输入 token，建立上下文对应的 Key/Value 张量。
- **Decode**：逐步生成输出 token，利用已有 KV，避免反复计算先前上下文。

因此，KV cache 既不是模型权重，也不是完整答案缓存。它与具体模型和输入上下文相关。

例如，很多请求都携带同一份长文档作为前缀：

- 没有可复用缓存时，每个请求都可能重新计算这部分输入。
- 有匹配 KV 时，可以复用已完成的前缀计算，再处理不同的后续输入。
- 主要收益通常体现在减少重复 prefill、降低首 token 延迟，并释放计算容量。

仓库对这个问题和缓存层次的说明见 [kvcache-offloading.rst，第 25–45 行](/Users/chl/chl-code/Job/aibrix/docs/source/features/kvcache-offloading.rst:25)。

```text
                         Engine Pod A
                    +--------------------+
                    | GPU KV cache       |
                    |        |           |
                    | Offload connector  |
                    |        |           |
                    | L1: host DRAM      |
                    +--------|-----------+
                             |
                        KV block put/get
                             |
                             v
                   +---------------------+
                   | L2 distributed cache|
                   | Worker 1 ... Worker N|
                   +---------------------+
                             ^
                             |
                        KV block put/get
                             |
                         Engine Pod B

                   Meta service
                        |
                        +--> cache cluster membership
```

这张图中：

- **GPU KV cache** 是引擎执行计算时使用的缓存。
- **L1** 是引擎进程管理的主机内存缓存，可以减少远程访问。
- **L2** 是多个引擎共享的外部缓存，让一个实例计算的 KV 有机会被另一个实例复用。
- **Connector** 对接引擎与缓存系统，执行缓存查询、读写和数据搬运。
- **Meta service** 提供缓存集群成员等信息；它与前面“模型名称对应哪些推理实例”的模型元数据职责不同。

L1、L2 可以分别启用，也可以组合使用；具体后端决定部署和通信方式。依据：[kvcache-offloading.rst，第 31–78 行](/Users/chl/chl-code/Job/aibrix/docs/source/features/kvcache-offloading.rst:31)。

代码上，`BaseKVCacheManager._acquire_impl` 包含 L1 查询和后续 L2 获取逻辑，同时存在零拷贝等特殊路径，因此并非所有配置都机械地先 L1 再 L2。见 [cache_manager.py，第 1237–1305 行](/Users/chl/chl-code/Job/aibrix/python/aibrix_kvcache/aibrix_kvcache/cache_manager.py:1237)。Redis 元数据实现的 `get_cluster_metadata` 见 [redis_meta_service.py，第 64–76 行](/Users/chl/chl-code/Job/aibrix/python/aibrix_kvcache/aibrix_kvcache/meta_service/redis_meta_service.py:64)。

**前缀路由与分布式缓存是互补关系：前者把请求送到缓存附近，后者把缓存提供给需要它的实例。**

| 功能 | 交换的主要信息 | 作用 |
|---|---|---|
| KV 事件同步 | 哪个实例存有、移除了哪些缓存块 | 帮助网关维护缓存位置索引 |
| 前缀感知路由 | 请求前缀与实例状态 | 选择更合适的实例 |
| KV offloading / 分布式缓存 | 实际 KV 张量 | 保存、搬运并复用计算结果 |

仓库明确区分了“公布缓存位置”与“搬运缓存内容”，见 [kvcache-offloading.rst，第 43–45 行](/Users/chl/chl-code/Job/aibrix/docs/source/features/kvcache-offloading.rst:43)。

分布式缓存也有成本。读取远程 KV 需要网络和内存搬运，收益取决于“省下的重算时间”是否大于“取回缓存的时间”。仓库因此提供选择性 offloading，避免无差别传输所有 KV，见 [aibrix_kvcache/README.md，第 8–11 行](/Users/chl/chl-code/Job/aibrix/python/aibrix_kvcache/README.md:8)。跨节点复用也不意味着不同模型、不同适配器或任意张量布局都能直接互用。

**你图中的 Cold Start Manager 和 Pod Deletion Controller，需要结合当前功能理解。**

冷启动管理的目标是缩短“没有可用模型实例”到“能够提供服务”之间的时间，包括准备文件、启动引擎、加载模型和等待就绪。

当前仓库存在相关的实验性 `ModelClaim` 功能：控制器选择预热的 GPU Pod，由 Runtime 启动独立引擎进程，在就绪后发布对应端口；网关还能触发睡眠模型唤醒。它有特定限制，不应直接当作图中通用 Cold Start Manager 的完整实现。见 [modelclaim.rst，第 9–25 行](/Users/chl/chl-code/Job/aibrix/docs/source/features/modelclaim.rst:9)及 [第 45–56 行](/Users/chl/chl-code/Job/aibrix/docs/source/features/modelclaim.rst:45)。

尤其是睡眠模型的请求处理：当前测试明确检查了**触发唤醒并返回可重试的 503**，而不是一直保留原请求等待加载完成。见 `TestValidateModelAvailabilityReturnsRetryableResponseForSleepingModelClaim`：[gateway_req_body_test.go，第 745–765 行](/Users/chl/chl-code/Job/aibrix/pkg/plugins/gateway/gateway_req_body_test.go:745)。

Pod 删除管理则涉及缩容与发布时的退出过程。当前 `drain.processPod` 会启动排空状态，并在配置的超时时间到达后删除 Pod；这段代码不能被解释为已经精确确认所有推理请求完成。见 [drain.go，第 132–171 行](/Users/chl/chl-code/Job/aibrix/pkg/controller/drain/drain.go:132)。

**最后，用图中的一次 LoRA2 请求串起这些组件。**

假设请求目标是已经注册的 LoRA2，且图中的 Pod 1、Pod 2 都已成功加载它：

1. **客户端提交请求。** `api_base` 指向请求入口，`model` 指定目标模型或适配器，`api_key` 用于调用侧认证。OpenAI compatible 描述接口兼容性，不表示计算必须由 OpenAI 执行。

2. **Envoy 接收请求，通过 `ext_proc` 让 AIBrix 插件处理相关信息。** 插件解析请求中的模型并查询可用实例。入口见 `Server.HandleRequestBody`：[gateway_req_body.go，第 43–97 行](/Users/chl/chl-code/Job/aibrix/pkg/plugins/gateway/gateway_req_body.go:43)。

3. **按配置执行准入与限流。** 超出用户配额或模型限制时，可以拒绝请求，不必让 GPU 承担无效排队。用户限流见 `Server.checkLimits`：[gateway_ratelimit.go，第 32–78 行](/Users/chl/chl-code/Job/aibrix/pkg/plugins/gateway/gateway_ratelimit.go:32)。

4. **从能服务 LoRA2 的候选实例中选择目标。** 根据你的图，候选是 Pod 1 和 Pod 2。前缀缓存、运行负载等信息可以影响选择；Pod 3 仅加载 LoRA1，不能因为空闲就接收 LoRA2 请求。

5. **插件返回目标地址，Envoy 转发到所选引擎。** 目标头的构造见 [gateway_req_body.go，第 201–205 行](/Users/chl/chl-code/Job/aibrix/pkg/plugins/gateway/gateway_req_body.go:201)。

6. **引擎使用基础模型与 LoRA2 执行推理。** 配置了 KV 复用时，先利用可用的兼容缓存，再计算未命中的部分，随后生成输出 token。这里执行模型计算的是引擎，Runtime 负责的是管理接口。

7. **响应返回，同时更新用量和运行状态。** token 用量回记可以用于后续 TPM 判断；新的负载指标也会影响后续请求的路由和扩缩决策。用量处理见 [gateway_rsp_body.go，第 300–305 行](/Users/chl/chl-code/Job/aibrix/pkg/plugins/gateway/gateway_rsp_body.go:300)。

8. **持续高负载触发控制平面的后续调整。** 自动扩缩器提高基础工作负载的副本数；新 Pod 就绪后，LoRA 控制器按配置协调适配器加载，更新服务发现，路由器才能把后续 LoRA2 请求分配过去。扩缩执行见 [podautoscaler_controller.go，第 905–908 行](/Users/chl/chl-code/Job/aibrix/pkg/controller/podautoscaler/podautoscaler_controller.go:905)；适配器协调见 [modeladapter_controller.go，第 447–495 行](/Users/chl/chl-code/Job/aibrix/pkg/controller/modeladapter/modeladapter_controller.go:447)。


# claude

我先说明范围和成功标准。这次是只读的"全面解释"任务，不改任何代码。

- **范围**：对照你贴的架构图，把 AIBrix 的控制平面和数据平面逐个组件讲清楚。
- **依据**：所有结论都以当前仓库代码为准，不凭文件名推断。
- **格式**：遵守你的两条规则。代码和引用都标注文件、类或函数和行号；图只画 ASCII 图，并附详细解释。

我先扫描仓库结构，再逐个组件读代码定位行号。

仓库内容比架构图丰富，涉及 `stormservice`、`roleset`、`modelclaim`、`drain`、`modelrouter`、`statesync` 等模块。我先从控制器注册入口读起，确认控制平面实际注册的控制器有哪些。

控制器注册入口已经拿到。接下来读 feature gate 的定义，弄清每个控制器默认是否开启，再读 `cmd/controllers/main.go` 的启动流程。

控制器全景清楚了。现在逐个组件深入，先看 **ModelAdapter（LoRA）控制器**，它对应图里的 "Lora1/2/3 + Custom Endpoint Slice"。

读 ModelAdapter 控制器的核心流程：注册与 watch 设置、`Reconcile` 主链路、加载逻辑，以及 Service 和 EndpointSlice 的创建。

关键发现：图中的"Customized Endpoint Slice Controller"并非独立控制器，而是 ModelAdapter 控制器内的一个步骤。接下来我会查看 `lora_client.go`，了解 LoRA 实际是如何加载进引擎 Pod 的，以及它与运行时 sidecar 的关系。

继续看 LoRA 调度策略（`scheduling/`）和 `BuildURLs`（控制面怎样决定直连引擎还是走 sidecar）。

ModelAdapter 部分够了。下一个是 **LLM 专用自动扩缩（PodAutoscaler）**，也就是图里的 "LLM Specific Autoscaler"。

读 PodAutoscaler 的注册与 `Reconcile` 主流程，再看三种算法（HPA/KPA/APA）。

再读自动扩缩的核心流水线：`reconcileCustomPA`、`computeScaleDecision`、`ComputeDesiredReplicas`，以及指标采集（KV cache 等推理感知指标从哪来）。

自动扩缩讲清楚了。下一个是 **RayClusterFleet（多节点分布式推理）**。

RayClusterFleet 确认为 Deployment→ReplicaSet→RayCluster 的三层结构，接下来查看 AI 引擎运行时（Runtime sidecar），即 Pod 内的 Model loader、Runtime Agent 和 Watch Dog。

我注意到上条命令的 `cd` 把工作目录切到了子目录，按 AGENTS.md 要求需回到仓库根目录，接下来的命令都会以根目录为基准。现在我先切回根目录，同时查看运行时 sidecar 的入口 `app.py` 及其 API 层代码。

运行时 sidecar 的接口已经清楚，其中 `/v1/runtime/models/activate|sleep|wake` 是图里没有的新接口，服务于 `ModelClaim` 控制器。接下来我会看下载/模型加载部分，并核实图中的 "Watch Dog"、"Cold Start Manager"、"Pod Deletion Controller" 在代码里是否有对应实现。

我正在逐一核实架构图中的组件是否在代码中真实存在，目前发现 "Watch Dog" 和 "Cold Start Manager" 几乎没有独立实现。接下来我会核对这些零星命中，并查找 "Pod Deletion Controller" 对应的具体实现。

"Watch Dog"和"Cold Start Manager"在代码中没有对应的命名组件，最接近的实现是`ModelClaim`及运行时的activate/wake接口，我继续深入阅读；"Pod Deletion Controller"则需要查看`drain/`包和pod-deletion-cost注解。

读 `drain` 包，它最可能对应 "Pod Deletion" 语义。同时看 `modelrouter` 控制器，它最可能对应图里的 "Model Metadata Controller"。

`drain` 是一个库而不是独立控制器，被 roleset/stormservice 调用。我确认网关怎么感知 draining Pod，并读 `modelrouter` 的实现。

`modelrouter` 已确认：它监听 Deployment / ModelAdapter / RayClusterFleet，为每个模型生成 Gateway API 的 `HTTPRoute`。接下来看数据平面，先查网关如何过滤 draining Pod，再阅读网关主体 `pkg/plugins/gateway`。

网关是数据平面的核心。读 `Server` 结构、ext_proc 处理循环 `Process` / `handleProcessingRequest`，以及路由入口 `selectTargetPod`。

网关是 Envoy ext_proc 的 gRPC 服务，分四个阶段处理。逐阶段读：请求头（鉴权 / 限流）、请求体（路由选 Pod）、`selectTargetPod`。

我找到了图中"Cold Start Manager"对应的真实实现：ModelClaim 处于 Sleeping 状态时，网关会调用 `RequestWake`（gateway_req_body.go:319-321）。接下来我会去读唤醒逻辑和限流实现的代码。

请求侧读完，再看响应侧怎样统计 token 用量并回写 TPM，以及路由算法的目录和注册表。

读路由器选择和多策略混合（评分归一化）的核心，再挑代表性算法：前缀缓存感知、负载感知。

读前缀缓存感知路由（图里 "prefix-cache-aware"）和负载感知路由的实现。

数据平面的 "Model Metadata Store" 在代码里就是 `pkg/cache`：它把 Pod / ModelAdapter 的 informer 数据聚合成 model→pods 映射，并周期抓取引擎指标。读它的结构。

pkg/cache 部分已看清楚，图中的 "Model Metadata Store" 在 Go 侧其实就是每个网关进程内的内存 Registry，并非独立存储。接下来查看 Python 那边的 metadata 服务（另一套 FastAPI）具体提供什么功能。

网关、路由、缓存这部分数据平面代码已经看完，接下来我会检查分布式 KV 缓存（Go 控制器+Python 运行时）、GPU 优化器，并确认仓库里是否存在你提到的"加速器诊断工具"。

"diagnos" 的搜索结果大多是无关日志。我将在 `docs/`、`development/`、`samples/` 中精确搜索"加速器诊断/故障模拟"相关实现，确认该功能在仓库中的实现状态。

仓库里有官方架构文档 `docs/source/designs/architecture.rst`。读它，并找出"加速器诊断"在文档里是怎么描述的。

找到了，`architecture.rst:16-37` 正是英文原文出处，"Accelerator Diagnose Tools" 仅在第28行出现。接下来核对代码中含 "diagnos" 的匹配项，确认是否确实对应这个工具的实现。

确认`Accelerator Diagnose Tools`在代码树中没有独立实现，仅有文档提及和一个开发用mock应用`development/app`。接下来查看分布式KV Cache运行时，先从Go侧的KVCache控制器和CRD入手。

Go 侧的 KVCache 编排清楚了。再看真正的缓存数据面：Python `aibrix_kvcache` 的分层结构（L1 / L2、元数据服务、与 vLLM 的集成）。

接下来查看 `pkg/kvevent`，了解 vLLM 如何向网关实时同步 KV block 事件以支持前缀感知路由。

KV 事件同步清楚了：网关经 ZMQ 订阅 vLLM Pod 的 `BlockStored/Removed/AllBlocksCleared` 事件，维护前缀索引。下一个是 **GPU 优化器**。

验证一个细节：样例里 `targetValue: "100"` 对推荐副本数意味着什么。我读官方文档的说法，避免误解释。

GPU 优化器到 PodAutoscaler 的链路已验证。接着读 **sidecar 注入与 Model loader（artifact 下载）**：`pkg/webhook` 里的注入逻辑，以及运行时的 `artifact_service.py`。

网关的 Envoy 接入方式清楚了。还要确认 Envoy 怎样把请求真正转发到 ext_proc 选出来的那个 Pod，也就是 `target-pod` 头是怎么被消费的。

再读 **ModelClaim**（对应图里最接近 "Cold Start Manager / 模型加载" 的机制）的 CRD 与控制器。

再读数据平面的 **PD 分离（Prefill / Decode）路由**，它是网关里最复杂的算法，也是最近提交的重点。

补几个关键核对点：可路由 Pod 的判据、指标抓取（推理感知指标从哪来），以及 "Topology Information" 在代码中是否存在。

再读自动扩缩的执行流水线 `executeScalingPipeline`，核对官方文档说的"秒级扩缩"在代码里到底对应什么。

补最后两个引用：官方样例里自动扩缩具体选用了哪些"推理感知指标"，以及 LoRA 样例的 YAML，便于我用真实配置举例。

我发现样例注释里提到的调度器名称（"default"、"least-adapters"）与代码中实际的策略名（如 `leastAdapters`）不一致，且实际配置由控制器启动参数决定。我需要进一步核对 `spec.schedulerName` 是否真的被使用，避免依据过时注释下结论。

写答案前再核对两个数字：指标窗口默认值（决定"秒级扩缩"这句话该怎么表述），以及记忆目录里是否已有相关条目。

记忆目录目前为空，我将把你提到的两条长期偏好（引用需带文件/类和行号，画图只用ASCII并附解释）记录下来，方便后续自动遵循。

落笔前再核对一件事：图中 "Fairness" 是否对应网关里的 `vtc` 路由，我读它的说明确认，避免凭名字下结论。

我先把所有 ASCII 图写进草稿文件，检查列宽和是否含非 ASCII 字符，保证在等宽字体里对齐后再放进回答。

图表对齐检查通过：全部是纯 ASCII，方框边界对齐。D4 / D5 / D6 的行偏宽，我改成"步骤字母加图后列表"的写法。最后补几个需要行号的文档引用。

图和引用都核对齐了，现在把最终版图定稿（D4 / D5 / D6 改为"字母标注 + 图后列表"，D3 的状态行改成不武断的写法），再做一次宽度和 ASCII 检查。

D1 里有一处箭头方向画反了（控制器调用 Pod 的 HTTP 箭头应该向上）。我修正后再验一次对齐，然后组织最终回答。

# AIBrix 全面解读（控制平面 + 数据平面）

**依据与约定**
- 这是对仓库 `main`（会话开始时 HEAD 为 `3deb9b2c`）的**静态阅读**，没有起集群，也没有跑测试。
- 官方文档 [architecture.rst:16-37](docs/source/designs/architecture.rst:16) 就是你贴的那段文字的英文原文。
- 文中每个 `文件:行号` 都是我读过的真实位置。图只用 ASCII，图中文件名省略了目录，完整路径见图后说明。

## 0. 先看结论

1. **进程构成**：
    - 一个 Go 控制器进程，入口 [main.go:113](cmd/controllers/main.go:113)。
    - 一个 Go 网关插件，是 Envoy ext_proc 的 gRPC 服务，默认 `:50052`，见 [main.go:110](cmd/plugins/main.go:110)。
    - 若干 Python 服务：runtime sidecar、gpu-optimizer、metadata-service。
    - 一个 Python KV 缓存库 `aibrix_kvcache`。
2. **控制器按开关注册**：
    - `--controllers` 默认 `*`，见 [main.go:152](cmd/controllers/main.go:152)。
    - 顶层开关共 7 个，见 [features.go:24-33](pkg/features/features.go:24)，注册逻辑在 [controller.go:51-100](pkg/controller/controller.go:51)。
    - RayCluster 相关控制器还要求集群已装 KubeRay CRD，见 [controller.go:68-87](pkg/controller/controller.go:68)。
3. **图和代码有实质出入**（§5 有详表）：
    - 图里的 Model Metadata Store，在代码里是网关进程内的内存缓存。
    - Custom EndpointSlice Controller 只是 ModelAdapter 控制器的一个步骤。
    - Watch Dog、Cold Start Manager、Pod Deletion Controller、Topology Information 没有同名实现。
4. 文档列出的 **Accelerator Diagnose Tools 只有文档，没有实现**，见 [architecture.rst:28](docs/source/designs/architecture.rst:28)。
5. 代码里有、图和文档清单都没写的：ModelClaim、StormService/RoleSet/PodSet、KVCache 控制器、ModelRoute、drain、metadata-service（§4.7）。

## 1. 组件地图

```text
+----------------------------------- DATA PLANE -----------------------------------+
|                                                                                  |
|  [R] Request Router                        [P] Serving pod                       |
|      Envoy Gateway + gateway-plugins           engine (vLLM/SGLang/..)    :8000  |
|      (Go, ext_proc gRPC :50052)                aibrix-runtime sidecar     :8080  |
|      auth, RPM/TPM, routing, PD,               base model + LoRA adapters        |
|      token accounting                                ^                |          |
|            |                                         | scrape         | KV       |
|            | reads                                   | /metrics       | blocks   |
|            v                                         |                v          |
|  [S] pkg/cache "Store"  -----------------------------+     [K] Distributed KV    |
|      model -> pods map, engine metrics,                       cache runtime      |
|      prefix index                                             (aibrix_kvcache)   |
|      (= "Model Metadata Store")                               L1 DRAM + L2       |
+----------------------------------------------------------------------------------+
     ^ [1] informers feed [S]                 ^ [2] controllers call pods over HTTP
     |     (Pod / ModelAdapter / ModelClaim)  |     (:8080 runtime, :8000 engine)
+--------------------------------- CONTROL PLANE ----------------------------------+
|  aibrix-controller-manager (Go, cmd/controllers)      Python services            |
|    ModelAdapter (LoRA)      PodAutoscaler KPA/APA/HPA   gpu-optimizer            |
|    RayClusterFleet / RS     KVCache                     metadata-service :8090   |
|    ModelRoute (HTTPRoute)   ModelClaim                  aibrix-runtime (agent)   |
|    StormService > RoleSet > PodSet      + admission webhooks                     |
+----------------------------------------------------------------------------------+
```

**图解**
- `[R]` 是 Envoy Gateway 加 `gateway-plugins`。Envoy 只负责收发流量，决策逻辑在 Go 插件里（§2、§3.1）。
- `[S]` 是网关进程内的内存缓存，保存 model→pods 映射、引擎指标和前缀索引，即图里的 "Model Metadata Store"（§3.2）。
- `[P]` 是每个模型 Pod：推理引擎 `:8000`，加可选的 `aibrix-runtime` sidecar `:8080`。sidecar 由 webhook 注入（§4.5）。
- `[K]` 是分布式 KV 缓存运行时（§3.3）。
- 箭头 `[1]`：控制面写 K8s 对象状态，例如 `ModelAdapter.status.instances` 和 Pod 注解。`[S]` 通过 informer 读取并据此路由，见 [cache_init.go:507-547](pkg/cache/cache_init.go:507) 和 [informers.go:287-299](pkg/cache/informers.go:287)。
- 箭头 `[2]`：控制器直接用 HTTP 调 Pod。例如加载 LoRA，或让 ModelClaim 激活模型。

## 2. 一个请求的完整生命周期

```text
 client
   |  POST /v1/chat/completions   {"model": "m", "messages": [...]}
   v
 Envoy Gateway   HTTPRoute "reserved-router" -> ext_proc filter
   |             (request body Buffered, response body Streamed)     gateway-plugin.yaml:184-247
   |
   +-[1] RequestHeaders --> gateway-plugins: HandleRequestHeaders     gateway_req_headers.go:61
   |                          a. API-key check (optional)             gateway_req_headers.go:104-123
   |                          b. user RPM/TPM check (needs Redis)     gateway_req_headers.go:125
   |                                                                    -> gateway_ratelimit.go:32
   |
   +-[2] RequestBody ----> gateway-plugins: HandleRequestBody         gateway_req_body.go:43
   |                          a. parse body -> model/prompt/stream    gateway_req_body.go:61-90
   |                          b. model known, has routable pods?      gateway_req_body.go:94 -> :314
   |                             (ModelClaim Sleeping -> wake + 503)  gateway_req_body.go:319-322
   |                          c. resolve routing strategy             gateway_req_body.go:110-124
   |                          d. per-model RPS limit                  gateway_req_body.go:157
   |                          e. no strategy  -> plain HTTPRoute path gateway_req_body.go:167-179
   |                             strategy set -> selectTargetPod      gateway_req_body.go:181-182
   |                                              -> gateway.go:707-799
   |                          f. reply headers: routing-strategy,     gateway_req_body.go:201-205
   |                             target-pod=<ip:port>
   v
 Envoy   header routing-strategy matches "original_route"             gateway.yaml:112-128
   |     -> cluster ORIGINAL_DST, destination = header "target-pod"   gateway.yaml:198-202
   v
 model Pod (engine)
   |   JSON / SSE response
   +-[3] ResponseHeaders/Body --> HandleResponseBody                  gateway_rsp_body.go:252
   |                          token usage -> Redis <user>_TPM_CURRENT gateway_rsp_body.go:300-318
   v
 client
```

`gateway_*.go` 在 `pkg/plugins/gateway/`。`gateway-plugin.yaml` 在 `config/gateway/gateway-plugin/`。`gateway.yaml` 在 `config/gateway/`。

**要点**
1. **Envoy 侧只是"哑路由"**：
    - `reserved-router` 匹配 OpenAI 风格路径，后端指到 `aibrix-gateway-plugins:50052`。文件注释直接称它是 dummy route，见 [gateway-plugin.yaml:180-223](config/gateway/gateway-plugin/gateway-plugin.yaml:180)。
    - 真正的决策靠 EnvoyExtensionPolicy 挂上的 ext_proc：请求体 `Buffered`，因为要读 `model` 和 prompt；响应体 `Streamed`，因为 SSE 要边到边发；超时 600s，见 [gateway-plugin.yaml:235-247](config/gateway/gateway-plugin/gateway-plugin.yaml:235)。
2. **头阶段做准入**：
    - 用户身份来自 `user` 请求头，见 [gateway_req_headers.go:41](pkg/plugins/gateway/gateway_req_headers.go:41)、[:74-75](pkg/plugins/gateway/gateway_req_headers.go:74)。
    - 限额状态从 Redis 读，见 [gateway_req_headers.go:226](pkg/plugins/gateway/gateway_req_headers.go:226)。
3. **体阶段做路由**：
    - 校验模型是否存在且有可路由 Pod，见 [gateway_req_body.go:314-352](pkg/plugins/gateway/gateway_req_body.go:314)。
    - 从 Pod 注解解析模型配置 profile，见 [:107-108](pkg/plugins/gateway/gateway_req_body.go:107)。
    - 策略优先级是"请求头 → profile → 环境变量"，见 [:110](pkg/plugins/gateway/gateway_req_body.go:110)。
4. **有两条转发路径**：
    - (A) 没指定路由策略：网关只写 `model` 头（[gateway_req_body.go:167-179](pkg/plugins/gateway/gateway_req_body.go:167)）。转发交给 ModelRoute 控制器建的 HTTPRoute（按 `model` 头精确匹配，后端是 Service，见 [modelrouter_controller.go:260-277](pkg/controller/modelrouter/modelrouter_controller.go:260)）。
    - (B) 指定了策略：网关自己选 Pod，写 `target-pod`，Envoy 用 ORIGINAL_DST 直连该 Pod。
5. **响应阶段计费**：从响应里取 token 用量，累加到 Redis 的 TPM 计数，见 [gateway_rsp_body.go:300-318](pkg/plugins/gateway/gateway_rsp_body.go:300)。

```go
// pkg/plugins/gateway/gateway.go — (*Server).selectTargetPod，节选自 L715-799
715  readyPods := utils.FilterRoutablePods(pods.All())
733  readyPods, err = utils.FilterPodsByLabelSelector(readyPods, externalFilterExpr)
743  readyPods = s.filterSaturatedReplicaInflight(readyPods, limit)
768  readyPods = routing.ApplyLoadImbalanceGate(routeCtx, s.cache, readyPods)
775  router, err := s.routerManager.Select(routeCtx)
799  return router.Route(routeCtx, &utils.PodArray{Pods: readyPods})
```

```yaml
# config/gateway/gateway.yaml L198-204（EnvoyPatchPolicy 新增的 cluster）
        name: original_destination_cluster
        type: ORIGINAL_DST
        original_dst_lb_config:
          use_http_header: true
          http_header_name: "target-pod"
        connect_timeout: 6s
        lb_policy: CLUSTER_PROVIDED
```

- "可路由 Pod"的定义是 [FilterReadyPod](pkg/utils/pod.go:203)：有 IP、未终止、**未 draining**、Ready。所以带 `aibrix.ai/draining` 注解的 Pod 会在被删除前先退出路由（[pod.go:83-85](pkg/utils/pod.go:83)）。
- 匹配 `routing-strategy` 头的 `original_route` 定义在 [gateway.yaml:112-128](config/gateway/gateway.yaml:112)。

## 3. 数据平面

### 3.1 Request Router（图中的 "API Gateway and Proxy"）

**a. 准入：RPM / TPM / RPS，以及"公平"**
- 用户的 RPM/TPM 存在 Redis，由 metadata-service 的 `/CreateUser` 写入，见 [users.py:84-109](python/aibrix/aibrix/metadata/api/v1/users.py:84)。未配置时默认 RPM=100，TPM=RPM×1000，见 [types.go:94-95](pkg/plugins/gateway/types.go:94) 和 [gateway_ratelimit.go:33-38](pkg/plugins/gateway/gateway_ratelimit.go:33)。
- 限流器是 Redis 固定窗口。用户维度窗口 1 分钟，模型维度（RPS）窗口 1 秒，见 [gateway.go:281-287](pkg/plugins/gateway/gateway.go:281) 和 [ratelimiter/README.md:3](pkg/plugins/gateway/ratelimiter/README.md:3)。没有 Redis，或设了 `AIBRIX_DISABLE_RATE_LIMITING`，就退化成 no-op，见 [main.go:216](cmd/plugins/main.go:216)。
- 检查顺序是 RPM、计数加一、TPM，见 [gateway_ratelimit.go:40-64](pkg/plugins/gateway/gateway_ratelimit.go:40)。
- **TPM 是事后计费**：放行时只比较已累计值（[:103-114](pkg/plugins/gateway/gateway_ratelimit.go:103)），token 用量到响应结束才累加（[gateway_rsp_body.go:305](pkg/plugins/gateway/gateway_rsp_body.go:305)）。由此可知，单个大请求可以略微越过上限。
- 图里的 "Fairness" 对应 `vtc-basic`（Virtual Token Counter）：按滑动窗口内每个客户已获得的 token 服务量打分，与 Pod 利用率分数合并，取最低，见 [vtc_readme.md:9-16](pkg/plugins/gateway/algorithms/vtc_readme.md:9)、[vtc_router.go:26](pkg/plugins/gateway/algorithms/vtc/vtc_router.go:26)。图里的 "resource oversell"（超卖），我没找到对应实现。

**b. 路由策略**（`routing-strategy` 的取值，均在 `pkg/plugins/gateway/algorithms/`）

| 类别 | 策略 |
|---|---|
| 负载感知 | `least-request` [:35](pkg/plugins/gateway/algorithms/least_request.go:35)、`least-kv-cache` [:27](pkg/plugins/gateway/algorithms/least_kv_cache.go:27)、`least-gpu-cache` [:26](pkg/plugins/gateway/algorithms/least_gpu_cache.go:26)、`least-busy-time` [:26](pkg/plugins/gateway/algorithms/least_busy_time.go:26)、`least-latency` [:25](pkg/plugins/gateway/algorithms/least_latency.go:25)、`least-utilization` [:26](pkg/plugins/gateway/algorithms/least_util.go:26)、`throughput` [:32](pkg/plugins/gateway/algorithms/throughput.go:32)、`power-of-two` [:32](pkg/plugins/gateway/algorithms/power_of_two.go:32)、`load-balance` [:34](pkg/plugins/gateway/algorithms/load_balance.go:34)、`random` [:27](pkg/plugins/gateway/algorithms/random.go:27) |
| 缓存感知 | `prefix-cache` [:55](pkg/plugins/gateway/algorithms/prefix_cache.go:55)、`prefix-cache-preble` [:34](pkg/plugins/gateway/algorithms/prefix_cache_preble.go:34) |
| 会话 / 公平 | `session-affinity` [:41](pkg/plugins/gateway/algorithms/simple_session_affinity.go:41)、`vtc-basic` [:26](pkg/plugins/gateway/algorithms/vtc/vtc_router.go:26) |
| SLO | `slo`、`slo-pack-load`、`slo-least-load`、`slo-least-load-pulling`，见 [slo.go:26-35](pkg/plugins/gateway/algorithms/slo.go:26) |
| PD 分离 | `pd`，见 [pd_disaggregation.go:46](pkg/plugins/gateway/algorithms/pd_disaggregation.go:46) |

**c. 多策略混合**（`RouterManager`）
- 写法是 `prefix-cache:2,least-latency:1`，权重范围 0 到 1000000，权重 0 表示跳过，见 [router.go:69-74](pkg/plugins/gateway/algorithms/router.go:69)、[:110-118](pkg/plugins/gateway/algorithms/router.go:110)。
- 各策略先各自打分，按"越大越好 / 越小越好"的极性归一化到 [0,1]，再加权求和取最高；同分按 Pod 名排序，保证结果稳定，见 [router.go:481-492](pkg/plugins/gateway/algorithms/router.go:481)、[:536-538](pkg/plugins/gateway/algorithms/router.go:536)。
- `pd` 和 `slo*` 是"排他"策略，混合时其它策略会被忽略，见 [router.go:126-134](pkg/plugins/gateway/algorithms/router.go:126)。
- 网关会**悄悄**把 `load-balance`（必要时加 `least-request`）混入你选的策略，见 [router.go:809-825](pkg/plugins/gateway/algorithms/router.go:809)。响应头和错误信息仍显示你选的原策略名。所以选了 `prefix-cache`，仍然会有负载均衡。

**d. prefix-cache（图里的 "prefix-cache-aware routing"）**
- 流程：
    1. 分词并分块哈希，与各 Pod 的前缀索引匹配（[prefix_cache.go:413-429](pkg/plugins/gateway/algorithms/prefix_cache.go:413)）。
    2. 按"匹配率降序、同率按在途请求数升序"排序，选第一个请求数不超过"均值 + σ·标准差"的 Pod（[:967-1003](pkg/plugins/gateway/algorithms/prefix_cache.go:967)）。
    3. 都不满足则回退到最少请求（[:444-455](pkg/plugins/gateway/algorithms/prefix_cache.go:444)）。
    4. 把这次前缀写回索引（[:462-464](pkg/plugins/gateway/algorithms/prefix_cache.go:462)）。
- 两种模式（[prefix_cache_readme.md:19-24](pkg/plugins/gateway/algorithms/prefix_cache_readme.md:19)）：
    - Standard：路由器自己维护本地哈希表。
    - KV Sync：由 vLLM 的块事件实时喂索引，对应 `kvSyncPrefixCacheRouter`（[prefix_cache.go:187](pkg/plugins/gateway/algorithms/prefix_cache.go:187)、[:778](pkg/plugins/gateway/algorithms/prefix_cache.go:778)）。

```go
// pkg/plugins/gateway/algorithms/prefix_cache.go — getTargetPodFromMatchedPodsFromCounts，L993-1000
993  // select targetpod with highest %prefixmatch and request_count within stddev
994  for _, podname := range podnames {
995  	reqCnt := float64(podRequestCount[podname])
996  	if reqCnt <= meanRequestCount+float64(stdDevFactor)*stdDevRequestCount {
997  		targetPodName = podname
998  		break
999  	}
1000 }
```

**e. PD 分离（Prefill / Decode）**
- Prefill Pod 处理 prompt 并生成 KV，Decode Pod 接手 KV 继续生成 token。路由器每个请求选一个 prefill 加一个 decode，见 [pd_readme.md:14-56](pkg/plugins/gateway/algorithms/pd_readme.md:14)。
- `pdRouter.Route` 的步骤：
    1. 校验引擎和请求体（[pd_disaggregation.go:484-490](pkg/plugins/gateway/algorithms/pd_disaggregation.go:484)）。
    2. 选 prefill/decode（[:495](pkg/plugins/gateway/algorithms/pd_disaggregation.go:495)）。
    3. 发起 prefill 请求，同步或异步取决于引擎（[:528](pkg/plugins/gateway/algorithms/pd_disaggregation.go:528)）。
    4. 把 **decode Pod** 的地址返回给 Envoy（[:545-546](pkg/plugins/gateway/algorithms/pd_disaggregation.go:545)）。
- Pod 用 `roleset-name` 和 `role-name` 标签分类（[pd_readme.md:57-64](pkg/plugins/gateway/algorithms/pd_readme.md:57)）。只有同时含 prefill 和 decode 的 roleset 才合格（[:565](pkg/plugins/gateway/algorithms/pd_readme.md:565)）。
- 还支持按 prompt 长度分桶，以及"combined" Pod 兜底，见 [pd_disaggregation.go:138](pkg/plugins/gateway/algorithms/pd_disaggregation.go:138)、[pd_readme.md:589](pkg/plugins/gateway/algorithms/pd_readme.md:589)。

**f. 多副本网关**
- 每个 Pod 的在途请求数存在 Redis。`GetPodsRunningRequests` 一次往返取回全部候选 Pod 的**跨网关**实时值，见 [least_request.go:66-72](pkg/plugins/gateway/algorithms/least_request.go:66)。
- 设 `AIBRIX_STATESYNC_ENABLED` 后，前缀哈希表还会经 Redis 在副本间增量同步，见 [main.go:220-231](cmd/plugins/main.go:220)。

### 3.2 Model Metadata Store = `pkg/cache`

- **接口**：`Cache` 聚合了 Pod / Model / Metric / RequestTracker / Profile，见 [cache_api.go:26-35](pkg/cache/cache_api.go:26)。
- **发现**：
    - `initDiscoveryProvider` 通过 Provider 的 Watch 接收 Pod / ModelAdapter / ModelClaim 事件，见 [cache_init.go:498-547](pkg/cache/cache_init.go:498)。
    - K8s 模式用 informer，且只有网关额外 watch ModelClaim（[:405-412](pkg/cache/cache_init.go:405)）。
    - standalone 模式用静态 Provider（[main.go:168-169](cmd/plugins/main.go:168)）。
- **登记规则**：
    - Pod 带 `model.aibrix.ai/name` 标签，或带 ModelClaim 注解，才会被登记；worker Pod 会被忽略（[informers.go:113-142](pkg/cache/informers.go:113)）。
    - ModelAdapter 的 `status.instances` 里每个 Pod 被映射到 adapter 名（[:287-299](pkg/cache/informers.go:287)、[:447-471](pkg/cache/informers.go:447)）。
- **指标**：
    - 周期性地把 Ready Pod 入队，worker 抓取 `podIP:metricPort` 上的引擎指标，见 [cache_init.go:475-494](pkg/cache/cache_init.go:475) 和 [cache_metrics.go:329-395](pkg/cache/cache_metrics.go:329)。
    - 指标名跨引擎映射，例如 `gpu_cache_usage_perc` 在 vLLM / SGLang / xLLM 上分别对应不同原名，见 [metrics.go:440-452](pkg/metrics/metrics.go:440)。
- **Redis 的用途**：网关快照、请求 trace、在途请求心跳，见 [cache_init.go:430-441](pkg/cache/cache_init.go:430)。
- **结论**：它不是"另一个存储"，而是每个网关副本自己维护的一份内存视图。图里 "Model Metadata Controller → Registration" 这一段，被拆成两处：`pkg/cache` 的 watch，以及 ModelRoute 建 HTTPRoute（§4.7）。

### 3.3 Distributed KV Cache Runtime

```text
 engine process (vLLM + AIBrix KV connector)                                          (a)
        |  put / get / acquire / exists
        v
 KVCacheManager                                                                       (b)
   |
   +-> L1Cache : DRAM tensor pool, eviction fifo | lru | s3fifo                       (c)
   |      |  hands blocks to L2 when:                                                 (d)
   |      |     ingestion HOT -> on hot access | ALL -> on put | else -> on evict
   |      v
   +-> L2Cache : async workers, key builder, placement policy                         (e)
   |      +-> connector: infinistore | hpkv | priskv | eic | rocksdb | shfs | mock    (f)
   |      +-> MetaService (redis) : cluster membership                                (g)
   v
 Kubernetes objects built by the KVCache controller (Go)                              (h)
   Redis metadata Pod+Service -> cache StatefulSet -> Service -> watcher SA/Role/Pod
```

**图注（均在 `python/aibrix_kvcache/aibrix_kvcache/` 下，除 (h)）**
- (a) 与 vLLM 的集成在 `integration/vllm/kv_connector/`（目录）。
- (b) 管理器是 [`BaseKVCacheManager`](python/aibrix_kvcache/aibrix_kvcache/cache_manager.py:591)，构造函数在 [:598-842](python/aibrix_kvcache/aibrix_kvcache/cache_manager.py:598)。
- (c) L1 见 [:752-764](python/aibrix_kvcache/aibrix_kvcache/cache_manager.py:752)，淘汰策略在 `l1/eviction_policy/`。
- (d) L1 到 L2 的落盘时机由回调决定，见 [:824-836](python/aibrix_kvcache/aibrix_kvcache/cache_manager.py:824)。
- (e) L2 见 [:766-798](python/aibrix_kvcache/aibrix_kvcache/cache_manager.py:766)。
- (f) 连接器实现在 `l2/connectors/`（目录）。
- (g) 元数据服务见 [:691-696](python/aibrix_kvcache/aibrix_kvcache/cache_manager.py:691)。
- (h) Go 侧 [distributed.go:58-99](pkg/controller/kvcache/backends/distributed.go:58) 依次创建 Redis 元数据 Pod+Service（[:63-67](pkg/controller/kvcache/backends/distributed.go:63)）、缓存 StatefulSet（[:70](pkg/controller/kvcache/backends/distributed.go:70)）、Service（[:75](pkg/controller/kvcache/backends/distributed.go:75)）、watcher 的 SA / Role / RoleBinding / Pod（[:79-96](pkg/controller/kvcache/backends/distributed.go:79)）。

**写路径（先 L1，再异步 L2）**

```python
# python/aibrix_kvcache/aibrix_kvcache/cache_manager.py — BaseKVCacheManager.put，L1589-1596
1589  # Bypass L1 if L2 cache is enabled with zero copy
1590  if self._l2_cache_has_zero_copy():
1591      return self._l2_put(prefix, query, kv_tensors)
1592
1593  # If L1Cache is enabled, we put kv tensors to L1Cache and leverage its
1594  # eviction policy to asynchronously ingest kv tensors to L2Cache.
1595  # Otherwise, we ingest kv tensors to L2Cache directly.
1596  if self._l1_cache is not None:
```

**其它要点**
- **Go 侧编排**：
    - KVCache 控制器按后端分发：`vineyard`、`hpkv`、`infinistore`，见 [kvcache_controller.go:62-75](pkg/controller/kvcache/kvcache_controller.go:62)。
    - 后端取自注解 `kvcache.orchestration.aibrix.ai/backend`，缺省是 `vineyard`，见 [kvcache_controller.go:188-197](pkg/controller/kvcache/kvcache_controller.go:188)、[constants/kvcache.go:39-42](pkg/constants/kvcache.go:39)。
    - CRD 里 `mode` 默认 `distributed`，元数据可选 Redis 或 Etcd，见 [kvcache_types.go:85-107](api/orchestration/v1alpha1/kvcache_types.go:85)。
- **watcher**：`cmd/kvcache-watcher` 从其常量与依赖看，用一致性哈希槽位和 Redis 成员键登记 cache 节点，见 [main.go:52-78](cmd/kvcache-watcher/main.go:52)。我没有深读它的逻辑。
- **KV 事件同步**（让网关的前缀索引与引擎真实缓存一致）：
    - 网关经 ZMQ 订阅每个 vLLM Pod，见 [kvevent/manager.go:38-59](pkg/kvevent/manager.go:38) 和 [:78-137](pkg/kvevent/manager.go:78)。
    - 事件分三类：`BlockStored`、`BlockRemoved`、`AllBlocksCleared`，见 [handler.go:39-61](pkg/kvevent/handler.go:39)。
    - 开启条件是"事件同步开关 **且** 远程 tokenizer 开关"同时为真，见 [main.go:199-204](cmd/plugins/main.go:199)。

## 4. 控制平面

### 4.1 Model Adapter (LoRA) 控制器

```text
 kubectl apply ModelAdapter   (spec: podSelector, artifactURL, replicas = nil | 1)
        |
        v
 ModelAdapter controller   Reconcile -> DoReconcile        modeladapter_controller.go:323 -> :427
   |
   +-[1] reconcileReplicas        which pods?              modeladapter_controller.go:448
   |       replicas=nil : every matching pod                 :640-643
   |       replicas=1   : ONE pod chosen by the scheduler    :644-647 -> :660-750
   |
   +-[2] reconcileLoading         load LoRA on each pod    modeladapter_controller.go:464
   |       loraClient.LoadAdapter                            lora_client.go:82
   |         sidecar present AND --enable-runtime-sidecar
   |           -> POST http://<podIP>:8080/v1/lora_adapter/load     lora_client.go:86-87, :44
   |              {lora_name, artifact_url, credentials}            lora_client.go:255-273
   |              the runtime downloads the artifact, then tells the engine
   |         otherwise
   |           -> POST http://<podIP>:8000/v1/load_lora_adapter     lora_client.go:48, utils.go:127
   |              {lora_name, lora_path}  (hf:// or local path only) lora_client.go:216-245
   |
   +-[3] reconcileService         headless Service, no selector    modeladapter_controller.go:483
   +-[4] reconcileEndpointSlice   IPs of status.instances          modeladapter_controller.go:495
   +-[5] syncReadinessStatus      derive phase + Ready condition   modeladapter_controller.go:509
        |
        v
 gateway pkg/cache: informer sees ModelAdapter.status.instances       informers.go:287-299
   -> model "<adapter name>" maps to exactly those pods
   -> a request {"model": "<adapter name>"} is routed only among them
```

`modeladapter_controller.go`、`lora_client.go`、`utils.go` 均在 `pkg/controller/modeladapter/`。

**要点**
- **CRD**：`replicas` 缺省表示装到所有匹配 Pod；`1` 表示由调度器选一个；其他值被拒绝，见 [modeladapter_types.go:50-56](api/model/v1alpha1/modeladapter_types.go:50)。
- **触发**：
    - 监听 ModelAdapter，以及它拥有的 Service 和 EndpointSlice，见 [:253-265](pkg/controller/modeladapter/modeladapter_controller.go:253)。
    - 监听带 `adapter.model.aibrix.ai/enabled=true` 且有模型标签的 Pod（[:203-225](pkg/controller/modeladapter/modeladapter_controller.go:203)、[:59-64](pkg/controller/modeladapter/modeladapter_controller.go:59)）。
    - 另外每 10s 全量重排队（[:66](pkg/controller/modeladapter/modeladapter_controller.go:66)、[:385-403](pkg/controller/modeladapter/modeladapter_controller.go:385)）。
- **两条加载路径**，由 [lora_client.go:86](pkg/controller/modeladapter/lora_client.go:86) 决定：
    - 走 sidecar，端口 8080，由 sidecar 负责下载。
    - 直连引擎，端口 8000。vLLM 与 SGLang 的路径不同，由 [BuildURLs](pkg/controller/modeladapter/utils.go:120) 屏蔽。
    - 直连模式只接受 `huggingface://`、`hf://` 或本地路径；`s3://` 等会直接报错并提示改用 sidecar，见 [lora_client.go:216-245](pkg/controller/modeladapter/lora_client.go:216)。
- **删除**：靠 finalizer 保证先从引擎卸载，见 [:356-376](pkg/controller/modeladapter/modeladapter_controller.go:356)。
- **失败重试**：`MaxLoadingRetries = 6`，`RetryBackoffSeconds = 5`，见 [:68-72](pkg/controller/modeladapter/modeladapter_controller.go:68)。
- **为什么 Service 没有 selector、还要手写 EndpointSlice**：
    - 哪些 Pod 装了这个 LoRA 取决于 `status.instances`，而不是标签。
    - Service 没有 selector，K8s 不会自动生成 Endpoints，所以控制器按 `status.instances` 手写 EndpointSlice（[:1029-1070](pkg/controller/modeladapter/modeladapter_controller.go:1029)）。
    - 这就是图里的 "Custom Endpoint Slice"，并且它是本控制器的一个步骤，不是独立控制器。

```go
// pkg/controller/modeladapter/resources.go — buildModelAdapterService，L96-99（没有 Selector 字段）
96  Spec: corev1.ServiceSpec{
97  	ClusterIP:                corev1.ClusterIPNone,
98  	PublishNotReadyAddresses: true,
99  	Ports:                    ports,
```

- **调度策略**：
    - 有 `random`、`leastAdapters`、`binPack`、`leastLatency`、`leastThroughput` 五种（[scheduler.go:35-50](pkg/controller/modeladapter/scheduling/scheduler.go:35)），默认 `leastAdapters`（[:121](pkg/controller/modeladapter/modeladapter_controller.go:121)）。
    - `binPack` 的容量还是写死的 `podCap := 10 // todo: replace mock data`（[bin_pack.go:49](pkg/controller/modeladapter/scheduling/bin_pack.go:49)）。
    - ⚠ 策略只由**控制器进程级**参数 `--model-adapter-scheduler-policy` 决定（[main.go:153](cmd/controllers/main.go:153)）。
    - CRD 的 `spec.schedulerName`（[modeladapter_types.go:37-40](api/model/v1alpha1/modeladapter_types.go:37)）在整个 `pkg/` 里没有任何读取它的地方（我 grep 过）。
    - 样例注释写的 "default / least-adapters"（[adapter.yaml:30-32](samples/adapter/adapter.yaml:30)）也与代码里的策略名对不上。

### 4.2 RayClusterFleet（多节点推理）

```text
 RayClusterFleet         Deployment-like : strategy, rollback, revision history       (a)
     |  owns 1..N  (one per pod-template revision)
     v
 RayClusterReplicaSet    ReplicaSet-like : keep N RayClusters alive                   (b)
     |  creates / deletes
     v
 RayCluster              KubeRay CRD : 1 head pod + worker pods = 1 multi-node replica

 StormService  -->  RoleSet  -->  PodSet  -->  Pod      (roles such as prefill / decode)  (c)
```

- (a) Fleet 拥有 `RayClusterReplicaSet` 和 `RayCluster`，见 [rayclusterfleet_controller.go:74-80](pkg/controller/rayclusterfleet/rayclusterfleet_controller.go:74)。
    - `Reconcile`（[:107-183](pkg/controller/rayclusterfleet/rayclusterfleet_controller.go:107)）依次处理：暂停（[:158](pkg/controller/rayclusterfleet/rayclusterfleet_controller.go:158)）、回滚（[:162](pkg/controller/rayclusterfleet/rayclusterfleet_controller.go:162)）、扩缩事件（[:166-173](pkg/controller/rayclusterfleet/rayclusterfleet_controller.go:166)），最后按策略分派 Recreate 或 Rolling（[:175-180](pkg/controller/rayclusterfleet/rayclusterfleet_controller.go:175)）。
    - 字段直接复用 `appsv1.DeploymentStrategy`（[rayclusterfleet_types.go:47](api/orchestration/v1alpha1/rayclusterfleet_types.go:47)），行为上是 Deployment 的克隆。
- (b) ReplicaSet 用 expectations 机制防止重复创建（[rayclusterreplicaset_controller.go:109-161](pkg/controller/rayclusterreplicaset/rayclusterreplicaset_controller.go:109)）。
    - 缩容排序是：未就绪优先、`pod-deletion-cost` 低的优先、旧的优先、名字兜底，见 [rayclusterreplicaset_utils.go:125-133](pkg/controller/rayclusterreplicaset/rayclusterreplicaset_utils.go:125)。
    - 判活规则：任何 Ray Pod 失败都当作不可恢复，重建整个 RayCluster，见 [:210-227](pkg/controller/rayclusterreplicaset/rayclusterreplicaset_utils.go:210)。
- **联动**：ModelRoute 会 watch Fleet 来建路由（[modelrouter_controller.go:130-133](pkg/controller/modelrouter/modelrouter_controller.go:130)）；自动扩缩对 Fleet 只统计 head Pod（[podautoscaler_controller.go:1274-1280](pkg/controller/podautoscaler/podautoscaler_controller.go:1274)）。
- (c) StormService 是更新一代的多角色编排。三层 owner 关系见 [stormservice_controller.go:60-66](pkg/controller/stormservice/stormservice_controller.go:60)、[roleset_controller.go:67-75](pkg/controller/roleset/roleset_controller.go:67)、[podset_controller.go:75-79](pkg/controller/podset/podset_controller.go:75)（§4.7）。

### 4.3 LLM 专用自动扩缩（图里的 "LLM Specific Autoscaler"）

```text
 PodAutoscaler CR : scalingStrategy KPA | APA | HPA ; metricsSources[] ; min/max
        |
        |  triggers: PodAutoscaler / HPA events + periodic resync every 10 s          (a)
        v
 Reconcile --> validate spec --> conflict check --> switch on strategy                (b)
   |
   +-- HPA ----------> reconcileHPA : a native Kubernetes HPA does the scaling        (c)
   |
   +-- KPA / APA ----> reconcileCustomPA                                              (d)
         [1] get the scale target (Deployment / RayClusterFleet / StormService role)
         [2] read current replicas
         [3] computeScaleDecision
               for every metricsSource --> executeScalingPipeline                     (e)
                   collect    pod: GET <podIP>:<port>/metrics
                              external: GPU-optimizer REST
                   aggregate  stable window / panic window
                   recommend  KPA or APA formula
               take the MAX over all metrics                                          (f)
               cooldown / stabilization window                                        (g)
               clamp to [minReplicas, maxReplicas]                                    (h)
         [4] write desired replicas through the scale subresource                     (i)

 GPU-optimizer loop (feeds an "external" metric source)
 gateway (Go) --writes--> Redis  aibrix:{model}_request_trace_{ts}                    (j)
 gpu-optimizer (Python): read traces -> cluster request shapes -> solve GPU mix       (k)
 gpu-optimizer serves  GET /metrics/{ns}/{deployment} = vllm:deployment_replicas      (l)
 PodAutoscaler (KPA, metricSourceType: external) polls that endpoint                  (m)
```

**图注**（(a)–(i) 在 `pkg/controller/podautoscaler/` 下，(j)–(m) 见 §4.4）
- (a) 周期常量见 [podautoscaler_controller.go:104](pkg/controller/podautoscaler/podautoscaler_controller.go:104)，循环见 [:659-677](pkg/controller/podautoscaler/podautoscaler_controller.go:659)，同时 watch 原生 HPA（[:212-216](pkg/controller/podautoscaler/podautoscaler_controller.go:212)）。
- (b) `Reconcile` 见 [:283-331](pkg/controller/podautoscaler/podautoscaler_controller.go:283)：校验在 [:306](pkg/controller/podautoscaler/podautoscaler_controller.go:306)，冲突检查在 [:308](pkg/controller/podautoscaler/podautoscaler_controller.go:308)，分派在 [:322-330](pkg/controller/podautoscaler/podautoscaler_controller.go:322)。
- (c) HPA 策略：`HPAAlgorithm` 只是占位，真正扩缩由 K8s 完成，见 [hpa.go:23-41](pkg/controller/podautoscaler/algorithm/hpa.go:23)；入口是 [`reconcileHPA`](pkg/controller/podautoscaler/podautoscaler_controller.go:777)。
- (d) [`reconcileCustomPA`](pkg/controller/podautoscaler/podautoscaler_controller.go:857)：取 scale 目标（:865）、取当前副本（:877）、算决策（:885）、写回（:908）。
- (e) [`executeScalingPipeline`](pkg/controller/podautoscaler/autoscaler.go:389)：采集 [:409](pkg/controller/podautoscaler/autoscaler.go:409)，窗口聚合 [:418-431](pkg/controller/podautoscaler/autoscaler.go:418)，出建议 [:466](pkg/controller/podautoscaler/autoscaler.go:466)。
    - 指标抓取：pod 源在 [fetcher.go:80-110](pkg/controller/podautoscaler/metrics/fetcher.go:80)，GPU-optimizer 源在 [:268-300](pkg/controller/podautoscaler/metrics/fetcher.go:268)。
    - 另外还支持 K8s resource / custom / external 指标（[:113](pkg/controller/podautoscaler/metrics/fetcher.go:113)、[:170](pkg/controller/podautoscaler/metrics/fetcher.go:170)、[:219](pkg/controller/podautoscaler/metrics/fetcher.go:219)）。
- (f) 多个指标各出一个建议，取**最大值**，见 [autoscaler.go:191-197](pkg/controller/podautoscaler/autoscaler.go:191)。
- (g) 冷却窗见 [podautoscaler_controller.go:1205-1209](pkg/controller/podautoscaler/podautoscaler_controller.go:1205) 与 [:1306](pkg/controller/podautoscaler/podautoscaler_controller.go:1306)，注解名见 [annotations.go:45-51](pkg/controller/podautoscaler/types/annotations.go:45)。
- (h) 上下界钳制见 [:1216-1224](pkg/controller/podautoscaler/podautoscaler_controller.go:1216)。
- (i) 写 scale 子资源见 [workload_scale.go:240](pkg/controller/podautoscaler/workload_scale.go:240)；按 role 扩缩见 [:289](pkg/controller/podautoscaler/workload_scale.go:289)。

**三种策略**（[podautoscaler_types.go:108-110](api/autoscaling/v1alpha1/podautoscaler_types.go:108)）
- **KPA**：Knative 式的稳定窗加恐慌窗，恐慌模式下只增不减，见 [kpa.go:42-83](pkg/controller/podautoscaler/algorithm/kpa.go:42) 和 [:168-188](pkg/controller/podautoscaler/algorithm/kpa.go:168)。
- **APA**：AIBrix 自研。按"当前每 Pod 指标 / 目标值"的比例扩缩，带上下容忍带和最大扩缩速率：

```go
// pkg/controller/podautoscaler/algorithm/apa.go — APAAlgorithm.computeTargetReplicas，L91-98
91  if currentUsePerPod/expectedUse > (1 + upTolerance) {
92  	maxScaleUp := int32(math.Ceil(context.GetMaxScaleUpRate() * currentPodCount))
93  	expectedPods := int32(math.Ceil(currentPodCount * (currentUsePerPod / expectedUse)))
94  	if expectedPods > maxScaleUp {
95  		expectedPods = maxScaleUp
96  	}
98  	return expectedPods
```

**指标源、目标与扩展字段**
- **指标源**有 `pod`、`resource`、`custom`、`external` 四类，`domain` 已弃用，见 [podautoscaler_types.go:204-218](api/autoscaling/v1alpha1/podautoscaler_types.go:204)。
- 官方样例用 **KV 缓存占用**做 KPA 指标：`gpu_cache_usage_perc`，目标 0.5，缩容冷却 3 分钟，见 [kpa.yaml:10](samples/autoscaling/kpa.yaml:10)、[:15-21](samples/autoscaling/kpa.yaml:15)。
- **扩缩目标**可以是 Deployment、RayClusterFleet，或 StormService 的某个 role（`subTargetSelector.roleName`），见 [podautoscaler_types.go:57-64](api/autoscaling/v1alpha1/podautoscaler_types.go:57)。
- 此外还有定时上下界 `schedules`（[:77-81](api/autoscaling/v1alpha1/podautoscaler_types.go:77)）和预测式扩缩 `predictive`（Preview / Auto，[:101-106](api/autoscaling/v1alpha1/podautoscaler_types.go:101)）。

**⚠ 关于"秒级扩缩"**
- 文档写的是 real-time、second-level（[architecture.rst:25](docs/source/designs/architecture.rst:25)）。
- 代码里：每 10s 触发一次（上文 (a)），指标粒度 1s（[autoscaler.go:99](pkg/controller/podautoscaler/autoscaler.go:99)）。
- **但默认稳定窗 180s、恐慌窗 60s**（[metrics/client.go:28-29](pkg/controller/podautoscaler/metrics/client.go:28)），可用 `observeWindowSeconds` 和 `panicWindowSeconds` 调小（[podautoscaler_types.go:87-99](api/autoscaling/v1alpha1/podautoscaler_types.go:87)）。
- 所以"秒级"是可以配到的能力，不是默认行为。

### 4.4 GPU Optimizer

- **形态**：Python Starlette 服务，不是 K8s 控制器。接口见 [gpu_optimizer/app.py](python/aibrix/aibrix/gpu_optimizer/app.py)：`/monitor`（[:180](python/aibrix/aibrix/gpu_optimizer/app.py:180)、[:200](python/aibrix/aibrix/gpu_optimizer/app.py:200)）、`/update_profile`（[:218](python/aibrix/aibrix/gpu_optimizer/app.py:218)）、`/scale`（[:234](python/aibrix/aibrix/gpu_optimizer/app.py:234)）、`/metrics`（[:272](python/aibrix/aibrix/gpu_optimizer/app.py:272)）。
- **输入有两路**：
    - 离线 profile，由 `gen_profile` 生成后写入 Redis，见 [README.md](python/aibrix/aibrix/gpu_optimizer/README.md)。
    - 在线负载：网关每个统计间隔把"输入 / 输出 token 的 log2 桶及其频次"写成 `aibrix:{model}_request_trace_{ts}`，见 [cache_trace.go:535-536](pkg/cache/cache_trace.go:535)。读取方在 [load_reader.py:225-262](python/aibrix/aibrix/gpu_optimizer/load_monitor/load_reader.py:225)。
- **处理**：移动 DBSCAN 聚类（[monitor.py:378-381](python/aibrix/aibrix/gpu_optimizer/load_monitor/monitor.py:378)），再用 Melange 求解器求各 GPU 类型的副本数和成本（[optimizer.py:114-132](python/aibrix/aibrix/gpu_optimizer/optimizer/optimizer.py:114)、[monitor.py:488-525](python/aibrix/aibrix/gpu_optimizer/load_monitor/monitor.py:488)）。
- **输出**：暴露 `vllm:deployment_replicas`，见 [app.py:272-292](python/aibrix/aibrix/gpu_optimizer/app.py:272)。`/scale` 可以人工覆盖，见 [:234-269](python/aibrix/aibrix/gpu_optimizer/app.py:234)。
- **接入扩缩**：PodAutoscaler 选 `KPA` 加 `external` 指标源，见 [optimizer-kpa.yaml:12-21](samples/autoscaling/optimizer-kpa.yaml:12)，对应 [fetcher.go:268-300](pkg/controller/podautoscaler/metrics/fetcher.go:268)。

### 4.5 AI Engine Runtime（sidecar）

- **注入**：
    - 给 Deployment（或 StormService）加注解 `model.aibrix.ai/sidecar-injection: "true"`。
    - webhook 把 `aibrix-runtime` 容器插到容器列表最前，并给所有容器挂共享 emptyDir `adapter-storage`，挂载点是 `/tmp/aibrix/adapters`。
    - 见 [deployment_webhook.go:56-64](pkg/webhook/deployment_webhook.go:56)、[:105-145](pkg/webhook/deployment_webhook.go:105)、[sidecar_injection.go:28-43](pkg/webhook/sidecar_injection.go:28)。
    - 探针在 [:77-98](pkg/webhook/sidecar_injection.go:77)，资源 100m/256Mi 到 500m/512Mi 在 [:99-108](pkg/webhook/sidecar_injection.go:99)。
- **能力**（[app.py](python/aibrix/aibrix/app.py)）：
    - LoRA 加载与卸载：[:140-220](python/aibrix/aibrix/app.py:140)。
    - 引擎模型列表：[:225-231](python/aibrix/aibrix/app.py:225)。
    - 模型下载与列表：[:234-249](python/aibrix/aibrix/app.py:234)。
    - 抓取引擎 `/metrics` 并重新暴露：[:88-116](python/aibrix/aibrix/app.py:88)。
    - 健康与就绪：[:257-268](python/aibrix/aibrix/app.py:257)。
- **抽象引擎**：`get_inference_engine(engine, version, endpoint)`，见 [app.py:119-125](python/aibrix/aibrix/app.py:119)。
- **下载（图里的 "Model loader"）**：
    - 按 URL scheme 选下载器：`s3`、`gcs`、`tos`、`huggingface`、`http(s)`，见 [downloaders.py:637-668](python/aibrix/aibrix/runtime/downloaders.py:637)。
    - 带完成标记、按 adapter 加锁、保留 `.part` 和 ETag 侧文件以便断点续传，见 [artifact_service.py:157-236](python/aibrix/aibrix/runtime/artifact_service.py:157)。
- **运行时模型接口**（ModelClaim 使用）：activate、deactivate、kv-limit、sleep、wake、list、snapshot，见 [model_runtime_api.py:53-204](python/aibrix/aibrix/runtime/model_runtime_api.py:53)。
- **何时启用**：控制器需带 `--enable-runtime-sidecar`，且 Pod 里确有 `aibrix-runtime` 容器；否则仍直连引擎，见 [main.go:154-155](cmd/controllers/main.go:154)、[lora_client.go:86](pkg/controller/modeladapter/lora_client.go:86)。

### 4.6 Accelerator Diagnose Tools

- **状态：仅文档**，见 [architecture.rst:28](docs/source/designs/architecture.rst:28)。
- **证据**：
    - 我在 `*.go`、`*.py`、`*.md`、`*.yaml` 里不区分大小写搜索 "diagnos"，命中都是无关用法：klog 诊断信息（[router.go:454](pkg/plugins/gateway/algorithms/router.go:454)）、响应头里的路由诊断（[gateway_rsp_headers.go:42-52](pkg/plugins/gateway/gateway_rsp_headers.go:42)）、drain 注解的备注（[constants/drain.go:31](pkg/constants/drain.go:31)）。
    - 搜 accelerator / fault / xid 只命中 console 里的 GPU 类型校验和 error injection。
- **沾边但不是它的东西**：
    - `development/app` 的 mock vLLM 应用，见 [development.rst:299-303](docs/source/development/development.rst:299)。
    - console 的按 job 错误注入，见 [design.md:5](apps/console/api/error_injection/doc/design.md:5)。
    - 二者都不是 GPU 故障诊断。

### 4.7 代码里有、你的清单和图里没有的

| 组件 | 作用 | 位置 |
|---|---|---|
| **ModelRoute** | 监听 Deployment / ModelAdapter / RayClusterFleet / LeaderWorkerSet，为模型建 Gateway API 的 `HTTPRoute`（14 条 OpenAI 风格路径 + `model` 头精确匹配，默认超时 600s）和跨命名空间的 `ReferenceGrant` | [modelrouter_controller.go:87-151](pkg/controller/modelrouter/modelrouter_controller.go:87)、[:243-334](pkg/controller/modelrouter/modelrouter_controller.go:243) |
| **ModelClaim** | 把"完整模型"以引擎进程的形式动态装到共享 GPU 的 warm Pod 上（kvcached），支持 sleep / wake。激活期间标记为不可路由（端口 0），就绪后才放开 | [modelclaim_types.go:30-68](api/model/v1alpha1/modelclaim_types.go:30)、[modelclaim_controller.go:389-449](pkg/controller/modelclaim/modelclaim_controller.go:389) |
| **StormService / RoleSet / PodSet** | 多角色（prefill / decode 等）编排，`Replica` 与 `Pooled` 两种模式；role 可配 `drain`、`podGroupSize`、拓扑策略 | [stormservice_types.go:86-109](api/orchestration/v1alpha1/stormservice_types.go:86)、[roleset_types.go:200-235](api/orchestration/v1alpha1/roleset_types.go:200)、[:154-170](api/orchestration/v1alpha1/roleset_types.go:154) |
| **drain** | 受控下线：先打 `aibrix.ai/draining` 等注解，等超时再删除；网关随即不再路由到它 | [drain.go:59-77](pkg/controller/drain/drain.go:59)、[:132-172](pkg/controller/drain/drain.go:132)、[:189-214](pkg/controller/drain/drain.go:189) |
| **admission webhooks** | 校验与默认值，含 sidecar 注入，覆盖 ModelAdapter、KVCache、StormService、Deployment、PodAutoscaler | [main.go:323-352](cmd/controllers/main.go:323) |
| **metadata-service**（Python） | OpenAI 兼容的 `/v1/models`、`/v1/files`、`/v1/batches`，另有用户与限额管理。Envoy 侧这几条路径直连它并跳过 ext_proc | [models.py:177-207](python/aibrix/aibrix/metadata/api/v1/models.py:177)、[gateway-plugin.yaml:144-178](config/gateway/gateway-plugin/gateway-plugin.yaml:144) |

**ModelClaim 与图里 "Cold Start" 最接近的机制**
- 网关遇到 Sleeping 状态的模型，会异步请求唤醒，并回 503 加 `Retry-After`，见 [gateway_req_body.go:316-322](pkg/plugins/gateway/gateway_req_body.go:316)、[modelclaim_wake.go:64-84](pkg/plugins/gateway/modelclaim_wake.go:64)。
- 这是"按请求唤醒"，不是通用的冷启动管理器。

```go
// pkg/controller/modelclaim/modelclaim_controller.go — ensureActivated，L430-437
430  // The engine is spawned but not yet serveable (boot/compile). Keep the
431  // model NOT routable — stamp the non-routable marker (port 0), record the
432  // instance as Activating with its real port — until reconcileInstanceHealth
433  // confirms the engine is ready, then it flips the annotation to the real
434  // port. This means the gateway never routes to a still-booting engine.
435  if err := r.annotateWarmPod(ctx, pm, pod, 0); err != nil {
436  	return err
437  }
```

## 5. 图 ↔ 代码 对照

| 图中元素 | 代码里对应什么 | 结论 |
|---|---|---|
| API Gateway and Proxy（OpenAI 兼容） | Envoy Gateway + gateway-plugins（[gateway.go:329](pkg/plugins/gateway/gateway.go:329)） | 有 |
| Fairness、TPM/RPM | RPM/TPM（[gateway_ratelimit.go:32](pkg/plugins/gateway/gateway_ratelimit.go:32)）；公平用 `vtc-basic`；resource oversell 未找到 | 部分 |
| 前缀缓存感知、负载感知路由 | [prefix_cache.go:399](pkg/plugins/gateway/algorithms/prefix_cache.go:399) 加一批 `least-*` 与 `pd` | 有 |
| Model Metadata Store | 网关进程内 `Store`（[cache_init.go:380](pkg/cache/cache_init.go:380)），非独立存储 | 有，形态不同 |
| Model Metadata Controller（Registration / Watch） | `pkg/cache` 的 watch（[cache_init.go:498-547](pkg/cache/cache_init.go:498)）+ ModelRoute 建路由 | 拆成两处 |
| Service base / Lora1 / Lora2 / Lora3 | 每个 adapter 一个 headless Service（[resources.go:64-102](pkg/controller/modeladapter/resources.go:64)） | 有 |
| Custom Endpoint Slice + Controller | ModelAdapter 控制器的一步（[:1029-1070](pkg/controller/modeladapter/modeladapter_controller.go:1029)） | 有，已合并 |
| Pod 内 Base / Lora | 有 | 有 |
| Pod 内 Mem / Mount | "Mem"没找到；最近似 "Mount" 的是注入的共享 emptyDir（[deployment_webhook.go:105-139](pkg/webhook/deployment_webhook.go:105)） | 部分 |
| Sidecar：Model loader | 下载服务（[artifact_service.py:157-236](python/aibrix/aibrix/runtime/artifact_service.py:157)） | 有 |
| Sidecar：Runtime Agent | `aibrix-runtime`（[app.py:140-268](python/aibrix/aibrix/app.py:140)） | 有 |
| Sidecar：Watch Dog | 无同名实现；最近似的是 sidecar 探针（[sidecar_injection.go:77-98](pkg/webhook/sidecar_injection.go:77)）和 ModelClaim 的实例健康检查（[modelclaim_controller.go:481](pkg/controller/modelclaim/modelclaim_controller.go:481)） | 无 |
| S3、high perf loader、Dataset（Artifact / credentials） | 下载器支持 s3 / gcs / tos / hf / http；凭证来自 `credentialsSecretRef`（[lora_client.go:261-273](pkg/controller/modeladapter/lora_client.go:261)）；"high perf loader"没找到特殊实现 | 部分 |
| Cold Start Manager（`/load`） | 无同名；最近似 ModelClaim 加 runtime activate / wake | 无 |
| LLM Specific Autoscaler | PodAutoscaler 控制器 | 有 |
| Pod Deletion Controller（Watch & Patch） | 无同名控制器；分散在 `drain` 库、role/podset 控制器（[rolesyncer.go:492](pkg/controller/roleset/rolesyncer.go:492)）与 RayCluster 缩容排序（[rayclusterreplicaset_utils.go:125](pkg/controller/rayclusterreplicaset/rayclusterreplicaset_utils.go:125)） | 无，分散 |
| Distributed KV Cache Runtime（Cache Worker Pod、metadata Server） | KVCache 控制器 + `aibrix_kvcache` | 有 |
| Topology Information（Pod 内 GPU / Memory / SSD 各放哪个模型） | `pkg/cache` 里没有 topology 概念（最近的是 pod→models 映射）；RoleSet 的 `TopologyPolicy` 是节点亲和，不是这个 | 无 |

## 6. 核对范围与局限

- **只做了静态阅读**：没有起集群，也没有跑 `make test`。
- **只读了注册和入口、没逐个读实现的部分**：
    - 其余 `least-*`、`power-of-two`、`throughput` 等策略，我只核对了注册。
    - SLO 与 VTC 路由的内部打分。
    - Redis 限流器实现。
    - vineyard / hpkv / infinistore 三个 KV 后端。
    - L2 连接器。
    - `kvcache-watcher` 的逻辑。
    - `statesync`。
    - batch、console、brixbench。
- **文档与代码的三处值得留意的差异**：
    - `spec.schedulerName` 未被读取（§4.1）。
    - "秒级扩缩"默认并非秒级（§4.3）。
    - Accelerator Diagnose Tools 仅文档（§4.6）。

另外，我把你的两条输出规则（引用带行号、图只用 ASCII 并附解释）记进了项目记忆，之后同类请求不用再重复。想深入哪一块，比如 PD 打分细节、KVCache 各后端、StormService 滚动升级，告诉我，我继续读代码展开。


