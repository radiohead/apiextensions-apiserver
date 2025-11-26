# Critical Paths Reference

**Quick reference for performance-critical code paths and files.**

## Critical Files Table

| File | Purpose | Lines | Importance | Notes |
|------|---------|-------|------------|-------|
| `pkg/apiserver/customresource_handler.go` | CR request handling | 1,746 | Critical - hot path | Atomic storage map for lock-free reads |
| `pkg/apiserver/schema/validation.go` | Schema validation | 15,303 | Critical - all CRs | Runs on every create/update |
| `pkg/apiserver/schema/cel/compilation.go` | CEL compilation | 361 | High - CEL caching | Compile once, evaluate many |
| `pkg/apiserver/schema/cel/validation.go` | CEL evaluation | 994 | High - validation | Cost-limited execution |
| `pkg/apiserver/apiserver.go` | Server initialization | 286 | High - startup | Entry point configuration |
| `pkg/apiserver/customresource_discovery_controller.go` | Discovery sync | 391 | High - kubectl | Updates API discovery |
| `pkg/controller/establish/establishing_controller.go` | Establishing CRDs | ~300 | Medium | Marks CRDs ready |
| `pkg/apis/apiextensions/v1/types.go` | CRD API types | ~1,500 | Medium | API definitions |

## Hot Paths (Performance Critical)

### 1. Custom Resource GET/LIST/WATCH

**Most common operations** - Optimized for read-heavy workload

```
HTTP Request → crdHandler.ServeHTTP()
    ↓
Parse GVR → atomic.Value.Load() (lock-free!)
    ↓
Lookup storage → Get from etcd
    ↓
Convert to requested version → Return response
```

**Optimization**: Lock-free reads via `atomic.Value` (~10x faster than RWMutex)

**Files involved**:
- `pkg/apiserver/customresource_handler.go` (main handler)
- `pkg/apiserver/conversion/converter.go` (version conversion)
- Storage layer (via `k8s.io/apiserver`)

### 2. Discovery Lookups

**kubectl queries on every command** - Heavily cached client-side

```
kubectl command → Discovery request
    ↓
versionDiscoveryHandler/groupDiscoveryHandler
    ↓
Return cached discovery document
```

**Client-side cache**: 10 minutes (kubectl default)

**Files involved**:
- `pkg/apiserver/customresource_discovery_controller.go`
- `pkg/apiserver/customresource_handler.go` (discovery handlers)

### 3. Informer Caching

**Controllers rely on fast local reads** - No API server round-trip

```
etcd event → Informer receives
    ↓
Update local cache → Trigger event handlers
    ↓
Controller reconciliation (using Lister)
```

**Files involved**:
- `pkg/client/informers/` (generated informers)
- `pkg/client/listers/` (generated listers)
- All controllers in `pkg/controller/`

### 4. CEL Validation

**Runs on every CR create/update** - Compilation cached

```
CR create/update → Schema validation
    ↓
CEL rule lookup (cached Program)
    ↓
Evaluate with cost tracking
    ↓
Accept or reject with error message
```

**Optimization**: Compile once at CRD registration, evaluate many times

**Files involved**:
- `pkg/apiserver/schema/cel/compilation.go` (compile & cache)
- `pkg/apiserver/schema/cel/validation.go` (evaluate)

## Optimization Strategies

### 1. Atomic Storage
- **Location**: `pkg/apiserver/customresource_handler.go`
- **Pattern**: `atomic.Value` for crdStorageMap
- **Benefit**: Lock-free reads for GET/LIST/WATCH operations
- **Trade-off**: Slightly higher write latency (rare)

### 2. Discovery Caching
- **Location**: Discovery handlers in `customresource_handler.go`
- **Pattern**: Pre-computed discovery documents updated atomically
- **Benefit**: No computation on discovery requests
- **Client-side**: kubectl caches for 10 minutes

### 3. CEL Compilation Caching
- **Location**: `pkg/apiserver/schema/cel/compilation.go`
- **Pattern**: Compile at CRD registration, cache Program
- **Benefit**: Avoid recompilation on every validation
- **Storage**: In-memory per CRD version

### 4. Storage Reuse
- **Location**: `pkg/apiserver/customresource_handler.go` (crdInfo)
- **Pattern**: Per-version storage instances cached
- **Benefit**: No recreation on each request
- **Lifecycle**: Recreated only on CRD spec changes

