
# Higress 控制面主线 + 数据面插件 全链路逻辑梳理

> 原始问题：
> 1，代码/引用必须备注/注释行数（哪个文件/类，哪一行）
> 2，如果要画图，只能画 ASCII 图（必须配合文字详细补充解释），不准画其他类型图
> 围绕 控制面主线代码，数据面插件，梳理项目全链路逻辑

> 所有引用格式：`文件路径:行号`（均已在源码阅读中核实）。`istio/`、`envoy/envoy` 是仓库内 git 子模块。

## 〇、全局架构图（ASCII）

```
+========================== 控制面 (higress-controller, Go) ==========================+
|                                                                                     |
|  cmd/higress/main.go:26  →  pkg/cmd/server.go:68-85  →  pkg/bootstrap/server.go:153 |
|                                                                                     |
|  +--- K8s API Watch (Informer) ------------------------------------------------+   |
|  | Ingress/V1Beta1  Gateway API  Secret  ConfigMap(higress-config)              |   |
|  | WasmPlugin(CRD)  McpBridge(CRD)  Http2Rpc(CRD)  Service(多集群)               |   |
|  +------------------------------+----------------------------------------------+   |
|                                 |                                                   |
|  pkg/ingress/config/ingress_config.go:290  List()  按需翻译(无本地缓存,查询时现算)      |
|      Ingress 注解 → annotations.Handler (ingress_config.go:484)                    |
|      转出 6 类 Istio IR: Gateway / VirtualService / DestinationRule /                |
|                        ServiceEntry / EnvoyFilter / WasmPlugin                     |
|                                 |                                                   |
|                    RegistryReconciler (registry/reconcile/reconcile.go:48)          |
|                      └─ nacos/consul/eureka/zk/direct watcher → ServiceEntry        |
|                                 |                                                   |
|  IngressTranslation (translation.go:163) = 环境 ConfigStore                          |
|                                 |                                                   |
|  xds.DiscoveryServer (server.go:385, 复用 istiod 的 ADS 服务器)                       |
|      ├─ 标准 xDS: LDS/RDS/CDS/EDS/SRDS  ← istio pilot 内建生成器                     |
|      │    └─ wasm 插件以 ECDS 引用内联进 HTTP 过滤器链                                 |
|      │       (istio/istio/pilot/pkg/networking/core/listener_builder.go:462-476)    |
|      └─ MCP-over-xDS: WasmPlugin/VS/DR/GW/SE/EF 原样打包为 mcp.Resource              |
|         (pkg/ingress/mcp/generator.go:186-236, 按 TypeUrl=GVK 订阅分发)              |
+=====================================+==============================================+
                                      | gRPC ADS (:15051, server.go:94/314-323)
                                      v
+========================== 数据面 (higress-gateway = Envoy fork) =====================+
|  Listener → HTTP Connection Manager 过滤器链（按相位排序）:                            |
|   [wasm(AUTHN) → jwt/rbac → wasm(AUTHZ) → cors → golang-filter(mcp-session/-server) |
|    → wasm(默认) → ... → router]                                                      |
|                                                                                     |
|  WASM 插件运行时: envoy/envoy/source/extensions/common/wasm/ (proxy-wasm VM)         |
|    └─ 插件产物: oci:// 镜像 或 file:///opt/plugins/*.wasm (WasmPlugin CR 的 url)      |
|  Go 原生过滤器: golang-filter.so (plugins/golang-filter/main.go:12-23 注册)           |
|    └─ mcp-session / mcp-server (AI 网关 MCP 协议转换)                                |
+=====================================================================================+
```

