# Performance Guide

This guide covers performance characteristics, scalability limits, optimization strategies, and bottleneck analysis for the Kubernetes API Extensions API Server.

## Overview

The apiextensions-apiserver is optimized for high-throughput custom resource operations with careful attention to hot paths, caching, and lock-free data structures. Understanding these characteristics is critical for capacity planning and performance tuning.

## Scalability Limits

### CRD Count

**Tested Limits**: 100s of CRDs

**Characteristics**:
- Memory per CRD: ~1-10 MB (storage instances + caches)
- Discovery document size grows linearly with CRD count
- kubectl startup time increases with more CRDs
- OpenAPI spec size grows with CRD count

**Degradation Patterns**:
- Discovery requests slower beyond 200 CRDs
- API server memory usage scales linearly
- Client-side discovery parsing takes longer
- OpenAPI spec generation slows down

**Recommendations**:
```bash
# Monitor CRD count
kubectl get crd --no-headers | wc -l

# Check discovery document size
kubectl get --raw /apis | jq '. | length'

# Monitor API server memory
kubectl top pod -n kube-system <apiserver-pod>
```

### Custom Resource Count

**Tested Limits**: Millions per CRD

**Characteristics**:
- etcd-limited (etcd typically handles millions of objects)
- Watch connection memory scales with CR count
- List operations paginated automatically
- Informer memory scales with total CR count

**Degradation Patterns**:
- List operations slower with large result sets
- Watch initialization takes longer
- etcd disk I/O increases
- Network bandwidth for list/watch increases

**Recommendations**:
```bash
# Use pagination for large lists
kubectl get myresources --limit 100 --continue <token>

# Use field selectors to reduce results
kubectl get myresources --field-selector metadata.name=specific

# Monitor etcd size
ETCDCTL_API=3 etcdctl endpoint status --write-out=table
```

### Watch Connections

**Tested Limits**: Thousands of concurrent watches

**Characteristics**:
- Memory per watch: ~10-100 KB (buffer + metadata)
- Watch events delivered to all watchers
- Connection pooling helps with large clusters
- HTTP/2 multiplexing reduces overhead

**Degradation Patterns**:
- Memory usage increases linearly with watch count
- Event fanout latency increases
- Network bandwidth for event distribution
- File descriptor limits can be reached

**Recommendations**:
```bash
# Monitor watch connections
ss -tan state established '( dport = :6443 or sport = :6443 )' | wc -l

# Check file descriptor usage
lsof -p <apiserver-pid> | wc -l

# Monitor watch event latency
# Use metrics: apiserver_watch_events_total, apiserver_watch_events_sizes
```

### Version Count

**Typical Range**: 10-20 versions per CRD

**Characteristics**:
- Memory per version: ~100 KB - 1 MB
- Separate storage instance per version
- Discovery includes all served versions
- Conversion overhead increases with version count

**Degradation Patterns**:
- More memory per CRD
- Increased discovery document size
- More conversion operations
- More complex version routing logic

**Recommendations**:
```yaml
# Deprecate old versions
versions:
- name: v1beta1
  served: true  # Still accessible
  storage: false
  deprecated: true
  deprecationWarning: "v1beta1 is deprecated, use v1"
- name: v1
  served: true
  storage: true
```

## Memory Usage

### Per CRD Memory Breakdown

**Storage Instances**: ~500 KB - 5 MB
- REST storage per version
- Request scopes (main, status, scale)
- Codec factories
- Admission handlers

**CEL Programs**: ~10-100 KB per version
- Compiled CEL expressions cached
- Validation rule programs
- Message expression programs
- Cost estimation data

**Discovery Data**: ~10-50 KB
- API group metadata
- Version metadata
- Resource metadata
- OpenAPI schema

**Total per CRD**: ~1-10 MB (varies with schema complexity)

### Memory Growth Patterns

**Linear Growth**:
- Total CRD count
- Total CR count in informer caches
- Watch connection count

**Constant per CRD**:
- Storage instances (one per version)
- CEL program cache
- Discovery metadata

**Example Calculation**:
```
100 CRDs × 2 versions each × 5 MB per CRD = 1 GB
+ 10,000 CRs × 10 KB each in informer = 100 MB
+ 1,000 watch connections × 50 KB each = 50 MB
= ~1.15 GB total
```

### Monitoring Memory Usage

