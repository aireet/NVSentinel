# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**NVSentinel** is a GPU Node Resilience System for Kubernetes that automatically detects, classifies, and remediates hardware and software faults in GPU nodes. It's a microservices-based system with event-driven architecture.

**Status:** Experimental/Preview Release (v0.5.0)

**Tech Stack:** Go 1.25+, Python 3.10, Kubernetes 1.25+, Helm 3.0+, MongoDB/PostgreSQL, gRPC, NVIDIA DCGM

---

## Common Development Commands

### Environment Setup
```bash
make dev-env-setup              # Install all development tools
make show-versions              # View all tool versions from .versions.yaml
```

### Building and Testing
```bash
# Run all linting and tests (CI-equivalent)
make lint-test-all

# Test specific module
make -C platform-connectors lint-test
make -C health-monitors/gpu-health-monitor lint-test
make -C fault-quarantine lint-test

# Individual module targets (via common.mk)
cd platform-connectors
make vet        # go vet
make lint       # golangci-lint
make test       # gotestsum with coverage
make build      # go build
make binary     # build main binary
```

### Docker / Containers
```bash
# Build all container images locally
make docker-all

# Build specific module locally (fast)
make -C health-monitors/syslog-health-monitor docker-build-local

# Build and publish to registry
make docker-publish-all

# Python GPU monitor (DCGM variants)
make -C health-monitors/gpu-health-monitor docker-build-dcgm3
make -C health-monitors/gpu-health-monitor docker-build-dcgm4
```

### Protocol Buffers
```bash
make protos-generate    # Regenerate all protobuf files
make protos-lint        # Check protobuf files are up to date
```

### Local Development with Tilt
```bash
make dev-env            # Create Kind cluster + start Tilt (recommended)
make tilt-up            # Start Tilt only
make cluster-create     # Create Kind cluster manually
make cluster-status     # Check cluster and registry status
make dev-env-clean      # Stop Tilt and delete cluster
```

### License Headers
```bash
make license-headers-add     # Add Apache 2.0 headers
make license-headers-lint    # Check license headers
```

---

## High-Level Architecture

### Data Flow

```
Health Monitors (detectors)
    ↓ gRPC
Platform Connectors (ingestion)
    ↓ persist
MongoDB/PostgreSQL (event store)
    ↓ change streams
Core Modules (responders)
    ↓ Kubernetes API
Kubernetes Cluster (actuator)
```

### Key Architectural Principles

1. **Event-Driven** - MongoDB/PostgreSQL change streams for reactive processing
2. **Decoupled** - Modules operate independently without direct communication
3. **Kubernetes-Native** - All actions via Kubernetes API
4. **Modular** - Pluggable health monitors with standardized interfaces

### Core Data Structure: HealthEvent

All communication uses the `HealthEvent` protobuf defined in `data-models/protobufs/`:

```protobuf
message HealthEvent {
  uint32 version = 1;
  string agent = 2;              // Monitor name
  string componentClass = 3;     // GPU, NIC, etc.
  string nodeName = 13;          // Kubernetes node
  string checkName = 4;          // Specific check
  bool isFatal = 5;             // Critical failure
  bool isHealthy = 6;           // Current state
  string message = 7;           // Human-readable
  RecommendedAction recommendedAction = 8;
  repeated string errorCode = 9;
  map<string, string> metadata = 11;
}
```

---

## Major Components

### 1. Health Monitors (`health-monitors/`)

**Purpose:** Detect hardware/software faults and send events via gRPC

| Module | Language | Purpose |
|--------|----------|---------|
| `gpu-health-monitor` | Python | NVIDIA DCGM monitoring (temp, ECC, XID) |
| `syslog-health-monitor` | Go | journalctl analysis (XID, kernel panics) |
| `csp-health-monitor` | Go | Cloud provider maintenance events |
| `kubernetes-object-monitor` | Go | CEL-based policy monitoring |

### 2. Platform Connectors (`platform-connectors/`)

**Purpose:** gRPC server that receives events, persists to database, updates K8s node status

- Implements `HealthEventOccurredV1` gRPC endpoint
- Validates events and inserts into MongoDB/PostgreSQL
- Updates Kubernetes Node conditions for fatal failures

### 3. Core Response Modules

| Module | Location | Purpose |
|--------|----------|---------|
| Fault Quarantine | `fault-quarantine/` | Cordons nodes based on CEL policies |
| Node Drainer | `node-drainer/` | Evicts workloads from cordoned nodes |
| Fault Remediation | `fault-remediation/` | Creates remediation CRDs |
| Health Events Analyzer | `health-events-analyzer/` | Pattern analysis and correlation |