图的文字说明：上半部分是 Go 编写的控制面，入口经 `main.go` 进入 bootstrap 组装，通过 K8s Informer 监听四类资源（Ingress/Gateway API、Secret、ConfigMap、三个 Higress CRD）；翻译层在每次 xDS 订阅时把 Ingress 现算成 6 类 Istio IR，注册中心支线（nacos 等）产出 ServiceEntry；这些统一挂在 `IngressTranslation` 这个 ConfigStore 上，由复用自 istiod 的 ADS 服务器通过两条通道下发——标准 xDS（含 wasm 插件的 ECDS 内联）和 MCP-over-xDS（原始 CR 透传）。下半部分是 Envoy 网关，收到配置后按相位组装 HTTP 过滤器链，WASM 插件由 proxy-wasm 虚拟机执行，另有 Go 编译的 golang-filter.so 承载 MCP 协议转换过滤器。

---

## 一、控制面主线

### 1.1 进程启动链

- `cmd/higress/main.go:26-31`：`main()` 只做一件事——执行 `cmd.GetRootCommand()`。
- `pkg/cmd/root.go:21-31`：注册 `serve` 子命令。
- `pkg/cmd/server.go:68-85`：`RunE` 中创建 `bootstrap.NewServer(serverArgs)` 并 `Start(stop)`。默认参数在 `pkg/cmd/server.go:89-103`：gRPC 地址 `:15051`、HTTP `:8888`；`pkg/cmd/server.go:105-106` 定义网关选择标签 `higress/higress-system-higress-gateway`（WasmPlugin 的 selector 就按它匹配，见 2.3）。

### 1.2 Server 组装（pkg/bootstrap/server.go:153-196）

`NewServer` 按顺序执行 7 个初始化函数（`server.go:170-178`）：

| 初始化函数 | 位置 | 职责 |
|---|---|---|
| `initKubeClient` | server.go:454-472 | 建 kube client，`EnableCrdWatcher`(470 行) 让 CRD（WasmPlugin/McpBridge 等）可被 watch |
| `initXdsServer` | server.go:383-408 | 核心，见 1.4 |
| `initHttpServer` | server.go:474-485 | `:8888` 调试口 + `/ready` 探针（495-504 行） |
| `initConfigController` | server.go:221-263 | 核心，见 1.3 |
| `initRegistryEventHandlers` | server.go:199-219 | 配置变更 → 全量 push 的桥 |
| `initAuthenticators` | server.go:422-432 | xDS 连接认证（客户端证书 + K8s JWT） |
| `initAutomaticHttps` | server.go:434-452 | ACME 自动证书服务（:8889） |

`Start()`（server.go:265-346）先做 CRD 版本检查（284 行），等全部 informer 缓存同步（309-311 行 `waitForCacheSync`），再调 `xdsServer.CachesSynced()`（313 行）开始接受 Envoy 连接，最后起 gRPC(314-323) 与 HTTP(324-333) 监听。

### 1.3 配置翻译层（Ingress → Istio IR）

`initConfigController`（server.go:221-263）创建 `translation.NewIngressTranslation`（240 行），它实现 istiod 的 `ConfigStoreController` + `IngressStore` 接口（`translation/translation.go:33-36`），并被赋为整个 xDS 环境的 ConfigStore（server.go:253）。**这是 Higress 替换 istiod 配置源的关键：istiod 的 LDS/RDS 生成器从这个 store 里"读" Istio 配置，而里面的内容全是 Ingress/注解翻译出来的。**

翻译是**查询时现算**（无持久化缓存）：Envoy 每次订阅触发 push 时调用 `IngressTranslation.List()`（`translation.go:163-192`，类型过滤 164-171 行）→ `IngressConfig.List()`（`config/ingress_config.go:290-327`）：

1. `listFromIngressControllers`（ingress_config.go:329-380）：各集群 Ingress controller 的 `List()`（`kube/ingressv1/controller.go:277`）取出全部 Ingress → 按创建时间排序（362 行）→ `createWrapperConfigs`（445-500 行）解析 Higress 全套注解（`annotationHandler.Parse`，484 行，灰度/超时/重试/canary 等都在注解里）→ 按 GVK 分发到 4 个转换器（366-377 行）：
   - `convertGateways`（502 行）：Ingress host → Istio Gateway（域名即监听器）；
   - `convertVirtualService`（555 行）：path/backend → HTTPRoute，注解变路由级策略，金丝雀在 `applyCanaryIngresses`（1632 行）合并权重；
   - `convertDestinationRule`（973 行）/ `convertServiceEntry`（931 行）；
   - `convertWasmPlugin`（907 行）：见 2.3。
