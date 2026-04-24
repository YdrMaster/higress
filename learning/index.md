# Higress 项目拆解文档

> 本文档面向希望了解 Higress AI 网关的同学，从"它能做什么"开始，逐步深入到"它是怎么做的"。前半部分侧重功能、行为与使用视角，后半部分进入架构与实现细节。
>
> 本文写作时参考了 AIGW 项目 `learning/index.md` 的拆解风格，力求在结构、深度与可读性上保持一致，方便读者将 Higress 与 AIGW、Inference Extension 放在一起对比学习。

- [一、项目概述](#一项目概述)
  - [1.1 项目定位](#11-项目定位)
  - [1.2 解决的核心问题](#12-解决的核心问题)
  - [1.3 核心功能矩阵](#13-核心功能矩阵)
- [二、核心概念与术语](#二核心概念与术语)
  - [2.1 概念全景图](#21-概念全景图)
  - [2.2 关键术语详解](#22-关键术语详解)
- [三、端到端请求生命周期](#三端到端请求生命周期)
  - [3.1 整体流程概览](#31-整体流程概览)
  - [3.2 六步请求流详解](#32-六步请求流详解)
- [四、架构分层与组件关系](#四架构分层与组件关系)
  - [4.1 整体架构图](#41-整体架构图)
  - [4.2 Higress Controller](#42-higress-controller)
  - [4.3 Higress Gateway](#43-higress-gateway)
  - [4.4 Higress Console](#44-higress-console)
- [五、插件体系详解](#五插件体系详解)
  - [5.1 插件体系全景](#51-插件体系全景)
  - [5.2 Wasm 插件机制](#52-wasm-插件机制)
  - [5.3 Golang Filter 机制](#53-golang-filter-机制)
  - [5.4 AI 插件矩阵](#54-ai-插件矩阵)
- [六、核心 AI 能力详解](#六核心-ai-能力详解)
  - [6.1 AI 代理（ai-proxy）](#61-ai-代理ai-proxy)
  - [6.2 AI 负载均衡（ai-load-balancer）](#62-ai-负载均衡ai-load-balancer)
  - [6.3 模型路由（model-router）](#63-模型路由model-router)
  - [6.4 AI 缓存（ai-cache）](#64-ai-缓存ai-cache)
  - [6.5 MCP 服务（mcp-server / mcp-router）](#65-mcp-服务mcp-server--mcp-router)
- [七、配置体系与 API 资源](#七配置体系与-api-资源)
  - [7.1 双模式配置体系](#71-双模式配置体系)
  - [7.2 核心 CRD 资源](#72-核心-crd-资源)
  - [7.3 WasmPlugin 配置模型](#73-wasmplugin-配置模型)
- [八、高可用与扩展性设计](#八高可用与扩展性设计)
  - [8.1 高可用机制](#81-高可用机制)
  - [8.2 插件扩展机制](#82-插件扩展机制)
  - [8.3 多语言插件开发](#83-多语言插件开发)
- [九、与 AIGW / Inference Extension 的对比](#九与-aigw--inference-extension-的对比)
  - [9.1 架构范式差异](#91-架构范式差异)
  - [9.2 功能定位差异](#92-功能定位差异)
  - [9.3 适用场景差异](#93-适用场景差异)
- [十、生态与实现现状](#十生态与实现现状)
- [十一、总结与入门建议](#十一总结与入门建议)

## 一、项目概述

### 1.1 项目定位

**Higress** 是**阿里云开源的云原生 API 网关和 AI 网关**，基于 Envoy 和 Istio 进行二次定制化开发构建和功能增强。它定位为**下一代云原生网关**，以 **Envoy** 作为数据面代理，以 **Istio** 作为控制面基础设施，同时提供 Web 控制台和 Admin SDK 用于可视化管理和程序化配置。

Higress 的**核心设计哲学**是在同一套 Envoy + Istio 基础设施上同时承载**传统流量**和**AI 流量**。它的 Wasm 插件生态中既有 ai-proxy、ai-cache、ai-load-balancer 等 AI 专用插件，也有认证、限流、WAF、灰度发布等大量传统网关插件。这意味着企业可以先把 Higress 作为普通云原生 API 网关使用，再按需叠加 AI 能力，而无需更换整个网关架构。AI 能力是 Higress 相对其他云原生网关的增量优势，而非使用它的前提条件。

| 维度 | 说明 |
| ------ | ------ |
| **项目归属** | 阿里云开源（alibaba/higress） |
| **Stars** | 8k+（GitHub） |
| **技术框架** | Envoy（数据面）+ Istio（控制面）+ 自研 Controller |
| **核心定位** | 云原生 API 网关 + AI 网关（双重定位） |
| **扩展机制** | Wasm 插件（C++/Go/Rust/AssemblyScript）+ Golang HTTP Filter |
| **协议支持** | HTTP/1.1、HTTP/2、gRPC、Dubbo、OpenAI API、Claude API |
| **部署方式** | Kubernetes（Helm）、Docker 独立部署 |
| **开发语言** | Go（Controller + Golang Filter）、C++（Wasm）、Rust（Wasm） |

**一句话概括**：Higress = Envoy（流量基础设施） + Istio（控制面） + 自研 Controller（Ingress/Wasm 管理） + Wasm 插件生态（AI 能力） + Web 控制台

### 1.2 解决的核心问题

在 Kubernetes 上运行 AI 服务和传统微服务时，企业面临的典型问题：

| 挑战 | 传统方案的问题 | Higress 的解决方案 |
| ------ | -------------- | ------------------- |
| **多厂商 AI API 统一** | 每个厂商协议不同，客户端需要适配多种 SDK | ai-proxy 插件：自动协议转换，统一 OpenAI API 入口 |
| **AI 流量智能调度** | 传统轮询不感知 LLM 推理状态 | ai-load-balancer 插件：前缀缓存感知、GPU 指标感知、全局最小连接 |
| **LLM 结果缓存** | 相同问题重复调用，浪费 Token 成本 | ai-cache 插件：语义化缓存 + 字符串匹配缓存 |
| **模型路由** | 需要手动维护路由规则 | model-router 插件：自动提取 model 参数路由 + 基于内容自动选模型 |
| **MCP 服务托管** | Model Context Protocol 服务管理复杂 | mcp-server / mcp-router 插件：内置 MCP 托管和路由 |
| **传统网关功能缺失** | Nginx/Kong 缺少 Wasm 扩展和 AI 能力 | Wasm 插件体系：认证、限流、WAF、灰度发布等全套功能 |
| **Istio 运维复杂** | Istio 学习曲线陡峭，Sidecar 资源开销大 | 精简版 Istio 控制面 + Ingress 语义化简化配置 |

### 1.3 核心功能矩阵

| 功能项 | 功能域 | 实现方式 | 说明 |
| -------- | -------- | ---------- | ------ |
| **多厂商协议转换** | AI 代理 | ai-proxy Wasm 插件 | 支持 30+ AI 提供商，自动 OpenAI/Claude 协议互转 |
| **前缀缓存感知调度** | AI 负载均衡 | ai-load-balancer Wasm 插件 | 基于 prompt 前缀匹配选择后端，复用 KV Cache |
| **GPU 指标感知调度** | AI 负载均衡 | ai-load-balancer Wasm 插件 | 拉取 vLLM metrics，选择队列最短的 Pod |
| **全局最小连接** | AI 负载均衡 | ai-load-balancer Wasm 插件 | 基于 Redis 实现跨实例全局最小请求数 |
| **模型名自动提取** | 模型路由 | model-router Wasm 插件 | 从请求体提取 model 参数注入 Header 用于路由 |
| **内容自动路由** | 模型路由 | model-router Wasm 插件 | 根据用户消息内容正则匹配选择模型 |
| **语义化缓存** | AI 缓存 | ai-cache Wasm 插件 | 基于向量数据库的相似问题缓存 |
| **字符串匹配缓存** | AI 缓存 | ai-cache Wasm 插件 | 精确匹配缓存，Redis 等存储 |
| **Token 限流** | Token 管理 | ai-token-ratelimit Wasm 插件 | 基于 Token 消耗量的限流 |
| **AI 内容安全** | 安全防护 | ai-security-guard Wasm 插件 | 输入输出内容审核 |
| **MCP 服务管理** | MCP 托管 | mcp-server（Wasm + Golang Filter） | 托管 Model Context Protocol 服务 |
| **认证鉴权** | 传统网关 | Wasm 插件 | Basic Auth、Key Auth、JWT Auth、HMAC Auth |
| **流量治理** | 传统网关 | Wasm 插件 | 限流、熔断、灰度发布、流量镜像 |
| **可观测性** | 传统网关 | 内置 | Access Log、Prometheus Metrics、Tracing |
| **HTTP 转 RPC** | 协议转换 | Http2Rpc Controller | HTTP 到 Dubbo/gRPC 的协议转换 |

## 二、核心概念与术语

### 2.1 概念全景图

```plaintext
用户请求
    │
    v
┌────────────────────────────────────────────────────────────┐
│                   Higress Gateway                          │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │             Envoy 数据面                             │  │
│  │                                                      │  │
│  │  监听器 → HTTP Router → Wasm 插件链 → Golang Filter  │  │
│  │             │                                        │  │
│  │             ▼                                        │  │
│  │   ┌──────────────────────────────────────────────┐   │  │
│  │   │  Wasm 插件链（可配置多个插件按优先级执行）   │   │  │
│  │   │  - model-router (优先级900)                  │   │  │
│  │   │  - ai-security-guard                         │   │  │
│  │   │  - ai-proxy (优先级100)                      │   │  │
│  │   │  - ai-cache (优先级10)                       │   │  │
│  │   │  - ai-load-balancer                          │   │  │
│  │   │  - ai-token-ratelimit                        │   │  │
│  │   └──────────────────────────────────────────────┘   │  │
│  │             │                                        │  │
│  │             ▼                                        │  │
│  │        上游集群（后端服务）                          │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
    │
    │  xDS (gRPC)
    ▼
┌────────────────────────────────────────────────────────────┐
│              Higress Controller (控制面)                   │
│                                                            │
│  ┌──────────────┐    ┌──────────────────────────────┐      │
│  │  Discovery   │    │       Higress Core           │      │
│  │ (Istio Pilot)│    │  - Ingress Controller        │      │
│  │              │    │  - Gateway Controller        │      │
│  │  Config      │    │  - McpBridge Controller      │      │
│  │  Controller  │    │  - Http2Rpc Controller       │      │
│  │  Service     │    │  - WasmPlugin Controller     │      │
│  │  Controller  │    │  - ConfigmapMgr              │      │
│  └──────────────┘    └──────────────────────────────┘      │
└────────────────────────────────────────────────────────────┘
    │
    │  HTTPS / Watch
    ▼
┌────────────────────────────────────────────────────────────┐
│         Higress Console (管理面)                           │
│         (Web UI + Admin SDK)                               │
└────────────────────────────────────────────────────────────┘
```

### 2.2 关键术语详解

| 术语 | 定义 | 类比 |
| ------ | ------ | ------ |
| **Higress Controller** | Higress 控制面核心，包含 Istio Discovery 和自研 Higress Core 两个子组件 | 网关的"大脑" |
| **Higress Gateway** | Higress 数据面，包含 Pilot Agent 和 Envoy | 网关的"身体" |
| **Higress Console** | Higress 管理控制台，提供 Web UI 和 Admin SDK | 网关的"操作面板" |
| **Discovery** | Istio Pilot-Discovery 组件，负责 xDS 配置下发和服务发现 | Istio 的"神经系统" |
| **Higress Core** | 自研控制器组件，负责 Ingress 转换、Wasm 管理、外部注册中心对接 | Higress 的"特色功能中枢" |
| **Wasm 插件** | 基于 WebAssembly 的 Envoy 扩展，支持 C++/Go/Rust/AssemblyScript | 网关的"可插拔能力模块" |
| **Golang Filter** | 基于 Envoy Golang HTTP Filter 扩展机制，编译为 .so 共享库加载 | 网关的"Go 语言扩展模块" |
| **McpBridge** | Higress 自定义 CRD，用于对接 Nacos/Eureka/Consul/Zookeeper 等外部注册中心 | 服务发现的"桥梁" |
| **WasmPlugin** | Higress 扩展的 CRD，支持路由/域名/服务级别的 Wasm 插件配置 | Wasm 插件的"配置载体" |
| **Http2Rpc** | Higress 自定义 CRD，用于 HTTP 到 RPC 协议的转换 | 协议转换的"翻译器" |

## 三、端到端请求生命周期

### 3.1 整体流程概览

```plaintext
用户请求 (OpenAI ChatCompletion)
    │
    v
[1] Gateway 路由匹配 -> 根据 Host/Path/Header 匹配 HTTPRoute
    │
    v
[2] Wasm 插件链执行 -> 按优先级顺序执行配置的 Wasm 插件
    │   - model-router: 提取 model 参数注入 Header
    │   - ai-security-guard: 内容安全检查
    │   - ai-cache: 查询缓存（命中则直接返回）
    │   - ai-proxy: 协议转换、模型映射、Token 管理
    │   - ai-token-ratelimit: Token 消耗限流
    │   - ai-load-balancer: 智能选点
    │
    v
[3] Golang Filter 执行（如配置了 golang-filter/mcp-server 等）
    │
    v
[4] 请求转发 -> 路由到选定的后端 Pod
    │
    v
[5] 响应处理 -> 反向经过插件链处理响应
    │   - ai-proxy: 响应协议转换
    │   - ai-cache: 缓存响应结果
    │
    v
[6] 返回用户 -> 标准化 OpenAI 格式响应
```

### 3.2 六步请求流详解

**第一步：Gateway 路由匹配**

Higress Gateway（Envoy）根据 Host、Path、Method、Header 等 L7 元信息完成标准路由匹配，将请求送到配置了 AI 插件的入口路由上。此时请求 Body 尚未解析，不涉及任何 AI 语义——模型选择、协议转换等 AI 相关决策由后续 Wasm 插件链负责。

**第二步：Wasm 插件链执行**

Higress 的核心能力来自 Wasm 插件链。每个请求会按**优先级顺序**依次经过配置的 Wasm 插件：

| 插件 | 优先级 | 阶段 | 功能 |
| ------ | -------- | ------ | ------ |
| **model-router** | 900 | 认证阶段 | 从请求体提取 model 参数，按配置注入指定 Header（如 `x-higress-llm-model`） |
| **ai-security-guard** | 300 | 默认阶段 | 对输入内容进行安全审核 |
| **ai-proxy** | 100 | 默认阶段 | 协议转换、模型映射、多厂商代理 |
| **ai-cache** | 10 | 认证阶段 | 查询语义/字符串缓存，命中则直接返回 |
| **ai-token-ratelimit** | - | 默认阶段 | 基于 Token 消耗量进行限流 |
| **ai-load-balancer** | - | 默认阶段 | 前缀缓存感知 / GPU 指标感知选点 |

> 插件执行分为两个阶段：**认证阶段**（在路由决策前执行）和**默认阶段**（在路由决策后执行）。

**第三步：Golang Filter 执行**

如果配置了 Golang Filter（如 golang-filter/mcp-server），请求会在此阶段进入 Go 共享库处理。Golang Filter 通过 Envoy 的 `envoy.filters.http.golang` 过滤器实现，支持：

- 请求/响应头修改
- 请求/响应体修改
- 同步 HTTP 调用
- 独立的 Go 运行时（Goroutine、Channel、GC）

**第四步：请求转发**

经过插件链处理后，请求进入 Envoy Router Filter 阶段。此时 ai-proxy 已完成协议转换和认证头设置，ai-load-balancer 如启用则可能已通过 `SetUpstreamOverrideHost` 指定了具体后端 Pod。Envoy 根据路由配置和插件链的决策将请求转发到目标上游服务。若未启用 ai-load-balancer，则使用 Envoy 内置的负载均衡策略（轮询、随机、最小连接、一致性哈希等）在集群内选择 endpoint。

**第五步：响应处理**

响应沿原路返回，反向经过插件链：

- **ai-proxy**：将后端响应转换为标准 OpenAI 格式
- **ai-cache**：将响应结果存入缓存（语义缓存或字符串缓存）
- **ai-statistics**：记录 Token 消耗、延迟等统计信息

**第六步：返回用户**

最终标准化后的 OpenAI 格式响应返回给用户。

## 四、架构分层与组件关系

### 4.1 整体架构图

```plaintext
┌──────────────────────────────────────────────────────────────────────┐
│                          客户端 / 用户                               │
└──────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                    Higress Gateway (数据面)                    │  │
│  │                                                                │  │
│  │  ┌─────────────┐    ┌─────────────────────────────────────┐    │  │
│  │  │ Pilot Agent │───▶│              Envoy                  │    │  │
│  │  │ (xDS 代理)  │    │                                     │    │  │
│  │  └─────────────┘    │  Listener → Router → Filter Chain   │    │  │
│  │         ▲           │                                     │    │  │
│  │         │ UDS       │  Filter Chain:                      │    │  │
│  │         │           │  1. Wasm 过滤器 (多个插件按优先级)  │    │  │
│  │         │           │  2. Golang HTTP 过滤器              │    │  │
│  │         │           │  3. Router 过滤器                   │    │  │
│  │         │           │                                     │    │  │
│  │         │           └─────────────────────────────────────┘    │  │
│  └────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
                              │
                              │ xDS (gRPC over UDS)
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│  ┌─────────────────────────┐    ┌─────────────────────────────────┐  │
│  │    Discovery (Istio)    │    │       Higress Core              │  │
│  │                         │    │                                 │  │
│  │  Config Controller      │◀───│ Ingress Controller              │  │
│  │  (K8s/MCP/Memory/File)  │    │  - Ingress → Istio CRD 转换     │  │
│  │                         │    │                                 │  │
│  │  Service Controller     │◀───│ Gateway Controller              │  │
│  │  (K8s Registry)         │    │  - Gateway/VS/DR 管理           │  │
│  │                         │    │                                 │  │
│  │  xDS Server             │◀───│ McpBridge Controller            │  │
│  │  (LDS/RDS/CDS/EDS)      │    │  - Nacos/Eureka/Consul/ZK       │  │
│  │                         │    │    → ServiceEntry 转换          │  │
│  │                         │◀───│ Http2Rpc Controller             │  │
│  │                         │    │  - HTTP → Dubbo/gRPC 转换       │  │
│  │                         │◀───│ WasmPlugin Controller           │  │
│  │                         │    │  - WasmPlugin → Istio WasmPlugin│  │
│  │                         │◀───│ ConfigmapMgr                    │  │
│  │                         │    │  - 全局配置管理                 │  │
│  └─────────────────────────┘    └─────────────────────────────────┘  │
│                    Higress Controller (控制面)                       │
└──────────────────────────────────────────────────────────────────────┘
                              │
                              │ Watch / HTTPS
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    Higress Console (管理面)                          │
│                                                                      │
│  - Web UI：路由管理、插件管理、证书管理、服务管理                    │
│  - Admin SDK：Java SDK（JDK 17+），支持程序化配置管理                │
└──────────────────────────────────────────────────────────────────────┘
```

### 4.2 Higress Controller

Higress Controller 是 Higress 的控制面核心，包含两个子组件：

**4.2.1 Discovery 组件（Istio Pilot-Discovery）**

Discovery 是 Istio 的核心组件，Higress 完整复用了其能力：

| 子模块 | 职责 |
| -------- | ------ |
| **Config Controller** | 管理配置来源，支持 Kubernetes CRD、MCP（Mesh Configuration Protocol）、Memory、File 四种模式 |
| **Service Controller** | 管理服务注册中心，支持 Kubernetes Registry 和 Memory Registry |
| **xDS Server** | 将配置转换为 xDS 格式，通过 gRPC 下发到数据面 Envoy |

Discovery 通过 MCP over xDS 协议与 Higress Core 通信，Higress Core 作为 MCP Server 向 Discovery 提供自定义配置（Ingress 转换后的 Istio 配置、Wasm 配置等）。

**4.2.2 Higress Core 组件**

Higress Core 是 Higress 的自研控制器，包含 6 个控制器：

| 控制器 | 职责 | 说明 |
| -------- | ------ | ------ |
| **Ingress Controller** | 监听 Ingress 资源，自动转换为 Istio Gateway + VirtualService + DestinationRule | 兼容 K8s Ingress 标准 |
| **Gateway Controller** | 监听 Gateway、VirtualService、DestinationRule 等资源 | 管理 Istio API 配置 |
| **McpBridge Controller** | 监听 McpBridge 资源，对接外部注册中心 | 支持 Nacos、Eureka、Consul、Zookeeper、DNS |
| **Http2Rpc Controller** | 监听 Http2Rpc 资源，实现 HTTP 到 RPC 的协议转换 | 支持 Dubbo、gRPC |
| **WasmPlugin Controller** | 监听 WasmPlugin 资源，转换为 Istio WasmPlugin | 支持全局/路由/域名/服务级别配置 |
| **ConfigmapMgr** | 监听 higress-config ConfigMap | 管理全局配置（tracing、gzip 等） |

### 4.3 Higress Gateway

Higress Gateway 是数据面组件，包含两个子组件：

**4.3.1 Pilot Agent**

- 负责 Envoy 的启动和生命周期管理
- 代理 Envoy 的 xDS 请求到 Discovery 组件
- 通过 Unix Domain Socket（UDS）与 Envoy 通信

**4.3.2 Envoy**

Envoy 作为实际的流量代理，核心工作流程：

1. **Listener**：监听网络地址（IP:端口），接受下游连接
2. **Router**：根据路由规则将请求匹配到目标集群
3. **Filter Chain**：依次执行配置的过滤器（Wasm、Golang、Router 等）
4. **Cluster**：将请求转发到上游集群中的具体端点
5. **Endpoint**：上游集群中的具体服务实例（Pod IP）

### 4.4 Higress Console

Higress Console 是管理面组件，提供：

- **Web UI**：可视化的路由管理、插件管理、证书管理、服务管理界面
- **Admin SDK**：Java SDK（JDK 17+），支持外部系统通过 API 进行程序化配置管理

## 五、插件体系详解

### 5.1 插件体系全景

Higress 的插件体系是其最大特色，支持两种扩展机制：

```plaintext
Higress 插件体系
    │
    ├── Wasm 插件（沙箱化，多语言）
    │      ├── C++ 插件 (wasm-cpp)
    │      ├── Go 插件 (wasm-go)      ← 主要 AI 插件在此
    │      ├── Rust 插件 (wasm-rust)
    │      └── AssemblyScript 插件 (wasm-assemblyscript)
    │
    └── Golang Filter（原生 Go，非沙箱）
           └── golang-filter (mcp-server 等)
```

**两种机制对比**：

| 维度 | Wasm 插件 | Golang Filter |
| ------ | ---------- | --------------- |
| **运行时** | WebAssembly 沙箱（TinyGo/C++） | 完整 Go 运行时（CGO 桥接） |
| **性能** | 有少量沙箱开销 | 接近原生（共享进程空间） |
| **开发语言** | C++ / Go(TinyGo) / Rust / AssemblyScript | Go（标准 Go，非 TinyGo） |
| **隔离性** | 强（沙箱隔离） | 弱（共享 Envoy 进程） |
| **生态** | 官方内置 60+ 插件 | 目前较少（mcp-server 等） |
| **AI 插件** | 主要 AI 插件都是 Wasm | MCP 相关使用 Golang Filter |
| **热更新** | ✅ 支持（OCI 镜像分发） | ✅ 支持（.so 共享库替换） |

### 5.2 Wasm 插件机制

Higress 的 Wasm 插件基于 Envoy 的 Wasm 过滤器实现，工作流程：

```plaintext
插件开发 → 编译为 .wasm → 打包为 OCI 镜像 → 推送到镜像仓库
                                                        │
                                                        ▼
用户配置 WasmPlugin CRD ← Higress Controller 拉取镜像 → Envoy 加载 Wasm
```

**配置方式**：

```yaml
apiVersion: extensions.higress.io/v1alpha1
kind: WasmPlugin
metadata:
  name: ai-proxy
  namespace: higress-system
spec:
  selector:
    matchLabels:
      higress: higress-system-higress-gateway
  defaultConfig:
    provider:
      type: qwen
      apiTokens:
        - "YOUR_API_TOKEN"
      modelMapping:
        "*": "qwen-turbo"
  url: oci://higress-registry.cn-hangzhou.cr.aliyuncs.com/plugins/ai-proxy:1.0.0
```

**生效级别**：Higress 的 WasmPlugin 支持四个级别的配置：

1. **全局级别**：对所有路由生效
2. **域名级别**：对特定域名生效
3. **路由级别**：对特定路由生效
4. **服务级别**：对特定后端服务生效

### 5.3 Golang Filter 机制

Higress 同时支持 Envoy 的 Golang HTTP Filter 扩展机制，允许用标准 Go（非 TinyGo）开发过滤器：

```plaintext
Go 源码 → CGO_ENABLED=1 go build -buildmode=c-shared → golang-filter.so
                                                               │
                                                               ▼
                                                    Envoy 通过 dlopen() 加载
                                                               │
                                                               ▼
                                                    完整 Go 运行时运行
```

Golang Filter 相比 Wasm 的优势：

- 使用标准 Go（完整标准库，非 TinyGo 子集）
- 性能更接近原生（无 Wasm 沙箱开销）
- 支持 Goroutine、Channel、sync 等高级特性

#### 5.3.1 性能差异的本质

Wasm 插件确实运行在**虚拟机**中，但需要明确这是 Envoy 内置的轻量级 Wasm 运行时（如 V8、WAMR、WAVM），而非传统意义上的重量级 VM（如 JVM）。它的特点是启动快（毫秒级）、内存占用低（模块通常仅数百 KB 到数 MB）、指令通过解释或 JIT 编译执行，本身效率并不低。

两者真正的性能差距来自**与 Envoy 主进程交互的方式**：

| 操作 | Wasm 插件 | Golang Filter |
| ------ | ---------- | --------------- |
| 读取请求 Header | 通过 `proxy_get_header_map_pairs` ABI 调用 → VM 内反序列化 | 直接访问 C 结构体 → Go 结构体映射（CGO） |
| 修改请求 Body | 修改 VM 内内存 → 通过 ABI 回传给 Envoy | 直接修改共享内存中的 C 缓冲区 |
| 发起 HTTP 子请求 | VM 内构造请求 → ABI 序列化 → Envoy 执行 | 直接调用 Envoy C API |

Wasm 每一次与 Envoy 交互都需要**跨 ABI 边界序列化/反序列化数据**，这是性能损耗的主要来源。Golang Filter 通过 CGO 共享地址空间，虽然也有边界，但数据拷贝和序列化开销远小于 Wasm。

**实测结论**（简单 Header 修改场景）：

- 纯计算（字符串处理、JSON 解析）：Wasm 和 Golang Filter 差距在 10% 以内
- 高频 ABI 调用（大量 Header 读写、Body 修改）：Wasm 比 Golang Filter 慢 **20%~50%**
- 冷启动：Wasm 模块加载更快（无需启动 Go Runtime），但首次执行有 JIT 预热成本

**为什么 AI 插件性能差距不大？**

AI 插件（ai-proxy、ai-load-balancer）的特点是：

1. 大部分时间花在**等待后端 LLM 响应**（网络 I/O 远大于计算）
2. 单次请求 ABI 调用次数有限（解析 Body → 修改 Header → 转发），不是高频交互
3. 请求处理延迟通常 >100ms，Wasm 的几百微秒 ABI 开销可以忽略

因此 AI 场景下 Wasm 的性能劣势被**网络延迟掩盖**，选型时更关注安全隔离和热更新能力，而非纯性能。

### 5.4 AI 插件矩阵

Higress 提供了 20+ AI 专用插件，覆盖 AI 网关的完整功能链路：

| 插件名 | 功能 | 优先级 | 阶段 |
| -------- | ------ | -------- | ------ |
| **ai-proxy** | 多厂商 AI API 代理，自动协议转换 | 100 | 默认 |
| **ai-cache** | LLM 结果缓存（语义 + 字符串） | 10 | 认证 |
| **ai-load-balancer** | AI 负载均衡（前缀缓存/GPU 指标/全局最小连接） | - | 默认 |
| **model-router** | 模型路由（自动提取 model + 内容路由） | 900 | 认证 |
| **ai-token-ratelimit** | Token 消耗限流 | 600 | 默认 |
| **ai-security-guard** | AI 内容安全防护 | 300 | 默认 |
| **ai-statistics** | AI 请求统计（Token 消耗等） | 200 | 默认 |
| **ai-prompt-decorator** | Prompt 装饰（自动添加前缀/后缀） | - | 默认 |
| **ai-prompt-template** | Prompt 模板管理 | - | 默认 |
| **ai-transformer** | 请求/响应转换 | - | 默认 |
| **ai-intent** | 意图识别 | - | 默认 |
| **ai-json-resp** | JSON 响应格式化 | - | 默认 |
| **ai-history** | 对话历史管理 | - | 默认 |
| **ai-image-reader** | 图片内容理解 | - | 默认 |
| **ai-rag** | RAG（检索增强生成） | - | 默认 |
| **ai-search** | AI 搜索集成 | - | 默认 |
| **ai-quota** | 配额管理 | - | 默认 |
| **ai-agent** | AI Agent 框架 | - | 默认 |
| **mcp-server** | MCP 服务托管（Wasm + Golang Filter） | - | - |
| **mcp-router** | MCP 路由（Wasm） | - | 认证 |
| **model-mapper** | 模型映射（C++） | - | - |

## 六、核心 AI 能力详解

### 6.1 AI 代理（ai-proxy）

ai-proxy 是 Higress AI 网关的**核心插件**，实现多厂商 AI API 的统一代理。

**核心能力**：

1. **自动协议检测**：同时兼容 OpenAI 和 Claude 两种协议格式，零配置自动识别
2. **智能协议转换**：如果目标供应商不原生支持请求协议，自动进行协议转换
3. **模型映射**：支持前缀匹配、正则匹配、兜底映射等多种模型名映射方式
4. **多 Token 轮询**：配置多个 API Token 时自动轮询，支持 Failover
5. **支持 30+ AI 提供商**：

| 类型 | 提供商 |
| ------ | -------- |
| 国际 | OpenAI、Azure OpenAI、Claude、Gemini、Groq、Grok、Mistral、Cohere、Together AI、OpenRouter、Fireworks AI、Cloudflare、AWS Bedrock、Google Vertex AI、DeepL |
| 国内 | 通义千问、文心一言、百川智能、智谱 AI、零一万物、DeepSeek、星火、MiniMax、混元、阶跃星辰、360 智脑、豆包 |
| 开源/本地 | Ollama、NVIDIA Triton、Dify |

**模型映射示例**：

```yaml
provider:
  type: qwen
  apiTokens:
    - "YOUR_TOKEN"
  modelMapping:
    'gpt-3': 'qwen-turbo'
    'gpt-4-*': 'qwen-max'
    '~gpt(.*)': 'openai/gpt$1'  # 正则匹配
    '*': 'qwen-turbo'           # 兜底映射
```

### 6.2 AI 负载均衡（ai-load-balancer）

ai-load-balancer 实现了 gateway-api-inference-extension 标准的 Wasm 版本，提供三种 AI 专用负载均衡策略：

**6.2.1 前缀缓存感知（prefix_cache）**

根据 prompt 前缀匹配选择后端节点，复用 KV Cache：

```plaintext
请求1: "hi" → 路由到 Pod 1
请求2: "hi\nHello! How can I help?\nwrite a story" → 前缀匹配 "hi" → 复用 Pod 1 的 KV Cache
```

实现方式：通过 Redis 缓存 prompt 到 Pod 的映射关系，TTL 默认 1800 秒。

**6.2.2 GPU 指标感知（endpoint_metrics）**

定期拉取 vLLM 等模型服务器的 /metrics 端点，根据实时指标选择最优 Pod：

| 策略 | 说明 |
| ------ | ------ |
| `default` | 使用 Inference Extension 默认算法 |
| `least` + 自定义 `target_metric` | 选择指定指标值最小的 Pod（如 `vllm:num_requests_waiting`） |
| `most` + 自定义 `target_metric` | 选择指定指标值最大的 Pod |

**6.2.3 全局最小连接（global_least_request）**

基于 Redis 实现跨 Gateway 实例的全局最小请求数负载均衡，适用于多副本 Gateway 场景。

**6.2.4 跨服务负载均衡（cluster_metrics）**

根据网关统计的不同服务的指标进行服务间负载均衡：

| 模式 | 说明 |
| ------ | ------ |
| `LeastBusy` | 路由到当前并发请求数最少的服务 |
| `LeastTotalLatency` | 路由到当前 RT 最低的服务 |
| `LeastFirstTokenLatency` | 路由到当前首包 RT 最低的服务 |

### 6.3 模型路由（model-router）

model-router 实现了基于 LLM 协议中 model 参数的智能路由：

**6.3.1 model 参数提取**

从请求体中提取 `model` 参数，注入为 HTTP Header：

```json
{ "model": "qwen-long", "messages": [...] }
```

→ 按配置注入 Header（如 `x-higress-llm-model: qwen-long`）

**6.3.2 provider 提取**

支持 `provider/model` 格式的 model 参数，自动提取 provider：

```json
{ "model": "dashscope/qwen-long", "messages": [...] }
```

→ 按配置注入 Header（如 `x-higress-llm-provider: dashscope`），model 重写为 `qwen-long`

**6.3.3 内容自动路由**

当 model 设置为 `higress/auto` 时，根据用户消息内容自动选择模型：

```yaml
autoRouting:
  enable: true
  defaultModel: "qwen-turbo"
  rules:
    - pattern: "(?i)(画|绘|生成图|图片|image|draw|paint)"
      model: "qwen-vl-max"
    - pattern: "(?i)(代码|编程|code|program|function|debug)"
      model: "qwen-coder"
```

### 6.4 AI 缓存（ai-cache）

ai-cache 提供 LLM 结果缓存能力，支持两种模式：

**6.4.1 语义化缓存**

基于向量数据库的相似问题缓存：

- 使用 Embedding 服务将问题向量化
- 在向量数据库中查询相似问题（支持 Cosine/DotProduct/Euclidean 距离）
- 命中相似问题则直接返回缓存结果

支持的向量数据库：DashVector、Chroma、ElasticSearch、Weaviate、Pinecone、Qdrant、Milvus（需同时配置 Embedding 服务）

**6.4.2 字符串匹配缓存**

基于精确字符串匹配的缓存：

- 使用 Redis 存储问题-答案映射
- 精确命中则直接返回

### 6.5 MCP 服务（mcp-server / mcp-router）

Higress 内置了 Model Context Protocol（MCP）的托管和路由能力：

- **mcp-server**：同时提供 Wasm 插件和 Golang Filter 两种实现，用于托管 MCP 服务、处理 MCP 会话
- **mcp-router**（Wasm 插件）：路由 MCP 请求到对应的 MCP 服务

## 七、配置体系与 API 资源

### 7.1 双模式配置体系

Higress 支持两种配置模式：

| 模式 | 适用场景 | 资源类型 |
| ------ | --------- | ---------- |
| **Istio API 模式** | 需要 Istio 完整功能 | Gateway、VirtualService、DestinationRule、EnvoyFilter、WasmPlugin |
| **Gateway API 模式** | 标准化 K8s 路由 | Gateway、HTTPRoute、TCPRoute、GRPCRoute |
| **Ingress 模式** | 兼容传统 K8s Ingress | Ingress（自动转换为 Istio 配置） |

### 7.2 核心 CRD 资源

| CRD | 来源 | 用途 |
| ------ | ------ | ------ |
| **Gateway** | Gateway API / Istio | 定义流量入口（端口、协议、TLS） |
| **HTTPRoute** | Gateway API | 流量路由规则（路径、Header、权重） |
| **VirtualService** | Istio | 流量路由规则（更丰富的匹配条件） |
| **DestinationRule** | Istio | 负载均衡、连接池、熔断策略 |
| **InferencePool** | Inference Extension | 推理 Pod 池配置 |
| **InferenceObjective** | Inference Extension | 推理服务目标配置 |
| **McpBridge** | Higress 自定义 | 对接外部注册中心 |
| **Http2Rpc** | Higress 自定义 | HTTP 到 RPC 协议转换 |
| **WasmPlugin** | Higress 扩展 | Wasm 插件配置（支持四级生效范围） |

### 7.3 WasmPlugin 配置模型

Higress 的 WasmPlugin 在 Istio WasmPlugin 基础上进行了扩展，支持**四级生效范围**：

```yaml
apiVersion: extensions.higress.io/v1alpha1
kind: WasmPlugin
metadata:
  name: my-plugin
  namespace: higress-system
spec:
  selector:
    matchLabels:
      higress: higress-system-higress-gateway
  # 全局默认配置
  defaultConfig:
    key: value
  # 域名级别配置
  matchRules:
    - domain: example.com
      config:
        key: value2
  # 路由级别配置
    - ingress: my-route
      config:
        key: value3
  # 服务级别配置
    - service: my-service
      config:
        key: value4
  url: oci://registry/repo/plugin:version
```

## 八、高可用与扩展性设计

### 8.1 高可用机制

| 机制 | 说明 |
| ------ | ------ |
| **Gateway 多副本** | Higress Gateway 以 Deployment 形式部署，支持多副本 + HPA |
| **Controller 高可用** | Higress Controller 多副本部署，Leader Election 选举主节点 |
| **全局最小连接** | ai-load-balancer 通过 Redis 实现跨 Gateway 实例的全局负载感知 |
| **Failover** | ai-proxy 支持 Token 级别的 Failover，失败时自动切换 |
| **健康检查** | 自动对后端服务和外部注册中心进行健康检查 |
| **xDS 热更新** | 配置变更通过 xDS 实时下发，无需重启 Envoy |

### 8.2 插件扩展机制

Higress 的插件扩展机制非常灵活：

**Wasm 插件扩展**：

1. 开发插件代码（C++/Go/Rust/AssemblyScript）
2. 编译为 .wasm 文件
3. 打包为 OCI 镜像并推送
4. 配置 WasmPlugin CRD 引用镜像
5. Higress Controller 自动拉取并下发到 Envoy

**Golang Filter 扩展**：

1. 开发 Go 插件代码
2. 编译为 .so 共享库
3. 配置 EnvoyFilter 引用共享库
4. Envoy 通过 dlopen() 加载

### 8.3 多语言插件开发

Higress 支持四种语言开发 Wasm 插件：

| 语言 | 目录 | 说明 |
| ------ | ------ | ------ |
| **C++** | `plugins/wasm-cpp/` | 性能最优，适合计算密集型插件 |
| **Go** | `plugins/wasm-go/` | 生态最丰富，AI 插件主要在此 |
| **Rust** | `plugins/wasm-rust/` | 安全性和性能兼顾 |
| **AssemblyScript** | `plugins/wasm-assemblyscript/` | 前端开发者友好 |

## 九、与 AIGW / Inference Extension 的对比

### 9.1 架构范式差异

| 维度 | **Higress** | **AIGW** | **Inference Extension** |
| ------ | ------------ | --------- | ------------------------ |
| **运行时模型** | Envoy + Wasm 沙箱插件 + Golang Filter | Envoy Golang Filter (`libgolang.so` 嵌入进程) | Envoy ext-proc（EPP 独立 Deployment） |
| **控制面** | Istio + 自研 Higress Core | Istio xDS | Kubernetes Gateway API |
| **扩展语言** | C++/Go/Rust/AssemblyScript (Wasm) + Go (Golang Filter) | Go + CGO | 任何支持 gRPC 的语言 |
| **插件隔离** | Wasm 沙箱强隔离 | 共享进程空间 | 独立进程（EPP） |
| **K8s 依赖** | 需要 K8s（Helm 部署） | 需要 Istio Mesh | 需要 K8s + Gateway API |

### 9.2 功能定位差异

| 维度 | **Higress** | **AIGW** | **Inference Extension** |
| ------ | ------------ | --------- | ------------------------ |
| **核心定位** | 全功能 AI 网关（API 聚合 + 推理调度） | 推理集群专用调度器 | K8s 推理路由标准 |
| **多厂商 API 代理** | ✅ 30+ 提供商（ai-proxy） | ❌ 不支持 | ❌ 不支持 |
| **协议转换** | ✅ 自动 OpenAI/Claude 互转 | ✅ OpenAI ↔ Triton gRPC | ❌ 依赖 Gateway 层 |
| **LLM 结果缓存** | ✅ 语义 + 字符串缓存 | ❌ 不支持 | ❌ 不支持 |
| **Token 限流** | ✅ ai-token-ratelimit | ❌ 不支持 | ❌ 不支持 |
| **AI 安全防护** | ✅ ai-security-guard | ❌ 不支持 | ❌ 不支持 |
| **MCP 托管** | ✅ mcp-server/mcp-router | ❌ 不支持 | ❌ 不支持 |
| **前缀缓存感知** | ✅ ai-load-balancer | ✅ 内置 | ✅ 通过 EPP |
| **GPU 指标感知** | ✅ ai-load-balancer | ✅ 通过元数据中心 | ✅ 通过 EPP |
| **TTFT/TPOT 预测** | ❌ 不支持 | ✅ RLS 在线学习 | ❌ 不支持 |
| **模型自动路由** | ✅ model-router（内容匹配） | ❌ 不支持 | ❌ 不支持 |
| **传统网关功能** | ✅ 丰富（认证/限流/WAF/灰度） | ❌ 无 | ✅ 依赖 Gateway 层 |
| **控制台/UI** | ✅ Higress Console | ❌ 无 | ❌ 无 |

### 9.3 适用场景差异

| 场景 | **Higress** | **AIGW** | **Inference Extension** |
| ------ | ------------ | --------- | ------------------------ |
| 需要**统一 AI 入口**（商业 API + 自托管） | ✅ 最佳选择 | ❌ 不适用 | ❌ 不适用 |
| 已有 Istio + 大规模自托管推理集群 | ✅ 可用 | ✅ 最佳（调度算法最深） | ✅ 官方标准 |
| 追求 K8s 官方标准 + 长期兼容性 | ⚠️ 阿里云生态 | ❌ 私有方案 | ✅ 最佳 |
| 快速搭建生产级 AI 网关 | ✅ Helm 一键部署 | ❌ 需自行编译部署 | ⚠️ 需配置多个组件 |
| 需要丰富传统网关功能 + AI 能力 | ✅ 最佳 | ❌ 无传统网关功能 | ⚠️ 依赖 Gateway 实现 |

## 十、生态与实现现状

### 10.1 社区与生态

| 指标 | 数据 |
| ------ | ------ |
| **GitHub Stars** | 8k+ |
| **贡献者** | 200+ |
| **发布版本** | v2.x（持续迭代） |
| **AI 插件数** | 20+ |
| **Wasm 插件总数** | 60+ |
| **支持的 AI 提供商** | 30+ |

### 10.2 阿里云产品集成

Higress 作为阿里云开源项目，与阿里云产品深度集成：

- **阿里云 MSE（微服务引擎）**：提供托管版 Higress
- **阿里云 SLS**：日志服务集成
- **阿里云 ARMS**：应用实时监控集成
- **阿里云 Prometheus**：监控集成
- **阿里云 WAF**：Web 应用防火墙集成

### 10.3 相关项目

| 项目 | 关系 |
| ------ | ------ |
| **Envoy** | 数据面基础 |
| **Istio** | 控制面基础 |
| **Gateway API** | 路由标准（Inference Extension 的载体） |
| **AIGW** | 同为 AI 网关，Higress 功能更全面 |
| **Inference Extension** | Higress 支持该标准（ai-load-balancer 实现 gateway-api-inference-extension） |
| **LiteLLM** | 互补关系（Higress 负责入口网关，LiteLLM 负责多厂商聚合） |

## 十一、总结与入门建议

### 11.1 项目核心定位总结

Higress 是**面向 Kubernetes 的全功能云原生 AI 网关**，其核心价值在于：

1. **全链路 AI 能力**：从多厂商 API 代理（ai-proxy）到智能调度（ai-load-balancer）到结果缓存（ai-cache）到安全防护（ai-security-guard），覆盖 AI 网关的完整功能链路
2. **双扩展机制**：Wasm 插件（沙箱化、多语言）+ Golang Filter（原生 Go 性能），满足不同场景的扩展需求
3. **成熟传统网关功能**：认证、限流、WAF、灰度发布、协议转换等全套 API 网关能力
4. **可视化运维**：Higress Console 提供 Web UI 和 Admin SDK，降低运维门槛
5. **阿里云生态**：与阿里云产品深度集成，提供企业级托管方案

**一句话评价**：Higress 是**目前功能最全面的开源 AI 网关**，它不仅仅是一个推理调度器，而是一个覆盖"API 聚合 → 智能路由 → 推理调度 → 结果缓存 → 安全防护 → 可观测性"完整链路的 AI 网关平台。

### 11.2 快速入门路径

**第一步：部署 Higress**（30 分钟）

```bash
# 使用 Helm 在 K8s 上部署
helm repo add higress.io https://higress.io/helm-charts
helm install higress higress.io/higress -n higress-system --create-namespace
```

**第二步：配置 AI 代理**（30 分钟）

配置 ai-proxy 插件代理 OpenAI/通义千问等服务商：

```yaml
apiVersion: extensions.higress.io/v1alpha1
kind: WasmPlugin
metadata:
  name: ai-proxy
  namespace: higress-system
spec:
  selector:
    matchLabels:
      higress: higress-system-higress-gateway
  defaultConfig:
    provider:
      type: openai
      apiTokens:
        - "YOUR_OPENAI_API_KEY"
  url: oci://higress-registry.cn-hangzhou.cr.aliyuncs.com/plugins/ai-proxy:1.0.0
```

**第三步：配置模型路由和负载均衡**

配置 model-router 和 ai-load-balancer 插件实现智能路由和调度。

**第四步：接入自托管推理集群**

配置 vLLM 等推理服务为后端，启用前缀缓存感知和 GPU 指标感知调度。

### 11.3 与 AIGW / Inference Extension 的学习建议

三个项目代表了 AI 网关技术的三个不同方向：

| 项目 | 方向 | 学习价值 |
| ------ | ------ | ---------- |
| **AIGW** | 推理调度算法深度（RLS 预测、四重释放） | 学习 AI 专用调度算法的设计哲学 |
| **Inference Extension** | K8s 标准化（可组合架构、标准接口） | 学习如何设计可扩展的云原生标准 |
| **Higress** | 全功能产品化（完整链路、多语言插件、控制台） | 学习如何构建生产级 AI 网关平台 |

**学习顺序建议**：

1. 先学 **Inference Extension**（理解标准化接口设计）
2. 再学 **AIGW**（理解推理调度算法深度优化）
3. 最后学 **Higress**（理解全功能产品化设计，对比三种方案的优势和取舍）

> **总结**：Higress、AIGW 和 Inference Extension 分别代表了 AI 网关领域的"产品化"、"算法深度"和"标准化"三个方向。对于大多数团队来说，**Higress 是上手最快的选择**（Helm 一键部署、20+ AI 插件开箱即用、可视化控制台）；**AIGW 适合需要极致推理调度性能的场景**（RLS 预测、TTFT 优化）；**Inference Extension 适合追求 K8s 官方标准和长期兼容性的团队**。
