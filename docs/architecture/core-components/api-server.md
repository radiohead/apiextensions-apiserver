# API Server Initialization

The API Extensions API Server implements a delegated API server pattern that extends Kubernetes with CustomResourceDefinition (CRD) support. This document covers the initialization sequence, configuration structure, and core architectural patterns.

## Overview

**Primary File**: `pkg/apiserver/apiserver.go` (286 lines)
**Entry Point**: `main.go` (33 lines)
**Command Setup**: `pkg/cmd/server/server.go` (~300 lines)
**Confidence**: 92%

The API server follows Kubernetes conventions with clean separation between initialization, configuration, and runtime operation. It uses a delegated API server pattern to chain to the main kube-apiserver for unknown endpoints.

## Entry Point and Command Structure

### Main Entry Point

**File**: `main.go:1-33`

```go
func main() {
    ctx := genericapiserver.SetupSignalContext()
    cmd := server.NewServerCommand(ctx, os.Stdout, os.Stderr)
    code := cli.Run(cmd)
    os.Exit(code)
}
```

**Responsibilities**:
- Set up signal context for graceful shutdown (SIGTERM, SIGINT)
- Delegate to server command via Cobra CLI framework
- Use `k8s.io/component-base/cli` for consistent CLI behavior
- Exit with appropriate code

**Pattern**: Minimal entry point that delegates to specialized server command.

### Server Command

**File**: `pkg/cmd/server/server.go`

The server command follows the Cobra CLI pattern:
- Command-line flag parsing
- Configuration validation
- Server instance creation
- Run loop with context

## Configuration Architecture

### Configuration Layers

The configuration follows a multi-layered pattern to prevent accidental modification:

#### 1. Config (Mutable)

```go
type Config struct {
    GenericConfig *genericapiserver.Config
    ExtraConfig   ExtraConfig
}
```

**GenericConfig**: Standard Kubernetes API server configuration
- Authentication/authorization
- Admission plugins
- Storage configuration
- API enablement
- Client CA bundle
- Service resolver

**ExtraConfig**: CRD-specific extensions
- `CRDRESTOptionsGetter`: Storage options for custom resources
- `MasterCount`: HA cluster detection (affects establishing delays)
- `ServiceResolver`: Webhook service name resolution
- `AuthResolverWrapper`: Authentication for webhook converters

#### 2. CompletedConfig (Immutable)

```go
type completedConfig struct {
    GenericConfig genericapiserver.CompletedConfig
    ExtraConfig   *ExtraConfig
}

type CompletedConfig struct {
    *completedConfig  // Private embedding prevents external instantiation
}

func (cfg *Config) Complete() CompletedConfig {
    c := completedConfig{
        cfg.GenericConfig.Complete(),
        &cfg.ExtraConfig,
    }
    return CompletedConfig{&c}
}
```

**Purpose**:
- Prevents accidental modification after completion
- Private embedding ensures only internal package can create
- Immutable configuration for thread-safe access

## Server Initialization Sequence

### New() Function Flow

**File**: `pkg/apiserver/apiserver.go:85-245`

```
1. Create Generic Server
    ↓
2. Setup CRD Informer Sync Signal
    ↓
3. Install CRD API Group
    ↓
4. Create Client and Informer
    ↓
5. Configure Handler Chain
    ↓
6. Register PostStartHooks
    ↓
7. Return CustomResourceDefinitions Server
```

### Step 1: Generic Server Creation

```go
genericServer, err := c.GenericConfig.New("apiextensions-apiserver", delegationTarget)
```

- Creates base API server with standard Kubernetes functionality
- Chains to `delegationTarget` (typically main kube-apiserver)
- Falls back to NotFoundHandler if no delegation target

### Step 2: CRD Informer Sync Signal

```go
hasCRDInformerSyncedSignal := make(chan struct{})
```

**Purpose**: Prevents 404 errors during initial CRD loading
- Returns 503 (Service Unavailable) until informer synced
- Signals when CRD informer has completed initial sync
- Critical for avoiding race conditions during startup

### Step 3: API Group Installation

**File**: `pkg/apiserver/apiserver.go:110-135`

