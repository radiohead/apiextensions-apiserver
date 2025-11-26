# Data Flows

This document describes the major data flows through the apiextensions-apiserver, showing how requests are processed and how different components interact. Understanding these flows is essential for debugging issues and optimizing performance.

## Overview

The apiextensions-apiserver handles three primary data flows:

1. **CRD Creation Flow** - Registering new custom resource types
2. **Custom Resource Create Flow** - Creating instances of custom resources
3. **Custom Resource Watch Flow** - Streaming updates to clients

Each flow involves multiple components coordinating to provide Kubernetes API semantics while supporting dynamic resource types.

## Flow 1: CRD Creation Flow

### Overview

When a CRD is created, the system must validate it, create storage infrastructure, update discovery, and mark it as ready to serve requests. This involves 12 coordinated steps across multiple controllers.

### Complete Flow Diagram

```
┌──────────────────────────────────────────────────────────────┐
│ 1. POST /apis/apiextensions.k8s.io/v1/customresourcedefinitions │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 2. Validate CRD                                               │
│    - Name format: <plural>.<group>                            │
│    - Schema structural requirements                           │
│    - Version configuration                                    │
│    - Conversion webhook config                                │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 3. Store in etcd                                              │
│    Key: /customresourcedefinitions/<name>                     │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 4. CRD Informer receives Add event                            │
│    - Local cache updated                                      │
│    - All controllers notified                                 │
└──────────────────────────────────────────────────────────────┘
                          ↓
        ┌────────────────┴────────────────┐
        ↓                                   ↓
┌────────────────────┐          ┌────────────────────┐
│ 5a. Naming         │          │ 5b. APIApproval    │
│     Controller     │          │     Controller     │
│                    │          │                    │
│ - Validate name    │          │ - Check *.k8s.io   │
│ - Check conflicts  │          │ - Verify approval  │
│ - Set condition:   │          │ - Set condition:   │
│   NamesAccepted    │          │   Conformant=True  │
│   = True           │          │                    │
└────────────────────┘          └────────────────────┘
        ↓                                   ↓
        └────────────────┬────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 6. NonStructuralSchema Controller                             │
│    - Validate schema is structural                            │
│    - Set condition: NonStructuralSchema = False               │
│    (or True with warning if non-structural)                   │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 7. Finalizer Controller                                       │
│    - Add finalizer: customresourcecleanup.apiextensions.k8s.io│
│    (ensures CR cleanup before CRD deletion)                   │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 8. crdHandler creates storage for each version               │
│                                                               │
│    For each version in spec.versions:                         │
│    ┌────────────────────────────────────────────────────┐   │
│    │ - Build CustomResourceStorage                       │   │
│    │ - Create RequestScope (validation, conversion)      │   │
│    │ - Build status/scale subresource scopes            │   │
│    │ - Compile CEL validation rules                      │   │
│    └────────────────────────────────────────────────────┘   │
│                                                               │
│    crdInfo struct created:                                    │
│    {                                                          │
│      storages: {                                              │
│        "v1": storage_v1,                                      │
│        "v1beta1": storage_v1beta1                             │
│      },                                                       │
│      requestScopes: {...},                                    │
│      storageVersion: "v1"                                     │
│    }                                                          │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 9. Atomic storage map update                                 │
│                                                               │
│    oldMap := customStorage.Load()                             │
│    newMap := clone(oldMap)                                    │
│    newMap[crd.UID] = crdInfo                                  │
│    customStorage.Store(newMap)  // Atomic!                    │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 10. Establishing Controller                                   │
│     - Verify storage created                                  │
│     - Wait 5 seconds (HA clusters for propagation)            │
│     - Set condition: Established = True                       │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 11. Discovery Controller updates discovery documents         │
│                                                               │
│     Updates:                                                  │
│     - /apis (add group if new)                                │
│     - /apis/{group} (add/update versions)                     │
│     - /apis/{group}/{version} (add resource)                  │
│     - Aggregated discovery (bulk update)                      │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 12. OpenAPI Controllers update specs                         │
│                                                               │
│     OpenAPI v2 Controller:                                    │
│     - Convert CRD schema to OpenAPI v2                        │
│     - Update /openapi/v2 (global spec)                        │
│                                                               │
│     OpenAPI v3 Controller:                                    │
│     - Convert CRD schema to OpenAPI v3                        │
│     - Update /openapi/v3/apis/{group}/{version}               │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ ✓ Custom resource endpoints now available:                   │
│   /apis/{group}/{version}/namespaces/{ns}/{resource}         │
│   /apis/{group}/{version}/{resource} (cluster-scoped)        │
│                                                               │
│   kubectl can now:                                            │
│   - kubectl get {resource}                                    │
│   - kubectl create -f {resource}.yaml                         │
│   - kubectl explain {resource}.spec                           │
└──────────────────────────────────────────────────────────────┘
```