## Bottlenecks

### 1. Webhook Calls

**Impact**: +10-50ms latency per webhook

```
CR Operation → [Admission/Conversion] Webhook
    ↓
Network roundtrip + webhook processing
    ↓
Continue request processing
```

**Mitigation**: Use CEL validation instead of admission webhooks when possible

**Affected operations**: Create, Update, Patch (with webhooks configured)

### 2. Storage Recreation

**Impact**: Full rebuild on CRD spec changes

```
CRD Update → Informer event
    ↓
crdHandler detects spec change
    ↓
Destroy old storage → Create new storage
    ↓
Update atomic.Value with new crdStorageMap
```

**Duration**: Typically < 1 second per CRD

**Affected**: All CRDs with spec changes (rare in production)

### 3. Schema Validation

**Impact**: Deep nesting can be expensive

- **Depth**: Nested object validation is recursive
- **Cost**: O(n) where n = number of fields
- **CEL Cost**: Limited to 1M cost units per rule

**Mitigation**: Keep schemas reasonably flat, use CEL cost estimation

### 4. Watch Connection Count

**Impact**: Scales with (CR count × watchers)

- **Memory**: Each watch maintains buffer
- **Network**: Each event multiplied by watchers
- **etcd**: Watch pressure on storage

**Typical scale**: Thousands of concurrent watches supported

## LOC Metrics Per Component

### API Server Core (15,000 LOC)
- **Main handler**: 1,746 lines
- **Discovery controller**: 391 lines
- **Server initialization**: 286 lines
- **Conversion system**: ~2,000 lines
- **Storage layer interfaces**: ~1,000 lines
- **Supporting code**: ~9,000 lines

### Schema Validation (35,000 LOC)
- **Core validation**: 15,303 lines
- **CEL integration**: ~2,000 lines
- **Defaulting**: ~3,000 lines
- **Pruning**: ~2,000 lines
- **ObjectMeta validation**: ~1,500 lines
- **Supporting code**: ~11,000 lines

### Controllers (10,000 LOC)
- **Establishing**: ~300 lines
- **Naming**: ~400 lines
- **NonStructuralSchema**: ~300 lines
- **APIApproval**: ~400 lines
- **Finalizer**: ~500 lines
- **Discovery**: ~1,000 lines
- **OpenAPI (v2 + v3)**: ~2,000 lines
- **Supporting code**: ~5,000 lines

### Client Libraries (25,000 LOC - mostly generated)
- **Clientsets**: ~8,000 lines
- **Informers**: ~6,000 lines
- **Listers**: ~3,000 lines
- **DeepCopy**: ~2,000 lines
- **ApplyConfigurations**: ~6,000 lines

### API Types (8,000 LOC)
- **v1 types**: ~4,500 lines
- **v1beta1 types**: ~3,500 lines

### Testing (26,000 LOC)
- **Integration tests**: 15,582 lines
- **Unit tests**: ~10,000 lines
- **Fixtures and helpers**: ~4,000 lines

### Total: 119,053 LOC across 296 Go files

## Request Flow Performance

### CRD Creation Flow
**Total time**: ~1-2 seconds (includes controller reconciliation)

1. Validate CRD: ~10ms
2. Store in etcd: ~50-100ms
3. Informer sync: ~100-500ms
4. Controllers reconcile: ~500-1000ms
5. Storage creation: ~100-200ms
6. Discovery update: ~50-100ms

### Custom Resource Create Flow
**Total time**: < 100ms (without webhooks)

1. Parse request: ~1ms
2. Apply defaulting: ~1-5ms
3. Schema validation: ~5-20ms
4. CEL validation: ~1-10ms
5. Store in etcd: ~20-50ms
6. Convert & return: ~1-5ms

**With webhooks**: +10-50ms per webhook call

### Custom Resource Watch Flow
**Steady state latency**: < 100ms per event

1. etcd event: ~10-30ms
2. Convert version: ~1-5ms
3. Send to client: ~1-10ms (network)

## Related Documentation

- **[Architecture Overview](../architecture.md)** - Detailed request flow explanations
- **[Dependencies](./dependencies.md)** - External dependencies affecting performance
- **[CLAUDE.md](../../../CLAUDE.md)** - Critical components quick reference

**Last Updated**: 2025-11-25
**Source**: Architecture analysis lines 539-557 + CLAUDE.md critical file reference