2. `listFromGatewayControllers`（ingress_config.go:382-397）：Gateway API 路径，并额外注入内置的 `higress-internal-ai-endpoint-picker` wasm 插件（403 行起）。
3. EnvoyFilter 单独走（336-353 行）：来自 `configmapMgr.ConstructEnvoyFilters()`（higress-config ConfigMap → 全局选项/gzip/tracing/mcp-server EnvoyFilter，见 2.5）+ 缓存的 http2rpc EnvoyFilter。
4. 翻译完的配置再过模板变量处理器（`List` 的 303-308 行，支持 `$from_secret` 等引用）。

`NewIngressConfig`（ingress_config.go:191-240）同时挂了 4 个 CR 控制器：McpBridge(217-220)、WasmPlugin(222-225)、Http2Rpc(227-230)、higress-config ConfigMap(232-234)，各自的 AddOrUpdate/Delete 回调是插件链路的入口（见第二节）。

### 1.4 xDS 下发

`initXdsServer`（server.go:383-408）：

- 385 行：直接 new 一个 istiod 的 `xds.DiscoveryServer`——**Higress 控制面就是一个精简版 istiod**，LDS/RDS/CDS/EDS 生成逻辑全部复用 `istio/` 子模块。
- 386-398 行：为 6 个 GVK 注册 Higress 自定义生成器（`mcp.WasmPluginGenerator` 等），其余 pilot 类型兜底到空生成器（396 行）。
- 399-401 行：`ProxyNeedsPush` 恒真（网关总是全量 push）。

生成器选择逻辑在 `istio/istio/pilot/pkg/xds/xdsgen.go:68-93` `findGenerator`：先按 `proxy.Metadata.Generator/typeURL` 查，最后 80 行按裸 `typeURL` 命中——所以**任何客户端用 GVK 字符串当 TypeUrl 发起 ADS 订阅（例如 `extensions.istio.io/v1alpha1/WasmPlugin`），就会被 387 行注册的 `WasmPluginGenerator` 接管**，拿到 MCP 包装的资源（这就是"控制面既发标准 xDS、又发 MCP-over-xDS"的分发点）。

配置变更触发 push：`initRegistryEventHandlers`（server.go:199-219）向 6 类 GVK 注册 `configHandler`，任何变更构造 `PushRequest{Full:true}` 调 `s.xdsServer.ConfigUpdate`（212 行）→ istiod 的 push 队列 → 各连接按 `findGenerator` 出数据。

### 1.5 服务发现（注册中心）支线

`AddOrUpdateMcpBridge`（ingress_config.go:1417-1496）在首次看到 `default` McpBridge 时创建 `RegistryReconciler`（1428-1488 行，并把 SE/DR/VS/WasmPlugin/EF 五类 handler 作为 `serviceUpdate` 回调闭包）：

- `registry/reconcile/reconcile.go:72-106` `Reconcile()`：diff 出增/删/改的 registry 配置（108-143 行），为每个 registry 生成 watcher。
- 以 nacos 为例（`registry/nacos/watcher.go:335-363`）：订阅回调把 nacos 实例列表转成 Istio `ServiceEntry`（365 行 `generateServiceEntry`）写入内存 cache，并 `UpdateService()` 触发 1.4 的 push——最终经 `convertServiceEntry`（ingress_config.go:931-971，921-927 行同样汇聚 registry 产出的 wasm 配置）进入 xDS。

---

## 二、插件链路（控制面 → 数据面）

### 2.1 WasmPlugin CR（Higress 扩展版）

CRD 定义在 `api/extensions/v1alpha1`，比 Istio 原生多了 `matchRules`（按 ingress/service/domain 三维匹配生效）。用户下发 CR 后：