### Key Timing Characteristics

| Step | Typical Latency | Notes |
|------|----------------|-------|
| 1-3: API request | < 100ms | Validation and etcd write |
| 4: Informer update | < 100ms | Watch event propagation |
| 5-7: Controller updates | 1-5 seconds | Async reconciliation |
| 8-9: Storage creation | < 500ms | In-memory structure |
| 10: Establishing | 5+ seconds | HA clusters (intentional delay) |
| 11-12: Discovery/OpenAPI | 1-2 seconds | Document regeneration |
| **Total** | **7-10 seconds** | CRD fully ready |

### Failure Scenarios

**Schema Validation Failure**
```
Step 2: Validate CRD
    ↓
Schema is not structural
    ↓
NonStructuralSchema condition = True (Warning)
    ↓
CRD created but CEL validation unavailable
```

**API Approval Failure**
```
Step 5b: APIApproval Controller
    ↓
Group is *.k8s.io but no approval annotation
    ↓
Conformant condition = False
    ↓
CRD created but flagged as non-conformant
```

**Storage Creation Failure**
```
Step 8: crdHandler creates storage
    ↓
CEL compilation fails (invalid rule)
    ↓
Storage not created
    ↓
Established condition = False
    ↓
Endpoints not available
```

### Controller Coordination

Controllers run **independently** but coordinate via **status conditions**:

```go
// Controllers check conditions before proceeding
func (c *DiscoveryController) Reconcile(crd *CRD) error {
    // Wait for CRD to be established
    if !IsEstablished(crd) {
        return nil  // Requeue, check later
    }

    // Now update discovery
    c.updateDiscovery(crd)
    return nil
}
```

This **eventually consistent** model means:
- Controllers may run in any order
- Some conditions may be set before others
- Final state is consistent but intermediate states vary

### Code References

- `pkg/apiextensions/apiserver/apiserver.go:150-200` - CRD API installation
- `pkg/controller/establish/establishing_controller.go:82-150` - Establishing logic
- `pkg/apiserver/customresource_handler.go:188-217` - Storage creation
- `pkg/apiserver/customresource_discovery_controller.go:90-180` - Discovery updates

## Flow 2: Custom Resource Create Flow

### Overview

Creating a custom resource involves validation (schema, CEL, admission), conversion, and storage. This is the **hot path** for custom resource operations.

### Complete Flow Diagram

