# NVSentinel 源码深度学习计划

> 本计划适用于希望从源码级别精通 NVSentinel 项目的开发者
> 适合通过抄写代码 + 深入设计思想的方式学习

---

## 学习目标

1. **掌握 NVIDIA/GPU 监控知识** - DCGM、XID 错误、GPU 拓扑
2. **掌握事件驱动架构** 在云原生场景的实际应用
3. **理解 Go 微服务设计** 的最佳实践
4. **学习 Kubernetes 控制器模式** 和 CRD 设计
5. **掌握数据库抽象层** 的设计思路
6. **深入理解 CEL (Common Expression Language)** 在策略引擎中的应用
7. **学习生产级 Go 项目** 的工程实践

---

## 前置知识要求

| 知识领域 | 要求程度 | 补充资源 |
|---------|---------|---------|
| NVIDIA/GPU 基础 | 了解 GPU 架构、驱动、监控 | 见下文 GPU 知识章节 |
| Go 语言 | 熟练（并发、接口、模块系统） | [Go by Example](https://gobyexample.com/) |
| Kubernetes | 了解基础（Pod、Node、Controller） | [Kubernetes Concepts](https://kubernetes.io/docs/concepts/) |
| Protocol Buffers | 了解基本概念 | [protobuf官方文档](https://protobuf.dev/) |
| 数据库基础 | 理解 SQL 和 Change Streams | MongoDB/PostgreSQL 文档 |
| gRPC | 了解 RPC 和流式通信 | [gRPC Go Quick Start](https://grpc.io/docs/languages/go/quickstart/) |

---

## 学习路径概览

```
Phase 0: NVIDIA/GPU 基础知识 ⭐ 必修！
    ├─ DCGM (Data Center GPU Manager)
    ├─ XID 错误系统
    ├─ GPU 拓扑结构 (NVLink、NVSwitch)
    ├─ GPU 元数据 (UUID、PCI 地址)
    └─ NVIDIA 驱动与内核交互

Phase 1: 基础设施层 (Foundation)
    ├─ Protobuf 数据模型
    ├─ 数据库抽象层
    └─ 公共工具库

Phase 2: 事件摄入层 (Ingestion)
    ├─ Platform Connectors 架构
    ├─ Pipeline 处理流程
    └─ gRPC 服务实现

Phase 3: 检测层 (Detection)
    ├─ GPU Health Monitor (Python)
    ├─ Syslog Health Monitor (Go)
    └─ 其他监控器模式

Phase 4: 响应层 (Response)
    ├─ Fault Quarantine (隔离模块)
    ├─ Node Drainer (排空模块)
    └─ Fault Remediation (修复模块)

Phase 5: 高级特性 (Advanced)
    ├─ Change Stream 实现
    ├─ CEL 策略引擎
    ├─ 熔断器模式
    └─ 状态机管理

Phase 6: 工程实践 (Engineering)
    ├─ 构建系统设计
    ├─ 测试策略
    └─ 部署架构
```

---

## 系统架构概览 🏗️

> **在开始深入学习代码之前，先理解整个系统的架构和模块协同方式**

### 整体架构图 (含代码路径)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        NVSentinel 系统架构                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────────  检测层 (Detection Layer)  ──────────────────┐        │
│  │                           health-monitors/                      │        │
│  │                                                                  │        │
│  │  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐               │        │
│  │  │   GPU       │   │   Syslog    │   │    CSP      │               │        │
│  │  │  Monitor    │   │   Monitor   │   │  Monitor    │               │        │
│  │  │  (Python)   │   │    (Go)     │   │    (Go)     │               │        │
│  │  │  DCGM集成   │   │  XID解析    │   │  云维护事件  │               │        │
│  │  │ gpu-health/ │   │ syslog-hm/  │   │ csp-health/ │               │        │
│  │  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘               │        │
│  │         │                 │                 │                       │        │
│  │         └─────────────────┼─────────────────┘                       │        │
│  │                           ▼                                         │        │
│  │                    gRPC: HealthEventOccurredV1()                     │        │
│  └───────────────────────────┼──────────────────────────────────────────┘        │
│                              ▼                                                │
│  ┌──────────────────  摄入层 (Ingestion Layer)  ──────────────────┐        │
│  │                        platform-connectors/                    │            │
│  │                                                              │            │
│  │  ┌─────────────────────────────────────────────────────────┐            │
│  │  │           Platform Connectors (Go)                      │            │
│  │  │  ┌─────────────────────────────────────────────────┐    │            │
│  │  │  │ Pipeline: 责任链处理 (Chain of Responsibility)  │    │            │
│  │  │  │           pkg/pipeline/                          │    │            │
│  │  │  │                                                  │    │            │
│  │  │  │  ┌─────────────────┐    ┌──────────────────┐   │    │            │
│  │  │  │  │ Metadata        │───▶│ Overrides        │   │    │            │
│  │  │  │  │ Transformer     │    │ Transformer      │   │    │            │
│  │  │  │  │ (GPU UUID等)    │    │ (CEL 行为覆盖)   │   │    │            │
│  │  │  │  │ transformers/   │    │ transformers/     │   │    │            │
│  │  │  │  └─────────────────┘    └──────────────────┘   │    │            │
│  │  │  └─────────────────────────────────────────────────┘    │            │
│  │  │                                                          │            │
│  │  │  ┌──────────────┐        ┌──────────────────┐          │            │
│  │  │  │ K8s Storage  │        │ DB Storage       │          │            │
│  │  │  │ (Node        │        │ (MongoDB/PG)     │          │            │
│  │  │  │ Condition)   │        │ connectors/       │          │            │
│  │  │  └──────────────┘        └────────┬─────────┘          │            │
│  │  └───────────────────────────────────┼─────────────────────┘            │
│  └──────────────────────────────────────┼──────────────────────────────────┘
│                                         ▼ Change Stream (实时推送)          │
│                                         store-client/                       │
│  ┌──────────────────  响应层 (Response Layer)  ──────────────────┐        │
│  │                                                              │            │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │            │
│  │  │  Fault       │  │    Node      │  │   Fault      │       │            │
│  │  │  Quarantine  │  │   Drainer    │  │ Remediation  │       │            │
│  │  │  (节点隔离)  │  │  (Pod排空)   │  │  (故障修复)  │       │            │
│  │  │  CEL规则     │  │  PDB尊重     │  │  CRD创建     │       │            │
│  │  │ fault-q/     │  │ node-drainer/│  │ fault-rem/   │       │            │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘       │            │
│  │         │                 │                 │               │            │
│  │         ▼                 ▼                 ▼               │            │
│  │    ┌─────────────────────────────────────────────┐        │            │
│  │    │         Kubernetes API Server               │        │            │
│  │    │  (Cordon, Drain, Taint, Label, CRD)        │        │            │
│  │    └─────────────────────────────────────────────┘        │            │
│  └────────────────────────────────────────────────────────────┘            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

═══════════════════════════════════════════════════════════════════════════════
                              辅助模块 (Supporting Modules)
═══════════════════════════════════════════════════════════════════════════════

┌─────────────────────────────────────────────────────────────────────────────┐
│  基础设施层 (Foundation Layer)                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐             │
│  │  data-models/   │  │  store-client/  │  │    commons/     │             │
│  │                 │  │                 │  │                 │             │
│  │  Protobuf定义   │  │  数据库抽象层   │  │  公共工具库     │             │
│  │  protobufs/     │  │  pkg/datastore/ │  │  pkg/logger/    │             │
│  │                 │  │  pkg/query/     │  │  pkg/k8sutils/  │             │
│  │  HealthEvent    │  │  pkg/watcher/   │  │  pkg/statemgr/  │             │
│  │  数据契约       │  │  Change Stream  │  │  状态管理器     │             │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│  其他监控器 (Additional Monitors)                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐             │
│  │ k8s-object-hm/  │  │ metadata-col/   │  │    labeler/     │             │
│  │                 │  │                 │  │                 │             │
│  │ K8s对象监控     │  │  元数据收集器   │  │  节点标签管理   │             │
│  │ CEL规则监控     │  │  GPU拓扑收集    │  │  自动标注驱动   │             │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│  高级功能模块 (Advanced Modules)                                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐             │
│  │    janitor/     │  │ events-analyzer │  │ event-exporter/ │             │
│  │                 │  │                 │  │                 │             │
│  │  修复执行器     │  │  事件模式分析   │  │  事件导出服务   │             │
│  │  GPU重置/重启   │  │  聚合关联分析   │  │  外部系统集成   │             │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 模块代码路径快速索引

| 模块名 | 代码路径 | 主要语言 | 关键文件 |
|-------|---------|---------|---------|
| **检测层 (Detection)** |
| GPU Health Monitor | `health-monitors/gpu-health-monitor/` | Python | `gpu_health_monitor/dcgm_watcher/dcgm.py` |
| Syslog Health Monitor | `health-monitors/syslog-health-monitor/` | Go | `pkg/xid/xid_handler.go`, `pkg/patterns/xid.go` |
| CSP Health Monitor | `health-monitors/csp-health-monitor/` | Go | `pkg/csp/csp_monitor.go` |
| K8s Object Monitor | `health-monitors/kubernetes-object-monitor/` | Go | `pkg/monitor/monitor.go` |
| **摄入层 (Ingestion)** |
| Platform Connectors | `platform-connectors/` | Go | `pkg/server/platform_connector_server.go` |
| Pipeline | `platform-connectors/pkg/pipeline/` | Go | `pipeline.go` |
| Transformers | `platform-connectors/pkg/transformers/` | Go | `metadata_transformer.go` |
| Connectors | `platform-connectors/pkg/connectors/` | Go | `kubernetes_storage.go`, `db_storage.go` |
| **响应层 (Response)** |
| Fault Quarantine | `fault-quarantine/` | Go | `pkg/evaluator/rule_evaluator.go` |
| Node Drainer | `node-drainer/` | Go | `pkg/reconciler/reconciler.go` |
| Fault Remediation | `fault-remediation/` | Go | `pkg/reconciler/remediation_reconciler.go` |
| Janitor | `janitor/` | Go | `pkg/reconciler/reconciler.go` |
| **基础设施 (Foundation)** |
| Data Models | `data-models/` | - | `protobufs/health_event.proto` |
| Store Client | `store-client/` | Go | `pkg/datastore/interfaces.go` |
| Commons | `commons/` | Go | `pkg/statemanager/statemanager.go` |
| **辅助模块 (Supporting)** |
| Metadata Collector | `metadata-collector/` | Go | `pkg/collector/collector.go` |
| Labeler | `labeler/` | Go | `pkg/reconciler/reconciler.go` |
| Health Events Analyzer | `health-events-analyzer/` | Go | `pkg/analyzer/analyzer.go` |
| Event Exporter | `event-exporter/` | Go | `pkg/exporter/exporter.go` |
| Log Collector | `log-collector/` | Go | `pkg/collector/collector.go` |
| **部署相关** |
| Helm Charts | `distros/kubernetes/nvsentinel/` | - | `values.yaml`, `Chart.yaml` |
| Tilt Config | `tilt/` | - | `Tiltfile` |
| Docker Files | `docker/` | - | 各模块的 Dockerfile |

---

### 模块协同工作流程 (完整数据流)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         GPU 故障检测到修复的完整流程                          │
└─────────────────────────────────────────────────────────────────────────────┘

时刻 T0: GPU 发生硬件故障 (例如：温度过高、ECC 错误)
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 1: 故障检测 (Health Monitors - 并行工作)                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ GPU Health Monitor (Python DaemonSet)                   │   │
│  │  ────────────────────────────────────────────────────    │   │
│  │  1. DCGM 定期轮询 (默认5秒间隔)                           │   │
│  │  2. 调用 DCGM API: dcgm_group.health.Check()            │   │
│  │  3. 检测到: GPU 温度 105°C (阈值 85°C)                   │   │
│  │  4. 识别错误码: DCGM_FR_THERMAL                          │   │
│  │  5. 创建 HealthEvent protobuf:                          │   │
│  │     {                                                   │   │
│  │       componentClass: "GPU",                            │   │
│  │       checkName: "thermal",                             │   │
│  │       isFatal: true,                                    │   │
│  │       errorCode: ["DCGM_FR_THERMAL"],                   │   │
│  │       gpuId: 0                                          │   │
│  │     }                                                   │   │
│  │  6. 批量发送 (最多100条/批次):                          │   │
│  │     gRPC HealthEventOccurredV1(events)                  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Syslog Health Monitor (Go DaemonSet)                    │   │
│  │  ────────────────────────────────────────────────────    │   │
│  │  1. 实时读取 journalctl 内核日志                         │   │
│  │     sdjournal.NewJournal()                              │   │
│  │     journal.AddMatch("SYSLOG_IDENTIFIER=kernel")        │   │
│  │  2. 匹配到: "NVRM: Xid (PCI:0000:b3:00.0): 79"         │   │
│  │  3. 正则解析 XID:                                        │   │
│  │     PCI=0000:b3:00.0, XID=79                            │   │
│  │  4. 查询元数据: PCI → GPU UUID                          │   │
│  │  5. 创建 HealthEvent:                                   │   │
│  │     {                                                   │   │
│  │       componentClass: "GPU",                            │   │
│  │       checkName: "xid-error",                           │   │
│  │       isFatal: true,                                    │   │
│  │       errorCode: ["79"],                                │   │
│  │       entitiesImpacted: [{                              │   │
│  │         entityType: "GPU_UUID",                         │   │
│  │         entityValue: "GPU-deadbeef-1234"               │   │
│  │       }]                                                │   │
│  │     }                                                   │   │
│  │  6. 发送到 Platform Connectors                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
    │
    │ gRPC: HealthEventOccurredV1(HealthEvents)
    │       events: [{ HealthEvent }, { HealthEvent }, ...]
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 2: 事件处理 (Platform Connectors)                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  gRPC 服务器接收 → Pipeline 责任链处理                           │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Transformer 1: Metadata Enrichment (元数据增强)         │   │
│  │  ────────────────────────────────────────────────────    │   │
│  │  输入: HealthEvent { gpuId: 0, nodeName: "gpu-node-1" } │   │
│  │                                                             │   │
│  │  处理:                                                      │   │
│  │    1. 查询 Metadata Collector: gpuId 0 → UUID            │   │
│  │    2. 查询节点标签: 驱动版本、DCGM 版本                   │   │
│  │    3. 查询云元数据: 实例类型、可用区 (GCP/AWS)            │   │
│  │                                                             │   │
│  │  输出: HealthEvent {                                       │   │
│  │    gpuId: 0,                                              │   │
│  │    metadata: {                                            │   │
│  │      gpu_uuid: "GPU-deadbeef-1234",                      │   │
│  │      driver_version: "535.129.03",                       │   │
│  │      dcgm_version: "3.3.4",                              │   │
│  │      instance_type: "p4d.24xlarge",                      │   │
│  │      availability_zone: "us-east-1a"                     │   │
│  │    }                                                      │   │
│  │  }                                                        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                          │                                      │
│                          ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Transformer 2: Overrides Application (行为覆盖 - 可选)  │   │
│  │  ────────────────────────────────────────────────────    │   │
│  │  允许用户通过 CEL 表达式覆盖默认行为:                     │   │
│  │                                                             │   │
│  │  规则示例:                                                 │   │
│  │    event.metadata.gpu_uuid == "GPU-TEST-1234"            │   │
│  │      → skip quarantine: true  (测试 GPU，不隔离)         │   │
│  │                                                             │   │
│  │    event.errorCode.exists(e, e == "48")                  │   │
│  │      → force drain: true      (ECC 错误强制排空)        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                          │                                      │
│                          ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Connector 1: Kubernetes Storage                          │   │
│  │  ────────────────────────────────────────────────────    │   │
│  │  更新 Kubernetes Node 状态:                               │   │
│  │    1. 更新 Node Condition:                               │   │
│  │       type: "GPUReady", status: "False"                  │   │
│  │    2. 可选添加污点:                                       │   │
│  │       key: "nvidia.com/gpu-unhealthy",                   │   │
│  │       effect: "NoExecute"                                │   │
│  └─────────────────────────────────────────────────────────┘   │
│                          │                                      │
│                          ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Connector 2: Database Storage                            │   │
│  │  ────────────────────────────────────────────────────    │   │
│  │  持久化到数据库 (MongoDB 或 PostgreSQL):                   │   │
│  │    {                                                      │   │
│  │      _id: ObjectId("..."),                               │   │
│  │      version: 1,                                         │   │
│  │      agent: "gpu-health-monitor",                        │   │
│  │      componentClass: "GPU",                              │   │
│  │      nodeName: "gpu-node-1",                             │   │
│  │      isFatal: true,                                      │   │
│  │      errorCode: ["DCGM_FR_THERMAL"],                     │   │
│  │      metadata: { gpu_uuid: "GPU-deadbeef-1234", ... },  │   │
│  │      generatedTimestamp: ISODate("2025-12-25T10:30:00Z"),│   │
│  │      status: "open"                                     │   │
│  │    }                                                      │   │
│  │                                                             │   │
│  │  ⚡ 触发 Change Stream 事件推送                            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
    │
    │ MongoDB Change Stream / PostgreSQL Logical Replication
    │ 实时推送到所有订阅者
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 3: 故障响应 (Core Modules - 独立监听，并行处理)            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  🔑 关键: 所有模块同时监听同一个 Change Stream                   │
│  每个模块独立判断是否处理该事件                                  │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Module 1: Fault Quarantine (故障隔离)                   │   │
│  │  ────────────────────────────────────────────────────    │   │
│  │  1. 收到 Change Stream 事件:                              │   │
│  │     { operation: "insert", fullDocument: { event } }     │   │
│  │                                                             │   │
│  │  2. 评估 CEL 规则 (按优先级排序):                         │   │
│  │     ┌─────────────────────────────────────────────┐       │   │
│  │     │ Rule Set 1: gpu-fatal-errors               │       │   │
│  │     │   expression:                               │       │   │
│  │     │     event.componentClass == "GPU" &&       │       │   │
│  │     │     event.isFatal == true                   │       │   │
│  │     │   priority: 100                             │       │   │
│  │     │   action: cordon + taint                    │       │   │
│  │     └─────────────────────────────────────────────┘       │   │
│  │                                                             │   │
│  │     ✅ 规则匹配!                                           │   │
│  │                                                             │   │
│  │  3. 检查熔断器 (Circuit Breaker):                          │   │
│  │     ┌─────────────────────────────────────────────┐       │   │
│  │     │ Sliding Window (5 minutes)                  │       │   │
│  │     │   Bucket 1: [node-1, node-2, node-3]        │       │   │
│  │     │   Bucket 2: [node-4]                        │       │   │
│  │     │                                              │       │   │
│  │     │ Total unique nodes: 4                        │       │   │
│  │     │ Cluster size: 10 GPUs                        │       │   │
│  │     │ Percentage: 40%                              │       │   │
│  │     │ Threshold: 30%                               │       │   │
│  │     │                                              │       │   │
│  │     │ ⚠️ 40% > 30% → 熔断器触发!                  │       │   │
│  │     │ 跳过隔离，记录告警                            │       │   │
│  │     └─────────────────────────────────────────────┘       │   │
│  │                                                             │   │
│  │     假设熔断器未触发:                                       │   │
│  │                                                             │   │
│  │  4. 执行隔离:                                               │   │
│  │     GET /api/v1/nodes/gpu-node-1                           │   │
│  │     node.Spec.Unschedulable = true                         │   │
│  │     node.Spec.Taints = append([{                          │   │
│  │       key: "nvidia.com/gpu",                              │   │
│  │       value: "fault",                                     │   │
│  │       effect: "NoSchedule"                                │   │
│  │     }])                                                   │   │
│  │     PATCH /api/v1/nodes/gpu-node-1                        │   │
│  │                                                             │   │
│  │  5. 更新状态标签:                                           │   │
│  │     node.Labels["nvsentinel-state"] = "quarantined"       │   │
│  │     node.Labels["nvsentinel-cordon-by"] = "fault-quarantine"│   │
│  │     node.Labels["nvsentinel-reason"] = "DCGM_FR_THERMAL"  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Module 2: Node Drainer (节点排空)                       │   │
│  │  ────────────────────────────────────────────────────    │   │
│  │  1. 监听节点标签变化:                                     │   │
│  │     nvsentinel-state: none → quarantined                │   │
│  │     nvsentinel-state: quarantined → draining            │   │
│  │                                                             │   │
│  │  2. 列出节点上所有 Pod:                                   │   │
│  │     GET /api/v1/nodes/gpu-node-1/pods                     │   │
│  │     Found: 12 Pods                                        │   │
│  │                                                             │   │
│  │  3. 过滤和排序:                                            │   │
│  │     跳过: DaemonSet (2), Mirror Pod (0)                   │   │
│  │     剩余: 10 Pods                                          │   │
│  │                                                             │   │
│  │     按 PDB 优先级排序:                                      │   │
│  │       Tier 1 (无 PDB): training-job-xxx                   │   │
│  │       Tier 2 (有 PDB): inference-service-yyy              │   │
│  │                                                             │   │
│  │  4. 逐个驱逐:                                               │   │
│  │     for pod in pods {                                      │   │
│  │       // 检查 PDB                                          │   │
│  │       allowedDisruptions = checkPDB(pod)                  │   │
│  │       if allowedDisruptions <= 0 {                        │   │
│  │         continue  // 跳过，保护可用性                      │   │
│  │       }                                                    │   │
│  │                                                             │   │
│  │       // 执行驱逐                                          │   │
│  │       POST /api/v1/namespaces/ns/pods/pod-name/eviction   │   │
│  │                                                             │   │
│  │       // 等待 Pod 删除                                     │   │
│  │       waitForPodDeletion(pod, timeout: 5min)               │   │
│  │                                                             │   │
│  │       // 记录进度到数据库                                   │   │
│  │       updateDrainStatus(pod, "evicted")                   │   │
│  │     }                                                      │   │
│  │                                                             │   │
│  │  5. 所有 Pod 驱逐完成:                                      │   │
│  │     node.Labels["nvsentinel-state"] = "drain-succeeded"   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Module 3: Fault Remediation (故障修复)                  │   │
│  │  ────────────────────────────────────────────────────    │   │
│  │  1. 监听状态: drain-succeeded                            │   │
│  │                                                             │   │
│  │  2. 根据 HealthEvent 选择修复策略:                         │   │
│  │     ┌─────────────────────────────────────────────┐       │   │
│  │     │ ErrorCode → RecommendedAction             │       │   │
│  │     ├─────────────────────────────────────────────┤       │   │
│  │     │ XID 79    → COMPONENT_RESET (GPU Reset)    │       │   │
│  │     │ XID 48    → RESTART_BM (节点重启)         │       │   │
│  │     │ XID 31    → RESTART_BM (驱动错误)         │       │   │
│  │     │ Thermal   → COMPONENT_RESET (温度问题)     │       │   │
│  │     └─────────────────────────────────────────────┘       │   │
│  │                                                             │   │
│  │  3. 创建 Remediation CRD:                                  │   │
│  │     apiVersion: nvidia.com/v1alpha1                       │   │
│  │     kind: Remediation                                     │   │
│  │     metadata:                                             │   │
│  │       name: gpu-node-1-thermal-remediation                │   │
│  │     spec:                                                 │   │
│  │       nodeName: gpu-node-1                                │   │
│  │       action:                                             │   │
│  │         type: GpuReset                                    │   │
│  │         params:                                           │   │
│  │           gpuId: "0"                                      │   │
│  │           force: true                                     │   │
│  │       healthEventRef: <event-id>                          │   │
│  │                                                             │   │
│  │  4. 更新状态:                                               │   │
│  │     node.Labels["nvsentinel-state"] = "remediating"       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Module 4: Janitor (修复执行器)                           │   │
│  │  ────────────────────────────────────────────────────    │   │
│  │  1. 监听 Remediation CRD                                  │   │
│  │                                                             │   │
│  │  2. 执行 GPU Reset:                                        │   │
│  │     - 调用 DCGM API: dcgmResetGPU(gpuId)                 │   │
│  │     - 等待 GPU 恢复: 30 秒                                │   │
│  │     - 验证 GPU 健康: dcgmCheckHealth()                   │   │
│  │                                                             │   │
│  │  3. 更新 CRD 状态:                                         │   │
│  │     status.phase = "remediation-succeeded"               │   │
│  │                                                             │   │
│  │  4. 最终状态:                                               │   │
│  │     node.Labels["nvsentinel-state"] =                    │   │
│  │       "remediation-succeeded"                            │   │
│  │                                                             │   │
│  │  5. 可选: 解除隔离                                          │   │
│  │     node.Spec.Unschedulable = false                       │   │
│  │     删除污点                                               │   │
│  │     节点恢复调度 ✅                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

最终结果: GPU 故障被检测 → 节点被隔离 → Pod 被驱逐 → GPU 被修复 → 节点恢复
```

---

### 核心设计特点

#### 1. 完全解耦的事件驱动架构

```go
// 每个模块独立监听 Change Stream，互不依赖
watcher, _ := store.NewChangeStreamWatcher(ctx, config)
go func() {
    for event := range watcher.Events() {
        // 处理事件，无需知道谁发送的
        if shouldHandle(event) {
            handleEvent(event)
        }
        // 标记已处理 (支持断点续传)
        watcher.MarkProcessed(ctx, event.Token)
    }
}()
```

**架构优势**:
- ✅ **模块独立**: 每个模块可以独立部署、升级、扩展
- ✅ **水平扩展**: 多个副本处理同一个 Change Stream (MongoDB Resume Token)
- ✅ **故障隔离**: 单个模块故障不影响其他模块
- ✅ **技术异构**: Python (GPU Monitor) 和 Go (其他模块) 可以共存

#### 2. 责任链模式 (Pipeline)

```go
// Platform Connectors 的事件处理管道
type Pipeline struct {
    transformers []Transformer
}

func (p *Pipeline) Process(ctx context.Context, event *pb.HealthEvent) {
    for _, t := range p.transformers {
        if err := t.Transform(ctx, event); err != nil {
            log.Warn("Transformer failed", "transformer", t.Name())
            // 继续处理下一个 Transformer
        }
    }
}
```

**处理链**:
```
原始 HealthEvent
    ↓
┌─────────────────────────────────────────────────┐
│ Transformer 1: Metadata Enrichment             │
│  - 添加 GPU UUID (从元数据服务查询)            │
│  - 添加驱动版本、DCGM 版本                      │
│  - 添加云提供商元数据 (实例类型、可用区)        │
└─────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────┐
│ Transformer 2: Overrides Application (可选)    │
│  - 检查 CEL 表达式是否匹配                      │
│  - 强制跳过/执行隔离或排空                      │
│  - 覆盖默认行为                                 │
└─────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────┐
│ Connector 1: Kubernetes Storage                 │
│  - 更新 Node Condition (GPUReady)              │
│  - 添加节点污点 (可选)                          │
└─────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────┐
│ Connector 2: Database Storage                   │
│  - 持久化到 MongoDB/PostgreSQL                  │
│  - 触发 Change Stream                          │
└─────────────────────────────────────────────────┘
```

#### 3. CEL 策略引擎

```go
// 故障隔离规则配置示例
[[rule-sets]]
  name = "gpu-fatal-errors"
  priority = 100

  [rule-sets.match]
    [[rule-sets.match.any]]
      kind = "event"
      # CEL 表达式
      expression = "event.componentClass == 'GPU' && event.isFatal == true"

  [rule-sets.taint]
    key = "nvidia.com/gpu"
    value = "fault"
    effect = "NoSchedule"

  [rule-sets.cordon]
    shouldCordon = true
```

**CEL 表达式示例**:
```go
// 简单条件
"event.componentClass == 'GPU'"

// 复杂逻辑
"event.componentClass == 'GPU' && event.isFatal == true"

// 错误码匹配
"event.errorCode.exists(e, e == 'XID-48' || e == 'XID-79')"

// 元数据查询
"event.metadata.gpu_uuid.matches('^GPU-[0-9a-f-]+$')"

// 时间窗口
"event.generatedTimestamp > now() - duration('1h')"
```

#### 4. 熔断器模式 (防止雪崩)

```go
// 滑动窗口熔断器实现
type slidingWindowBreaker struct {
    buckets     []int          // 环形缓冲区
    percentage  int            // 最大百分比 (如30%)
    duration    time.Duration  // 时间窗口 (如5分钟)
    nodeToIndex map[string]int // 去重节点
}

func (b *slidingWindowBreaker) Record(nodeName string) error {
    // 添加到窗口
    b.addToBucket(nodeName)

    // 检查百分比
    if b.percentageExceeded() {
        b.state = StateTripped
        return errors.New("circuit breaker tripped: too many nodes quarantined")
    }
    return nil
}
```

**防止场景**:
```
集群有 10 个 GPU 节点

正常情况:
  1 个 GPU 故障 → 隔离 1 个节点 (10%) → ✅ 正常

异常情况:
  冷却系统故障 → 4 个 GPU 同时高温
  → 熔断器检测到 40% 节点被隔离
  → 超过 30% 阈值
  → ⚠️ 熔断器触发，停止隔离
  → 保护集群剩余 60% GPU 可用性
```

#### 5. 为什么选择数据库 Change Stream 而不是消息队列？

| 特性 | 数据库 Change Stream | 消息队列 (Kafka/RabbitMQ) |
|-----|---------------------|--------------------------|
| **持久化** | ✅ 内置，事件即数据 | ❌ 需要额外配置 |
| **断点续传** | ✅ Resume Token 原生支持 | ⚠️ 需要消费者组管理 |
| **历史查询** | ✅ SQL 查询任意历史事件 | ❌ 消息消费后难查询 |
| **运维成本** | ✅ 复用现有数据库 | ❌ 额外维护消息队列 |
| **复杂性** | ✅ 无需额外组件 | ❌ 增加系统复杂度 |
| **实时性** | ⚠️ 毫秒级延迟 | ✅ 微秒级延迟 |
| **吞吐量** | ⚠️ 适合中等规模 | ✅ 适合超大规模 |

**NVSentinel 的选择理由**:
1. GPU 故障频率较低 (每天可能几次)，不需要超高频处理
2. 事件历史对故障分析非常重要
3. 简化架构，减少组件数量
4. MongoDB/PostgreSQL 通常已是集群标配

---

### 各模块职责详解

| 模块 | 语言 | 部署形态 | 职责 | 关键技术 |
|-----|------|---------|------|---------|
| **GPU Health Monitor** | Python | DaemonSet | 通过 DCGM 监控 GPU 硬件健康 | DCGM Python API、gRPC 客户端 |
| **Syslog Health Monitor** | Go | DaemonSet | 解析内核日志的 XID 错误 | sdjournal、正则解析 |
| **CSP Health Monitor** | Go | Deployment | 监控云厂商维护事件 | GCP/AWS API 集成 |
| **Kubernetes Object Monitor** | Go | Deployment | CEL 规则监控 K8s 对象 | CEL、Informer |
| **Platform Connectors** | Go | Deployment | gRPC 服务 + 事件处理管道 | Pipeline 模式、Transformer |
| **Fault Quarantine** | Go | Deployment | 基于规则的节点隔离 | CEL 规则引擎、熔断器 |
| **Node Drainer** | Go | Deployment | 优雅驱逐 Pod | PDB 尊重、优先级排序 |
| **Fault Remediation** | Go | Deployment | 创建修复 CRD | CRD 控制器 |
| **Janitor** | Go | Deployment | 执行修复操作 | GPU Reset、节点重启 API |
| **Metadata Collector** | Go | DaemonSet | 收集 GPU 拓扑信息 | DCGM 拓扑 API |
| **Labeler** | Go | DaemonSet | 自动标注节点 | Pod 监控、标签管理 |
| **Health Events Analyzer** | Go | Deployment | 事件模式分析 | 时间窗口聚合 |
| **Event Exporter** | Go | Deployment | 导出事件到外部系统 | CloudEvents 格式 |
| **Store Client** | Go | 共享库 | 数据库抽象层 | MongoDB/PostgreSQL 双支持 |
| **Commons** | Go | 共享库 | 公共工具库 | 日志、配置、K8s 客户端 |

---

### 学习检查点 - 架构理解

- [ ] 能画出 NVSentinel 的完整架构图
- [ ] 理解为什么采用事件驱动架构
- [ ] 知道 Platform Connectors 的 Pipeline 是如何工作的
- [ ] 理解 Change Stream 如何实现模块解耦
- [ ] 知道熔断器的作用和实现原理
- [ ] 能解释 CEL 策略引擎的优势
- [ ] 理解为什么选择数据库而非消息队列

---

## Phase 0: NVIDIA/GPU 基础知识 ⭐⭐⭐⭐⭐

> **重要提示**: 这是理解 NVSentinel 的**必修章节**。如果不理解 GPU 监控的基本概念，将无法深入理解整个系统的设计意图。

---

### 0.1 DCGM (Data Center GPU Manager) ⭐⭐⭐⭐⭐
**学习重点**: NVIDIA 的数据中心 GPU 监控框架

**什么是 DCGM？**

DCGM (Data Center GPU Manager) 是 NVIDIA 提供的一套用于监控和管理数据中心的 GPU 的套件。它包含：

1. **DCGM Engine**: 后台守护进程，负责采集 GPU 数据
2. **DCGM Python Bindings**: Python API 接口
3. **Health Checks**: 内置的健康检查系统
4. **Field Monitoring**: 数百个 GPU 监控指标

**核心概念**:

```python
# DCGM 健康监控类型 (Health Watches)
DCGM_HEALTH_WATCH_MEM             # 显存健康
DCGM_HEALTH_WATCH_PROC            # 处理器核心健康
DCGM_HEALTH_WATCH_NVLINK          # NVLink 互连健康
DCGM_HEALTH_WATCH_PCI             # PCI 总线健康
DCGM_HEALTH_WATCH_THERMAL         # 热管理健康
DCGM_HEALTH_WATCH_POWER           # 电源管理健康
DCGM_HEALTH_WATCH_DRIVER          # 驱动程序健康
```

**学习文件**:
- `health-monitors/gpu-health-monitor/gpu_health_monitor/dcgm_watcher/dcgm.py`

**代码解析**:

```python
class DCGMWatcher:
    """DCGM 监控器核心类"""

    def _get_available_health_watches(self) -> dict[int, str]:
        """动态获取所有可用的健康监控类型"""
        health_watches = {}
        for var in dir(dcgm_structs):
            if (var.startswith("DCGM_HEALTH_WATCH")
                and not var.endswith("ALL")
                and not "_COUNT_" in var):
                health_watches[getattr(dcgm_structs, var)] = var
        return health_watches

    def _perform_health_check(self, dcgm_group: pydcgm.DcgmGroup):
        """执行 DCGM 健康检查"""
        # 调用 DCGM API 执行健康检查
        health_details = dcgm_group.health.Check()

        # 解析健康检查结果
        for i in range(health_details.incidentCount):
            incident = health_details.incidents[i]
            watch_name = self._health_watches[incident.system]
            gpu_id = incident.entityInfo.entityId
            error_code = self._error_codes[incident.error.code]
```

**设计亮点**:
- ✅ **动态发现**: 不硬编码健康监控类型，从 DCGM 库动态获取
- ✅ **分组管理**: 使用 DcgmGroup 统一管理多个 GPU 和 NVSwitch
- ✅ **错误累积**: 同一 GPU 的多个错误会合并到一个事件中
- ✅ **连接性检测**: 超时或错误会被检测到并触发重连

**相关知识点**:
- DCGM Field IDs: `DCGM_FI_DEV_*` 开头的数百个字段
- DCGM Error Codes: `DCGM_FR_*` 开头的错误码
- DCGM Entity Types: `DCGM_FE_GPU`, `DCGM_FE_SWITCH` 等

**抄写建议**:
1. 完整抄写 `dcgm.py` 文件
2. 理解 DCGM 健康检查的返回结构
3. 学习如何使用 DCGM Python API
4. 尝试运行 `dcgmdiscover` 和 `dcgmi` 命令行工具

---

### 0.2 XID 错误系统 ⭐⭐⭐⭐⭐
**学习重点**: NVIDIA GPU 的错误报告机制

**什么是 XID？**

XID (NVIDIA XID) 是 NVIDIA 内核驱动报告 GPU 错误的标准格式。每个 XID 错误码代表一种特定的 GPU 硬件或软件问题。

**XID 格式**:

```
NVRM: Xid (PCI:0000:b3:00.0): 79, pid=1234, name=process, Ch 00000001
│     │   │                │    │       │              │
│     │   │                │    │       │              └─ Channel/额外信息
│     │   │                │    │       └─ 触发进程名
│     │   │                │    └─ 触发进程 PID
│     │   │                └─ XID 错误码 (关键！)
│     │   └─ PCI 地址
│     └─ 固定格式
└─ NVIDIA 驱动前缀
```

**常见 XID 错误码**:

| XID | 含义 | 严重程度 | 建议操作 |
|-----|------|---------|---------|
| 13  | GPU 图形引擎寄存器损坏 | 致命 | 重置/更换 GPU |
| 31  | GPU 驱动程序内部错误 | 致命 | 重启节点/更新驱动 |
| 43  | GPU 停止响应 | 致命 | 重置 GPU |
| 48  | 双位 ECC 错误 | 致命 | 重置/更换 GPU |
| 61  | GPU 翻转 (热问题) | 致命 | 检查散热/重置 |
| 62  | 翻转后 IO 错误 | 致命 | 重置/更换 GPU |
| 74  | IOCTL 错误 | 非致命 | 重启进程 |
| 79  | GPU 处于糟糕状态 | 致命 | 重置 GPU |

**学习文件**:
- `health-monitors/syslog-health-monitor/pkg/patterns/xid.go` - XID 正则表达式
- `health-monitors/syslog-health-monitor/pkg/xid/xid_handler.go` - XID 处理器

**代码解析**:

```go
// XID 匹配模式 (patterns/xid.go)
var XIDPattern = regexp.MustCompile(
    `NVRM: Xid \(PCI:([0-9a-fA-F:.]+)\): (\d+)(?:, pid=(\d+))?(?:, name=([^,]+))?(?:, Ch ([0-9a-fA-F]+))?`,
)

// XID 处理器 (xid_handler.go)
func (xidHandler *XIDHandler) ProcessLine(message string) (*pb.HealthEvents, error) {
    // 1. 解析 XID 格式日志
    xidResp, err := xidHandler.parser.Parse(message)

    // 2. 提取 PCI 地址和 GPU UUID
    pciID := xidResp.Result.PCIE
    gpuUUID := xidHandler.getGPUUUID(pciID)

    // 3. 创建健康事件
    event := &pb.HealthEvent{
        ComponentClass: "GPU",
        ErrorCode:      []string{xidResp.Result.DecodedXIDStr},
        EntitiesImpacted: []*pb.Entity{
            {EntityType: "PCI", EntityValue: pciID},
            {EntityType: "GPU_UUID", EntityValue: gpuUUID},
        },
        IsFatal: xidHandler.determineFatality(recommendedAction),
        RecommendedAction: recommendedAction,
    }
}
```

**设计亮点**:
- ✅ **正则匹配**: 单一正则表达式解析所有 XID 格式
- ✅ **PCI 到 UUID 映射**: 动态维护 PCI 地址到 GPU UUID 的映射
- ✅ **XID 分析服务**: 可选的外部服务进行深度 XID 分析
- ✅ **特定元数据**: 不同 XID 有不同的元数据（如 XID 13 的 GPC/TPC/SM）

**相关知识点**:
- 内核日志: 通过 `journalctl` 读取内核日志
- PCI 地址: GPU 在 PCIe 总线上的地址 (如 `0000:b3:00.0`)
- GPU UUID: GPU 的唯一标识符 (如 `GPU-deadbeef-1234-5678-abcd`)

**抄写建议**:
1. 抄写 `patterns/xid.go` 理解正则表达式
2. 抄写 `xid_handler.go` 理解 XID 处理流程
3. 查看 `xid/parser/` 下的不同解析器实现
4. 使用 `dmesg | grep Xid` 查看实际系统的 XID 错误

---

### 0.3 GPU 拓扑结构 ⭐⭐⭐⭐
**学习重点**: 多 GPU 系统的互连架构

**核心概念**:

```
┌─────────────────────────────────────────────────────┐
│                    节点 (Node)                       │
│                                                       │
│  ┌─────────┐ NVLink ┌─────────┐ NVLink ┌─────────┐ │
│  │ GPU 0   │<──────>│ GPU 1   │<──────>│ GPU 2   │ │
│  │ H100    │        │ H100    │        │ H100    │ │
│  └────┬────┘        └────┬────┘        └────┬────┘ │
│       │                 │                   │       │
│       └─────────────────┼───────────────────┘       │
│                         │                           │
│                    ┌────▼────┐                      │
│                    │ NVSwitch │                      │
│                    │  (NDR)   │                      │
│                    └─────────┘                      │
│                         │                           │
│                    PCIe 4.0/5.0                     │
│                         │                           │
│                    ┌────▼────┐                      │
│                    │  CPU    │                      │
│                    └─────────┘                      │
└─────────────────────────────────────────────────────┘
```

**关键组件**:

| 组件 | 说明 | DCGM 字段 |
|-----|------|----------|
| **GPU** | 图形处理单元 | `DCGM_FE_GPU` |
| **NVLink** | GPU 间高速互连 | `DCGM_HEALTH_WATCH_NVLINK` |
| **NVSwitch** | 多 GPU 交换机 | `DCGM_FE_SWITCH` |
| **PCIe** | 主机连接总线 | `DCGM_HEALTH_WATCH_PCI` |

**DCGM 拓扑发现**:

```python
# 创建 DCGM 组，包含所有 GPU 和 NVSwitch
dcgm_system = dcgm_handle.GetSystem()

# 获取所有支持的 GPU
supported_gpus = dcgm_system.discovery.GetEntityGroupEntities(
    dcgm_fields.DCGM_FE_GPU,  # GPU 实体类型
    True  # 只返回支持的实体
)

# 获取所有支持的 NVSwitch
supported_switches = dcgm_system.discovery.GetEntityGroupEntities(
    dcgm_fields.DCGM_FE_SWITCH,  # NVSwitch 实体类型
    True
)

# 将所有实体添加到监控组
for gpu in supported_gpus:
    dcgm_group.AddEntity(dcgm_fields.DCGM_FE_GPU, gpu)

for switch in supported_switches:
    dcgm_group.AddEntity(dcgm_fields.DCGM_FE_SWITCH, switch)
```

**Metadata Collector 模块**:

**文件**: `metadata-collector/` (独立模块)

该模块负责收集 GPU 拓扑信息：
- GPU UUID 和序列号
- PCI 地址和总线位置
- NVLink 连接拓扑
- NVSwitch 配置
- 机箱序列号

**元数据 JSON 示例**:

```json
{
  "gpus": [
    {
      "gpu_id": 0,
      "uuid": "GPU-deadbeef-1234-5678-abcd",
      "pci_address": "0000:b3:00.0",
      "serial": "0324517892345"
    },
    {
      "gpu_id": 1,
      "uuid": "GPU-deadbeef-abcd-efgh-ijkl",
      "pci_address": "0000:b4:00.0",
      "serial": "0324517892346"
    }
  ],
  "chassis_serial": "CHS123456789",
  "nvlink_topology": "..."
}
```

**抄写建议**:
1. 查看 `metadata-collector/` 模块了解拓扑收集
2. 学习 GPU UUID 和 PCI 地址的映射关系
3. 使用 `nvidia-smi topo -m` 查看实际拓扑
4. 理解 NVLink 带宽和 NVSwitch 的作用

---

### 0.4 GPU 元数据管理 ⭐⭐⭐⭐
**学习重点**: GPU 标识和追踪

**核心文件**:
- `health-monitors/gpu-health-monitor/gpu_health_monitor/metadata/reader.py`
- `health-monitors/syslog-health-monitor/pkg/metadata/`

**GPU 标识符对比**:

| 标识符 | 格式示例 | 稳定性 | 用途 |
|--------|---------|--------|------|
| **GPU ID** | `0, 1, 2...` | 不稳定 | DCGM 内部使用 |
| **PCI Address** | `0000:b3:00.0` | 稳定 | 内核日志引用 |
| **GPU UUID** | `GPU-deadbeef-1234...` | 最稳定 | 唯一标识 |
| **Serial Number** | `0324517892345` | 最稳定 | 硬件追踪 |

**元数据读取器设计**:

```python
class MetadataReader:
    """线程安全、延迟加载的 GPU 元数据读取器"""

    def __init__(self, metadata_path: str):
        self._path = metadata_path
        self._metadata = None
        self._lock = threading.RLock()
        self._loaded = False

    def _ensure_loaded(self):
        """双重检查锁定的延迟加载模式"""
        if self._loaded:
            return

        with self._lock:
            if self._loaded:
                return

            with open(self._path, "r") as f:
                self._metadata = json.load(f)
            self._loaded = True

    def get_gpu_uuid(self, gpu_id: int) -> Optional[str]:
        """通过 DCGM GPU ID 获取 GPU UUID"""
        self._ensure_loaded()

        for gpu in self._metadata.get("gpus", []):
            if gpu.get("gpu_id") == gpu_id:
                return gpu.get("uuid")
        return None

    def get_pci_address(self, gpu_id: int) -> Optional[str]:
        """通过 DCGM GPU ID 获取 PCI 地址"""
        # ... 类似实现
```

**设计亮点**:
- ✅ **延迟加载**: 只在首次使用时加载元数据
- ✅ **线程安全**: 使用 RLock 保证并发安全
- ✅ **容错处理**: 文件不存在时优雅降级
- ✅ **多格式支持**: JSON 文件格式灵活

**PCI 地址规范化**:

```go
// XID 处理中的 PCI 地址规范化
func normalizePCI(pci string) string {
    // "0000:b3:00.0" -> "0000:b3:00"
    // 去除函数号，只保留总线地址
    if idx := strings.Index(pci, "."); idx != -1 {
        return pci[:idx]
    }
    return pci
}
```

**抄写建议**:
1. 抄写 `metadata/reader.py` 完整实现
2. 理解双重检查锁定模式
3. 学习 GPU ID 到 UUID 的映射逻辑
4. 查看 `metadata-collector` 如何生成元数据文件

---

### 0.5 NVIDIA 驱动与内核交互 ⭐⭐⭐
**学习重点**: GPU 错误如何从硬件到内核日志

**错误传播路径**:

```
GPU 硬件故障
    ↓
GPU 内部寄存器/传感器
    ↓
DCGM 引擎 (用户空间)
    ↓
NVIDIA 内核驱动 (nvidia.ko)
    ↓
内核日志 (dmesg/journalctl)
    ↓
Syslog Health Monitor 读取
    ↓
创建 HealthEvent
```

**内核日志源**:

```go
// Syslog Monitor 读取内核日志
journal, _ := sdjournal.NewJournal()
journal.AddMatch("SYSLOG_IDENTIFIER=kernel")  // 只看内核日志
journal.AddMatch("_TRANSPORT=kernel")         // 内核传输
journal.SeekTail()                            // 从最新日志开始

// 读取新日志
for journal.Next() > 0 {
    entry, _ := journal.GetEntry()
    message := entry.Fields["MESSAGE"]

    // 解析 XID 错误
    if strings.Contains(message, "NVRM: Xid") {
        // 处理 XID
    }
}
```

**DCGM vs 内核日志监控**:

| 特性 | DCGM Monitor | Syslog Monitor |
|-----|-------------|----------------|
| **监控方式** | 主动轮询 DCGM API | 被动读取内核日志 |
| **监控内容** | 系统化的健康检查 | 驱动报告的原始错误 |
| **覆盖范围** | GPU、NVSwitch 等所有组件 | 主要是 GPU XID 错误 |
| **延迟** | 轮询间隔 (通常秒级) | 日志产生即读取 (毫秒级) |
| **语言** | Python | Go |

**两种监控互补**:
- **DCGM Monitor**: 定期全面健康检查
- **Syslog Monitor**: 实时捕获关键错误

**抄写建议**:
1. 查看 `syslog-health-monitor/pkg/syslog-monitor/` 理解 journald 集成
2. 使用 `journalctl -k` 查看内核日志
3. 使用 `dmesg -w` 实时监控内核消息
4. 理解 sd-go 库如何访问 systemd journal

---

### 0.6 DCGM 字段监控 ⭐⭐⭐
**学习重点**: 数百个 GPU 监控指标

**DCGM 字段分类**:

```python
# 常见的 DCGM 监控字段
DCGM_FI_DEV_GPU_TEMP           # GPU 温度
DCGM_FI_DEV_POWER_USAGE        # 功耗
DCGM_FI_DEV_FB_FREE            # 显存可用量
DCGM_FI_DEV_ECC_SBE_AGG_TOTAL  # 单位 ECC 错误总数
DCGM_FI_DEV_ECC_DBE_AGG_TOTAL  # 双位 ECC 错误总数
DCGM_FI_DEV_PCIE_REPLAY_COUNT  # PCIe 重放计数
DCGM_FI_DEV_CLOCK_THROTTLE     # 时钟降频原因
DCGM_FI_DEV_GPU_UTIL           # GPU 利用率
DCGM_FI_DEV_MEM_UTIL           # 显存利用率
```

**字段监控 vs 健康检查**:

| 特性 | 健康检查 (Health Check) | 字段监控 (Field Watch) |
|-----|------------------------|----------------------|
| **API** | `dcgm_group.health.Check()` | `dcgm_group.WatchFields()` |
| **返回** | 通过/失败 + 错误详情 | 实时字段值 |
| **适用场景** | 故障检测 | 性能监控/趋势分析 |
| **NVSentinel 用法** | GPU Health Monitor | 可扩展用于性能监控 |

**抄写建议**:
1. 查看 DCGM 文档了解所有字段
2. 使用 `dcgmi dmon -e 155,157` 监控特定字段
3. 理解为什么 NVSentinel 主要使用健康检查而非字段监控

---

### Phase 0 学习检查点

- [ ] 能解释 DCGM 是什么，它如何监控 GPU
- [ ] 理解 XID 错误的格式和常见错误码
- [ ] 知道 GPU UUID、PCI Address、GPU ID 的区别
- [ ] 理解 NVLink 和 NVSwitch 在 GPU 集群中的作用
- [ ] 能画出从 GPU 硬件故障到 HealthEvent 的完整路径
- [ ] 知道 DCGM Monitor 和 Syslog Monitor 的区别和互补性

---

## Phase 1: 基础设施层 (Foundation)

### 1.1 Protobuf 数据模型 ⭐⭐⭐⭐⭐
**学习重点**: 理解系统核心数据契约

**文件**: `data-models/protobufs/health_event.proto`

**学习内容**:
```protobuf
message HealthEvent {
  uint32 version = 1;                    // 版本号 - 向后兼容的关键
  string agent = 2;                      // 监控器标识
  string componentClass = 3;             // 组件类型 (GPU/NIC/DISK等)
  string nodeName = 13;                  // K8s 节点名
  string checkName = 4;                  // 检查项名称
  bool isFatal = 5;                      // 是否致命 - 触发自动隔离
  bool isHealthy = 6;                    // 健康状态
  string message = 7;                    // 人可读描述
  RecommendedAction recommendedAction = 8;  // 建议操作
  repeated string errorCode = 9;         // 错误码列表
  repeated Entity entitiesImpacted = 10; // 受影响的实体
  map<string, string> metadata = 11;     // 扩展元数据
  google.protobuf.Timestamp generatedTimestamp = 12;  // 生成时间
  BehaviourOverrides quarantineOverrides = 14;  // 隔离行为覆盖
  BehaviourOverrides drainOverrides = 15;         // 排空行为覆盖
}
```

**设计亮点**:
- ✅ **版本字段**: 支持协议演化和向后兼容
- ✅ **双重标志**: `isFatal` 和 `isHealthy` 分离故障程度和健康状态
- ✅ **覆盖机制**: 允许单个事件覆盖全局行为（高级特性）
- ✅ **时间戳**: 使用 protobuf 标准时间类型，支持跨语言
- ✅ **扩展性**: `metadata` map 允许任意扩展字段

**抄写建议**:
1. 完整抄写 `health_event.proto`
2. 理解每个字段的使用场景
3. 思考: 为什么需要 `recommendedAction` 枚举？
4. 思考: `BehaviourOverrides` 如何实现策略灵活性？

**相关知识**:
- Protobuf 字段演化规则（添加字段安全、删除字段危险）
- Protobuf 枚举设计模式

---

### 1.2 数据库抽象层 ⭐⭐⭐⭐⭐
**学习重点**: 如何设计数据库无关的抽象接口

**核心文件**:
- `store-client/pkg/datastore/interfaces.go` - 接口定义
- `store-client/pkg/datastore/factory.go` - 工厂模式
- `store-client/pkg/query/builder.go` - 查询构建器

**学习内容**:

#### 1.2.1 接口设计

```go
// 数据存储抽象接口
type DataStore interface {
    MaintenanceEventStore() MaintenanceEventStore
    HealthEventStore() HealthEventStore
    Ping(ctx context.Context) error
    Close(ctx context.Context) error
    Provider() DataStoreProvider
}

// 健康事件存储接口
type HealthEventStore interface {
    UpdateHealthEventStatus(ctx context.Context, id string, status HealthEventStatus) error
    FindHealthEventsByNode(ctx context.Context, nodeName string) ([]HealthEventWithStatus, error)
    FindHealthEventsByQuery(ctx context.Context, builder QueryBuilder) ([]HealthEventWithStatus, error)
    UpdateHealthEventsByQuery(ctx context.Context, queryBuilder QueryBuilder, updateBuilder UpdateBuilder) error
}
```

**设计亮点**:
- ✅ **接口隔离**: 每个接口职责单一
- ✅ **Context 传递**: 所有方法支持超时和取消
- ✅ **数据库无关**: 接口不暴露 MongoDB/PostgreSQL 特定类型
- ✅ **返回接口**: 返回 `HealthEventWithStatus` 而非具体类型

#### 1.2.2 查询构建器模式

```go
// 数据库无关的查询构建
type Builder struct {
    root Condition
}

// 使用示例
query := query.New().
    Build(query.And(
        query.Eq("nodename", "node-1"),
        query.Gte("createdAt", time.Now().Add(-24*time.Hour)),
    ))

// 自动转换为:
// MongoDB:   {"nodename": "node-1", "createdAt": {"$gte": timestamp}}
// PostgreSQL: nodename = $1 AND createdAt >= $2
```

**设计亮点**:
- ✅ **Builder 模式**: 链式调用，流畅 API
- ✅ **双重生成**: 一个接口生成两种数据库查询
- ✅ **类型安全**: 编译时检查查询结构
- ✅ **SQL 注入防护**: 参数化查询

**抄写建议**:
1. 先抄写 `interfaces.go` - 理解接口设计
2. 抄写 `factory.go` - 学习工厂模式如何选择数据库实现
3. 抄写 `query/builder.go` - 深入理解查询抽象
4. 对比 `client/mongodb_*` 和 `client/postgresql_*` - 看同一接口的不同实现

**相关知识**:
- Go 接口设计原则（小接口、组合优于继承）
- Builder 模式在 Go 中的实现
- MongoDB 和 PostgreSQL 的差异（文档型 vs 关系型）

---

### 1.3 公共工具库 ⭐⭐⭐
**学习重点**: 提取可复用组件的思路

**核心子包**:
- `commons/pkg/logger/` - 结构化日志
- `commons/pkg/configmanager/` - 配置管理
- `commons/pkg/flags/` - 命令行标志
- `commons/pkg/statemanager/` - 状态管理（核心！）

#### 1.3.1 状态管理器 ⭐⭐⭐⭐⭐

**文件**: `commons/pkg/statemanager/statemanager.go`

```go
// 节点状态转换机
type StateManager struct {
    k8sClient kubernetes.Interface
}

// 状态转换图
// none → quarantined → draining → drain-succeeded → remediating → remediation-succeeded
//                                              ↘ remediation-failed (终态)
```

**设计亮点**:
- ✅ **状态机模式**: 明确的状态转换规则
- ✅ **Kubernetes Label 存储**: 状态持久化在节点标签上
- ✅ **终态设计**: 有明确的成功/失败终态
- ✅ **可恢复性**: 重启后可以从 Label 恢复状态

**抄写建议**:
1. 抄写状态常量定义
2. 抄写状态转换逻辑
3. 绘制状态转换图
4. 思考: 为什么需要终态？如何防止状态回退？

---

## Phase 2: 事件摄入层 (Ingestion)

### 2.1 Platform Connectors 架构 ⭐⭐⭐⭐⭐
**学习重点**: 事件处理管道的设计

**核心文件**:
- `platform-connectors/pkg/server/platform_connector_server.go` - gRPC 服务
- `platform-connectors/pkg/pipeline/pipeline.go` - 处理管道
- `platform-connectors/pkg/connectors/` - 连接器实现

**学习内容**:

#### 2.1.1 gRPC 服务实现

```go
type platformConnectorServer struct {
    pb.UnimplementedPlatformConnectorServer
    pipeline *pipeline.Pipeline
}

func (s *platformConnectorServer) HealthEventOccurredV1(
    ctx context.Context,
    req *pb.HealthEvents,
) (*emptypb.Empty, error) {
    // 处理健康事件
    for _, event := range req.Events {
        if err := s.pipeline.Process(ctx, event); err != nil {
            return nil, err
        }
    }
    return &emptypb.Empty{}, nil
}
```

**设计亮点**:
- ✅ **Pipeline 模式**: 责任链处理事件
- ✅ **批量处理**: 单次 gRPC 调用可包含多个事件
- ✅ **异步处理**: 可扩展为异步队列

#### 2.1.2 处理管道

```go
type Pipeline struct {
    connectors   []Connector      // 持久化连接器
    transformers []Transformer    // 转换器链
}

type Connector interface {
    Process(ctx context.Context, event *pb.HealthEvent) error
}

type Transformer interface {
    Transform(ctx context.Context, event *pb.HealthEvent) (*pb.HealthEvent, error)
}
```

**处理流程**:
```
gRPC 请求
    ↓
Transformer 1: Metadata Enrichment (添加 GPU UUID、驱动版本)
    ↓
Transformer 2: Overrides Application (应用行为覆盖)
    ↓
Connector 1: Kubernetes Storage (更新 Node Condition)
    ↓
Connector 2: Database Storage (持久化到 MongoDB/PostgreSQL)
    ↓
完成
```

**设计亮点**:
- ✅ **责任链模式**: 每个 Transformer 职责单一
- ✅ **可扩展**: 新增 Transformer 无需修改现有代码
- ✅ **有序处理**: Transformers 按顺序执行
- ✅ **多持久化**: 同时写入 K8s 和数据库

**抄写建议**:
1. 抄写 `pipeline.go` 理解核心流程
2. 抄写一个 Transformer 实现（如 `metadata_transformer.go`）
3. 抄写一个 Connector 实现（如 `kubernetes_storage.go`）
4. 思考: 如何添加新的 Transformer？

---

## Phase 3: 检测层 (Detection)

### 3.1 GPU Health Monitor (Python) ⭐⭐⭐⭐
**学习重点**: Python 如何与 gRPC 服务通信

**核心文件**:
- `health-monitors/gpu-health-monitor/gpu_health_monitor/platform_connector/platform_connector.py`
- `health-monitors/gpu-health-monitor/gpu_health_monitor/dcgmtypes.py`

**学习内容**:

#### 3.1.1 gRPC 客户端实现

```python
class PlatformConnectorEventProcessor:
    def __init__(self, socket_path: str, node_name: str, ...):
        self._socket_path = socket_path  # Unix Domain Socket
        self._node_name = node_name
        self.entity_cache = {}           # 去重缓存

    def send_health_event(self, health_event):
        """通过 gRPC 发送健康事件"""
        # 批量处理
        with self._lock:
            self._batch.append(health_event)
            if len(self._batch) >= self._batch_size:
                self._flush_batch()

    def _flush_batch(self):
        """批量发送事件"""
        events = pb.HealthEvents()
        for event in self._batch:
            events.events.append(event)

        self.stub.HealthEventOccurredV1(events)
```

**设计亮点**:
- ✅ **Unix Domain Socket**: 本地高效通信
- ✅ **批量发送**: 减少 gRPC 调用开销
- ✅ **去重缓存**: 防止重复事件
- ✅ **连接重试**: 自动重连机制

#### 3.1.2 DCGM 集成

```python
class DCGMWatcher:
    """监控 DCGM 字段变化"""
    def __init__(self, callback: dcgmtypes.CallbackInterface):
        self._callback = callback

    def on_field_value_change(self, field, value):
        """DCGM 字段变化回调"""
        if self._is_error_condition(field, value):
            health_event = self._create_health_event(field, value)
            self._callback.send_health_event(health_event)
```

**抄写建议**:
1. 抄写 `platform_connector.py` 理解 gRPC 客户端
2. 抄写 `dcgmtypes.py` 理解事件构建
3. 了解 DCGM Python API 使用方式
4. 思考: Python 和 Go 如何共享 Protobuf 定义？

---

### 3.2 Syslog Health Monitor (Go) ⭐⭐⭐
**学习重点**: 如何监控 journalctl

**核心文件**:
- `health-monitors/syslog-health-monitor/pkg/monitor/journald_monitor.go`
- `health-monitors/syslog-health-monitor/pkg/parser/xid_parser.go`

**学习内容**:

```go
type JournaldMonitor struct {
    ctx       context.Context
    parsers   []Parser
    sender    EventSender
}

func (m *JournaldMonitor) Start() {
    // 使用 sd-go 库读取 journal
    journal, _ := sdjournal.NewJournal()
    journal.AddMatch("SYSLOG_IDENTIFIER=kernel")
    journal.SeekTail()

    for {
        count, err := journal.Next()
        if count == 0 {
            time.Sleep(1 * time.Second)
            continue
        }

        entry, _ := journal.GetEntry()
        // 使用 parsers 解析日志
        for _, parser := range m.parsers {
            if event := parser.Parse(entry); event != nil {
                m.sender.Send(event)
                break
            }
        }
    }
}
```

**设计亮点**:
- ✅ **解析器链**: 多个解析器依次尝试
- ✅ **增量读取**: 只读取新日志
- ✅ **系统调用**: 使用 sd-go 直接访问 journald

---

## Phase 4: 响应层 (Response)

### 4.1 Fault Quarantine (故障隔离) ⭐⭐⭐⭐⭐
**学习重点**: CEL 策略引擎 + Kubernetes 控制器

**核心文件**:
- `fault-quarantine/pkg/evaluator/rule_evaluator.go` - CEL 评估
- `fault-quarantine/pkg/reconciler/reconciler.go` - K8s 控制器
- `fault-quarantine/pkg/breaker/breaker.go` - 熔断器

**学习内容**:

#### 4.1.1 CEL 策略评估 ⭐⭐⭐⭐⭐

```go
type HealthEventRuleEvaluator struct {
    expression string
    program    cel.Program
}

// 规则表达式示例:
// "event.componentClass == 'GPU' && event.isFatal == true"
// "event.errorCode.exists(e, e == 'XID-48')"
// "event.metadata.gpu_uuid.matches('^GPU-[0-9a-f-]+$')"

func (r *HealthEventRuleEvaluator) Evaluate(event *pb.HealthEvent) (bool, error) {
    vars, _ :=CELVariable("event", event)
    result, _, err := r.program.ContextEval(vars)
    return result.Value().(bool), err
}
```

**设计亮点**:
- ✅ **声明式策略**: 用户无需写代码
- ✅ **类型安全**: CEL 编译时检查类型
- ✅ **高性能**: 编译后的表达式执行很快
- ✅ **可组合**: 支持复杂的逻辑表达式

#### 4.1.2 熔断器模式 ⭐⭐⭐⭐⭐

```go
type slidingWindowBreaker struct {
    cfg         Config
    buckets     []int          // 环形缓冲区
    startTime   time.Time
    state       State          // Closed or Tripped
    nodeToIndex map[string]int // 去重节点
}

// 防止在时间窗口内隔离超过 X% 的节点
func (b *slidingWindowBreaker) Record(nodeName string) error {
    if b.state == StateTripped {
        return errors.New("circuit breaker tripped")
    }

    b.addToBucket(nodeName)
    if b.percentageExceeded() {
        b.state = StateTripped
        return errors.New("circuit breaker tripped")
    }
    return nil
}
```

**设计亮点**:
- ✅ **滑动窗口**: 精确统计时间范围内的操作
- ✅ **百分比限制**: 防止过多节点被隔离
- ✅ **自动恢复**: 窗口过后自动重置
- ✅ **去重节点**: 同一节点只计数一次

#### 4.1.3 Kubernetes 控制器

```go
type FaultQuarantineReconciler struct {
    client.Client
    scheme *runtime.Scheme
    evaluator *RuleEvaluator
    breaker   *Breaker
}

func (r *FaultQuarantineReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 1. 获取节点
    node := &corev1.Node{}
    if err := r.Get(ctx, req.NamespacedName, node); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 2. 检查是否需要隔离
    shouldCordon, taint := r.evaluator.Evaluate(event)

    // 3. 检查熔断器
    if err := r.breaker.Record(node.Name); err != nil {
        // 熔断器触发，跳过隔离
        return ctrl.Result{RequeueAfter: time.Minute}, nil
    }

    // 4. 执行隔离
    if shouldCordon {
        node.Spec.Unschedulable = true
        node.Spec.Taints = append(node.Spec.Taints, taint)
        return r.Update(ctx, node)
    }

    return ctrl.Result{}, nil
}
```

**设计亮点**:
- ✅ **标准控制器模式**: 使用 controller-runtime
- ✅ **幂等性**: Reconcile 可安全重试
- ✅ **错误处理**: 区分可重试和不可重试错误
- ✅ **重试机制**: RequeueAfter 实现延迟重试

**抄写建议**:
1. 抄写 `rule_evaluator.go` - 理解 CEL 集成
2. 抄写 `breaker.go` - 理解熔断器实现
3. 抄写 `reconciler.go` - 理解控制器模式
4. 思考: 如何添加新的 CEL 函数？

---

### 4.2 Node Drainer (节点排空) ⭐⭐⭐⭐
**学习重点**: 如何优雅驱逐 Pod

**核心文件**:
- `node-drainer/pkg/reconciler/reconciler.go` - 控制器
- `node-drainer/pkg/evaluator/drain_evaluator.go` - 排空策略

**学习内容**:

```go
type NodeDrainerReconciler struct {
    client.Client
    kubeClient kubernetes.Interface
    drainEvaluator *DrainEvaluator
}

func (r *NodeDrainerReconciler) drainNode(ctx context.Context, node *corev1.Node) error {
    // 1. 获取节点上的所有 Pod
    pods, _ := r.listPodsOnNode(node)

    // 2. 按 PDB 优先级排序
    sortedPods := r.sortPodsByPriority(pods)

    // 3. 逐个驱逐
    for _, pod := range sortedPods {
        // 跳过 DaemonSet 和 Mirror Pod
        if r.isSkippablePod(pod) {
            continue
        }

        // 检查 PDB
        if !r.canEvict(pod) {
            continue
        }

        // 执行驱逐
        err := r.kubeClient.CoreV1().Pods(pod.Namespace).EvictV1(ctx, &metav1.Eviction{
            ObjectMeta: metav1.ObjectMeta{Name: pod.Name, Namespace: pod.Namespace},
        })

        if err != nil {
            // 处理驱逐失败
        }
    }

    return nil
}
```

**设计亮点**:
- ✅ **PDB 尊重**: 优先驱逐不受 PDB 保护的 Pod
- ✅ **超时机制**: 强制删除超时的 Pod
- ✅ **跳过 DaemonSet**: 不会驱逐 DaemonSet Pod
- ✅ **状态跟踪**: 记录排空进度到数据库

---

### 4.3 Fault Remediation (故障修复) ⭐⭐⭐⭐
**学习重点**: CRD 设计和自定义资源

**核心文件**:
- `fault-remediation/pkg/reconciler/remediation_reconciler.go`
- `api/remediation/v1alpha1/remediation_types.go`

**学习内容**:

```go
// Remediation CRD
type Remediation struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   RemediationSpec   `json:"spec,omitempty"`
    Status RemediationStatus `json:"status,omitempty"`
}

type RemediationSpec struct {
    NodeName   string            `json:"nodeName"`
    Action     RemediationAction `json:"action"`
    Reason     string            `json:"reason"`
    HealthEventRef string        `json:"healthEventRef"`
}

type RemediationAction struct {
    Type   string              `json:"type"`   // "RebootNode", "TerminateNode", "GpuReset"
    Params map[string]string   `json:"params"`
}

// 状态
type RemediationStatus struct {
    Phase      RemediationPhase `json:"phase"`
    Message    string           `json:"message,omitempty"`
    StartedAt  *metav1.Time     `json:"startedAt,omitempty"`
    FinishedAt *metav1.Time     `json:"finishedAt,omitempty"`
}
```

**设计亮点**:
- ✅ **CRD 即状态**: 利用 Kubernetes 作为状态存储
- ✅ **声明式 API**: 用户描述期望状态
- ✅ **审计跟踪**: Kubernetes 自动记录操作历史
- ✅ **可扩展**: 新增修复类型只需添加新 CR

---

## Phase 5: 高级特性 (Advanced)

### 5.1 Change Stream 实现 ⭐⭐⭐⭐⭐
**学习重点**: 如何实现数据库无关的变更流

**核心文件**:
- `store-client/pkg/storewatcher/change_stream.go` - 抽象接口
- `store-client/pkg/client/mongodb_changestream.go` - MongoDB 实现
- `store-client/pkg/client/postgresql_changestream.go` - PostgreSQL 实现

**学习内容**:

```go
// 数据库无关的变更流接口
type ChangeStreamWatcher interface {
    Start(ctx context.Context)
    Events() <-chan Event
    MarkProcessed(ctx context.Context, token []byte) error
    Close(ctx context.Context) error
}

type Event struct {
    OperationType OperationType
    Document      []byte
    Token         ResumeToken  // 用于断点续传
}

// MongoDB 实现
type mongoChangeStream struct {
    stream *mongo.ChangeStream
    events chan Event
}

func (m *mongoChangeStream) Start(ctx context.Context) {
    for m.stream.Next(ctx) {
        var change struct {
            OperationType string `bson:"operationType"`
            FullDocument  *pb.HealthEvent `bson:"fullDocument"`
            _id           primitive.RawValue `bson:"_id"`
        }
        m.stream.Decode(&change)

        m.events <- Event{
            OperationType: OperationTypeInsert,
            Document:      mustMarshal(change.FullDocument),
            Token:         m.stream.ResumeToken(),
        }
    }
}

// PostgreSQL 实现 (使用逻辑复制)
type pgChangeStream struct {
    conn *pgconn.PgConn
    slot string
    events chan Event
}

func (p *pgChangeStream) Start(ctx context.Context) {
    // 使用 PostgreSQL 逻辑复制槽
    // 解析 WAL 日志生成变更事件
}
```

**设计亮点**:
- ✅ **统一接口**: MongoDB 和 PostgreSQL 使用相同 API
- ✅ **Resume Token**: 支持断点续传
- ✅ **背压处理**: channel 缓冲防止内存溢出
- ✅ **错误恢复**: 自动重连和恢复

**抄写建议**:
1. 抄写 `change_stream.go` 接口定义
2. 选择一个数据库实现深入学习（推荐 PostgreSQL）
3. 理解 Resume Token 如何实现
4. 思考: 如何处理背压？

---

### 5.2 CEL 策略引擎深入 ⭐⭐⭐⭐
**学习重点**: 如何扩展 CEL 函数

**核心文件**:
- `fault-quarantine/pkg/evaluator/cel_functions.go`
- `platform-connectors/pkg/transformers/overrides_transformer.go`

**学习内容**:

```go
// 自定义 CEL 函数
func registerCustomFunctions(env *cel.Env) (*cel.Env, error) {
    return env.Extend(
        cel.Function("matches",
            cel.MemberOverload("string_matches", []*cel.Type{cel.StringType, cel.StringType}, cel.BoolType),
            cel.BinaryBinding(func(lhs, rhs ref.Val) ref.Val {
                return types.Bool(regexp.MustCompile(rhs.Value().(string)).MatchString(lhs.Value().(string)))
            }),
        ),
        // 添加更多自定义函数...
    )
}

// 使用
env, _ := cel.NewEnv(registerCustomFunctions)
ast, _ := env.Compile(`event.metadata.gpu_uuid.matches('^GPU-[0-9]+$')`)
```

**设计亮点**:
- ✅ **可扩展**: 轻松添加新函数
- ✅ **类型安全**: 编译时检查类型
- ✅ **高性能**: 编译后执行

---

## Phase 6: 工程实践 (Engineering)

### 6.1 构建系统设计 ⭐⭐⭐⭐
**学习重点**: 如何设计大型项目的构建系统

**核心文件**:
- `make/common.mk` - 统一构建逻辑
- `.versions.yaml` - 版本管理

**学习内容**:

```makefile
# 自动检测项目类型
ifeq ($(shell ls go.mod 2>/dev/null), go.mod)
    IS_GO_MODULE := yes
else ifeq ($(shell ls pyproject.toml 2>/dev/null), pyproject.toml)
    IS_PYTHON_MODULE := yes
endif

# Go 模块通用目标
ifeq ($(IS_GO_MODULE), yes)
build: $(GO_BIN)
    $(GO) build -o $(BIN_DIR)/$(MODULE_NAME) $(CMD_DIR)

test: $(GOTESTSUM)
    $(GOTESTSUM) --format testname -- $(TESTPKGS) -coverprofile=$(COVERAGE_FILE)

lint: $(GOLANGCI_LINT)
    $(GOLANGCI_LINT) run --config=$(ROOT_DIR)/.golangci.yml
endif

# Python 模块通用目标
ifeq ($(IS_PYTHON_MODULE), yes)
test:
    poetry run pytest

lint:
    poetry run pylint
endif
```

**设计亮点**:
- ✅ **自动检测**: 根据文件类型自动选择构建逻辑
- ✅ **版本集中**: `.versions.yaml` 单一版本来源
- ✅ **一致性**: 所有模块使用相同命令
- ✅ **并行构建**: 支持多模块并行构建

---

### 6.2 测试策略 ⭐⭐⭐⭐
**学习重点**: 如何测试 Kubernetes 控制器

**核心文件**:
- 各种 `*_test.go` 文件

**学习内容**:

```go
// 使用 envtest 测试控制器
func TestFaultQuarantineReconciler(t *testing.T) {
    // 启动真实的 API Server
    testEnv := &envtest.Environment{
        CRDDirectoryPaths: []string{filepath.Join("..", "config", "crd", "bases")},
    }

    cfg, _ := testEnv.Start()
    defer testEnv.Stop()

    // 创建真实的 k8s client
    client, _ := kubernetes.NewForConfig(cfg)

    // 创建 reconciler
    reconciler := &FaultQuarantineReconciler{
        Client: client,
        // ...
    }

    // 测试用例
    t.Run("should cordon node on fatal GPU error", func(t *testing.T) {
        // 创建测试节点
        node := &corev1.Node{
            ObjectMeta: metav1.ObjectMeta{Name: "test-node"},
        }
        client.Create(ctx, node)

        // 创建测试事件
        event := &pb.HealthEvent{
            NodeName:       "test-node",
            ComponentClass: "GPU",
            IsFatal:        true,
        }

        // 执行 Reconcile
        req := ctrl.Request{NamespacedName: types.NamespacedName{Name: "test-node"}}
        result, err := reconciler.Reconcile(ctx, req)

        // 验证节点被隔离
        updatedNode := &corev1.Node{}
        client.Get(ctx, req.NamespacedName, updatedNode)
        assert.True(t, updatedNode.Spec.Unschedulable)
    })
}
```

**设计亮点**:
- ✅ **envtest**: 使用真实 API Server，不是 fake client
- ✅ **表驱动测试**: 一个测试函数覆盖多个场景
- ✅ **可重复**: 每个测试独立命名空间

---

## 学习顺序建议

### 快速路径 (5周)
Week 1: Phase 0 (GPU 基础) + Phase 1 (基础设施)
Week 2: Phase 2 (摄入层) + Phase 3.1 (GPU Monitor)
Week 3: Phase 3.2 (XID 解析) + Phase 4.1 (Fault Quarantine)
Week 4: Phase 4.2-4.3 (Node Drainer + Remediation)
Week 5: Phase 5.1 (Change Stream) + Phase 6.1 (构建系统)

### 深度路径 (10周)
Week 1: Phase 0 (完整理解 GPU/DCGM/XID)
Week 2-3: Phase 1 (完整抄写基础设施层)
Week 4: Phase 2 + Phase 3 (检测层完整学习)
Week 5-6: Phase 4 (所有响应模块深入)
Week 7: Phase 5 (高级特性)
Week 8: Phase 6 (工程实践)
Week 9-10: 综合项目 (添加新功能)

---

## 学习检查点

### Level 0: GPU 基础 ⭐
- [ ] 能解释 DCGM 是什么，它如何监控 GPU
- [ ] 理解 XID 错误的格式和常见错误码含义
- [ ] 知道 GPU UUID、PCI Address、GPU ID 的区别和用途
- [ ] 理解 NVLink 和 NVSwitch 在 GPU 集群中的作用
- [ ] 能画出从 GPU 硬件故障到 HealthEvent 的完整路径
- [ ] 知道 DCGM Monitor 和 Syslog Monitor 的区别和互补性

### Level 1: 基础理解
- [ ] 能解释 HealthEvent 的每个字段含义
- [ ] 理解为什么需要数据库抽象层
- [ ] 能画出事件处理的完整流程图
- [ ] 知道 GPU 故障如何触发节点隔离

### Level 2: 代码熟悉
- [ ] 能不看代码实现一个简单的 Transformer
- [ ] 理解 CEL 表达式如何编译和执行
- [ ] 能解释熔断器的工作原理
- [ ] 理解 XID 解析的正则表达式

### Level 3: 深入理解
- [ ] 能实现一个新的 Health Monitor
- [ ] 能添加新的 CEL 函数
- [ ] 理解 Change Stream 的实现细节
- [ ] 能理解 DCGM 健康检查的分组管理

### Level 4: 架构掌握
- [ ] 能设计一个新的响应模块
- [ ] 理解为什么选择数据库而非消息队列
- [ ] 能提出架构改进建议
- [ ] 理解 GPU 隔离与节点隔离的权衡

### Level 5: 源码精通
- [ ] 能重构一个模块保持接口不变
- [ ] 能为项目添加新数据库支持
- [ ] 能指导他人学习项目
- [ ] 能扩展 DCGM 支持新的健康检查类型

---

## 实践项目建议

### 初级项目
1. **添加新的 Health Monitor**: 监控网络健康 (参考 syslog-monitor)
2. **理解 XID 错误码**: 收集并整理常见 XID 错误的含义
3. **GPU 元数据解析**: 编写工具解析 `nvidia-smi -q` 输出

### 中级项目
4. **实现新的 Remediation Action**: 添加 GPU 单卡隔离功能
5. **添加新的 CEL 函数**: 实现时间窗口内的错误计数
6. **DCGM 字段监控**: 扩展 GPU Monitor 支持自定义字段阈值

### 高级项目
7. **实现新的数据库支持**: 添加 MySQL 支持
8. **XID 分析服务**: 实现一个外部服务深度分析 XID 错误
9. **GPU 预测性维护**: 基于 DCGM 历史数据预测 GPU 故障
10. **优化性能**: 批量处理事件减少数据库调用

---

## 学习资源

### 项目内部
- `docs/designs/` - 架构设计文档 (26 篇设计文档)
- `DEVELOPMENT.md` - 开发指南
- `CLAUDE.md` - 项目概览
- `LEARN-PLAN.md` - 本学习计划

### NVIDIA/GPU 相关资源 ⭐
- [NVIDIA DCGM 官方文档](https://developer.nvidia.com/dcgm)
- [DCGM Python API](https://github.com/NVIDIA/dcgm-diagnostic-aggregator/blob/main/README.md)
- [NVIDIA XID 错误码解释](https://docs.nvidia.com/deploy/driver-powers/index.html) - 搜索 "Xid"
- [nvidia-smi 用户手册](https://developer.nvidia.com/nvidia-management-library-nvml)
- [NVLink 和 NVSwitch 架构](https://www.nvidia.com/en-us/data-center/nvlink/)
- [NVIDIA 数据中心 GPU 技术概述](https://www.nvidia.com/en-us/data-center/gpu-technology/)

### 云原生相关
- [Kubernetes Controller Runtime](https://book.kubebuilder.io/controller-runtime.html)
- [CEL Language Guide](https://github.com/google/cel-spec)
- [Kubernetes 控制器模式](https://kubernetes.io/docs/concepts/architecture/controller/)
- [Pod Disruption Budgets](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)

### 数据库相关
- [MongoDB Change Streams](https://www.mongodb.com/docs/manual/changeStreams/)
- [PostgreSQL Logical Replication](https://www.postgresql.org/docs/current/logicaldecoding.html)
- [Go MongoDB Driver](https://www.mongodb.com/docs/drivers/go/)
- [PostgreSQL pq driver](https://github.com/lib/pq)

---

## 学习建议

### 通用建议
1. **不要只看不练**: 必须动手抄写代码
2. **边抄边思考**: 为什么这样设计？换种方式行不行？
3. **画图辅助**: 画架构图、时序图、状态机图
4. **写笔记**: 记录自己的理解和疑问
5. **提 Issue**: 发现问题或疑问可以提 GitHub Issue
6. **写测试**: 为你理解的功能写测试

### GPU 相关学习建议
1. **实际操作**:
   - 在有 GPU 的机器上运行 `nvidia-smi`、`dcgmi` 命令
   - 使用 `journalctl -k | grep -i xid` 查看实际 XID 错误
   - 使用 `nvidia-smi topo -m` 查看 GPU 拓扑

2. **深入理解**:
   - 学习 GPU 硬件架构 (SM、GPC、TPC、HBM)
   - 理解 PCIe 协议和 NVLink 互连
   - 了解 ECC 内存和单/双位错误

3. **实战练习**:
   - 尝试手动触发 DCGM 健康检查
   - 编写 Python 脚本调用 DCGM API
   - 解析不同类型的 XID 错误并理解其含义

### 代码抄写技巧
1. **第一遍**: 完整抄写，理解每行代码的作用
2. **第二遍**: 不看原版，尝试自己实现
3. **第三遍**: 对比自己的实现和原版，找出差异
4. **总结**: 记录学到的设计模式和技巧

---

**Happy Learning! 🚀**