```bash
# API server memory metrics
kubectl top pod -n kube-system <apiserver-pod>

# Process memory details
kubectl exec -n kube-system <apiserver-pod> -- ps aux

# Go memory stats (if metrics endpoint available)
curl https://<apiserver>:6443/metrics | grep go_memstats
```

## Latency Targets

### CRD Operations

**Target**: < 100ms (p99)

**Operations**:
- Create CRD
- Update CRD
- Delete CRD
- Get CRD
- List CRDs

**Typical Breakdown**:
```
Request parsing:     ~1ms
Validation:          ~5ms
Admission:           ~10ms (if webhooks configured)
etcd write:          ~10-50ms
Response encoding:   ~1ms
Total:               ~30-70ms
```

**Optimization Tips**:
```bash
# Check CRD operation latency
kubectl create -f crd.yaml --dry-run=server -v=8 2>&1 | grep "Response Status"

# Monitor via metrics
# apiserver_request_duration_seconds{resource="customresourcedefinitions"}
```

### Custom Resource Operations (No Webhooks)

**Target**: < 50ms (p99) for Get/List

**Get Operation**:
```
Request routing:     ~1ms
Storage lookup:      ~5-20ms (etcd read)
Conversion:          ~1-5ms (if version differs from storage)
Response encoding:   ~1ms
Total:               ~10-30ms
```

**List Operation**:
```
Request routing:     ~1ms
Storage list:        ~10-50ms (depends on result size)
Conversion:          ~1-10ms per object
Response encoding:   ~5-20ms (depends on result size)
Total:               ~20-80ms (for 100 objects)
```

**Create/Update Operation**:
```
Request parsing:     ~1ms
Defaulting:          ~1-5ms
Admission (mutating): ~0ms (if no webhooks)
Schema validation:   ~5-20ms (depends on complexity)
CEL validation:      ~5-50ms (depends on rule complexity)
Admission (validating): ~0ms (if no webhooks)
Conversion:          ~1-5ms (if needed)
etcd write:          ~10-50ms
Response encoding:   ~1ms
Total:               ~25-130ms
```

**Optimization Tips**:
```yaml
# Minimize schema complexity
schema:
  type: object
  properties:
    # Keep nesting shallow (< 5 levels)
    # Avoid large enums (< 100 values)
    # Use string patterns sparingly

# Optimize CEL rules
x-kubernetes-validations:
- rule: "self.replicas >= 0"  # Simple comparison: ~1ms
- rule: "self.items.map(i, i.name).unique()"  # Complex: ~10-50ms
```

### Webhook Operations

**Latency Impact**: +10-50ms per webhook call

**Breakdown**:
```
Network round-trip:  ~5-20ms (depends on webhook location)
Webhook processing:  ~5-30ms (depends on webhook logic)
Retry overhead:      ~0-100ms (on failure)
Total overhead:      ~10-50ms typical, up to 150ms worst case
```

**Multiple Webhooks**:
- Mutating webhooks run sequentially
- Validating webhooks can run in parallel
- Total latency = sum(mutating) + max(validating)

**Optimization Tips**:
```yaml
# Configure reasonable timeouts
timeoutSeconds: 10  # Default is 30s, reduce for faster failures

# Use failurePolicy: Ignore for non-critical webhooks
failurePolicy: Ignore  # Don't block on webhook failures

# Co-locate webhooks in same cluster
clientConfig:
  service:
    namespace: default
    name: webhook-service  # In-cluster is faster than external URL
```

**Monitoring Webhooks**:
```bash
# Check webhook latency
kubectl get --raw /metrics | grep apiserver_admission_webhook_admission_duration_seconds

# Test webhook directly
curl -k -X POST https://<webhook-service>/validate \
  -H "Content-Type: application/json" \
  -d @admission-review.json
```

### Discovery Operations

**Target**: < 10ms (p99)

**Characteristics**:
- Client-side caching (kubectl caches for 10 minutes)
- Small response size (< 100 KB for typical clusters)
- No etcd access (served from memory)

**Optimization Tips**:
```bash
# Use aggregated discovery (single request)
kubectl get --raw /apis?aggregated=true

# Monitor discovery latency
kubectl get --raw /apis -v=8 2>&1 | grep "Response Status"
```

## Critical Path Analysis

### Hot Paths (Performance Critical)

#### 1. Custom Resource GET/LIST/WATCH

**Why Critical**: Most common operations, latency-sensitive