```
┌──────────────────────────────────────────────────────────────┐
│ 1. POST /apis/mygroup.io/v1/namespaces/default/myresources  │
│    Content-Type: application/json                            │
│    Body: { "apiVersion": "mygroup.io/v1", ... }              │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 2. crdHandler receives request                               │
│    - Parse URL: group=mygroup.io, version=v1, resource=...   │
│    - Lookup storage map: customStorage.Load()                │
│    - Find crdInfo for this CRD                                │
│    - Select RequestScope for version                          │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 3. Parse request body                                         │
│    JSON/YAML → unstructured.Unstructured                      │
│                                                               │
│    {                                                          │
│      "apiVersion": "mygroup.io/v1",                           │
│      "kind": "MyResource",                                    │
│      "metadata": {"name": "example"},                         │
│      "spec": {"replicas": 3}                                  │
│    }                                                          │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 4. Apply defaulting from schema                              │
│    (pkg/apiserver/schema/defaulting/)                         │
│                                                               │
│    Schema defines:                                            │
│      spec.replicas: {type: integer, default: 1}               │
│                                                               │
│    If field missing → add default value                       │
│    Example: spec.port defaults to 8080                        │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 5. Run mutating admission webhooks                           │
│    (k8s.io/apiserver admission chain)                         │
│                                                               │
│    Webhooks can:                                              │
│    - Add labels/annotations                                   │
│    - Modify spec fields                                       │
│    - Inject sidecars                                          │
│                                                               │
│    MutatingWebhookConfiguration → AdmissionReview             │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 6. Schema validation                                          │
│    (pkg/apiserver/schema/validation.go - 15,303 lines!)      │
│                                                               │
│    Validation layers:                                         │
│    ┌────────────────────────────────────────────────────┐   │
│    │ 6a. Type validation                                 │   │
│    │     - Check field types (string, int, bool, etc.)   │   │
│    │     - Verify array items                            │   │
│    │     - Validate object properties                    │   │
│    └────────────────────────────────────────────────────┘   │
│    ┌────────────────────────────────────────────────────┐   │
│    │ 6b. Format validation                               │   │
│    │     - date-time, email, hostname, ipv4, ipv6        │   │
│    │     - uuid, uri, byte (base64)                      │   │
│    └────────────────────────────────────────────────────┘   │
│    ┌────────────────────────────────────────────────────┐   │
│    │ 6c. Constraint validation                           │   │
│    │     - minimum, maximum, exclusiveMin/Max            │   │
│    │     - minLength, maxLength, pattern (regex)         │   │
│    │     - minItems, maxItems, uniqueItems               │   │
│    │     - minProperties, maxProperties                  │   │
│    │     - enum, required fields                         │   │
│    └────────────────────────────────────────────────────┘   │
│    ┌────────────────────────────────────────────────────┐   │
│    │ 6d. Object metadata validation                      │   │
│    │     - Kubernetes-specific rules                     │   │
│    │     - Name format, label syntax, annotation size    │   │
│    └────────────────────────────────────────────────────┘   │
│    ┌────────────────────────────────────────────────────┐   │
│    │ 6e. List type validation                            │   │
│    │     - x-kubernetes-list-type: atomic/set/map        │   │
│    │     - Uniqueness for set/map types                  │   │
│    └────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 7. CEL validation rules                                      │
│    (pkg/apiserver/schema/cel/validation.go)                  │
│                                                               │
│    For each x-kubernetes-validations rule:                    │
│    ┌────────────────────────────────────────────────────┐   │
│    │ - Bind 'self' variable to field value              │   │
│    │ - Execute compiled CEL program                      │   │
│    │ - Track cost (max 1M units)                         │   │
│    │ - Evaluate message/messageExpression if fails       │   │
│    └────────────────────────────────────────────────────┘   │
│                                                               │
│    Example rule:                                              │
│      rule: "self.replicas >= 1 && self.replicas <= 100"      │
│      → Evaluates to true/false                                │
│                                                               │
│    All rules must pass                                        │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 8. Run validating admission webhooks                         │
│    (k8s.io/apiserver admission chain)                         │
│                                                               │
│    Webhooks can:                                              │
│    - Reject invalid resources                                │
│    - Enforce policies                                         │
│    - Check external dependencies                             │
│                                                               │
│    ValidatingWebhookConfiguration → AdmissionReview           │
│    Response: allowed=true/false + message                     │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 9. Convert to storage version (if needed)                    │
│    (pkg/apiserver/conversion/)                                │
│                                                               │
│    If request version ≠ storage version:                      │
│    ┌────────────────────────────────────────────────────┐   │
│    │ Strategy: None                                      │   │
│    │   → Just update apiVersion field                    │   │
│    └────────────────────────────────────────────────────┘   │
│    ┌────────────────────────────────────────────────────┐   │
│    │ Strategy: Webhook                                   │   │
│    │   → POST to webhook service                         │   │
│    │   → ConversionReview request/response               │   │
│    │   → Latency: +10-50ms                               │   │
│    └────────────────────────────────────────────────────┘   │
│                                                               │
│    Result: Resource in storage version (e.g., v1)             │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 10. Store in etcd                                            │
│     (pkg/registry/customresource/storage.go)                 │
│                                                               │
│     Key: /registry/{group}/{resource}/{namespace}/{name}     │
│     Value: Serialized resource (storage version)             │
│                                                               │
│     - Atomic create operation                                │
│     - Conflict if name exists                                │
│     - ResourceVersion assigned                               │
│     - CreationTimestamp set                                  │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 11. Convert to requested version (if different)              │
│                                                               │
│     If storage version ≠ request version:                     │
│       storage v1 → request v1beta1                            │
│                                                               │
│     Same conversion logic as step 9                           │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 12. Return HTTP 201 Created response                         │
│     Content-Type: application/json                           │
│                                                               │
│     Response body: Created resource in requested version     │
│     {                                                         │
│       "apiVersion": "mygroup.io/v1",                          │
│       "kind": "MyResource",                                   │
│       "metadata": {                                           │
│         "name": "example",                                    │
│         "resourceVersion": "12345",                           │
│         "creationTimestamp": "2025-11-26T..."                 │
│       },                                                      │
│       "spec": {"replicas": 3}                                 │
│     }                                                         │
└──────────────────────────────────────────────────────────────┘
```

