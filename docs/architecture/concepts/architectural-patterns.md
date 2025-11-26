# Architectural Patterns

This document describes the five core architectural patterns that form the foundation of the apiextensions-apiserver design. These patterns are used consistently throughout the codebase and understanding them is essential for working with the system.

## Overview

The apiextensions-apiserver applies proven Kubernetes architectural patterns to solve the unique challenges of dynamic resource management. Each pattern addresses specific requirements:

- **Controller Pattern** - Autonomous lifecycle management
- **Delegated API Server** - Modular API composition
- **Dynamic Handler Registration** - Runtime endpoint creation
- **Multi-Version Support** - API evolution without breaking changes
- **Atomic Updates with Optimistic Locking** - Safe concurrent modifications

## Pattern 1: Controller Pattern

### Description

The Controller Pattern implements autonomous reconciliation loops that continuously work to make the actual state match the desired state. Each controller watches specific resources and reacts to changes.

### Implementation in apiextensions-apiserver

The system uses **seven independent controllers** to manage CRD lifecycle:

```
┌─────────────────────────────────────────────────────────┐
│                     CRD Resource                         │
└─────────────────────────────────────────────────────────┘
     │    │    │    │    │    │    │
     ↓    ↓    ↓    ↓    ↓    ↓    ↓
┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐
│Name │ │Estab│ │NonSt│ │APIAp│ │Final│ │Disco│ │OpenA│
│ ing │ │lish │ │ruct │ │prova│ │izer │ │very │ │  PI │
└─────┘ └─────┘ └─────┘ └─────┘ └─────┘ └─────┘ └─────┘
   ↓       ↓       ↓       ↓       ↓       ↓       ↓
┌──────────────────────────────────────────────────────┐
│              Status Conditions Updated                │
└──────────────────────────────────────────────────────┘
```

#### Controllers and Their Responsibilities

1. **Naming Controller** (`pkg/controller/status/`)
   - Validates `<plural>.<group>` naming format
   - Checks for conflicts with existing CRDs
   - Sets `NamesAccepted` condition

2. **Establishing Controller** (`pkg/controller/establish/`)
   - Marks CRDs as `Established` when storage ready
   - HA-aware: 5-second delay in multi-master clusters
   - Prevents premature API advertising

3. **NonStructuralSchema Controller** (`pkg/controller/nonstructuralschema/`)
   - Validates structural schema requirements
   - Warns if CEL/defaulting features unavailable
   - Sets `NonStructuralSchema` warning condition

4. **APIApproval Controller** (`pkg/controller/apiapproval/`)
   - Enforces approval for `*.k8s.io` groups
   - Requires `api-approved.kubernetes.io` annotation
   - Prevents namespace squatting

5. **Finalizer Controller** (`pkg/controller/finalizer/`)
   - Adds `customresourcecleanup.apiextensions.k8s.io` finalizer
   - Deletes all CR instances before allowing CRD deletion
   - Prevents orphaned resources

6. **Discovery Controller** (`pkg/apiserver/customresource_discovery_controller.go`)
   - Updates API discovery documents
   - Synchronizes with OpenAPI updates
   - Enables kubectl discovery

7. **OpenAPI Controllers** (v2 and v3)
   - Generates OpenAPI specifications
   - Updates specs on CRD changes
   - Supports schema-based tooling

### Key Characteristics

**Watch-Based Event Processing**
```go
// Typical controller structure
func (c *Controller) Run(workers int, stopCh <-chan struct{}) {
    defer utilruntime.HandleCrash()
    defer c.queue.ShutDown()

    // Wait for cache sync
    if !cache.WaitForCacheSync(stopCh, c.crdsSynced) {
        return
    }

    // Start workers
    for i := 0; i < workers; i++ {
        go wait.Until(c.runWorker, time.Second, stopCh)
    }

    <-stopCh
}
```

**Retry with Exponential Backoff**
- Rate-limited work queue
- Automatic retry on transient errors
- Exponential backoff prevents thundering herd
- Max retries for permanent failures

**Condition-Based Status Reporting**
```go
// Status conditions communicate state
type CustomResourceDefinitionStatus struct {
    Conditions []CustomResourceDefinitionCondition
    // Established, NamesAccepted, NonStructuralSchema, etc.
}
```

### Benefits

- **Independent Reconciliation**: Each controller operates autonomously
- **Clear Separation**: Single responsibility per controller
- **Easy Testing**: Controllers can be tested in isolation
- **Standard Pattern**: Familiar to Kubernetes developers
- **Graceful Degradation**: One controller failure doesn't affect others

### Trade-offs

- **Eventually Consistent**: State may be temporarily inconsistent
- **Ordering Challenges**: Controllers run independently, no guaranteed order
- **Complexity**: Multiple controllers to understand for complete picture
- **Resource Usage**: Each controller needs its own goroutines and caches