**Request Flow**:
```
HTTP Handler
    ↓ (< 1ms)
crdHandler.ServeHTTP
    ↓ (< 1ms)
Load storage from atomic.Value (lock-free!)
    ↓ (< 1ms)
Select request scope (version-specific)
    ↓ (5-20ms)
Storage.Get/List/Watch (etcd access)
    ↓ (1-5ms, optional)
Convert storage version → requested version
    ↓ (< 1ms)
Encode response (JSON/YAML/Protobuf)
```

**Optimization**: Atomic storage map (`atomic.Value`)
- Lock-free reads (~10x faster than RWMutex)
- Writes are rare (only on CRD changes)
- Critical for high-throughput read workloads

**Code Reference**: `pkg/apiserver/customresource_handler.go:ServeHTTP`

#### 2. Discovery Lookups

**Why Critical**: kubectl queries on every command

**Request Flow**:
```
HTTP Handler
    ↓ (< 1ms)
Discovery Handler (versionDiscoveryHandler or groupDiscoveryHandler)
    ↓ (< 1ms)
Load cached discovery document (atomic read)
    ↓ (< 1ms)
Encode response
```

**Optimization**: Pre-computed discovery documents
- Updated only on CRD changes (via controller)
- Atomic read for zero-lock overhead
- Client-side caching (10 min kubectl)

**Code Reference**: `pkg/apiserver/customresource_discovery_controller.go`

#### 3. Informer Caching

**Why Critical**: Controllers rely on fast local reads

**Data Flow**:
```
Watch Event from etcd
    ↓
Informer receives event
    ↓
Update local cache (map access)
    ↓
Invoke event handlers
    ↓
Controller reconciliation logic
```

**Optimization**: Local in-memory cache
- No API server round-trip
- Fast map lookups (O(1))
- Eventual consistency acceptable

**Usage Pattern**:
```go
informer.Lister().Get(name)  // Fast local read
```

#### 4. CEL Validation

**Why Critical**: Runs on every CR create/update

**Validation Flow**:
```
Schema Validation (types, formats, constraints)
    ↓ (5-20ms)
CEL Validation
    ↓
    Load compiled CEL program from cache (fast!)
        ↓ (5-50ms)
    Evaluate program with variables (self, oldSelf)
        ↓ (optional)
    Evaluate message expression
```

**Optimization**: CEL compilation caching
- Compile once at CRD registration
- Cache programs per version
- Evaluation-only cost at runtime

**Code Reference**: `pkg/apiserver/schema/cel/compilation.go`

**Cost Management**:
```yaml
# CEL cost limit: 1,000,000 units per rule
x-kubernetes-validations:
- rule: "self.size() < 1000"  # Low cost: ~100 units
- rule: "self.items.map(i, i.validate()).all(v, v)"  # High cost: ~10,000+ units
```

## Optimization Strategies

### 1. Atomic Storage Map

**Problem**: Lock contention on read-heavy workload

**Solution**: Use `atomic.Value` for crdStorageMap

**Implementation**:
```go
type crdHandler struct {
    customStorage atomic.Value  // crdStorageMap
}

// Read (hot path) - lock-free!
storageMap := handler.customStorage.Load().(crdStorageMap)
storage := storageMap[crdName]

// Write (rare) - atomic swap
newMap := make(crdStorageMap)
// ... populate newMap ...
handler.customStorage.Store(newMap)
```

**Performance Impact**:
- ~10x faster reads than RWMutex
- No lock contention on read path
- Acceptable write latency (only on CRD changes)

**Code Reference**: `pkg/apiserver/customresource_handler.go`

### 2. Discovery Caching

**Problem**: Discovery requests on every kubectl command

**Solution**: Pre-compute discovery documents, client-side caching

**Implementation**:
```go
// Server-side: Atomic discovery document
type versionDiscoveryHandler struct {
    discoveryDocument atomic.Value  // *metav1.APIResourceList
}

// Update only on CRD changes (via controller)
handler.discoveryDocument.Store(newDocument)

// Client-side: kubectl caches for 10 minutes
```

**Performance Impact**:
- Discovery served from memory (< 1ms)
- Client caching reduces requests by ~100x
- Controller overhead negligible

**Code Reference**: `pkg/apiserver/customresource_discovery_controller.go`

### 3. CEL Compilation Caching

**Problem**: CEL compilation expensive (10-100ms)

**Solution**: Compile once, cache per version