### Performance Profile

| Stage | Typical Latency | Notes |
|-------|----------------|-------|
| 1-3: Parse | < 1ms | JSON unmarshalling |
| 4: Defaulting | < 1ms | Schema-based defaults |
| 5: Mutating webhooks | +10-50ms | If configured |
| 6: Schema validation | 1-5ms | Depends on complexity |
| 7: CEL validation | 1-10ms | Depends on rules |
| 8: Validating webhooks | +10-50ms | If configured |
| 9: Conversion | +10-50ms | If version differs + webhook |
| 10: etcd write | 5-20ms | Network + disk |
| 11: Conversion back | +10-50ms | If version differs + webhook |
| **Total (no webhooks)** | **10-40ms** | Typical case |
| **Total (with webhooks)** | **50-200ms** | Includes webhook calls |

### Validation Failure Example

```
Step 7: CEL Validation
    ↓
Rule: "self.replicas >= 1 && self.replicas <= 100"
Value: self.replicas = 0
    ↓
Validation fails
    ↓
HTTP 422 Unprocessable Entity
{
  "kind": "Status",
  "status": "Failure",
  "message": "replicas must be between 1 and 100",
  "reason": "Invalid",
  "code": 422
}
```

### Code References

- `pkg/apiserver/customresource_handler.go:450-650` - Request handling
- `pkg/apiserver/schema/validation.go` - Schema validation
- `pkg/apiserver/schema/cel/validation.go:200-400` - CEL evaluation
- `pkg/registry/customresource/storage.go:150-300` - Storage layer

## Flow 3: Custom Resource Watch Flow

### Overview

Watch operations establish long-lived HTTP connections to stream resource changes. This enables reactive controllers and real-time updates.

### Complete Flow Diagram