## Pattern 2: Delegated API Server

### Description

The Delegated API Server pattern enables modular API composition by chaining specialized API servers. Unknown requests fall through to the next server in the chain.

### Implementation in apiextensions-apiserver

```
┌────────────────────────────────────────────┐
│           Client (kubectl/code)            │
└────────────────────────────────────────────┘
                    ↓
┌────────────────────────────────────────────┐
│         kube-apiserver (main)              │
│  - Core resources (pods, services, etc.)   │
│  - API aggregation layer                   │
└────────────────────────────────────────────┘
                    ↓ (unknown endpoint)
┌────────────────────────────────────────────┐
│      apiextensions-apiserver               │
│  - CustomResourceDefinitions               │
│  - Custom Resources (dynamically created)  │
└────────────────────────────────────────────┘
                    ↓ (unknown endpoint)
┌────────────────────────────────────────────┐
│         Other Aggregated Servers           │
│  - metrics-server, custom API servers      │
└────────────────────────────────────────────┘
```

### Request Routing

```go
type crdHandler struct {
    customStorage atomic.Value  // Handle CRs
    delegate      http.Handler  // Fall back for unknown
}

func (r *crdHandler) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    // Try custom resources first
    if info, ok := r.lookupCustomResource(req); ok {
        info.ServeHTTP(w, req)
        return
    }

    // Fall back to delegate
    r.delegate.ServeHTTP(w, req)
}
```

### Key Characteristics

**Chain of Responsibility**
- Each server handles its known endpoints
- Unknown requests passed to next server
- Final server returns 404 if not found

**Independent Deployment**
- API servers can be deployed separately
- Independent scaling and updates
- Isolated failure domains

**Standard Aggregation Layer**
- Uses Kubernetes APIService resources
- Automatic routing configuration
- Certificate-based authentication

### Benefits

- **Modularity**: Clear separation of concerns between API servers
- **Extensibility**: Add new API servers without modifying core
- **Independent Evolution**: Each server can evolve independently
- **Resource Isolation**: Memory, CPU isolated per server

### Trade-offs

- **Additional Latency**: Extra network hop for delegated requests
- **Complexity**: Multiple processes to deploy and monitor
- **Authentication**: Must handle certificate-based auth correctly
- **Discovery**: Aggregated discovery needs coordination

## Pattern 3: Dynamic Handler Registration

### Description

Dynamic Handler Registration enables creating and serving new API endpoints at runtime without server restarts. This is essential for CRDs where resource types are defined by users.

### Implementation in apiextensions-apiserver

```
CRD Created → Storage Built → Atomic Update → Endpoints Available
```

#### Storage Management

```go
type crdStorageMap map[types.UID]*crdInfo

type crdInfo struct {
    storages        map[string]customresource.CustomResourceStorage
    requestScopes   map[string]*handlers.RequestScope
    storageVersion  string
    waitGroup       *utilwaitgroup.SafeWaitGroup
}

// Atomic update pattern
func (r *crdHandler) updateStorage(newMap crdStorageMap) {
    r.customStorage.Store(newMap)  // Atomic!
}

// Lock-free read pattern
func (r *crdHandler) getStorage(uid types.UID) (*crdInfo, bool) {
    storageMap := r.customStorage.Load().(crdStorageMap)
    info, ok := storageMap[uid]
    return info, ok
}
```

#### Registration Flow

```
1. CRD Informer detects new/updated CRD
        ↓
2. Build storage for each version
   - REST storage implementation
   - Request scopes (main, status, scale)
   - Validation and conversion setup
        ↓
3. Create new storage map (copy-on-write)
        ↓
4. Atomic swap via atomic.Value.Store()
        ↓
5. Discovery and OpenAPI updated
        ↓
6. Endpoints immediately available
```

### Key Characteristics

**Atomic Updates**
- Copy-on-write storage map
- Single atomic swap operation
- No partial visibility of changes
- No locks on read path

**Per-CRD Storage**
- Independent storage per CRD
- Multiple versions per CRD
- Isolated request handling

**Graceful Teardown**
- WaitGroup tracks in-flight requests
- New requests rejected during deletion
- Existing requests complete gracefully

```go
// Request handling with graceful shutdown
func (r *crdInfo) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    if !r.waitGroup.Add(1) {
        // CRD being deleted, reject new requests
        http.Error(w, "CRD is terminating", http.StatusGone)
        return
    }
    defer r.waitGroup.Done()

    // Process request...
}
```

### Benefits

- **No Restart Required**: New endpoints available immediately
- **High Performance**: Lock-free reads via atomic.Value
- **Safe Concurrent Access**: Atomic updates prevent races
- **Clean Lifecycle**: Graceful creation and deletion