**Implementation**:
```go
type CompilationResult struct {
    Program     cel.Program  // Compiled and ready
    UsesOldSelf bool
    MaxCost     uint64
}

// Compile on CRD registration
compiledPrograms := compileValidationRules(schema)

// Cache in validator
validator.programs = compiledPrograms

// Evaluate on each request (fast!)
result, cost, err := program.Eval(variables)
```

**Performance Impact**:
- Compilation: ~10-100ms (one-time cost)
- Evaluation: ~5-50ms (per request)
- Total savings: ~10-100ms per request

**Code Reference**: `pkg/apiserver/schema/cel/compilation.go`

### 4. Storage Instance Reuse

**Problem**: Creating storage instances expensive

**Solution**: Cache storage per CRD version

**Implementation**:
```go
type crdInfo struct {
    storages map[string]customresource.CustomResourceStorage  // Per version
    // ... other fields ...
}

// Create on CRD registration
for _, version := range crd.Spec.Versions {
    storage := createStorage(version)
    info.storages[version.Name] = storage
}

// Reuse on every request
storage := info.storages[requestedVersion]
```

**Performance Impact**:
- Eliminates per-request storage creation
- Consistent memory usage
- Fast version lookup (map access)

**Code Reference**: `pkg/apiserver/customresource_handler.go`

## Performance Bottlenecks

### 1. Webhook Calls

**Problem**: Network latency for conversion/admission webhooks

**Latency Impact**: +10-50ms per webhook call

**Mitigation Strategies**:

**A. Co-locate Webhooks**:
```yaml
# Use in-cluster service (faster than external URL)
clientConfig:
  service:
    namespace: webhook-namespace
    name: webhook-service
    port: 443
```

**B. Optimize Webhook Logic**:
```go
// Keep webhook handlers fast (< 10ms)
func (h *Handler) Handle(ctx context.Context, req admission.Request) admission.Response {
    // Avoid slow operations:
    // - External API calls
    // - Database queries
    // - Complex computations

    // Use in-memory validation
    if !validateInMemory(req.Object) {
        return admission.Denied("validation failed")
    }
    return admission.Allowed("")
}
```

**C. Use failurePolicy Wisely**:
```yaml
# Non-critical webhooks: fail open
failurePolicy: Ignore  # Don't block on webhook failures

# Critical webhooks: fail closed
failurePolicy: Fail  # Block on webhook failures
```

**D. Configure Reasonable Timeouts**:
```yaml
# Default is 30s, reduce for faster failures
timeoutSeconds: 10
```

**Monitoring**:
```bash
# Check webhook latency
kubectl get --raw /metrics | grep apiserver_admission_webhook_admission_duration_seconds

# Check webhook failures
kubectl get --raw /metrics | grep apiserver_admission_webhook_rejection_count
```

### 2. Storage Recreation

**Problem**: Full storage rebuild on CRD spec changes

**Latency Impact**: ~100-500ms per CRD update

**Mitigation Strategies**:

**A. Minimize CRD Updates**:
```bash
# Batch multiple changes into single update
kubectl apply -f updated-crd.yaml
```

**B. Use Strategic Updates**:
```bash
# Update only changed fields
kubectl patch crd mycrd --type=merge -p '{"spec":{"versions":[...]}}'
```

**C. Accept Brief Unavailability**:
- Storage recreation happens asynchronously
- In-flight requests complete before teardown
- New requests wait for new storage

**Monitoring**:
```bash
# Watch CRD status during updates
kubectl get crd mycrd -w

# Check for Established condition
kubectl get crd mycrd -o jsonpath='{.status.conditions[?(@.type=="Established")].status}'
```

### 3. Schema Validation

**Problem**: Deep nesting and complex schemas slow validation

**Latency Impact**: ~5-50ms (depends on schema complexity)

**Mitigation Strategies**:

**A. Keep Schemas Shallow**:
```yaml
# Prefer flat structures (< 5 levels deep)
properties:
  config:
    type: object
    properties:
      setting1: { type: string }
      setting2: { type: integer }
    # Avoid: deep nesting beyond 3-4 levels
```

**B. Minimize Validation Constraints**:
```yaml
# Avoid large enums
enum: ["value1", "value2"]  # OK: small enum
# enum: ["v1", "v2", ..., "v100"]  # Slow: large enum

# Use simple patterns
pattern: "^[a-z]+$"  # OK: simple pattern
# pattern: "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)(?=.*[@$!%*?&])[A-Za-z\\d@$!%*?&]{8,}$"  # Slow: complex pattern
```

