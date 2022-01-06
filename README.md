# kiali-handbook 👀


## Introduction
> kiali-handbook is a Chinese handbook about kiali, which introduces kiali from architecture, how-to-use and configuration, source code and in-action. :)

# 前言

本文档的目录与大纲：

- **一、Kiali 介绍**
  - Kiali 架构与原理
  - Kiali 源码解析 
    - Kiali 编译与运行
    - Topology 计算源码解析
    - Metrics 计算源码解析
  - Kiali 功能汇总
- **二、部署与配置**
  - Kiali 配置注解
  - Kiali 部署模式
- **三、API与高级功能**
  - Kiali API
  - Kiali 高级功能

# Kiali 介绍

> Kiali 是一款由 RedHat 发起的围绕 Istio 的开源项目，提供了拓补图计算、链路跟踪、指标遥测、配置校验、健康检查等功能，具有强大的可观测性。
> 在 Kiali 对比 Argus 前，介绍一下 **Kiali**，主要内容如下：

- Kiali 架构与原理
- Kiali 源码解析 
  - Kiali 编译与运行
  - Topology 计算源码解析
  - Metrics 计算源码解析
- Kiali 功能汇总

## Kiali 架构与原理

Kiali 目前是个单体应用，它由一个**前端应用**和一个**后端应用**组成，它们同时都被打包进了一个容器。
Kiali 依赖一些外部服务和组件，下图展示了 Kiali 中涉及的组件和交互：

<img src="https://picreso.oss-cn-beijing.aliyuncs.com/tcm/image-20220105192348076.png" alt="image-20220105192348076" style="zoom:50%;" />

### Kiali 后端

