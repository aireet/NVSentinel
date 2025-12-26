# Syslog Health Monitor - Runtime Journal Support

本文档总结了 syslog-health-monitor 的工作原理，以及添加运行时 journal 支持的设计考虑。

## 目录

1. [系统架构](#系统架构)
2. [XID 错误检测机制](#xid-错误检测机制)
3. [Journal 存储机制](#journal-存储机制)
4. [Kata vs Regular 模式](#kata-vs-regular-模式)
5. [运行时 Journal 支持方案](#运行时-journal-支持方案)
6. [PR 策略](#pr-策略)

---

## 系统架构

### 整体数据流

```
┌─────────────────────────────────────────────────────────────────┐
│                     Health Monitors (检测器)                    │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  syslog-health-monitor (读取 journal)                       ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
                            │ gRPC
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Platform Connector (接入层)                  │
│  - 接收 HealthEvent                                             │
│  - 持久化到 MongoDB/PostgreSQL                                   │
│  - 更新 Kubernetes Node 状态                                    │
└─────────────────────────────────────────────────────────────────┘
                            │ Change Streams
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Core Modules (响应器)                        │
│  - fault-quarantine: 隔离不健康节点                             │
│  - node-drainer: 排水工作负载                                   │
│  - fault-remediation: 执行修复动作                              │
└─────────────────────────────────────────────────────────────────┘
```

### Syslog Health Monitor 组件

```
syslog-health-monitor/
├── main.go                    # 入口，检查配置和启动
├── pkg/syslog-monitor/        # Journal 读取逻辑
│   ├── syslogmonitor.go      # 核心监控逻辑
│   ├── journal_real.go       # 真实 journal (systemd)
│   ├── journal_stub.go       # 测试用 stub
│   └── journal_iface.go      # Journal 接口定义
├── pkg/xid/                   # XID 错误处理
│   ├── xid_handler.go        # XID 处理器
│   └── parser/               # XID 解析器
│       ├── csv.go            # 本地 CSV 解析
│       └── sidecar.go        # 外部服务解析
├── pkg/sxid/                  # SXID 错误处理
├── pkg/gpufallen/             # GPU 掉落检测
└── pkg/patterns/              # 正则表达式模式
    └── xid.go                # XID 匹配模式
```

---

## XID 错误检测机制

### XID 是什么

**XID** = NVIDIA GPU Error ID，用于标识 GPU 硬件/软件错误。

示例：`NVRM: Xid (PCI:0000:b3:00.0): 79, pid=1234, name=process`

### 检测流程

```go
// pkg/patterns/xid.go
var XIDPattern = regexp.MustCompile(
    `NVRM: Xid \(PCI:([0-9a-fA-F:.]+)\): (\d+)(?:, pid=(\d+))?(?:, name=([^,]+))?(?:, Ch ([0-9a-fA-F]+))?`
)
```

```
输入日志: "NVRM: Xid (PCI:0000:b3:00.0): 79, pid=1234, name=cuda"
                                            ↓
                        正则匹配 (XIDPattern)
                                            ↓
                    ┌───────────────────────┐
                    │  提取:                │
                    │  - PCI 地址           │
                    │  - XID 错误码 (79)    │
                    │  - 进程信息 (可选)    │
                    └───────────────────────┘
                                            ↓
                        ┌───────────────────────┐
                        │  查询错误处理建议      │
                        │  (CSV 或 Sidecar)     │
                        └───────────────────────┘
                                            ↓
                    ┌───────────────────────┐
                    │  生成 HealthEvent     │
                    │  - isFatal           │
                    │  - recommendedAction │
                    │  - entitiesImpacted  │
                    └───────────────────────┘
                                            ↓
                        发送到 Platform Connector
```

### 特殊 XID 处理

| XID | 特殊处理 | 额外信息 |
|-----|---------|---------|
| **XID 13** | 引擎错误 | GPC, TPC, SM 单元位置 |
| **XID 74** | NVLink 错误 | NVLink 链路号 + 7 个寄存器值 |
| **XID 154** | GPU 恢复动作 | 从日志中解析具体动作 |

```go
// csv.go:51-52
reXid13Pattern = regexp.MustCompile(`\(GPC\s+(\d+),\s*TPC\s+(\d+),\s*SM\s+(\d+)\)`)
reXid74NVLinkPattern = regexp.MustCompile(`link\s+(\d+)\((0x[0-9a-fA-F]+),...`)
```

### 健康事件结构

```protobuf
message HealthEvent {
  uint32 version = 1;
  string agent = 2;              // "syslog-health-monitor"
  string componentClass = 3;     // "GPU"
  string nodeName = 13;          // Kubernetes 节点名
  string checkName = 4;          // "SysLogsXIDError"
  bool isFatal = 5;             // 是否致命错误
  bool isHealthy = 6;           // 当前健康状态
  string message = 7;           // 原始日志消息
  RecommendedAction recommendedAction = 8;  // 建议动作
  repeated string errorCode = 9; // ["79"]
  repeated Entity entitiesImpacted = 10;   // GPU_PCI, GPU_UUID
  map<string, string> metadata = 11;
}
```

---

## Journal 存储机制

### Systemd Journal 两种模式

| | 运行时 Journal | 持久化 Journal |
|---|---------------|---------------|
| **路径** | `/run/log/journal/` | `/var/log/journal/` |
| **存储** | tmpfs (内存) | 磁盘 |
| **重启后** | ❌ 清空 | ✅ 保留 |
| **默认** | ✅ 是 | ❌ 需手动创建目录 |
| **大小限制** | 默认 4G | 可配置 |

### 配置文件

`/etc/systemd/journald.conf`:

```ini
[Journal]
Storage=persistent    # auto|persistent|volatile
SystemMaxUse=8G       # 持久化时最大使用空间
MaxFileSize=512M      # 单个文件最大
MaxRetentionSec=90day # 保留时间
```

### Journal 目录结构

```
/var/log/journal/
└── 4f57b2b0da6e4a75ba48fcc2272f484a/  ← Machine ID
    ├── system.journal                     ← 当前活跃日志
    ├── system@xxxxxx.journal              ← 已归档日志
    └── ...

/run/log/journal/
└── 4f57b2b0da6e4a75ba48fcc2272f484a/  ← Machine ID
    └── system.journal                     ← 仅当前启动
```

### 启用持久化 Journal

```bash
# 1. 创建目录
MACHINE_ID=$(cat /etc/machine-id)
sudo mkdir -p /var/log/journal/$MACHINE_ID

# 2. 设置权限
sudo chown root:systemd-journal /var/log/journal/$MACHINE_ID
sudo chmod 2750 /var/log/journal/$MACHINE_ID

# 3. 设置 ACL
sudo setfacl -R -m g:systemd-journal:rx /var/log/journal/$MACHINE_ID
sudo setfacl -R -dm g:systemd-journal:rx /var/log/journal/$MACHINE_ID

# 4. 配置持久化
sudo sed -i 's/#Storage=auto/Storage=persistent/' /etc/systemd/journald.conf

# 5. 重启 journald
sudo systemctl restart systemd-journald
```

### Journal Cursor 机制

```
Cursor 格式: s=文件偏移;i=内部位置;b=bootID;m=时间戳
示例: s=12f34;i=2345;b=abc123def456;m=56789
```

| Journal 类型 | 重启后 Cursor 行为 |
|-------------|-------------------|
| **持久化** | 文件仍在，cursor 可能失效，从头读取可能重复 |
| **运行时** | 新空文件，cursor 重置，从头读取（无旧数据） |

---

## Kata vs Regular 模式

### 为什么需要两个版本

**Kata Containers** 是一种使用轻量级 VM 运行容器的技术：

```
Regular 节点:
┌─────────────────────────────────────────────────────────────┐
│  宿主机内核                                                 │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  容器 (runc) - 共享内核                               │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  日志位置: /var/log (文件系统)                             │
└─────────────────────────────────────────────────────────────┘

Kata 节点:
┌─────────────────────────────────────────────────────────────┐
│  宿主机内核                                                 │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Kata VM (独立内核)                                   │  │
│  │  ┌─────────────────────────────────────────────────┐ │  │
│  │  │  容器                                          │ │  │
│  │  └─────────────────────────────────────────────────┘ │  │
│  │                                                      │  │
│  │  VM 内的 journal → 无法直接从宿主机访问              │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  宿主机 systemd journal (/run/log/journal)                 │
│    ↑ 记录 Kata VM 相关事件                                 │
└─────────────────────────────────────────────────────────────┘
```

### 两个 DaemonSet 的区别

#### Regular DaemonSet

```yaml
# 挂载配置
volumeMounts:
  - name: var-log-vol
    mountPath: /nvsentinel/var/log    # 映射 /var/log

volumes:
  - name: var-log-vol
    hostPath:
      path: /var/log

# 命令行参数
args:
  - "--kata-enabled"
  - "false"

# 节点选择器
nodeSelector:
  nvsentinel.dgxc.nvidia.com/kata.enabled: "false"
```

#### Kata DaemonSet

```yaml
# 挂载配置
volumeMounts:
  - name: host-journal
    mountPath: /nvsentinel/var/log/journal
  - name: host-systemd
    mountPath: /run/systemd/journal
  - name: host-machine-id
    mountPath: /etc/machine-id

volumes:
  - name: host-journal
    hostPath:
      path: /var/log/journal
  - name: host-systemd
    hostPath:
      path: /run/systemd/journal

# 命令行参数
args:
  - "--kata-enabled"
  - "true"

# 节点选择器
nodeSelector:
  nvsentinel.dgxc.nvidia.com/kata.enabled: "true"
```

### Labeler 自动检测

```go
// Labeler 监听节点变化
for _, node := range nodes {
    // 检测 kata 运行时
    if isKataRuntime(node) {
        node.Labels["nvsentinel.dgxc.nvidia.com/kata.enabled"] = "true"
    } else {
        node.Labels["nvsentinel.dgxc.nvidia.com/kata.enabled"] = "false"
    }
}

// DaemonSet 根据 label 自动调度
```

---

## 运行时 Journal 支持方案

### 问题分析

**当前代码** (main.go:125-129):

```go
checks = append(checks, fd.CheckDefinition{
    Name:        c,
    JournalPath: "/nvsentinel/var/log/journal/",  // ← 硬编码
})
```

这导致：
- Regular 节点只挂载 `/var/log` → `/nvsentinel/var/log`
- 代码尝试访问 `/nvsentinel/var/log/journal/`
- 如果 `/var/log/journal/` 不存在 → ❌ 错误

### 解决方案

#### 1. 修改挂载配置

`templates/_helpers.tpl`:

```yaml
# Regular mode volumeMounts
{{- else }}
- name: var-log-vol
  mountPath: /nvsentinel/var/log
  readOnly: true
- name: run-log-journal-vol        # 新增
  mountPath: /nvsentinel/run/log/journal
  readOnly: true
{{- end }}

# 对应的 volumes
{{- else }}
- name: var-log-vol
  hostPath:
    path: /var/log
    type: Directory
- name: run-log-journal-vol        # 新增
  hostPath:
    path: /run/log/journal
    type: Directory
{{- end }}
```

#### 2. 修改代码逻辑

`main.go`:

```go
// 新增辅助函数
func getJournalPath(kataEnabled string) string {
    if stringutil.IsTruthyValue(kataEnabled) {
        return "/nvsentinel/var/log/journal/"
    }

    // regular 模式：优先持久化，不存在则用运行时
    if _, err := os.Stat("/nvsentinel/var/log/journal/"); err == nil {
        return "/nvsentinel/var/log/journal/"
    }
    return "/nvsentinel/run/log/journal/"
}

// 在创建 checks 时使用
func run() error {
    // ...
    journalPath := getJournalPath(*kataEnabled)
    for c := range strings.SplitSeq((*checksList), ",") {
        checks = append(checks, fd.CheckDefinition{
            Name:        c,
            JournalPath: journalPath,
        })
    }
    // ...
}
```

### 运行时行为

```
启动时检测:
┌─────────────────────────────────────────────────────────────┐
│  getJournalPath()                                            │
│  ┌─────────────────────────────────────────────────────────┐│
│  │  kata模式?                                              ││
│  │    ├─ Yes → /nvsentinel/var/log/journal/               ││
│  │    └─ No → 检查持久化 journal                           ││
│  │           ├─ 存在 → /nvsentinel/var/log/journal/         ││
│  │           └─ 不存在 → /nvsentinel/run/log/journal/ ✅   ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

---

## PR 策略

### 标题

```
[Compatibility] Support runtime journal for real-time monitoring scenarios
```

### Motivation

NVSentinel 的核心目标是**实时 GPU 健康监控和自愈**：

```
实时监控 → 检测错误 → 标记节点 → 自动排水 → 恢复
```

对于这个核心功能：
- ✅ 实时检测 XID 错误
- ✅ 触发隔离/排水工作流
- ✅ 启用自愈机制

**历史日志** 不是必需的。运行时 journal 完全满足实时监控需求。

### Use Cases

此改动使 NVSentinel 能部署在以下环境：

- ✅ 开发/测试环境
- ✅ 未配置持久化 journal 的系统
- ✅ 资源受限的嵌入式系统
- ✅ 临时/短暂工作负载
- ✅ 不需要历史诊断的场景

### Changes

1. **Helm Chart**: 添加 `/run/log/journal` 挂载 (regular 模式)
2. **代码**: 自动检测并选择可用的 journal 路径

### Behavior

| 场景 | 行为 |
|-----|------|
| 持久化 journal 存在 | 使用 `/var/log/journal` (默认) |
| 持久化 journal 不存在 | 使用 `/run/log/journal` (自动回退) |
| Kata 模式 | 使用 `/var/log/journal` (不变) |

### Limitations

运行时 journal 模式下的限制：

- ⚠️ 重启后历史日志丢失（设计如此）
- ⚠️ 无法跨重启进行事后分析
- ✅ 不影响实时监控功能
- ✅ 不影响自愈工作流

### Risk Assessment

- ✅ 无破坏性变更 - 现有用户不受影响
- ✅ 向后兼容 - 持久化 journal 仍是首选
- ✅ 仅在持久化不可用时回退
- ✅ 不改变任何现有行为

### Testing

```go
func TestJournalPathSelection(t *testing.T) {
    tests := []struct {
        name           string
        kataEnabled    string
        persistentExists bool
        expected       string
    }{
        {"kata mode", "true", false, "/nvsentinel/var/log/journal/"},
        {"persistent exists", "false", true, "/nvsentinel/var/log/journal/"},
        {"fallback to runtime", "false", false, "/nvsentinel/run/log/journal/"},
    }
    // ...
}
```

### Documentation

更新用户文档：

```markdown
## Journal Storage

syslog-health-monitor requires systemd journal for GPU error detection.

Two storage modes are supported:
- **Persistent** (default): `/var/log/journal/` - Survives reboots
- **Runtime**: `/run/log/journal/` - Current boot only

The monitor automatically detects and uses the available mode.

For production, enable persistent journal:
```bash
sudo mkdir -p /var/log/journal/$(cat /etc/machine-id)
sudo systemctl restart systemd-journald
```
```

---

## 反驳潜在反对意见

| 反对意见 | 回应 |
|---------|------|
| "历史诊断很重要" | NVSentinel 的重点是**实时监控和自愈**，不是历史分析。历史日志可通过其他方式收集 |
| "重启后状态丢失" | 重启后本应重新评估节点健康状态，而不是依赖历史 |
| "非标准配置" | systemd 官方支持运行时 journal，这是标准配置之一 |
| "增加复杂度" | 逻辑简单，只有一个 `os.Stat()` 检查，低维护成本 |

---

## 核心要点总结

1. **NVSentinel 的核心价值**：实时监控 + 自动自愈，不是历史分析
2. **运行时 journal 完全够用**：满足核心功能，无需持久化
3. **自动检测降低门槛**：无需手动配置，开箱即用
4. **向后兼容**：不改变现有行为，零风险
5. **扩大适用场景**：支持更多部署环境

---

## 参考资料

- [systemd.journald.conf(5)](https://www.freedesktop.org/software/systemd/man/journald.conf.html)
- [NVSentinel Architecture](../README.md)
- [Kata Containers Integration](../CLAUDE.md#kata-containers-detection)
