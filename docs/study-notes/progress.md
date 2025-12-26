# NVSentinel 学习进度追踪

> 开始日期: 2025-12-25
> 分支: `learn/study-codebase`

## 总体进度

| 阶段 | 主题 | 状态 | 完成日期 | 笔记链接 |
|------|------|------|----------|----------|
| 1 | 数据模型 - HealthEvent | ⚪ 未开始 | - | - |
| 2 | 数据库抽象层 - Store Client | ⚪ 未开始 | - | - |
| 3 | 健康监控器 - GPU Health Monitor | ⚪ 未开始 | - | - |
| 4 | 健康监控器 - Syslog Health Monitor | ⚪ 未开始 | - | - |
| 5 | Platform Connectors - gRPC 服务 | ⚪ 未开始 | - | - |
| 6 | 故障隔离 - Fault Quarantine | ⚪ 未开始 | - | - |
| 7 | 节点排空 - Node Drainer | ⚪ 未开始 | - | - |
| 8 | Kubernetes Controller 模式 | ⚪ 未开始 | - | - |
| 9 | 构建系统和部署 | ⚪ 未开始 | - | - |
| 10 | 测试策略 | ⚪ 未开始 | - | - |

---

## 详细进度

### 阶段 1: 数据模型 - HealthEvent

| 任务 | 状态 | 完成日期 |
|------|------|----------|
| 阅读 health_event.proto | ⚪ | - |
| 抄写 Protobuf 定义 | ⚪ | - |
| 运行 protos-generate | ⚪ | - |
| 理解 HealthEvent 字段 | ⚪ | - |
| 回答自检问题 | ⚪ | - |

---

### 阶段 2: 数据库抽象层 - Store Client

| 任务 | 状态 | 完成日期 |
|------|------|----------|
| 阅读核心接口 (interfaces.go) | ⚪ | - |
| 抄写 Query Builder | ⚪ | - |
| 对比 MongoDB 和 PostgreSQL 实现 | ⚪ | - |
| 理解 Change Stream 模式 | ⚪ | - |
| 回答自检问题 | ⚪ | - |

---

### 阶段 3: 健康监控器 - GPU Health Monitor

| 任务 | 状态 | 完成日期 |
|------|------|----------|
| 阅读 CLI 入口点 | ⚪ | - |
| 抄写 DCGM Watcher 核心逻辑 | ⚪ | - |
| 抄写 gRPC 客户端发送 | ⚪ | - |
| 理解 DCGM 回调机制 | ⚪ | - |
| 回答自检问题 | ⚪ | - |

---

### 阶段 4: 健康监控器 - Syslog Health Monitor

| 任务 | 状态 | 完成日期 |
|------|------|----------|
| 阅读 main.go 入口 | ⚪ | - |
| 抄写 Journal 接口 | ⚪ | - |
| 抄写 XID 处理器 | ⚪ | - |
| 理解 XID 错误码解析 | ⚪ | - |
| 回答自检问题 | ⚪ | - |

---

### 阶段 5: Platform Connectors - gRPC 服务

| 任务 | 状态 | 完成日期 |
|------|------|----------|
| 阅读 gRPC 服务实现 | ⚪ | - |
| 抄写事件验证逻辑 | ⚪ | - |
| 理解数据库持久化 | ⚪ | - |
| 回答自检问题 | ⚪ | - |

---

### 阶段 6: 故障隔离 - Fault Quarantine

| 任务 | 状态 | 完成日期 |
|------|------|----------|
| 阅读 Change Stream 监听 | ⚪ | - |
| 抄写 CEL 策略评估 | ⚪ | - |
| 抄写 Reconcile 逻辑 | ⚪ | - |
| 理解 Node Cordon 操作 | ⚪ | - |
| 回答自检问题 | ⚪ | - |

---

### 阶段 7: 节点排空 - Node Drainer

| 任务 | 状态 | 完成日期 |
|------|------|----------|
| 阅读工作队列实现 | ⚪ | - |
| 抄写 Pod 驱逐逻辑 | ⚪ | - |
| 理解 PDB 处理 | ⚪ | - |
| 回答自检问题 | ⚪ | - |

---

### 阶段 8: Kubernetes Controller 模式

| 任务 | 状态 | 完成日期 |
|------|------|----------|
| 阅读 main.go 初始化 | ⚪ | - |
| 理解 Informer 模式 | ⚪ | - |
| 理解 Workqueue | ⚪ | - |
| 理解 Reconcile 循环 | ⚪ | - |
| 回答自检问题 | ⚪ | - |

---

### 阶段 9: 构建系统和部署

| 任务 | 状态 | 完成日期 |
|------|------|----------|
| 阅读主 Makefile | ⚪ | - |
| 阅读 common.mk | ⚪ | - |
| 抄写模块 Makefile | ⚪ | - |
| 理解 Tilt 配置 | ⚪ | - |
| 回答自检问题 | ⚪ | - |

---

### 阶段 10: 测试策略

| 任务 | 状态 | 完成日期 |
|------|------|----------|
| 阅读单元测试示例 | ⚪ | - |
| 理解 Table-Driven Tests | ⚪ | - |
| 理解 envtest | ⚪ | - |
| 回答自检问题 | ⚪ | - |

---

## 笔记索引

- [阶段 1 笔记](stage-1-protobuf.md)
- [阶段 2 笔记](stage-2-store-client.md)
- [阶段 3 笔记](stage-3-gpu-monitor.md)
- [阶段 4 笔记](stage-4-syslog-monitor.md)
- [阶段 5 笔记](stage-5-platform-connectors.md)
- [阶段 6 笔记](stage-6-fault-quarantine.md)
- [阶段 7 笔记](stage-7-node-drainer.md)
- [阶段 8 笔记](stage-8-controller-pattern.md)
- [阶段 9 笔记](stage-9-build-system.md)
- [阶段 10 笔记](stage-10-testing.md)

---

## 实验代码索引

- [阶段 1 实验](../experiments/stage-1-data-models/)
- [阶段 2 实验](../experiments/stage-2-store-client/)
- [阶段 3 实验](../experiments/stage-3-gpu-monitor/)
- [阶段 4 实验](../experiments/stage-4-syslog-monitor/)
- [阶段 5 实验](../experiments/stage-5-platform-connectors/)
- [阶段 6 实验](../experiments/stage-6-fault-quarantine/)
- [阶段 7 实验](../experiments/stage-7-node-drainer/)
- [阶段 8 实验](../experiments/stage-8-controller-pattern/)
- [阶段 9 实验](../experiments/stage-9-build-system/)
- [阶段 10 实验](../experiments/stage-10-testing/)

---

## 学习统计

- 已完成阶段: 0 / 10
- 已完成任务: 0 / 50+
- 抄写代码行数: 0
- 学习笔记字数: 0

---

## 下一步行动

- [ ] 开始阶段 1: 阅读 `data-models/protobufs/health_event.proto`