```
┌──────────────────────────────────────────────────────────────┐
│ 1. GET /apis/mygroup.io/v1/namespaces/default/myresources?  │
│        watch=true&resourceVersion=12345                       │
│                                                               │
│    HTTP/1.1 connection (long-lived)                           │
│    Accept: application/json                                   │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 2. crdHandler establishes watch on etcd                      │
│    (via storage layer)                                        │
│                                                               │
│    Watch parameters:                                          │
│    - Key prefix: /registry/mygroup.io/myresources/default/   │
│    - ResourceVersion: 12345 (start from this version)         │
│    - Filters: label selectors, field selectors               │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 3. etcd watch stream established                             │
│                                                               │
│    etcd sends events:                                         │
│    - PUT: Resource created/updated                            │
│    - DELETE: Resource deleted                                 │
│                                                               │
│    Each event includes:                                       │
│    - Resource in storage version                              │
│    - ResourceVersion (etcd revision)                          │
└──────────────────────────────────────────────────────────────┘
                          ↓
                ┌─────────────────┐
                │  For each event  │
                └─────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 4. Read resource from event                                  │
│                                                               │
│    Event type: PUT                                            │
│    Resource in storage version (v1)                           │
│    {                                                          │
│      "apiVersion": "mygroup.io/v1",                           │
│      "kind": "MyResource",                                    │
│      "metadata": {                                            │
│        "resourceVersion": "12346"                             │
│      },                                                       │
│      "spec": {"replicas": 5}                                  │
│    }                                                          │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 5. Convert to watch request version                          │
│    (if storage version ≠ watch version)                       │
│                                                               │
│    Example: Client watching v1beta1                           │
│      Storage v1 → Watch v1beta1                               │
│                                                               │
│    Conversion via:                                            │
│    - Strategy: None → Simple apiVersion change                │
│    - Strategy: Webhook → Call conversion webhook              │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 6. Determine watch event type                                │
│                                                               │
│    etcd event → Kubernetes watch event:                       │
│    ┌────────────────────────────────────────────────────┐   │
│    │ PUT (new) → ADDED                                   │   │
│    │   (resource didn't exist before)                    │   │
│    └────────────────────────────────────────────────────┘   │
│    ┌────────────────────────────────────────────────────┐   │
│    │ PUT (existing) → MODIFIED                           │   │
│    │   (resource existed, now updated)                   │   │
│    └────────────────────────────────────────────────────┘   │
│    ┌────────────────────────────────────────────────────┐   │
│    │ DELETE → DELETED                                    │   │
│    │   (resource removed)                                │   │
│    └────────────────────────────────────────────────────┘   │
│    ┌────────────────────────────────────────────────────┐   │
│    │ BOOKMARK (periodic)                                 │   │
│    │   (checkpoint with current resourceVersion)         │   │
│    └────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 7. Send WatchEvent to client                                │
│    (JSON-encoded, newline-delimited)                         │
│                                                               │
│    {                                                          │
│      "type": "MODIFIED",                                      │
│      "object": {                                              │
│        "apiVersion": "mygroup.io/v1",                         │
│        "kind": "MyResource",                                  │
│        "metadata": {                                          │
│          "name": "example",                                   │
│          "resourceVersion": "12346"                           │
│        },                                                     │
│        "spec": {"replicas": 5}                                │
│      }                                                        │
│    }                                                          │
│    \n  ← Newline separator for next event                     │
└──────────────────────────────────────────────────────────────┘
                          ↓
                ┌─────────────────┐
                │  Repeat for      │
                │  more events     │
                └─────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 8. Client receives event stream                             │
│    (typically via client-go watch interface)                 │
│                                                               │
│    Client code:                                               │
│    watcher, _ := client.Watch(ctx, metav1.ListOptions{       │
│      Watch: true,                                             │
│      ResourceVersion: "12345",                                │
│    })                                                         │
│                                                               │
│    for event := range watcher.ResultChan() {                  │
│      switch event.Type {                                      │
│      case watch.Added:                                        │
│        // Handle new resource                                 │
│      case watch.Modified:                                     │
│        // Handle update                                       │
│      case watch.Deleted:                                      │
│        // Handle deletion                                     │
│      }                                                        │
│    }                                                          │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 9. Informer updates local cache                             │
│    (if using SharedInformerFactory)                          │
│                                                               │
│    Informer cache synchronized:                               │
│    - Add event → Add to cache                                 │
│    - Modified → Update in cache                               │
│    - Deleted → Remove from cache                              │
│                                                               │
│    Cache provides fast local lookups (no API calls)           │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│ 10. Event handlers invoked                                   │
│     (registered via informer.AddEventHandler)                │
│                                                               │
│     informer.AddEventHandler(cache.ResourceEventHandlerFuncs{│
│       AddFunc: func(obj interface{}) {                        │
│         // Handle resource addition                           │
│       },                                                      │
│       UpdateFunc: func(old, new interface{}) {                │
│         // Handle resource update                             │
│       },                                                      │
│       DeleteFunc: func(obj interface{}) {                     │
│         // Handle resource deletion                           │
│       },                                                      │
│     })                                                        │
│                                                               │
│     Controllers reconcile based on events                     │
└──────────────────────────────────────────────────────────────┘
```