**C. Use CEL for Complex Validation**:
```yaml
# CEL is faster than equivalent JSON schema
x-kubernetes-validations:
- rule: "self.replicas >= 0 && self.replicas <= 100"
  # Faster than: minimum: 0, maximum: 100 with additional checks
```

**Monitoring**:
```bash
# Profile validation time
kubectl create -f resource.yaml --dry-run=server -v=8 2>&1 | grep "validation"
```

### 4. Watch Connection Count

**Problem**: Memory and fanout latency scale with watch count

**Latency Impact**: ~1-10ms additional fanout per 1000 watchers

**Mitigation Strategies**:

**A. Use Informers (Shared Watches)**:
```go
// BAD: Each controller creates separate watch
watch, err := client.Watch(ctx, metav1.ListOptions{})

// GOOD: Use shared informer factory
informerFactory := informers.NewSharedInformerFactory(client, 10*time.Minute)
informer := informerFactory.Mygroup().V1().MyResources().Informer()
informer.AddEventHandler(handler)
```

**B. Configure Informer Resync Period**:
```go
// Longer resync reduces watch churn
informerFactory := informers.NewSharedInformerFactory(client, 30*time.Minute)
```

**C. Use Label Selectors**:
```go
// Watch subset of resources
listOptions := metav1.ListOptions{
    LabelSelector: "app=myapp",
}
```

**Monitoring**:
```bash
# Count active watches
kubectl get --raw /metrics | grep apiserver_watch_list_total

# Check watch event latency
kubectl get --raw /metrics | grep apiserver_watch_events_duration_seconds
```

## Monitoring and Metrics

### Key Metrics to Monitor

**API Server Request Latency**:
```promql
histogram_quantile(0.99,
  sum(rate(apiserver_request_duration_seconds_bucket{resource="myresources"}[5m])) by (le, verb)
)
```

**Custom Resource Operations Rate**:
```promql
sum(rate(apiserver_request_total{resource="myresources"}[5m])) by (verb)
```

**Webhook Latency**:
```promql
histogram_quantile(0.99,
  sum(rate(apiserver_admission_webhook_admission_duration_seconds_bucket[5m])) by (le, name)
)
```

**Watch Connection Count**:
```promql
apiserver_longrunning_requests{verb="WATCH"}
```

**Memory Usage**:
```promql
process_resident_memory_bytes{job="apiserver"}
```

### Performance Testing

**Load Test Script**:
```bash
#!/bin/bash
# Create CRD
kubectl apply -f crd.yaml

# Wait for established
kubectl wait --for=condition=Established crd/myresources.mygroup.example.com

# Create 1000 resources
for i in {1..1000}; do
  kubectl create -f - <<EOF
apiVersion: mygroup.example.com/v1
kind: MyResource
metadata:
  name: test-$i
spec:
  field: value-$i
EOF
done

# Measure list latency
time kubectl get myresources

# Measure get latency
time kubectl get myresource test-500
```

**Watch Performance Test**:
```go
func BenchmarkWatchPerformance(b *testing.B) {
    // Start multiple watchers
    for i := 0; i < 100; i++ {
        go func() {
            watch, _ := client.Watch(ctx, metav1.ListOptions{})
            for event := range watch.ResultChan() {
                // Process event
            }
        }()
    }

    // Create resources and measure fanout
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        client.Create(ctx, newResource(), metav1.CreateOptions{})
    }
}
```

## Best Practices

### Schema Design
- Keep nesting shallow (< 5 levels)
- Minimize validation constraints
- Use CEL for complex validation
- Avoid large enums (< 100 values)

### Webhook Design
- Co-locate webhooks in cluster
- Keep handlers fast (< 10ms)
- Use reasonable timeouts (10s)
- Configure failurePolicy appropriately

### Client Design
- Use shared informers
- Configure appropriate resync periods
- Use label/field selectors
- Enable client-side throttling

### Operational
- Monitor key metrics (latency, rate, errors)
- Set up alerts for P99 latency
- Load test before production
- Capacity plan based on CRD/CR count

## Related Documentation

- [Custom Resource Handler](../core-components/custom-resource-handler.md) - Request handling implementation
- [Schema Validation](../core-components/schema-validation.md) - Validation pipeline details
- [CEL Integration](../core-components/cel-integration.md) - CEL validation performance
- [Troubleshooting Guide](./troubleshooting.md) - Performance issue debugging