```
用户 apply WasmPlugin CR (higress 扩展 CRD, url: oci://... 或 file://...)
   │
   v  informer 事件
pkg/ingress/kube/wasmplugin/controller.go (NewController, AddEventHandler)
   │
   v
IngressConfig.AddOrUpdateWasmPlugin      config/ingress_config.go:1351-1389
   ├─ 1368-1371: 先对所有 wasmPluginHandlers 触发 EventUpdate
   │              (带 AlwaysPushLabel, 强制全量 push, 不做配置 diff)
   └─ 1372:     convertIstioWasmPlugin     config/ingress_config.go:1199 起
                  ├─ 1200-1205: selector = 网关标签 (higress/higress-system-higress-gateway)
                  ├─ 1230-1231: defaultConfig / DefaultConfigDisable
                  └─ 1240-1319: matchRules 展开成 PluginConfig 里的
                                 _match_route_ / _match_service_ / _match_domain_ 字段
                                 (1276/1295/1314 行)
   │
   v  存入 m.wasmPlugins (1387 行)，等 Envoy 订阅时被 List 读走
```

图的文字说明：WasmPlugin CR 经 informer 进入 `AddOrUpdateWasmPlugin`，先带 `AlwaysPushLabel` 标签触发一次强制全量 push（不做新旧配置 diff、保证必推），再把 Higress 扩展字段（matchRules、defaultConfig）规整成 Istio 原生 `extensions.WasmPlugin` 的形状存入内存 map；真正翻译成 Envoy 配置发生在之后 Envoy 订阅、`List()` 被调用时。

### 2.2 下发通道 A：标准 xDS（LDS/ECDS，主通道）

Envoy 网关以普通代理身份订阅 ADS：

1. push 时 `PushContext.WasmPluginsByListenerInfo`（`istio/istio/pilot/pkg/model/push_context.go:2235`）从 ConfigStore 聚合 WasmPlugin（selector 匹配网关）。
2. HTTP 监听器构建时（`istio/istio/pilot/pkg/networking/core/listener_builder.go:462-476`）按相位把 wasm 过滤器插入过滤器链：AUTHN 相位在 jwt 之前（476 行）、AUTHZ 相位在 rbac 之后（478 行）等；TCP 场景在 `networkfilter.go:119-140`。
3. `toEnvoyHTTPFilter`（`istio/.../core/extension/wasmplugin.go:75-109`）插入的只是 **ECDS 引用**（`ConfigDiscovery` + wasm type_url），真正的 wasm 配置（OCI url、vmConfig、PluginConfig）由 `InsertedExtensionConfigurations`（同文件 132 行起）作为 ExtensionConfig 资源随 ADS 下发。**好处：改插件配置只需推 ECDS，不重建 Listener。**

### 2.3 下发通道 B：MCP-over-xDS（原样透传）

`WasmPluginGenerator.Generate`（`pkg/ingress/mcp/generator.go:157-161`）→ 公共 `generate()`（186-236 行）：把 CR spec 塞进 `istio.io/api/mcp/v1alpha1.Resource{Body, Metadata}`，226 行 Any 打包、230 行 `discovery.Resource` 返回；240-268 行甚至用 proto 未知字段（field 100）夹带 `config.Extra`。订阅方是以 GVK 为 TypeUrl 的 MCP 客户端（生成器兜底逻辑见 `xdsgen.go:80,92`），用于需要原始 CR 的组件（如控制台/plugin-server 等）。

### 2.4 数据面：Envoy 加载与运行 WASM 插件