### Watch Lifecycle

```
Connection Established
    ↓
Events stream continuously
    ↓
    ├─→ ADDED events
    ├─→ MODIFIED events
    ├─→ DELETED events
    └─→ BOOKMARK events (periodic checkpoints)
    ↓
Connection closes on:
    - Client disconnection
    - Server shutdown
    - Watch timeout
    - Error condition
```

### ResourceVersion Semantics

```yaml
# Watch from specific version
?watch=true&resourceVersion=12345
  → Events starting after 12345

# Watch from current state
?watch=true&resourceVersion=0
  → Initial LIST of current state, then watch

# Watch from most recent
?watch=true&resourceVersion=""
  → Watch from now, no historical events
```

### Performance Characteristics

| Aspect | Typical Value | Notes |
|--------|--------------|-------|
| Event latency | < 100ms | etcd → client propagation |
| Conversion overhead | +10-50ms | Per event if version differs |
| Bookmark interval | ~60 seconds | Periodic checkpoints |
| Max watch duration | Hours/days | Until connection closed |
| Concurrent watches | 1000s | Per API server instance |

### Failure Handling

**Watch Disconnection**
```
etcd connection lost
    ↓
Watch stream closes with error
    ↓
Client receives error event
    ↓
Client re-establishes watch
    ↓
Specify last seen resourceVersion
    ↓
Resume from last known state
```

**Too Old ResourceVersion**
```
Client requests resourceVersion=1000
    ↓
etcd compaction cleaned up history
    ↓
Server returns: "too old resource version"
    ↓
Client performs full LIST
    ↓
Establish watch from current version
```

### Code References

- `pkg/apiserver/customresource_handler.go:650-800` - Watch handling
- `pkg/registry/customresource/storage.go:300-450` - Storage watch
- `k8s.io/apiserver/pkg/storage/cacher/cacher.go` - Watch caching

## Cross-Flow Interactions

### CRD Update Affects CR Operations

```
CRD Updated (new validation rule)
    ↓
crdHandler receives informer event
    ↓
Rebuild storage with new schema
    ↓
Atomic swap storage map
    ↓
New CR requests use updated validation
    ↓
Existing watch connections unaffected
```

### Watch During CRD Deletion

```
CRD Deletion Started
    ↓
Finalizer controller deletes all CRs
    ↓
Watch clients receive DELETED events
    ↓
CRD storage removed from map
    ↓
New requests → 404 Not Found
    ↓
Active watches closed gracefully
```

## Performance Optimization Strategies

### Hot Path Optimization (CR Create)

1. **Lock-free storage lookup** - atomic.Value for zero contention
2. **CEL compilation caching** - compile once, evaluate many times
3. **Schema validation optimization** - early returns on type mismatches
4. **Admission webhook caching** - cache webhook configurations

### Watch Scaling

1. **Watch caching** - Shared cache for multiple watch clients
2. **Bookmark events** - Reduce reconnection overhead
3. **Label/field selector filtering** - Server-side event filtering
4. **Connection pooling** - Reuse etcd watch connections

### Discovery Optimization

1. **Aggregated discovery** - Single request for all resources
2. **Client-side caching** - 10 minute TTL (kubectl default)
3. **Atomic updates** - Consistent discovery snapshots
4. **Lazy OpenAPI generation** - Generate on first request

## Summary

Understanding these three data flows is essential for:

1. **Debugging** - Trace requests through the system
2. **Performance tuning** - Identify bottlenecks in flows
3. **Feature development** - Know where to add new functionality
4. **Troubleshooting** - Understand failure modes

Key takeaways:

- **CRD creation** involves 12 coordinated steps with eventual consistency
- **CR create** goes through 5 validation layers before storage
- **Watch** establishes long-lived connections with event streaming
- **Conversion** may add latency when versions differ
- **Controllers** coordinate via status conditions, not direct dependencies

These flows demonstrate how the architectural patterns work together to provide reliable, performant, and extensible custom resource management.