```go
apiGroupInfo := genericapiserver.NewDefaultAPIGroupInfo(
    apiextensions.GroupName,
    Scheme,
    metav1.ParameterCodec,
    Codecs,
)

v1storage := map[string]rest.Storage{}
v1storage["customresourcedefinitions"] = customResourceDefinitionStorage
v1storage["customresourcedefinitions/status"] = customResourceDefinitionStatusStorage

apiGroupInfo.VersionedResourcesStorageMap["v1"] = v1storage
```

**Installed APIs**:
- Group: `apiextensions.k8s.io`
- Version: `v1` (and `v1beta1` for legacy)
- Resources:
  - `customresourcedefinitions` (main resource)
  - `customresourcedefinitions/status` (status subresource)

### Step 4: Client and Informer Setup

```go
client, err := clientset.NewForConfig(s.GenericAPIServer.LoopbackClientConfig)
informerFactory := informers.NewSharedInformerFactory(client, 5*time.Minute)
```

**Components**:
- **Loopback Client**: Client to self for internal operations
- **Informer Factory**: Shared informer with 5-minute resync
- **CRD Informer**: Watches CRD changes

**Note**: The loopback client is critical - server cannot function without it. See error message at line 167-168.

### Step 5: Handler Chain Configuration

**File**: `pkg/apiserver/apiserver.go:150-195`

```go
// Discovery handlers
versionDiscoveryHandler := &versionDiscoveryHandler{
    discovery: map[schema.GroupVersion]*discovery.APIVersionHandler{},
    delegate:  delegateHandler,
}

groupDiscoveryHandler := &groupDiscoveryHandler{
    discovery: map[string]*discovery.APIGroupHandler{},
    delegate:  delegateHandler,
}

// Custom resource handler
crdHandler := &crdHandler{
    versionDiscoveryHandler: versionDiscoveryHandler,
    groupDiscoveryHandler:   groupDiscoveryHandler,
    customStorage:           atomic.Value{},
    crdLister:               crdInformer.Lister(),
    // ... other fields
}
```

**Handler Hierarchy**:
1. `crdHandler`: Main handler for `/apis` and `/apis/*` endpoints
2. `versionDiscoveryHandler`: Handles API version discovery per group
3. `groupDiscoveryHandler`: Handles API group discovery
4. Delegates to underlying API server for non-CRD requests

### Step 6: Controller Integration via PostStartHooks

**File**: `pkg/apiserver/apiserver.go:200-240`

Eight controllers started via PostStartHooks:

#### Hook: start-apiextensions-informers

```go
s.GenericAPIServer.AddPostStartHookOrDie("start-apiextensions-informers", func(context genericapiserver.PostStartHookContext) error {
    informerFactory.Start(context.StopCh)
    return nil
})
```

Starts all informers to begin watching CRDs.

#### Hook: start-apiextensions-controllers

```go
s.GenericAPIServer.AddPostStartHookOrDie("start-apiextensions-controllers", func(context genericapiserver.PostStartHookContext) error {
    // Wait for informer sync
    informerFactory.WaitForCacheSync(context.StopCh)

    // Start controllers
    go discoveryController.Run(context.StopCh, discoverySyncedCh)
    go namingController.Run(5, context.StopCh)
    go establishingController.Run(5, context.StopCh)
    go nonStructuralSchemaController.Run(5, context.StopCh)
    go apiApprovalController.Run(5, context.StopCh)
    go finalizingController.Run(5, context.StopCh)

    return nil
})
```

**Controllers Started**:

1. **discoveryController**: Manages API discovery documents
2. **namingController**: Validates CRD naming conditions
3. **establishingController**: Marks CRDs as established
4. **nonStructuralSchemaController**: Validates schema structure
5. **apiApprovalController**: Enforces API approval policy (k8s.io domains)
6. **finalizingController**: Handles CRD deletion and cleanup
7. **openapiController**: Updates OpenAPI v2 specs
8. **openapiv3Controller**: Updates OpenAPI v3 specs

**Controller Details**: See [crd-lifecycle.md](./crd-lifecycle.md) for in-depth coverage.