```
Envoy (higress envoy-1.36 fork)
  │  1. ECDS 收到 ExtensionConfig → envoy.extensions.filters.http.wasm.v3.Wasm
  │     (含 url=oci://higress-registry.../plugins/request-block:1.0.0)
  │  2. WasmConfig 构造: envoy/envoy/source/extensions/common/wasm/plugin.cc:20-59
  │     (capability 白名单 24 行、VM 环境变量合并 26-58 行)
  │  3. 镜像/远程 wasm 拉取: source/extensions/common/wasm/remote_async_datasource.cc:22-53
  │     (RemoteAsyncDataProvider, 抖动指数退避重试, 经 cluster_manager 走 envoy 集群)
  │  4. proxy-wasm VM 启动 (v8/wamr), 执行插件 .wasm 的 _start → Go init()
  v
每条请求: wasm filter 实例化 Context → 回调插件的 proxy_on_request_headers 等
```

图的文字说明：插件生命周期分四步——收到 ECDS 配置、构造 WasmConfig（处理能力白名单与 VM 环境变量）、按 `url` 拉取 wasm 产物（OCI 镜像经 cluster_manager 以 Envoy 集群方式拉取并解层，本地 file:// 则直接读文件，失败按抖动指数退避重试）、最后拉起 proxy-wasm 虚拟机执行模块入口。请求到来时 Envoy 为其创建 wasm Context 并依次回调导出的各阶段函数。

fork 对 wasm 运行时的增强（`#if defined(HIGRESS)`，`envoy/envoy/source/extensions/common/wasm/context.h`）：

- 220-224 行：`injectEncodedDataToFilterChain`（向过滤链注入响应体，插件本地直接回包用）、`getUpstreamHosts`/`setUpstreamOverrideHost`（**插件指定上游地址**，模型路由的关键能力）；
- 347-349 行：`setBuffer`（重写请求/响应体）；
- 415-417 行：`redisInit`/`redisCall`（插件直连 Redis，限流/缓存类插件用）；
- 432-434 行：自定义 ABI 0.2.100——改请求头后自动 `clearRouteCache` 支持重路由。

### 2.5 Go 原生过滤器（golang-filter，非 WASM）

- 注册：`plugins/golang-filter/main.go:12-14` 在 `init()` 里注册 `mcp-session`、`mcp-server` 两个过滤器，编译为 `golang-filter.so`（`Makefile.core.mk:206-221`，打进网关镜像，挂载路径 `/var/lib/istio/envoy/golang-filter.so`）。
- 配置注入：控制面把 higress-config ConfigMap 翻译成 EnvoyFilter——`pkg/ingress/kube/configmap/mcp_server.go:350-460` `ConstructEnvoyFilters()`：HTTP_FILTER 以 `config_discovery`(ECDS) 引用插入 cors 之后（363-371、395-405 行），EXTENSION_CONFIG 提供 `typed_config`（374-377 行），其中 626-629 行指定 `library_id/plugin_name` 并由 `mcp-server/config.go:45-49` 的 `Parser.Parse` 以 `xds.TypedStruct` 解析出 servers 列表（56-153 行），`FilterFactory`（178-187 行）实例化过滤器。
- 请求处理（MCP 协议转换，`plugins/golang-filter/mcp-session/filter.go`）：`DecodeHeaders`:50 → 按上游类型分流 `processMcpRequestHeadersForRestUpstream`:104 / `ForSSEUpstream`:162；`DecodeData`:173、`EncodeHeaders`:218、`EncodeData`:246（REST→SSE 270 行 / SSE→SSE 303 行的协议互转），`rewriteEndpointUrl`:369 处理 SSE endpoint 重写（对应最近提交 ddbaea08 修复的场景）。`mcp-server` 则在网关本地直接托管 MCP server（SSE 端点），配置来自 `mcp-server/config.go:109-143` 的注册式 server（nacos/gorm/rag 等）。

### 2.6 插件编程模型（wasm-go 框架视角）

以 `plugins/wasm-go/examples/request-block/main.go` 为例：

