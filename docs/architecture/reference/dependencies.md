# Dependencies Reference

**Quick reference for external dependencies and integration points.**

## Core Dependencies

### Kubernetes Ecosystem

| Dependency | Purpose | Type |
|------------|---------|------|
| `k8s.io/apiserver` | Generic API server framework | Framework |
| `k8s.io/apimachinery` | Schema and runtime machinery | Library |
| `k8s.io/client-go` | Client library foundations | Library |
| `k8s.io/code-generator` | Code generation tools | Tool |
| `k8s.io/kube-openapi` | OpenAPI spec generation | Library |

### Third-Party Dependencies

| Dependency | Purpose | Type |
|------------|---------|------|
| `github.com/google/cel-go` | CEL expression engine | Library |

## Integration Points

### External Systems

| System | Integration Method | Purpose | Latency Impact |
|--------|-------------------|---------|----------------|
| etcd | Direct storage backend | Persistence layer | N/A (required) |
| kube-apiserver | Delegation target | Fallback for unknown endpoints | N/A (chained) |
| kubectl | HTTP REST client | Primary CLI interface | None (client-side) |
| admission-webhooks | HTTPS callbacks | External validation/mutation | +10-50ms |
| conversion-webhooks | HTTPS callbacks | Version conversion | +10-50ms |

### Internal Integration

```
┌──────────────────────────────────────────────────────────────┐
│         API Extensions API Server (this component)           │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  CRD API Endpoints ←→ Custom Resource Handler                │
│           ↓                     ↓                             │
│      Controllers         Validation Pipeline                 │
│           ↓                     ↓                             │
│  Discovery/OpenAPI     Storage Layer (etcd)                  │
│                                                               │
└──────────────────────────────────────────────────────────────┘
           ↑                      ↑                 ↑
           │                      │                 │
    ┌──────┴──────┐    ┌─────────┴─────────┐   ┌──┴──────┐
    │   kubectl   │    │ admission-webhooks │   │  etcd   │
    └─────────────┘    └────────────────────┘   └─────────┘
           ↑
           │
    ┌──────┴──────────┐
    │ kube-apiserver  │
    │  (delegator)    │
    └─────────────────┘
```

## Dependency Graph

### Request Flow Dependencies

```
Client Request
    ↓
kube-apiserver (delegation)
    ↓
apiextensions-apiserver
    ↓
├─→ CRD Informer (watches etcd)
├─→ Schema Validator (uses CEL)
├─→ Admission Webhooks (external)
├─→ Conversion Webhooks (external)
└─→ etcd Storage (persistence)
```

### Controller Dependencies

```
CRD Event (from etcd via informer)
    ↓
├─→ Establishing Controller → Updates status
├─→ Naming Controller → Validates names
├─→ NonStructuralSchema Controller → Checks schema
├─→ APIApproval Controller → Enforces policy
├─→ Finalizer Controller → Manages cleanup
├─→ Discovery Controller → Updates discovery docs
└─→ OpenAPI Controllers (v2/v3) → Updates specs
```

## Version Constraints

### Kubernetes Compatibility

This component is version-locked with the Kubernetes release it's part of. See the main Kubernetes repository for version compatibility matrices.

**Staging Repository**: This is a read-only mirror from `k8s.io/kubernetes/staging/src/k8s.io/apiextensions-apiserver`

### API Versions

| API Group | Versions | Status |
|-----------|----------|--------|
| `apiextensions.k8s.io` | v1 | Stable |
| `apiextensions.k8s.io` | v1beta1 | Deprecated |

### Webhook Protocols

| Protocol | Version | Status |
|----------|---------|--------|
| ConversionReview | v1 | Stable |
| ConversionReview | v1beta1 | Deprecated |

## External Service Requirements

### Admission Webhooks (Optional)

**Required for**: Custom validation/mutation logic beyond CEL

- **Protocol**: HTTPS with TLS
- **Authentication**: Service account tokens
- **Timeout**: Configurable (default: 10s)
- **CA Bundle**: Required for validation
- **Impact**: Blocks request if webhook unavailable

### Conversion Webhooks (Optional)

**Required for**: Multi-version CRDs with different schemas

- **Protocol**: HTTPS with TLS
- **Authentication**: Service account tokens
- **Timeout**: Configurable (default: 30s)
- **CA Bundle**: Required for validation
- **Impact**: Blocks all CR operations if webhook unavailable

## Storage Requirements

### etcd

**Required**: Yes (mandatory)

- **Purpose**: Persistence layer for CRDs and custom resources
- **Access**: Via `k8s.io/apiserver` storage interface
- **Encryption**: Supports encryption-at-rest (if configured)
- **Communication**: TLS required in production

### Memory

| Component | Memory Usage | Notes |
|-----------|--------------|-------|
| Per CRD | ~1-10 MB | Storage + caches |
| Per Version | ~100 KB - 1 MB | Per CRD version |
| CEL Programs | ~10-100 KB | Cached per version |
| Informer Caches | Scales with CR count | Proportional to resource count |

## Related Documentation

- **[Architecture Overview](../architecture.md)** - Full architecture with dependency explanations
- **[Critical Paths](./critical-paths.md)** - Performance-critical dependency paths
- **[CLAUDE.md](../../../CLAUDE.md)** - Development guide with dependency notes

**Last Updated**: 2025-11-25
**Source**: Architecture analysis lines 710-726