Kiali 的后端是由 Golang 编写，可以在 [kiali/kiali](https://github.com/kiali/kiali) 中查看。Kiali 需要获取 Istio 的数据和配置，这些数据和配置通过 Prometheus 和 Cluster API (Kubernetes API) 提供。
通过 Cluster API ，Kiali 可以获取 Service、Deployment、Pod 等等。同时也可以获取 Istio 的配置，如 VirtualService、DestinationRule, Gateway 等等。所以图中对 Istio 的依赖是虚线相连，因为实际上和 Istio 的交互是通过其他外部服务实现。

Kiali 的后端不提供存储功能，所有的数据都是对 Prometheus Metrics 即时计算和查询的结果。

- 当 Kiali 通过 Kiali Operator 安装时，后端配置通过 Kiali CR 进行管理。
- 当 Kiali 通过 Helm 安装时，后端配置通过 ConfigMap 进行管理。

### Kiali 前端

前端是一个单页 Web 的无状态应用程序，除了一些证书保存在浏览器中。

使用 Patternfly、React、Typescript 和 Redux 构建。代码可以在 [kiali/kiali-ui](https://github.com/kiali/kiali-ui) 查看。在标准部署中，后端服务于前端，前端查询 Kiali 后端以获取数据并呈现给用户。 

### Prometheus

Prometheus 是 Istio 的依赖项，启用 Istio 遥测时，metrics 数据存储在 Prometheus 中，Kiali 使用存储在 Prometheus 中的数据来计算网格拓扑、展示指标、健康状态、潜在问题等。
所以 Prometheus 是 Kiali 的强依赖，如果没有它，Kiali 大部分功能都不能使用。

**目前，Kiali 依赖于 Istio 部分的默认指标：**
对于 HTTP，HTTP/2 和 gRPC 流量，Istio 通过 WASM ( istio/proxy [stats filter](https://github.com/istio/proxy/blob/master/extensions/stats/plugin.cc) ) 采集并生成以下的默认 Metrics：

- **请求数** ( `istio_requests_total` )： 这都是一个  `COUNTER`  类型的指标，用于记录 Istio 代理处理的总请求数。
- **请求时长** ( `istio_request_duration_milliseconds` )： 这是一个  `DISTRIBUTION`  类型的指标，用于测量请求的持续时间。
- **请求体大小** ( `istio_request_bytes` )： 这是一个  `DISTRIBUTION`  类型的指标，用来测量 HTTP 请求主体大小。
- **响应体大小** ( `istio_response_bytes` )： 这是一个  `DISTRIBUTION`  类型的指标，用来测量 HTTP 响应主体大小。
- **gRPC 请求消息数** ( `istio_request_messages_total` )： 这是一个  `COUNTER`  类型的指标，用于记录从客户端发送的 gRPC 消息总数。
- **gRPC 响应消息数** ( `istio_response_messages_total` )： 这是一个  `COUNTER`  类型的指标，用于记录从服务端发送的 gRPC 消息总数。

对于 TCP 流量，Istio 生成以下的 Metrics：

- **TCP 发送字节大小** ( `istio_tcp_sent_bytes_total` )： 这是一个  `COUNTER`  类型的指标，用于测量在 TCP 连接情况下响应期间发送的总字节数。
- **TCP 接收字节大小** ( `istio_tcp_received_bytes_total` )： 这是一个  `COUNTER`  类型的指标，用于测量在 TCP 连接情况下请求期间接收到的总字节数。
- **TCP 已打开连接数** ( `istio_tcp_connections_opened_total` )： 这是一个  `COUNTER`  类型的指标，用于记录 TCP 已打开的连接总数。
- **TCP 已关闭连接数** ( `istio_tcp_connections_closed_total` )： 这是一个  `COUNTER`  类型的指标，用于记录 TCP 已关闭的连接总数。

目前还没有用到的指标有：

- **gRPC 请求消息数** ( `istio_request_messages_total` )： 这是一个  `COUNTER`  类型的指标，用于记录从客户端发送的 gRPC 消息总数。
- **gRPC 响应消息数** ( `istio_response_messages_total` )： 这是一个  `COUNTER`  类型的指标，用于记录从服务端发送的 gRPC 消息总数。
- **TCP 已打开连接数** ( `istio_tcp_connections_opened_total` )： 这是一个  `COUNTER`  类型的指标，用于记录 TCP 已打开的连接总数。
- **TCP 已关闭连接数** ( `istio_tcp_connections_closed_total` )： 这是一个  `COUNTER`  类型的指标，用于记录 TCP 已关闭的连接总数。

**目前，Kiali 依赖于 Istio 部分的默认 Label：**

- **Reporter**： 标识请求指标的上报端。如果指标由服务端 Istio 代理上报，则设置为  `destination` ，如果指标由客户端 Istio 代理或网关上报，则设置为  `source` 。
- **request_protocol**: 标识请求的协议。设置为请求或连接协议。
- **response_code**: 标识请求的响应代码。此标签仅出现在 HTTP 指标上。
- **response_flags**: 有关来自代理的响应或连接的其他详细信息。如果是 Envoy，请参阅 [Envoy 访问日志](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage#config-access-log-format-response-flags)中的 `％RESPONSE_FLAGS％` 获取更多信息。
- **source_principal**: 标识流量源的对等主体。当使用对等身份验证时设置。
- **source_workload**: 标识源工作负载的名称，如果缺少源信息，则标识为 “unknown”。
- **source_workload_namespace**: 标识源工作负载的命名空间，如果缺少源信息，则标识为 “unknown”。
- **destination_workload**: 标识目标工作负载的名称，如果目标信息丢失，则标识为 “unknown”。
- **destination_workload_namespace**: 标识目标工作负载的命名空间，如果目标信息丢失，则标识为 “unknown”。
- **destination_principal**: 标识流量目标的对等主体。使用对等身份验证时设置。
- **destination_service**: 标识负责传入请求的目标服务主机。 例如： `details.default.svc.cluster.local` 。
- **destination_service_name**: 标识目标服务名称。例如： `details` 。
- **destination_service_namespace**: 标识目标服务名称。例如： `details` 。
- **connection_security_policy**: 标识请求的服务认证策略。当 Istio 使用安全策略来保证通信安全时，如果指标由服务端 Istio 代理上报，则将其设置为  `mutual_tls` 。
- **Canonical Service**: Canonical 服务是 Istio 中特有的概念，用于对 Service 进行版本管理。Canonical 服务具有名称和修订版本号，因此会根据来源或目的服务产生以下标签：
  - **source_canonical_revision**
  - **source_canonical_service**
  - **destination_canonical_revision**
  - **destination_canonical_service**
- **job**: Prometheus 的 Label
- **grpc_response_status**: 只有在 gRPC 请求时，才会带上该 Label，标识 gRPC 响应的状态

### Jaeger

Jaeger 是可选的。如果可用时，Kiali 侧边栏会出现 Jaeger 的链接，不过这需要正确地配置 [Jaeger](https://kiali.io/docs/features/tracing/)， 同时 Jaeger 正常工作需要在 Istio 中启用[分布式追踪](https://istio.io/latest/docs/tasks/observability/distributed-tracing/)。

### Grafana

Grafana 是可选的。如果可用，Kiali 的指标页面将显示一个 Grafana 的链接。如果您需要此功能，需要正确配置[Grafana](https://github.com/kiali/kiali#grafana)。
Kiali 默认已经具有基本的 Metrics 展示功能。它可以显示 Workload、App 和 Service 的默认 Istio 指标，并且也有展示的 Dashboard 嵌入 Kiali 前端中，它允许获取不同时间范围的指标以及加入一些筛选。
但是 Kiali 不允许自定义视图，也不允许自定义 Prometheus 查询。如果需要这些功能，则需要安装 Grafana。



## Kiali 功能汇总

### 概览

对 Mesh 中整体的 Nodes 的健康性、指标、IstioConfig 做集中展示，也支持三种维度：App、Workload、Services，同时也支持三种展示的视图：

<img src="https://picreso.oss-cn-beijing.aliyuncs.com/tcm/image-20220106085157879.png" alt="image-20220106085157879" style="zoom:30%;" />

同时可以快速定位目标的相关监控如：Graph、Applications、Workload、Service、IstioConfig，也能开启或关闭自动注入

<img src="https://picreso.oss-cn-beijing.aliyuncs.com/tcm/image-20220106085631852.png" alt="image-20220106085631852" style="zoom:30%;" /> 

### Graph

Graph 部分是 Kiali 的核心功能部分：

![image-20220106091432819](https://picreso.oss-cn-beijing.aliyuncs.com/tcm/image-20220106091432819.png)

支持 Namespace 筛选：

<img src="https://picreso.oss-cn-beijing.aliyuncs.com/tcm/image-20220106092137463.png" alt="image-20220106092137463" style="zoom:50%;" />

支持多种流量筛选规则，主要体现在 Edges 的 metrics 采集计算上：

<img src="https://picreso.oss-cn-beijing.aliyuncs.com/tcm/image-20220106091914433.png" alt="image-20220106091914433" style="zoom:50%;" />

点击指定 Edge 可以看到 协议以及 metrics，包括：

+ RPS
+ Error:
  + 成功率
  + 响应码比例：3xx, 4xx, 5xx, 2xx
+ Latency：
  + avg
  + p50
  + p90
  + p99

<img src="https://picreso.oss-cn-beijing.aliyuncs.com/tcm/image-20220106092258626.png" alt="image-20220106092258626" style="zoom:50%;" />

可以通过 Flag 标记 response code 比例

<img src="https://picreso.oss-cn-beijing.aliyuncs.com/tcm/image-20220106094445662.png" alt="image-20220106094445662" style="zoom:50%;" />

同时可以根据 hosts 展示 response code 比例

<img src="https://picreso.oss-cn-beijing.aliyuncs.com/tcm/image-20220106092700522.png" alt="image-20220106092700522" style="zoom:50%;" />

支持多种拓扑计算维度（VersionedApp (根据 version label 展示 App)、App、Service、Workload）

<img src="https://picreso.oss-cn-beijing.aliyuncs.com/tcm/image-20220106091749097.png" alt="image-20220106091749097" style="zoom:50%;" />

**支持多种展示方式：**

在 Show Edge Labels 中可以勾选 Response Time (响应时间)、Throughput (速率)、Traffic Distribution (流量分布)、Traffic Rate (每秒请求的次数)，在拓扑图中便会在 edges 上展示这些额外的信息

<img src="https://picreso.oss-cn-beijing.aliyuncs.com/tcm/image-20220106095116557.png" alt="image-20220106095116557" style="zoom:50%;" />

在 Show 中勾选展示方式，如 cluster boxes 即为以 cluster 维度归类拓扑关系，namespace boxes 即为以 namespace 维度归类拓扑关系，下面以 namespace 维度归类为例，现有 n1 n2 两个 ns，展示如下图：

<img src="https://picreso.oss-cn-beijing.aliyuncs.com/tcm/image-20220106095845204.png" alt="image-20220106095845204" style="zoom:50%;" />

（Kiali 在指定 ns 下会展示关联此 ns 的 traffic 源，如上图中在 namespace n5 的 a10 以及 n7 的 a13，标明了入口流量，同时会有 **带箭头紫色的标签** 标注 traffic 源 Node）

还有一些额外的筛选规则如下：

+ 展示响应慢的流量，大于 1 秒
+ 展示不健康的节点
+ 展示无法识别的节点

<img src="https://picreso.oss-cn-beijing.aliyuncs.com/tcm/image-20220106100122892.png" alt="image-20220106100122892" style="zoom:50%;" />

以及隐藏规则如下：

+ 隐藏健康的 nodes，便于快速定位故障
+ 隐藏无法识别的 nodes，去除无关信息

<img src="https://picreso.oss-cn-beijing.aliyuncs.com/tcm/image-20220106100156720.png" alt="image-20220106100156720" style="zoom:50%;" />

Kiali 拓扑图中的形状对应的含义，可点击 Legend 查看：

<img src="https://picreso.oss-cn-beijing.aliyuncs.com/tcm/image-20220106101120269.png" alt="image-20220106101120269" style="zoom:50%;" />

Kiali 拓扑图交互的方式，由以下控制：

<img src="https://picreso.oss-cn-beijing.aliyuncs.com/tcm/image-20220106101223384.png" alt="image-20220106101223384" style="zoom:50%;" />

从左到右分别含义为：

+ 可手动拖拽图标，自定义展示方式

+ 放大/缩小

+ 自动调节到合适的布局（如放大和缩小后，点击会以最合适的视图复原）

+ 三种 Layout 方式：

  + 从上至下排列
  + **中心化排列**：在节点多的情况下，这个展示方式最友好；社区在1月14号会发布一个 zoom 模式，应对大规模的拓扑场景，实现方式是类似于放大镜的效果

  + 从左至右排列

### Application/Workload/Service 治理

<img src="https://picreso.oss-cn-beijing.aliyuncs.com/tcm/image-20220106103337256.png" alt="image-20220106103337256" style="zoom:50%;" />

Kiali 对 App/ Workload/ Service 三种 Node Type 进行汇总，以 Workload 为例，介绍相关功能：

主要功能有：

+ Overview
+ Traffic
+ Logs（只有 Workload 有，Service 和 Application 没有）
+ Inbound Metrics
+ Outbound Metrics（Service 没有 Outbound 只有 Inbound）
+ Envoy（只有 Workload 有，Service 和 Application 没有）
+ Trace （默认无，仅当配置并开启 Jaeger 后会展示）

#### Overview

展示此 Node 的信息和健康状态以及相关拓扑关系

<img src="https://picreso.oss-cn-beijing.aliyuncs.com/tcm/image-20220106103320954.png" alt="image-20220106103320954" style="zoom:50%;" />

支持以某个特定 Node 为维度展示其 Graph（之前是以 Namespace 维度），如上图所示

同时在这种展示方式下，支持两种交互方式：

<img src="https://picreso.oss-cn-beijing.aliyuncs.com/tcm/image-20220106102651586.png" alt="image-20220106102651586" style="zoom:30%;" />

+ full graph：本质上是展示了这个节点所在 Namespace 的拓扑图（回到了 Namespace 维度的拓扑）
+ node graph：本质上是调用同一个接口，只是展示方式变化了，并且可以像 NS 维度那样添加更多的筛选规则，如下图

<img src="https://picreso.oss-cn-beijing.aliyuncs.com/tcm/image-20220106103656660.png" alt="image-20220106103656660" style="zoom:50%;" />

#### Traffic 

展示此 Node 关联的 Node 的流量信息：

<img src="https://picreso.oss-cn-beijing.aliyuncs.com/tcm/image-20220106104059009.png" alt="image-20220106104059009" style="zoom:50%;" />

#### Logs

展示此工作负载的Pod日志，容器级别（如下图展示，可以看到 topoee 容器 和 istio-proxy 的日志）：

<img src="https://picreso.oss-cn-beijing.aliyuncs.com/tcm/image-20220106104216890.png" alt="image-20220106104216890" style="zoom:50%;" />

#### Inbound/Outbound Metrics

加入 Dashboard 展示此 Node 的 Metrics（Kiali 自带，非 Grafana 嵌入）

![image-20220106104348538](https://picreso.oss-cn-beijing.aliyuncs.com/tcm/image-20220106104348538.png)

支持多种筛选规则：

<img src="https://picreso.oss-cn-beijing.aliyuncs.com/tcm/image-20220106104519306.png" alt="image-20220106104519306" style="zoom:30%;" />

#### Envoy

展示 Envoy 的相关配置、状态、Cluster、Listener、Routes等信息：

<img src="https://picreso.oss-cn-beijing.aliyuncs.com/tcm/image-20220106105022613.png" alt="image-20220106105022613" style="zoom:50%;" />

### IstioConfig

<img src="https://picreso.oss-cn-beijing.aliyuncs.com/tcm/image-20220106105123899.png" alt="image-20220106105123899" style="zoom:50%;" />

这一栏主要是和 Istio 交互的，功能如下：

+ 查看/修改/删除/新增 IstioConfig
+ 支持新建的资源类型：

<img src="https://picreso.oss-cn-beijing.aliyuncs.com/tcm/image-20220106105253918.png" alt="image-20220106105253918" style="zoom:40%;" />



## Kiali 源码解析

### Kiali 编译与运行

Kiali 的运行，有官方的 scripts 和 makefile 方便对 Kiali 进行编译和本地运行（同样依赖 Cluster API 和 Prometheus）

``` makefile
# Checkout the source code
mkdir kiali_sources
cd kiali_sources
export KIALI_SOURCES=$(realpath .)

git clone https://github.com/kiali/kiali.git
git clone https://github.com/kiali/kiali-ui.git
git clone https://github.com/kiali/kiali-operator.git
git clone https://github.com/kiali/helm-charts.git

ln -s $KIALI_SOURCES/kiali-operator kiali/operator

# Build the back-end and run the tests
cd $KIALI_SOURCES/kiali
make build test

# Build the front-end and run the tests
cd $KIALI_SOURCES/kiali-ui

# Run Kiali Locally
cd $KIALI_SOURCES/kiali/hack
./run-kiali.sh
```

> 本地运行主要是做 Local Development，如果直接使用 Kiali，最好直接在集群内部署

Kiali 运行实质上执行的命令是：

``` shell
/opt/kiali/kiali -config /kiali-configuration/config.yaml
```

对 Kiali 的配置，也是在 `config.yaml` 中实现的，后面在**部署方案设计**会详细解释。

### Topology 计算原理

先归纳一下 Kiali Topology 计算支持的类型：

+ 基于 Namespace 的 Topology 计算
+ 基于 App/VersionedApp 的 Topology 计算
+ 基于 Service 的 Topology 计算
+ 基于 Workload 的 Topology 计算

以 Namespace 的拓扑计算原理作为例子，其他逻辑类似，讲解相关源码：

#### 基于 Namespace 的 Topology 计算

Routes：`/api/namespaces/graph`

Response Structure:

``` json
{
    "timestamp": 1641408246,
    "duration": 60,
    "graphType": "versionedApp",
    "elements": {
        "nodes": [
            {
                "data": {
                    "id": "6715e1a6fa64f8bfeba0ea6629ce3823",
                    "nodeType": "app",
                    "cluster": "Kubernetes",
                    "namespace": "n1",
                    "workload": "a1-v1",
                    "app": "a1",
                    "version": "v1",
                    "traffic": [
                        {
                            "protocol": "http",
                            "rates": {
                                "httpOut": "9.74"
                            }
                        }
                    ],
                    "isRoot": true
                }
            }      
        ],
        "edges": [
            {
                "data": {
                    "id": "4d3adf06ecb87e3dc987adf9bd31d1e1",
                    "source": "05b3aab4b317f5137620a92ffb723cfd",
                    "target": "0fc8c8b4808e98a08990079cb8559bdc",
                    "traffic": {
                        "protocol": "http",
                        "rates": {
                            "http": "1.91",
                            "http3xx": "0.96",
                            "httpPercentReq": "19.7"
                        },
                        "responses": {
                            "200": {
                                "flags": {
                                    "-": "50.0"
                                },
                                "hosts": {
                                    "a7.n1.svc.cluster.local": "50.0"
                                }
                            },
                            "301": {
                                "flags": {
                                    "-": "50.0"
                                },
                                "hosts": {
                                    "a7.n1.svc.cluster.local": "50.0"
                                }
                            }
                        }
                    }
                }
            },
        ]
    }
}
```

Graph 是由两部分构成：Node 即节点（可以为以下几种类型：Service、App、VersionedApp、Workload），Edge 即 Node 间的关系与流量，响应结构体中，包含了每个 `Node` 和 `Edege` 的信息。

1. 对应路由

Path: `/routing/routes.go`

``` go
		// swagger:route GET /namespaces/graph graphs graphNamespaces
		// ---
		// The backing JSON for a namespaces graph.
		//
		//     Produces:
		//     - application/json
		//
		//     Schemes: http, https
		//
		// responses:
		//      400: badRequestError
		//      500: internalError
		//      200: graphResponse
		//
		{
			"GraphNamespaces",
			"GET",
			"/api/namespaces/graph",
			handlers.GraphNamespaces,
			true,
		},
```

2. 路由对应 Handlers，获取参数和 Client 用于计算 Namespace 的 Graph

**Func(): GraphNamespaces**

Path:  `/handlers/graph.go`

``` go
// GraphNamespaces is a REST http.HandlerFunc handling graph generation for 1 or more namespaces
func GraphNamespaces(w http.ResponseWriter, r *http.Request) {
	defer handlePanic(w)
	# 传入 Graph 计算的相关参数
	o := graph.NewOptions(r)
	# 通过 Request 中 AuthInfo 来创建 Kubernetes/Prometheus/Jaeger 的 ClientFactory 建立连接和交互
	business, err := getBusiness(r)
	graph.CheckError(err)
	# 传入相关参数和 Client 去计算 Graph
	code, payload := api.GraphNamespaces(business, o)
	respond(w, code, payload)
}
```

3. 根据提供的 Options 计算指定 Namespace 下的拓扑图

**Func(): GraphNamespaces**

Path: `/graph/api/api.go`

``` go
// GraphNamespaces 根据提供的 Options 计算指定 Namespace 下的拓扑图
func GraphNamespaces(business *business.Layer, o graph.Options) (code int, config interface{}) {
	// time 生成拓扑图的时间
	promtimer := internalmetrics.GetGraphGenerationTimePrometheusTimer(o.GetGraphKind(), o.TelemetryOptions.GraphType, o.InjectServiceNodes)
	defer promtimer.ObserveDuration()
	// 判断 Mesh 是否支持，o.TelemetryVendor 默认为 Istio
	switch o.TelemetryVendor {
	case graph.VendorIstio:
		prom, err := prometheus.NewClient()
		graph.CheckError(err)
  // 通过 graphNamespacesIstio 计算 Graph 返回 Config
		code, config = graphNamespacesIstio(business, prom, o)
	default:
		graph.Error(fmt.Sprintf("TelemetryVendor [%s] not supported", o.TelemetryVendor))
	}

	// 更新 metrics
	internalmetrics.SetGraphNodes(o.GetGraphKind(), o.TelemetryOptions.GraphType, o.InjectServiceNodes, 0)

	return code, config
}
```

4. 计算 trafficMap 并通过 Options 生成指定的 Graph

**Func(): graphNamespacesIstio**

Path: `/graph/api/api.go`

``` go
// graphNamespacesIstio provides a test hook that accepts mock clients
func graphNamespacesIstio(business *business.Layer, prom *prometheus.Client, o graph.Options) (code int, config interface{}) {

	// 创建全局对象存储 business
	globalInfo := graph.NewAppenderGlobalInfo()
  // 创建全局对象存储 business
	globalInfo.Business = business
	// 基于 Mesh 类型、prometheus Client 和 全局对象 生成 trafficMap
	trafficMap := istio.BuildNamespacesTrafficMap(o.TelemetryOptions, prom, globalInfo)
  // 基于 Options 里面的参数通过 trafficMap 生成指定的 Graph
	code, config = generateGraph(trafficMap, o)

	return code, config
}
```

5. 生成 TrafficMap 原理

TrafficMap 的结构：

```go
type TrafficMap map[string]*Node // 底层为 Map
```

Node 的结构：

``` go
type Node struct {
	ID        string   // ID：Node 的唯一标识，用于 Edge 间计算关系
	NodeType  string   // Node 的类型：App、VersionedApp、Service、Workload
	Cluster   string   // Cluster：集群名
	Namespace string   // Namespace：命名空间
	Workload  string   // Workload name：工作负载名
	App       string   // Workload app label value：工作负载 App label 值
	Version   string   // Workload version label value 工作负载 Version label 值
	Service   string   // Service name：Service 名
	Edges     []*Edge  // Edge 的集合
	Metadata  Metadata // 底层为 map[MetadataKey]interface{} MetadataKey 为 string 类型
}
```

Edge 的结构：

``` go
type Edge struct {
	Source   *Node		// 来源 Node 的指针
	Dest     *Node		// 目的 Node 的指针
	Metadata Metadata // 底层为 map[MetadataKey]interface{} MetadataKey 为 string 类型
}
```

**Func(): BuildNamespacesTrafficMap** 生成每个 NS 下 TrafficMap 然后合并，同时根据 Appender 增/减 额外信息

Path: `/graph/telemetry/istio/istio.go`

``` go
// BuildNamespacesTrafficMap
func BuildNamespacesTrafficMap(o graph.TelemetryOptions, client *prometheus.Client, globalInfo *graph.AppenderGlobalInfo) graph.TrafficMap {
	log.Tracef("Build [%s] graph for [%d] namespaces [%v]", o.GraphType, len(o.Namespaces), o.Namespaces)

	appenders := appender.ParseAppenders(o)
	trafficMap := graph.NewTrafficMap()

	for _, namespace := range o.Namespaces {
		log.Tracef("Build traffic map for namespace [%v]", namespace)
    // 生成某 namespace 下 namespaceTrafficMap
		namespaceTrafficMap := buildNamespaceTrafficMap(namespace.Name, o, client)
    // 获取某 namespace 下的信息
		namespaceInfo := graph.NewAppenderNamespaceInfo(namespace.Name)
    // 根据 appenders 在生成的 namespaceTrafficMap 添加额外信息
		for _, a := range appenders {
			appenderTimer := internalmetrics.GetGraphAppenderTimePrometheusTimer(a.Name())
			a.AppendGraph(namespaceTrafficMap, globalInfo, namespaceInfo)
			appenderTimer.ObserveDuration()
		}
    // 合并所有 namespace 下的 trafficMaps
		telemetry.MergeTrafficMaps(trafficMap, namespace.Name, namespaceTrafficMap)
	}
	// 标记请求 namespaces 之外的 Nodes
	telemetry.MarkOutsideOrInaccessible(trafficMap, o)
	// 标记请求 namespaces 之内的 Nodes
	telemetry.MarkTrafficGenerators(trafficMap)
  // 如果 NodeType 为 Service，在 trafficMap 中删除不需要的信息
	if graph.GraphTypeService == o.GraphType {
		trafficMap = telemetry.ReduceToServiceGraph(trafficMap)
	}

	return trafficMap
}
```

Appender 是一个接口，可以根据需要在 graph 中注入更详细的信息，它的定义如下：

``` go
// Appender is implemented by any code offering to append a service graph with
// supplemental information.  On error the appender should panic and it will be
// handled as an error response.
type Appender interface {
	// AppendGraph performs the appender work on the provided traffic map. The map
	// may be initially empty. An appender is allowed to add or remove map entries.
	AppendGraph(trafficMap TrafficMap, globalInfo *AppenderGlobalInfo, namespaceInfo *AppenderNamespaceInfo)

	// Name returns a unique appender name and which is the name used to identify the appender (e.g in 'appenders' query param)
	Name() string
}
```

在 /graph/telemetry/istio/appender 下，一共有如下实现：

+ AggregateNodeAppender
+ DeadNodeAppender 
+ HealthConfigAppender 
+ IdleNodeAppender 
+ IstioAppender 
+ ResponseTimeAppender 
+ SecurityPolicyAppender 
+ ServiceEntryAppender 
+ SidecarsCheckAppender
+ ThroughputAppender
+ WorkloadEntryAppender 

**Func(): buildNamespaceTrafficMap** 通过 构造 PromQL 查询 Istio 默认 Metrics

Path: `/graph/telemetry/istio/istio.go`

``` go
func buildNamespaceTrafficMap(namespace string, o graph.TelemetryOptions, client *prometheus.Client) graph.TrafficMap {
	// create map to aggregate traffic by protocol and response code
	trafficMap := graph.NewTrafficMap()
	duration := o.Namespaces[namespace].Duration
	idleCondition := "> 0"
	if o.IncludeIdleEdges {
		idleCondition = ""
	}

	// HTTP/GRPC 流量
	if o.Rates.Http == graph.RateRequests || o.Rates.Grpc == graph.RateRequests {
		metric := "istio_requests_total"
		groupBy := "source_cluster,source_workload_namespace,source_workload,source_canonical_service,source_canonical_revision,destination_cluster,destination_service_namespace,destination_service,destination_service_name,destination_workload_namespace,destination_workload,destination_canonical_service,destination_canonical_revision,request_protocol,response_code,grpc_response_status,response_flags"

		// 0) Incoming: 从源 istio-proxy 获取遥测数据去捕捉 Namespace 下无 Services 的 inbound 流量特征
		query := fmt.Sprintf(`sum(rate(%s{reporter="source",source_workload_namespace!="%s",destination_workload_namespace="unknown",destination_workload="unknown",destination_service=~"^.+\\.%s\\..+$"} [%vs])) by (%s) %s`,
			metric,
			namespace,
			namespace,
			int(duration.Seconds()), // range duration for the query
			groupBy,
			idleCondition)
		incomingVector := promQuery(query, time.Unix(o.QueryTime, 0), client.API())
		populateTrafficMap(trafficMap, &incomingVector, metric, o)

		// 1) Incoming: 从目的 istio-proxy 获取遥测数据去捕捉 Namespace 下 Services 的 inbound 流量特征
		query = fmt.Sprintf(`sum(rate(%s{reporter="destination",destination_workload_namespace="%s"} [%vs])) by (%s) %s`,
			metric,
			namespace,
			int(duration.Seconds()), // range duration for the query
			groupBy,
			idleCondition)
		incomingVector = promQuery(query, time.Unix(o.QueryTime, 0), client.API())
		populateTrafficMap(trafficMap, &incomingVector, metric, o)

		// 2) Outgoing: 从源 istio-proxy 获取遥测数据去捕捉 Namespace 下 Workloads 的 outbound 流量特征
		query = fmt.Sprintf(`sum(rate(%s{reporter="source",source_workload_namespace="%s"} [%vs])) by (%s) %s`,
			metric,
			namespace,
			int(duration.Seconds()), // range duration for the query
			groupBy,
			idleCondition)
		outgoingVector := promQuery(query, time.Unix(o.QueryTime, 0), client.API())
		populateTrafficMap(trafficMap, &outgoingVector, metric, o)
	}

	// GRPC 流量
	if o.Rates.Grpc != graph.RateNone && o.Rates.Grpc != graph.RateRequests {
		var metrics []string
		groupBy := "source_cluster,source_workload_namespace,source_workload,source_canonical_service,source_canonical_revision,destination_cluster,destination_service_namespace,destination_service,destination_service_name,destination_workload_namespace,destination_workload,destination_canonical_service,destination_canonical_revision"

		switch o.Rates.Grpc {
		case graph.RateReceived:
			metrics = []string{"istio_response_messages_total"}
		case graph.RateSent:
			metrics = []string{"istio_request_messages_total"}
		case graph.RateTotal:
			metrics = []string{"istio_request_messages_total", "istio_response_messages_total"}
		default:
			metrics = []string{}
		}

		for _, metric := range metrics {
			// 0) Incoming: 从源 istio-proxy 获取遥测数据去捕捉 Namespace 下无 Services 的 inbound 流量特征
			query := fmt.Sprintf(`sum(rate(%s{reporter="source",source_workload_namespace!="%s",destination_workload_namespace="unknown",destination_workload="unknown",destination_service=~"^.+\\.%s\\..+$"} [%vs])) by (%s) %s`,
				metric,
				namespace,
				namespace,
				int(duration.Seconds()), // range duration for the query
				groupBy,
				idleCondition)
			incomingVector := promQuery(query, time.Unix(o.QueryTime, 0), client.API())
			populateTrafficMap(trafficMap, &incomingVector, metric, o)

			// 1) Incoming: 从目的 istio-proxy 获取遥测数据去捕捉 Namespace 下 Services 的 inbound 流量特征
			query = fmt.Sprintf(`sum(rate(%s{reporter="destination",destination_workload_namespace="%s"} [%vs])) by (%s) %s`,
				metric,
				namespace,
				int(duration.Seconds()), // range duration for the query
				groupBy,
				idleCondition)
			incomingVector = promQuery(query, time.Unix(o.QueryTime, 0), client.API())
			populateTrafficMap(trafficMap, &incomingVector, metric, o)

			// 2) Outgoing: 从源 istio-proxy 获取遥测数据去捕捉 Namespace 下 Workloads 的 outbound 流量特征
			query = fmt.Sprintf(`sum(rate(%s{reporter="source",source_workload_namespace="%s"} [%vs])) by (%s) %s`,
				metric,
				namespace,
				int(duration.Seconds()), // range duration for the query
				groupBy,
				idleCondition)
			outgoingVector := promQuery(query, time.Unix(o.QueryTime, 0), client.API())
			populateTrafficMap(trafficMap, &outgoingVector, metric, o)
		}
	}

	// TCP 流量
	if o.Rates.Tcp != graph.RateNone {
		var metrics []string
		groupBy := "source_cluster,source_workload_namespace,source_workload,source_canonical_service,source_canonical_revision,destination_cluster,destination_service_namespace,destination_service,destination_service_name,destination_workload_namespace,destination_workload,destination_canonical_service,destination_canonical_revision,response_flags"

		switch o.Rates.Tcp {
		case graph.RateReceived:
			metrics = []string{"istio_tcp_received_bytes_total"}
		case graph.RateSent:
			metrics = []string{"istio_tcp_sent_bytes_total"}
		case graph.RateTotal:
			metrics = []string{"istio_tcp_sent_bytes_total", "istio_tcp_received_bytes_total"}
		default:
			metrics = []string{}
		}

		for _, metric := range metrics {
			// 0) Incoming: 从源 istio-proxy 获取遥测数据去捕捉 Namespace 下无 Services 的 inbound 流量特征
			query := fmt.Sprintf(`sum(rate(%s{reporter="source",source_workload_namespace!="%s",destination_workload_namespace="unknown",destination_workload="unknown",destination_service=~"^.+\\.%s\\..+$"} [%vs])) by (%s) %s`,
				metric,
				namespace,
				namespace,
				int(duration.Seconds()), // range duration for the query
				groupBy,
				idleCondition)
			incomingVector := promQuery(query, time.Unix(o.QueryTime, 0), client.API())
			populateTrafficMap(trafficMap, &incomingVector, metric, o)

			// 1) Incoming: 从目的 istio-proxy 获取遥测数据去捕捉 Namespace 下 Services 的 inbound 流量特征
			query = fmt.Sprintf(`sum(rate(%s{reporter="destination",destination_workload_namespace="%s"} [%vs])) by (%s) %s`,
				metric,
				namespace,
				int(duration.Seconds()), // range duration for the query
				groupBy,
				idleCondition)
			incomingVector = promQuery(query, time.Unix(o.QueryTime, 0), client.API())
			populateTrafficMap(trafficMap, &incomingVector, metric, o)

			// 2) Outgoing: 从源 istio-proxy 获取遥测数据去捕捉 Namespace 下 Workloads 的 outbound 流量特征
			query = fmt.Sprintf(`sum(rate(%s{reporter="source",source_workload_namespace="%s"} [%vs])) by (%s) %s`,
				metric,
				namespace,
				int(duration.Seconds()), // range duration for the query
				groupBy,
				idleCondition)
			outgoingVector := promQuery(query, time.Unix(o.QueryTime, 0), client.API())
			populateTrafficMap(trafficMap, &outgoingVector, metric, o)
		}
	}

	return trafficMap
}
```



### Metrics 计算原理

先归纳一下 Kiali Metrics 计算支持的类型：

+ 基于 Namespace 的 Metrics 计算
+ 基于 App/VersionedApp 的 Metrics 计算
+ 基于 Service 的 Metrics 计算
+ 基于 Workload 的 Metrics 计算

就以 Workload Metrics 计算为例，其他逻辑类似，讲解相关源码：

Path: `/routing/routes.go`

1. Routes

``` go
		// swagger:route GET /namespaces/{namespace}/workloads/{workload}/metrics workloads workloadMetrics
		// ---
		// Endpoint to fetch metrics to be displayed, related to a single workload
		//
		//     Produces:
		//     - application/json
		//
		//     Schemes: http, https
		//
		// responses:
		//      400: badRequestError
		//      503: serviceUnavailableError
		//      200: metricsResponse
		//
		{
			"WorkloadMetrics",
			"GET",
			"/api/namespaces/{namespace}/workloads/{workload}/metrics",
			handlers.WorkloadMetrics,
			true,
		},
```

2. Handler

Path: `/handlers/metrics.go`

``` go
// WorkloadMetrics is the API handler to fetch metrics to be displayed, related to a single workload
func WorkloadMetrics(w http.ResponseWriter, r *http.Request) {
	getWorkloadMetrics(w, r, defaultPromClientSupplier)
}

// getWorkloadMetrics 在 request 中获取参数
func getWorkloadMetrics(w http.ResponseWriter, r *http.Request, promSupplier promClientSupplier) {
	vars := mux.Vars(r)
	namespace := vars["namespace"]
	workload := vars["workload"]

	metricsService, namespaceInfo := createMetricsServiceForNamespace(w, r, promSupplier, namespace)
	if metricsService == nil {
		// any returned value nil means error & response already written
		return
	}
	// 构造查询的 struct IstioMetricsQuery
	params := models.IstioMetricsQuery{Namespace: namespace, Workload: workload}
  // 补充缺省值并检验查询的 struct IstioMetricsQuery
	err := extractIstioMetricsQueryParams(r, &params, namespaceInfo)
	if err != nil {
		RespondWithError(w, http.StatusBadRequest, err.Error())
		return
	}
	// 通过构造的 struct 查询 metrics
	metrics, err := metricsService.GetMetrics(params, nil)
	if err != nil {
		RespondWithError(w, http.StatusServiceUnavailable, err.Error())
		return
	}
  // 返回 metrics 的结构
	RespondWithJSON(w, http.StatusOK, metrics)
}
```

3. Metrics 计算

Path: `/models/metrics.go`

Metrics 结构体

``` go
type Metric struct {
	Labels     map[string]string `json:"labels"`
	Datapoints []Datapoint       `json:"datapoints"`
	Stat       string            `json:"stat,omitempty"`
	Name       string            `json:"name"`
}

type Datapoint struct {
	Timestamp int64
	Value     float64
}
```

Path: `/business/metrics.go`

获取 Metric

``` go
// 根据 Label 和 IstioMetricsQuery 获取 Metrics
func (in *MetricsService) GetMetrics(q models.IstioMetricsQuery, scaler func(n string) float64) (models.MetricsMap, error) {
	lb := createMetricsLabelsBuilder(&q)
	grouping := strings.Join(q.ByLabels, ",")
	return in.fetchAllMetrics(q, lb, grouping, scaler)
}
// 生成 Metrics 的 Labels
func createMetricsLabelsBuilder(q *models.IstioMetricsQuery) *MetricsLabelsBuilder {
	lb := NewMetricsLabelsBuilder(q.Direction)
	lb.Reporter(q.Reporter)

	namespaceSet := false
	if q.Service != "" {
		lb.Service(q.Service, q.Namespace)
		namespaceSet = true
	}
	if q.Workload != "" {
		lb.Workload(q.Workload, q.Namespace)
		namespaceSet = true
	}
	if q.App != "" {
		lb.App(q.App, q.Namespace)
		namespaceSet = true
	}
	if !namespaceSet && q.Namespace != "" {
		lb.Namespace(q.Namespace)
	}
	if q.RequestProtocol != "" {
		lb.Protocol(q.RequestProtocol)
	}
	if q.Aggregate != "" {
		lb.Aggregate(q.Aggregate, q.AggregateValue)
	}
	return lb
}
// 获取所有 Metrics
func (in *MetricsService) fetchAllMetrics(q models.IstioMetricsQuery, lb *MetricsLabelsBuilder, grouping string, scaler func(n string) float64) (models.MetricsMap, error) {
	labels := lb.Build()
	labelsError := lb.BuildForErrors()

	var wg sync.WaitGroup
	fetchRate := func(p8sFamilyName string, metric *prometheus.Metric, lbl []string) {
		defer wg.Done()
		m := in.prom.FetchRateRange(p8sFamilyName, lbl, grouping, &q.RangeQuery)
		*metric = m
	}

	fetchHisto := func(p8sFamilyName string, histo *prometheus.Histogram) {
		defer wg.Done()
		h := in.prom.FetchHistogramRange(p8sFamilyName, labels, grouping, &q.RangeQuery)
		*histo = h
	}

	type resultHolder struct {
		metric     prometheus.Metric
		histo      prometheus.Histogram
		definition istioMetric
	}
	maxResults := len(istioMetrics)
	if len(q.Filters) != 0 {
		maxResults = len(q.Filters)
	}
	results := make([]*resultHolder, maxResults)

	for _, istioMetric := range istioMetrics {
		// 通过 Filters 做筛选，如果 Filter 为空，获取所有 Metrics
		doFetch := len(q.Filters) == 0
		if !doFetch {
			for _, filter := range q.Filters {
				if filter == istioMetric.kialiName {
					doFetch = true
					break
				}
			}
		}
		if doFetch {
			wg.Add(1)
			result := resultHolder{definition: istioMetric}
			results = append(results, &result)
			if istioMetric.isHisto {
				go fetchHisto(istioMetric.istioName, &result.histo)
			} else {
				labelsToUse := istioMetric.labelsToUse(labels, labelsError)
				go fetchRate(istioMetric.istioName, &result.metric, labelsToUse)
			}
		}
	}
	wg.Wait()

	// 每个 Reporter 返回一个 Metrics
	metrics := make(models.MetricsMap)
	for _, result := range results {
		if result != nil {
			conversionParams := models.ConversionParams{Scale: 1.0}
			if scaler != nil {
				scale := scaler(result.definition.kialiName)
				if scale != 0.0 {
					conversionParams.Scale = scale
				}
			}
			var converted []models.Metric
			var err error
			if result.definition.isHisto {
				converted, err = models.ConvertHistogram(result.definition.kialiName, result.histo, conversionParams)
				if err != nil {
					return nil, err
				}
			} else {
				converted, err = models.ConvertMetric(result.definition.kialiName, result.metric, conversionParams)
				if err != nil {
					return nil, err
				}
			}
			metrics[result.definition.kialiName] = append(metrics[result.definition.kialiName], converted...)
		}
	}
	return metrics, nil
}
```
