# Core Components

This directory documents the core architectural components of the Kubernetes API Extensions API Server.

## Component Index

The apiextensions-apiserver implements a delegated API server pattern with eight major components:

### 1. [API Server](./api-server.md)
**File**: `pkg/apiserver/apiserver.go`

Initialization and configuration of the delegated API server. Covers the startup sequence, configuration patterns, controller integration via PostStartHooks, and the delegation chain to the main kube-apiserver.

**Key Topics**:
- Entry point and command structure
- Configuration layers (Config, CompletedConfig, ExtraConfig)
- Server initialization sequence
- PostStartHook controller startup
- Signal-based coordination

### 2. [CRD Lifecycle](./crd-lifecycle.md)
**Files**: `pkg/controller/establish/`, `pkg/controller/status/`, etc.

CustomResourceDefinition lifecycle management through seven independent controllers. Each controller manages a specific aspect of CRD state from creation through deletion.

**Key Topics**:
- CRD API structure and versioning
- Establishing Controller (marks CRDs ready)
- Naming Controller (validates names)
- NonStructuralSchema Controller (schema validation)
- API Approval Controller (k8s.io namespace protection)
- Finalizer Controller (cleanup on deletion)
- Status conditions and transitions

### 3. [Custom Resource Handler](./custom-resource-handler.md)
**File**: `pkg/apiserver/customresource_handler.go` (1,746 lines)

Dynamic HTTP handler serving all custom resource CRUD operations. Uses atomic storage management for lock-free reads and handles versioning, validation, and transformation pipelines.

**Key Topics**:
- crdHandler architecture
- Atomic storage map (lock-free reads)
- CRUD operations (Create, Read, Update, Delete)
- List and Watch operations
- Subresources (status, scale)
- Serialization and content negotiation

### 4. [Schema Validation](./schema-validation.md)
**Files**: `pkg/apiserver/schema/`

Comprehensive JSON Schema v3-based validation with Kubernetes extensions. Enforces "structural schemas" required for CEL validation, defaulting, and server-side apply.

**Key Topics**:
- Structural schema concept and requirements
- Validation pipeline (type, format, constraints)
- Defaulting system
- Pruning of unknown fields
- Object metadata validation
- List type validation (atomic, set, map)

### 5. [CEL Validation](./cel-validation.md)
**Files**: `pkg/apiserver/schema/cel/`

Common Expression Language (CEL) integration for expressive validation rules. Compiles expressions at CRD registration time, enforces cost limits, and supports transition rules with `oldSelf`.

**Key Topics**:
- CEL rule definition and compilation
- Cost management system
- Variables (self, oldSelf)
- Transition rules for updates
- Message expressions
- CEL standard library

### 6. [Version Conversion](./version-conversion.md)
**Files**: `pkg/apiserver/conversion/`

Multi-version CRD support through webhook-based conversion or simple apiVersion changes. Enables seamless API evolution with multiple versions served simultaneously.

**Key Topics**:
- Conversion strategies (None, Webhook)
- Storage version management
- Webhook protocol (ConversionReview)
- Performance impact and optimization
- Storage version transitions

### 7. [Discovery and OpenAPI](./discovery-openapi.md)
**Files**: `pkg/apiserver/customresource_discovery_controller.go`, `pkg/controller/openapi/`, `pkg/controller/openapiv3/`

Dynamic API discovery publishing and OpenAPI v2/v3 specification generation. Synchronizes discovery documents as CRDs are registered, enabling kubectl and other clients to discover resources.

**Key Topics**:
- Discovery controller and handlers
- Discovery endpoints (legacy and aggregated)
- OpenAPI v2 controller and spec generation
- OpenAPI v3 controller (per-group specs)
- Schema conversion and extension handling
- Deprecation warnings

### 8. [Client Libraries](./client-libraries.md)
**Files**: `pkg/client/`

Generated Go clients for type-safe CRD interaction. Includes clientsets, informers, listers, and deep copy functions following standard Kubernetes patterns.

**Key Topics**:
- Code generation system
- Clientsets for typed API access
- Informers for watching and caching
- Listers for cached lookups
- Deep copy methods
- Fake clients for testing
- Apply configurations for SSA

## Component Dependency Graph

```
┌─────────────────────────────────────────────────────────────────┐
│                        API Server                                │
│                   (Initialization & Setup)                       │
└────────────────┬─────────────────────────────────────┬──────────┘
                 │                                     │
                 ↓                                     ↓
    ┌────────────────────┐                  ┌──────────────────┐
    │   CRD Lifecycle    │                  │  Client Libraries│
    │   (Controllers)    │                  │  (Generated Code)│
    └──────────┬─────────┘                  └──────────────────┘
               │
               ↓
    ┌──────────────────────┐
    │ Custom Resource      │
    │      Handler         │
    │ (crdHandler)         │
    └──────┬───────────────┘
           │
           ├─────────────┬─────────────┬──────────────┬─────────────┐
           ↓             ↓             ↓              ↓             ↓
    ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
    │ Schema   │  │   CEL    │  │ Version  │  │Discovery │  │ Storage  │
    │Validation│  │Validation│  │Conversion│  │& OpenAPI │  │  (etcd)  │
    └──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘
```