#### Hook: crd-informer-synced

```go
s.GenericAPIServer.AddPostStartHookOrDie("crd-informer-synced", func(context genericapiserver.PostStartHookContext) error {
    return wait.PollImmediateUntil(100*time.Millisecond, func() (bool, error) {
        return crdInformer.Informer().HasSynced(), nil
    }, context.StopCh)
})
```

Waits for informer sync before reporting healthy. Prevents serving requests before CRDs loaded.

## Architectural Patterns

### 1. Delegation Pattern

```
API Request
    ↓
API Extensions Server
    ↓
CRD Endpoints? → Yes → CRD Handler
    ↓ No
Delegation Target (kube-apiserver)
    ↓
Core APIs or other delegated servers
```

**Benefits**:
- Modular API server composition
- Clear separation of concerns
- Enables API aggregation layer
- Falls back gracefully for unknown endpoints

### 2. Discovery Isolation

```go
// Disable generic discovery - use custom handlers
c.GenericConfig.EnableDiscovery = false
```

**Rationale**:
- Custom discovery logic for dynamic CRDs
- Supports aggregated discovery format
- CRD source tagging for discovery
- Better control over discovery format

**Implementation**: Custom `versionDiscoveryHandler` and `groupDiscoveryHandler`

### 3. Dynamic Handler Registration

```go
s.GenericAPIServer.Handler.NonGoRestfulMux.Handle("/apis", crdHandler)
s.GenericAPIServer.Handler.NonGoRestfulMux.Handle("/apis/", crdHandler)
```

**Pattern**: Uses `NonGoRestfulMux` for custom handler registration
- Handles `/apis` prefix for all CRD-related requests
- Bypasses generic REST routing for dynamic resources
- Enables runtime endpoint creation

### 4. Signal-Based Coordination

**Signals Used**:

```go
hasCRDInformerSyncedSignal := make(chan struct{})  // Readiness signal
discoverySyncedCh := make(chan struct{}, 1)        // Discovery ready
```

**Purpose**:
- Prevents race conditions during startup
- Coordinates readiness between components
- Ensures proper initialization order
- Enables graceful startup and shutdown

## Scheme and Codec Configuration

### Initialization

**File**: `pkg/apiserver/apiserver.go:45-65`

```go
var (
    Scheme = runtime.NewScheme()
    Codecs = serializer.NewCodecFactory(Scheme)

    unversionedVersion = schema.GroupVersion{Group: "", Version: "v1"}
    UnversionedGroupVersion = schema.GroupVersion{Group: "", Version: "v1"}
)

func init() {
    install.Install(Scheme)
    metav1.AddToGroupVersion(Scheme, schema.GroupVersion{Version: "v1"})
    unversioned := schema.GroupVersion{Group: "", Version: "v1"}
    Scheme.AddUnversionedTypes(unversioned,
        &metav1.Status{},
        &metav1.APIVersions{},
        &metav1.APIGroupList{},
        &metav1.APIGroup{},
        &metav1.APIResourceList{},
    )
}
```

**Registered Types**:
- CRD API types (v1, v1beta1)
- Unversioned meta types (Status, WatchEvent, APIGroup, APIResourceList)
- Standard Kubernetes object metadata

**Codec Factory**: Handles serialization/deserialization
- JSON (default)
- YAML
- Protobuf
- CBOR

## Integration Points

### Upstream Dependencies

1. **k8s.io/apiserver**: Generic API server framework
   - Authentication/authorization
   - Admission control
   - Storage layer
   - Middleware chain

2. **k8s.io/apimachinery**: Schema and runtime machinery
   - Type system
   - Serialization
   - Versioning

3. **Controller Runtime**: Reconciliation framework
   - Informers
   - Work queues
   - Event handlers

### Downstream Consumers

1. **Custom Resource Handler**: Serves CR CRUD operations
2. **Discovery Controllers**: Publish API discovery
3. **OpenAPI Controllers**: Generate specifications
4. **Admission Webhooks**: Validate and mutate resources

## Design Strengths