### Trade-offs

- **Memory Overhead**: Copy-on-write creates temporary duplicates
- **Write Latency**: Must rebuild entire storage map
- **Complexity**: More complex than static registration
- **Testing**: Race conditions require careful testing

## Pattern 4: Multi-Version Support

### Description

Multi-Version Support enables serving multiple API versions simultaneously while maintaining a single storage representation. This allows API evolution without breaking existing clients.

### Implementation in apiextensions-apiserver

```
┌─────────────────────────────────────────────────────────┐
│                  Client Requests                         │
├─────────────┬─────────────┬─────────────┬───────────────┤
│   v1alpha1  │    v1beta1  │      v1     │     v2beta1   │
└─────────────┴─────────────┴─────────────┴───────────────┘
      ↓              ↓              ↓              ↓
┌─────────────────────────────────────────────────────────┐
│              Version Conversion (webhook)                │
└─────────────────────────────────────────────────────────┘
                          ↓
              ┌───────────────────────┐
              │  Storage Version (v1) │
              │        in etcd        │
              └───────────────────────┘
```

#### Version Configuration

```go
type CustomResourceDefinitionVersion struct {
    Name    string  // e.g., "v1", "v1beta1"
    Served  bool    // Accept requests for this version
    Storage bool    // Use as storage version (exactly one)
    Schema  *CustomResourceValidation
    Subresources *CustomResourceSubresources
}
```

#### Conversion Strategies

**Strategy: None**
- Only changes `apiVersion` field
- Identical schemas across versions
- Zero overhead

```yaml
conversion:
  strategy: None
```

**Strategy: Webhook**
- External service performs conversion
- Supports complex transformations
- Standard ConversionReview protocol

```yaml
conversion:
  strategy: Webhook
  webhook:
    conversionReviewVersions: ["v1", "v1beta1"]
    clientConfig:
      service:
        name: my-converter
        namespace: default
        path: /convert
```

#### Request Flow with Conversion

```
1. Client sends request in v1beta1
        ↓
2. Admission/Validation in v1beta1
        ↓
3. Convert v1beta1 → v1 (storage version)
        ↓
4. Store in etcd as v1
        ↓
5. Read from etcd as v1
        ↓
6. Convert v1 → v1beta1 (requested version)
        ↓
7. Return response in v1beta1
```

### Key Characteristics

**Storage Version Management**
- Exactly ONE version marked `storage: true`
- All resources stored in this version
- Changing requires migration

**Version-Specific Features**
- Different schemas per version
- Different subresources
- Independent deprecation

**Tracked Storage Versions**
```go
type CustomResourceDefinitionStatus struct {
    StoredVersions []string  // Versions actually stored in etcd
}
```

### Benefits

- **API Evolution**: Add new versions without breaking existing clients
- **Graceful Migration**: Clients migrate at their own pace
- **Deprecation Support**: Mark versions as deprecated
- **Flexibility**: Different features per version

### Trade-offs

- **Conversion Overhead**: Every operation may require conversion
- **Webhook Dependency**: Webhook downtime blocks operations
- **Migration Complexity**: Changing storage version requires manual migration
- **Storage Tracking**: Must clean up storedVersions manually

## Pattern 5: Atomic Updates with Optimistic Locking

### Description

Atomic Updates with Optimistic Locking enable safe concurrent modifications without blocking reads. ResourceVersion-based conflict detection ensures consistency.

### Implementation in apiextensions-apiserver

#### ResourceVersion-Based Concurrency

```
Client A reads resource (resourceVersion: "100")
Client B reads resource (resourceVersion: "100")
    ↓
Client A updates (expects version "100") → Success (new version "101")
Client B updates (expects version "100") → Conflict! (current is "101")
    ↓
Client B re-reads (version "101"), re-applies change → Success (version "102")
```

#### Conflict Detection

```go
// Storage layer enforces optimistic locking
func (s *CustomResourceStorage) Update(ctx context.Context,
    name string, objInfo rest.UpdatedObjectInfo, ...) (runtime.Object, error) {

    // Get current object
    existing, err := s.Get(ctx, name, &metav1.GetOptions{})
    if err != nil {
        return nil, err
    }

    // Apply update
    updated, err := objInfo.UpdatedObject(ctx, existing)
    if err != nil {
        return nil, err
    }

    // Check ResourceVersion match
    if existing.GetResourceVersion() != updated.GetResourceVersion() {
        return nil, errors.NewConflict(...)
    }

    // Atomic update in etcd
    return s.storage.Update(ctx, key, updated, existing.GetResourceVersion(), ...)
}
```

#### Server-Side Apply (SSA)

Server-Side Apply provides declarative updates with field ownership:

```yaml
# Client A manages spec.replicas
apiVersion: mygroup.io/v1
kind: MyResource
metadata:
  name: example
  managedFields:
  - manager: client-a
    fieldsV1:
      spec.replicas: {}
spec:
  replicas: 3

# Client B manages spec.template
# No conflict - different fields
```

**Field Ownership Tracking**
- Each field has an owner (manager)
- Conflicts only on same-field updates
- Declarative semantics (apply desired state)
- Automatic conflict resolution

### Key Characteristics

**Optimistic Concurrency**
- No locks during reads
- Conflict detection on write
- Client retry on conflict
- Eventual consistency

**Atomic etcd Operations**
- Compare-and-swap in etcd
- Prevents lost updates
- Linearizable consistency

**Retry Pattern**
```go
// Standard retry pattern
err := retry.RetryOnConflict(retry.DefaultBackoff, func() error {
    // Read current state
    current, err := client.Get(ctx, name, metav1.GetOptions{})
    if err != nil {
        return err
    }

    // Apply modifications
    current.Spec.Replicas = 5

    // Attempt update (may conflict)
    _, err = client.Update(ctx, current, metav1.UpdateOptions{})
    return err
})
```

### Benefits

- **High Concurrency**: Reads never blocked
- **Data Safety**: Prevents lost updates
- **Simple Model**: Easy to reason about
- **Standard Pattern**: Used throughout Kubernetes

### Trade-offs

- **Client Complexity**: Clients must handle conflicts
- **Retry Storms**: Many clients can cause contention
- **Latency**: Conflicts require retry with re-read
- **Fairness**: No fairness guarantees in high contention

## Pattern Interactions

These patterns work together to create a cohesive system:

```
Controller Pattern
    ↓ (manages)
Dynamic Handler Registration
    ↓ (creates)
Delegated API Server Endpoints
    ↓ (handle)
Multi-Version Requests
    ↓ (use)
Atomic Updates with Optimistic Locking
```

**Example: Creating a CRD**
1. **Controller Pattern**: Naming controller validates names
2. **Dynamic Handler Registration**: Storage created for each version
3. **Multi-Version Support**: Endpoints for all served versions
4. **Atomic Updates**: Storage map atomically updated
5. **Delegated API Server**: New endpoints available via delegation

## Anti-Patterns to Avoid

### Blocking Reads with Locks
**Wrong**: Using RWMutex for storage map
```go
// Anti-pattern
type crdHandler struct {
    mu      sync.RWMutex
    storage map[types.UID]*crdInfo
}
```

**Right**: Using atomic.Value for lock-free reads
```go
// Correct pattern
type crdHandler struct {
    customStorage atomic.Value  // crdStorageMap
}
```

### Synchronous Controller Dependencies
**Wrong**: Controller A waits for Controller B
```go
// Anti-pattern
func (c *ControllerA) Reconcile(crd *CRD) {
    controllerB.DoWork(crd)  // Blocking!
    c.doMyWork(crd)
}
```

**Right**: Controllers work independently via status conditions
```go
// Correct pattern
func (c *ControllerA) Reconcile(crd *CRD) {
    c.doMyWork(crd)
    c.updateCondition(crd, "ControllerAReady", "True")
}

func (c *ControllerB) Reconcile(crd *CRD) {
    if !isConditionTrue(crd, "ControllerAReady") {
        return nil  // Wait for Controller A
    }
    c.doMyWork(crd)
}
```

### Manual Conflict Resolution
**Wrong**: Overwriting conflicts without retry
```go
// Anti-pattern
obj.Spec.Replicas = 5
client.Update(ctx, obj, metav1.UpdateOptions{})  // May lose data!
```

**Right**: Using retry loop
```go
// Correct pattern
retry.RetryOnConflict(retry.DefaultBackoff, func() error {
    current, _ := client.Get(ctx, name, metav1.GetOptions{})
    current.Spec.Replicas = 5
    _, err := client.Update(ctx, current, metav1.UpdateOptions{})
    return err
})
```

## Summary

These five architectural patterns form the foundation of the apiextensions-apiserver:

1. **Controller Pattern** - Independent reconciliation loops for lifecycle management
2. **Delegated API Server** - Modular API composition via chaining
3. **Dynamic Handler Registration** - Runtime endpoint creation without restarts
4. **Multi-Version Support** - API evolution with backward compatibility
5. **Atomic Updates with Optimistic Locking** - Safe concurrent modifications

Understanding these patterns is essential for:
- Debugging issues in the system
- Adding new features that fit the architecture
- Optimizing performance bottlenecks
- Reasoning about failure modes

Each pattern has been proven in production Kubernetes deployments at massive scale, and their consistent application throughout the codebase enables a robust, performant, and maintainable system.