### 4. Supporting Services

| Module | Location | Purpose |
|--------|----------|---------|
| Labeler | `labeler/` | Auto-label nodes with DCGM/driver versions |
| Metadata Collector | `metadata-collector/` | GPU/NVSwitch topology |
| Event Exporter | `event-exporter/` | Stream events to external systems |
| Janitor | `janitor/` | Execute node reboots/terminations |
| Log Collector | `log-collector/` | Diagnostic log aggregation |

### 5. Shared Libraries

| Module | Purpose |
|--------|---------|
| `commons/` | Shared Go utilities (k8sutils, logging, config) |
| `store-client/` | Database abstraction (MongoDB/PostgreSQL) |
| `data-models/` | Protocol Buffer definitions and Go structs |

---

## Important Code Patterns

### Critical: Error Handling in Retry Loops

```go
// ✅ CORRECT - Return error as-is for retry logic
err := retry.RetryOnConflict(retry.DefaultBackoff, func() error {
    _, err := client.Update(ctx, obj, metav1.UpdateOptions{})
    return err  // Don't wrap!
})

// ❌ WRONG - Wrapping breaks retry detection
err := retry.RetryOnConflict(retry.DefaultBackoff, func() error {
    _, err := client.Update(ctx, obj, metav1.UpdateOptions{})
    return fmt.Errorf("failed: %w", err)  // Breaks retry!
})
```

### Database Operations (Preferred API)

```go
// Modern database-agnostic API (recommended)
ds := datastore.NewDataStore(ctx, config)
healthStore := ds.HealthEventStore()
q := query.New().Build(query.Eq("nodename", "node-1"))
events, _ := healthStore.FindHealthEventsByQuery(ctx, q)
```

### Kata Containers Detection

- **Input label:** `katacontainers.io/kata-runtime` on nodes
- **Output label:** `nvsentinel.dgxc.nvidia.com/kata.enabled: "true"|"false"`
- **Truthy values:** `"true"`, `"enabled"`, `"1"`, `"yes"` (case-insensitive)
- Separate DaemonSets for kata vs regular nodes (different log mounts)

### Module Organization

- Each service is a separate Go module with its own `go.mod`
- All Go modules include `make/common.mk` for consistent targets
- Use `replace` directives for local module dependencies
- Use `commons/` for shared utilities

---

## Testing

```bash
# Unit tests (table-driven, use testify)
cd platform-connectors
make test

# Kubernetes controller tests (use envtest, not fake clients)
go test ./pkg/controllers/... -envtest

# Python tests (GPU monitor)
cd health-monitors/gpu-health-monitor
poetry run pytest -v
```

**Requirements:**
- Use `testify/assert` and `testify/require`
- Use `envtest` for Kubernetes controller tests
- Coverage goal: >80% on critical paths
- Test naming: `TestFunctionName_Scenario_ExpectedBehavior`

---

## Configuration

### Version Management
All tool versions centralized in `.versions.yaml` - single source of truth.

### Helm Charts
- **Main chart:** `distros/kubernetes/nvsentinel/`
- **Values files:** `values.yaml`, `values-full.yaml`, `values-tilt.yaml`
- **Component charts:** `charts/` subdirectory

### Module Enablement
```yaml
global:
  dryRun: false
  gpuHealthMonitor:
    enabled: true
  faultQuarantine:
    enabled: false  # Disabled by default - enable for production
  nodeDrainer:
    enabled: false
```

---

## File Structure Notes

```
NVSentinel/
├── api/                      # API definitions and CRDs
├── commons/                  # Shared Go utilities
├── data-models/              # Protocol buffers and shared data structures
├── distros/kubernetes/       # Helm charts for deployment
├── docker/                   # Docker build orchestration
├── health-monitors/          # Detection modules
├── platform-connectors/      # gRPC ingestion service
├── fault-quarantine/         # Node cordon logic
├── fault-remediation/        # Break-fix automation
├── health-events-analyzer/   # Pattern analysis
├── labeler/                  # Node labeling
├── log-collector/            # Log aggregation
├── make/                     # Build system templates (common.mk)
├── metadata-collector/       # GPU topology
├── node-drainer/             # Workload eviction
├── store-client/             # Database abstraction
├── tests/                    # E2E and integration tests
└── tilt/                     # Local dev environment (Tiltfile)
```

---

## Key References

- `README.md` - Project overview and quick start
- `DEVELOPMENT.md` - Comprehensive development guide
- `.github/copilot-instructions.md` - AI assistant coding guidelines
- `make/common.mk` - Shared build variables and patterns
- `.versions.yaml` - Tool version configuration