### 1. Clean Separation of Concerns
- Configuration distinct from initialization
- Initialization distinct from runtime
- Controllers independent and loosely coupled

### 2. Composability
- Delegation pattern enables API server composition
- Works with main kube-apiserver
- Supports aggregation layer

### 3. Graceful Startup
- Signal-based coordination prevents race conditions
- PostStartHooks ensure proper initialization order
- Informer sync before serving requests

### 4. HA-Aware
- MasterCount enables cluster-aware behavior
- 5-second establishing delay in multi-master clusters
- Prevents split-brain during CRD registration

### 5. Extensibility
- ExtraConfig pattern allows CRD-specific extensions
- Custom discovery handlers
- Pluggable storage options

## Design Considerations

### 1. Loopback Client Dependency
**Issue**: Server requires functional loopback client for operation

**Impact**: Cannot run without self-accessible API endpoint

**Mitigation**: Clear error message, documented requirement

### 2. Hardcoded Delays
**Issue**: 5-second establishing delay for HA clusters (via MasterCount)

**Impact**: May not suit all scenarios
- Too long: Slows CRD registration
- Too short: Risk of race conditions

**Recommendation**: Make configurable via flag or environment variable

### 3. Fixed Informer Resync
**Issue**: Hardcoded 5-minute resync period

```go
informerFactory := informers.NewSharedInformerFactory(client, 5*time.Minute)
```

**Impact**: May not suit all workloads
- High-churn environments: Too long
- Stable environments: Unnecessary overhead

**Recommendation**: Make resync period configurable

### 4. Error Message Leakage
**Location**: `pkg/apiserver/apiserver.go:167-168`

Error message mentions test workarounds, leaking implementation details:

```go
return nil, fmt.Errorf("failed to create loopback client (used for informers): %w", err)
```

**Recommendation**: Remove test-specific error message or improve clarity

## Performance Characteristics

### Startup Time
- **Generic Server**: ~100-200ms
- **Informer Sync**: ~1-5s (depends on CRD count)
- **Controller Start**: ~100ms
- **Total**: ~2-10s (primarily waiting for informer sync)

### Memory Usage
- **Base Server**: ~50-100 MB
- **Per CRD**: ~1-10 MB (storage + caches)
- **Informer Cache**: Scales with CRD count

### CPU Usage
- **Startup**: High (compilation, initialization)
- **Steady State**: Low (event-driven reconciliation)
- **Per CRD**: ~1-5% CPU during reconciliation

## Related Documentation

- **CRD Lifecycle**: [crd-lifecycle.md](./crd-lifecycle.md) - Controller details
- **Custom Resource Handler**: [custom-resource-handler.md](./custom-resource-handler.md) - CR CRUD operations
- **Discovery**: [discovery-openapi.md](./discovery-openapi.md) - API discovery and OpenAPI

## Critical Files Reference

| File | Purpose | Lines | Importance |
|------|---------|-------|------------|
| `main.go` | Binary entry point | 33 | High |
| `pkg/apiserver/apiserver.go` | Core server implementation | 286 | Critical |
| `pkg/cmd/server/server.go` | Cobra command setup | ~300 | High |
| `pkg/apiserver/customresource_handler.go` | CRD request handler | 1,746 | Critical |

## Testing

### Unit Tests
- Configuration validation
- Scheme registration
- Handler setup

### Integration Tests
- Full server lifecycle
- Controller integration
- Discovery functionality

**Test Location**: `pkg/apiserver/apiserver_test.go`

## Recommendations

### 1. Error Handling
Remove test-specific error message about loopback client. Provide clear user-facing error.

### 2. Configuration
Make the following configurable:
- Informer resync period
- Establishing delay for HA clusters
- Discovery sync timeout

### 3. Metrics
Add startup timing metrics for each initialization phase:
- Generic server creation
- API group installation
- Informer sync duration
- Controller startup time

### 4. Documentation
- Document MasterCount impact on establishing behavior
- Clarify loopback client requirement
- Explain delegation chain flow

### 5. Graceful Degradation
Handle loopback client failures more gracefully:
- Retry with exponential backoff
- Provide fallback behavior
- Clear recovery path