## Request Flow Through Components

### CRD Creation Flow
```
1. API Request → API Server (apiserver.go)
2. Validate CRD → Schema Validation
3. Store in etcd → Storage Layer
4. CRD Informer → Event to Controllers
5. Naming Controller → Validate names
6. NonStructuralSchema Controller → Check schema
7. API Approval Controller → Check approval
8. Custom Resource Handler → Create storage
9. Establishing Controller → Mark established
10. Discovery Controller → Update discovery
11. OpenAPI Controllers → Update specs
12. Custom resource endpoints → Available
```

### Custom Resource Create Flow
```
1. POST /apis/{group}/{version}/{resource}
2. Custom Resource Handler → Parse request
3. Schema Validation → Type/format/constraints
4. CEL Validation → Custom rules
5. Admission Webhooks → External validation
6. Version Conversion → To storage version
7. Store in etcd → Persist
8. Version Conversion → To requested version
9. Response → Return created object
```

## Cross-Component Integration Points

### Storage Management
- **Custom Resource Handler**: Creates and manages per-CRD storage
- **CRD Lifecycle**: Establishing controller signals when storage ready
- **Version Conversion**: Converts objects to storage version before persisting

### Validation Pipeline
- **Schema Validation**: First layer (types, formats, constraints)
- **CEL Validation**: Second layer (custom expression rules)
- **Admission Webhooks**: Third layer (external validation)
- Integration happens in Custom Resource Handler

### Discovery Synchronization
- **CRD Lifecycle**: Controllers update CRD status
- **Custom Resource Handler**: Creates REST endpoints
- **Discovery Controller**: Publishes API groups/versions/resources
- **OpenAPI Controllers**: Generate specifications

### Client Generation
- **Client Libraries**: Generated from API types
- **API Server**: Serves typed clients
- **Informers**: Watch CRD changes via client
- **Controllers**: Use informers for event processing

## Performance Critical Paths

### Hot Paths (Performance Critical)
1. **Custom Resource GET/LIST/WATCH**: Most common operations
   - Optimization: Atomic storage map for lock-free reads
2. **Discovery Lookups**: kubectl queries on every command
   - Optimization: Client-side caching, aggregated discovery
3. **CEL Validation**: Runs on every CR create/update
   - Optimization: Compiled programs cached per version
4. **Watch Connections**: Real-time event streams
   - Optimization: Efficient etcd watch integration

### Bottlenecks
1. **Webhook Calls**: Network latency for conversion/admission
2. **Storage Recreation**: Full rebuild on CRD spec changes
3. **Schema Validation**: Deep nesting can be expensive
4. **Watch Connection Count**: Scales with CR count × watchers

## Key Design Patterns

### 1. Controller Pattern
Independent reconciliation loops for each concern:
- Establishing, Naming, NonStructuralSchema, APIApproval, Finalizer, Discovery, OpenAPI

### 2. Atomic Storage Management
Lock-free reads via `atomic.Value`:
- ~10x faster than RWMutex for read-heavy workloads
- Rare writes only on CRD changes

### 3. Dynamic Handler Registration
Runtime endpoint creation without server restart:
- Atomic storage map updates
- Discovery synchronization
- Seamless CRD registration

### 4. Multi-Version Support
Storage version with conversion:
- Single canonical version in etcd
- Conversion on read/write
- Webhook-based transformation

### 5. Signal-Based Coordination
Prevents race conditions during startup:
- `hasCRDInformerSyncedSignal`: Readiness coordination
- `discoverySyncedCh`: Discovery synchronization

## Testing Strategy

Each component has comprehensive testing:
- **Unit Tests**: Individual component logic
- **Integration Tests**: Component interactions (~15,582 lines)
- **Table-Driven Tests**: Standard Kubernetes pattern
- **Fake Clients**: Testing without real API server

## Further Reading

- **Request Flow**: See [../request-flow.md](../request-flow.md) for detailed request processing
- **Data Flow**: See [../data-flow.md](../data-flow.md) for object lifecycle
- **Performance**: See [../performance.md](../performance.md) for optimization strategies
- **Architecture Overview**: See [../README.md](../README.md) for high-level architecture

## Contributing

When modifying core components:
1. Run component-specific tests: `go test ./pkg/apiserver/...`
2. Run integration tests: `go test ./test/integration/...`
3. Update relevant documentation in this directory
4. Follow table-driven test patterns
5. Consider performance impact on hot paths