- `main.go:32-41`：`main()` 为空，`init()` 调 `wrapper.SetCtx` 注册三个回调：`ParseConfigBy`(54 行 parseConfig)、`ProcessRequestHeadersBy`(131 行)、`ProcessRequestBodyBy`(187 行)。SDK 为 `github.com/higress-group/wasm-go`（外部模块，AGENTS.md 说明非仓内 SDK）。
- 请求头阶段（131-185 行）用 `proxywasm.GetHttpRequestHeader` 读 `:path`，命中规则即 `proxywasm.SendHttpResponseWithDetail`(143/149/155 行) 直接回 403——对应 2.4 的 `injectEncodedDataToFilterChain`。
- **路由级匹配在 SDK 内完成**：模块缓存 `wasm-go@v1.0.2.../pkg/matcher/rule_matcher.go:51-54` 定义 `_match_route_/_match_domain_/_match_service_` 键（正是 2.1 中控制面 1276/1295/1314 行生成的字段），`GetMatchConfig`(127 行) 在每条请求上按 route 名/host/service 选出生效的那份 PluginConfig——**这就是 Higress 插件"全局配置 + 按路由覆盖"语义的完整闭环**。

---

## 三、一条 HTTP 请求的端到端生命周期（汇总）

```
客户端
 │ ① DNS → LB → 网关 Pod :80/:443
 v
Envoy Listener（LDS 来自 convertGateways 翻译的 Ingress host）
 │ ② HTTP 过滤器链（顺序即 listener_builder.go:462-481 的插入顺序）:
 │    wasm(AUTHN 相位, 如 key-auth)
 │    → 认证/鉴权 → wasm(AUTHZ 相位)
 │    → cors → golang-filter mcp-session/mcp-server (若启用, ECDS 动态配置)
 │    → wasm(默认相位, 如 ai-proxy/request-block)
 │ ③ 路由匹配 (RDS, VS 由 Ingress path/注解翻译; 插件可 clearRouteCache 重路由)
 │    └─ 每个插件经 SDK rule_matcher.GetMatchConfig 拿到本路由的配置
 │ ④ 集群选择 (CDS/EDS: K8s Service 或 nacos 等 ServiceEntry; 插件可
 │    setUpstreamOverrideHost 强指定实例)
 v
上游 (K8s Pod / nacos 实例 / LLM API / Dubbo 经 http2rpc EnvoyFilter)
 │ ⑤ 响应回程: EncodeHeaders/EncodeData 走 wasm 插件的响应阶段
 │    + mcp-session 做 REST/SSE MCP 协议互转
 v
客户端
```

文字补充要点：② 中过滤器顺序由 WasmPlugin 的 `phase` 字段决定（翻译于 ingress_config.go:1213，插入于 listener_builder.go:476-481）；③ 中路由名是 `_match_route_` 匹配的键（ingress 翻译生成的路由名即 Ingress 的 host/path 派生名）；④ 中 EDS 端点来自 K8s informer 或 registry watcher（nacos watcher.go:344-363 的订阅回调实时刷新）；控制面任何变更（Ingress 注解、WasmPlugin、ConfigMap、nacos 实例）都会经 server.go:201-213 的 `configHandler` 或 registry 的 `serviceUpdate` 触发 `ConfigUpdate` → 防抖后全量/增量推送到 Envoy，改动秒级生效且不需要重启网关。

---

## 四、两条容易混淆的链路辨析

1. **"控制面名字里的 mcp" 有两处，含义不同**：`pkg/ingress/mcp/generator.go` 是 *配置分发协议*（MCP-over-xDS，把 CR 原样推给订阅者）；`plugins/golang-filter/mcp-*` 是 *Model Context Protocol*（AI 网关的 MCP server/session 能力）。二者无直接关系。
2. **WasmPlugin 走标准 ECDS，不走 MCP 通道**：2.2 的 ECDS 是插件配置到达 Envoy 的主路径；2.3 的 MCP 通道只是把原始 CR 额外暴露给需要原始资源的客户端。WasmPlugin CR 的 `url` 支持 `oci://`（生产）与 `file:///opt/plugins/...`（e2e 本地挂载，AGENTS.md 记录了未构建时 fail-closed 返回 500 的坑）。
