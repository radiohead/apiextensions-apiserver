# CustomResourceDefinition Lifecycle

The CRD lifecycle management system implements a sophisticated controller-based architecture that manages the complete lifecycle of CustomResourceDefinitions from creation through deletion. Seven independent controllers work in concert to validate, establish, and clean up CRDs.

## Overview

**Primary Location**: `pkg/controller/`
**API Types**: `pkg/apis/apiextensions/v1/types.go` (~1,500 lines)
**Confidence**: 90%

The lifecycle system uses the controller pattern - each controller manages a specific aspect of CRD state, updating status conditions to reflect the current state of the CRD.

## CRD API Structure

### Core Resource Type

**File**: `pkg/apis/apiextensions/v1/types.go:70-150`

```go
type CustomResourceDefinition struct {
    metav1.TypeMeta
    metav1.ObjectMeta

    Spec   CustomResourceDefinitionSpec
    Status CustomResourceDefinitionStatus
}

type CustomResourceDefinitionSpec struct {
    Group    string                           // API group (e.g., "mygroup.example.com")
    Names    CustomResourceDefinitionNames    // Resource names
    Scope    ResourceScope                    // Cluster or Namespaced
    Versions []CustomResourceDefinitionVersion // Multi-version support
    Conversion *CustomResourceConversion      // Version conversion strategy
    PreserveUnknownFields bool               // Schema pruning control (deprecated)
}
```

### Names Structure

```go
type CustomResourceDefinitionNames struct {
    Plural     string   // REST API endpoint name (required)
    Singular   string   // CLI display name
    Kind       string   // Type name in manifests (required)
    ListKind   string   // Collection type name
    ShortNames []string // CLI shortcuts
    Categories []string // CLI grouping (e.g., "all")
}
```

**Validation Rules**:
- Plural: lowercase, valid DNS subdomain
- Kind: PascalCase
- ListKind: Defaults to `{Kind}List`
- ShortNames: Must be unique cluster-wide

### Scope Types

```go
type ResourceScope string

const (
    ClusterScoped   ResourceScope = "Cluster"
    NamespaceScoped ResourceScope = "Namespaced"
)
```

**ClusterScoped**: Resources available cluster-wide (e.g., Node, PersistentVolume)
**NamespaceScoped**: Resources scoped to namespace (e.g., Pod, Service)

### Versions Array

```go
type CustomResourceDefinitionVersion struct {
    Name    string                      // Version name (e.g., "v1")
    Served  bool                        // Available via API
    Storage bool                        // Storage version (exactly one)
    Deprecated bool                     // Mark as deprecated
    DeprecationWarning string           // Warning message
    Schema  *CustomResourceValidation   // Per-version schema
    Subresources *CustomResourceSubresources // Status, scale
    AdditionalPrinterColumns []CustomResourceColumnDefinition // kubectl columns
}
```

**Multi-Version Rules**:
- At least one version must be served
- Exactly ONE version must be `storage: true`
- Deprecated versions continue to work

## Lifecycle Controllers

### 1. Establishing Controller

**Location**: `pkg/controller/establish/establishing_controller.go` (~300 lines)
**Purpose**: Mark CRDs as "Established" when ready to serve requests

#### Workflow

```
CRD Created/Updated
    ↓
Establishing Controller Notified
    ↓
Check if CRD storage is created and ready
    ↓
Apply HA-aware delay (if multi-master)
    ↓
Update "Established" condition to True
    ↓
CRD ready to accept custom resources
```

#### HA-Aware Delay

**File**: `pkg/controller/establish/establishing_controller.go:100-120`

```go
func (ec *EstablishingController) Reconcile(crd *apiextensionsv1.CustomResourceDefinition) error {
    if ec.masterCount > 1 {
        // Multi-master cluster: wait 5 seconds
        // Ensures all API servers have processed the CRD
        time.Sleep(5 * time.Second)
    }

    // Mark as established
    crd.Status.Conditions = append(crd.Status.Conditions,
        apiextensionsv1.CustomResourceDefinitionCondition{
            Type:   apiextensionsv1.Established,
            Status: apiextensionsv1.ConditionTrue,
            Reason: "InitialNamesAccepted",
        })
}
```

**Rationale**: In HA clusters, prevents race conditions where CRD is advertised before all API servers are ready.

#### Integration

- Tightly coupled with `crdHandler` storage creation
- Prevents premature API advertising in discovery
- Critical for zero-downtime CRD updates

### 2. Naming Controller

**Location**: `pkg/controller/status/naming_controller.go` (~200 lines)
**Purpose**: Validate CRD naming conventions and conflicts

#### Validations

**Format Checks**:
- Name format: `<plural>.<group>` (e.g., `widgets.example.com`)
- Group: Valid DNS subdomain
- Plural: Valid DNS label
- Singular: Valid DNS label
- Kind: Valid identifier

**Conflict Checks**:
- No naming conflicts with existing CRDs
- Plural names unique across all groups
- Short names unique cluster-wide
- Kind names unique within group

#### Workflow

```
CRD Created/Updated
    ↓
Naming Controller Validates
    ↓
Check Name Format
    ↓
Check for Conflicts
    ↓
Update "NamesAccepted" Condition
    ↓
If accepted: Update status.acceptedNames
```

#### Status Update

**File**: `pkg/controller/status/naming_controller.go:150-180`

```go
if namesAccepted {
    crd.Status.Conditions = updateCondition(crd.Status.Conditions,
        apiextensionsv1.CustomResourceDefinitionCondition{
            Type:   apiextensionsv1.NamesAccepted,
            Status: apiextensionsv1.ConditionTrue,
            Reason: "NoConflicts",
        })
    crd.Status.AcceptedNames = crd.Spec.Names
} else {
    crd.Status.Conditions = updateCondition(crd.Status.Conditions,
        apiextensionsv1.CustomResourceDefinitionCondition{
            Type:    apiextensionsv1.NamesAccepted,
            Status:  apiextensionsv1.ConditionFalse,
            Reason:  "NamingConflict",
            Message: conflictMessage,
        })
}
```

### 3. NonStructuralSchema Controller

**Location**: `pkg/controller/nonstructuralschema/nonstructuralschema_controller.go` (~200 lines)
**Purpose**: Enforce structural schema requirements

#### Structural Schema Requirements

A schema is "structural" if:
1. All values explicitly typed
2. No type ambiguity (no oneOf/anyOf/allOf)
3. Clear object properties and array items
4. No unresolved references

**Why Required**:
- CEL validation rules need clear types
- Pruning behavior requires structure
- Server-side apply needs field identification
- Defaulting requires predictable types

#### Workflow

```
CRD Created/Updated
    ↓
Extract Schema from Each Version
    ↓
Validate Structural Requirements
    ↓
If Non-Structural: Set Warning Condition
    ↓
Log Warning but Allow CRD
```

#### Condition Update

```go
if !isStructural(schema) {
    crd.Status.Conditions = updateCondition(crd.Status.Conditions,
        apiextensionsv1.CustomResourceDefinitionCondition{
            Type:    apiextensionsv1.NonStructuralSchema,
            Status:  apiextensionsv1.ConditionTrue,
            Reason:  "Violations",
            Message: "schema is not structural",
        })
}
```

**Impact**: Non-structural schemas work but lose advanced features (CEL, defaulting, SSA).

**See Also**: [schema-validation.md](./schema-validation.md) for structural schema details.

### 4. API Approval Controller

**Location**: `pkg/controller/apiapproval/api_approval_controller.go` (~300 lines)
**Purpose**: Enforce Kubernetes API approval process

#### Protected Namespaces

```go
const KubeAPIApprovedAnnotation = "api-approved.kubernetes.io"

var protectedGroups = []string{
    "*.k8s.io",
    "*.kubernetes.io",
}
```

**Policy**: CRDs in protected groups require explicit approval to prevent namespace squatting.

#### Approval Validation

**File**: `pkg/controller/apiapproval/api_approval_controller.go:120-160`

```go
func (c *APIApprovalController) checkApproval(crd *apiextensionsv1.CustomResourceDefinition) bool {
    // Check if group is protected
    if !isProtectedGroup(crd.Spec.Group) {
        return true // Not protected, no approval needed
    }

    // Check for approval annotation
    annotation, ok := crd.Annotations[KubeAPIApprovedAnnotation]
    if !ok {
        return false // Protected group without approval
    }

    // Validate annotation value
    if strings.HasPrefix(annotation, "unapproved") {
        return false // Explicitly unapproved
    }

    // Check if annotation is valid URL (PR or issue link)
    if _, err := url.Parse(annotation); err != nil {
        return false // Invalid approval reference
    }

    return true // Approved
}
```

#### Condition Update

```go
crd.Status.Conditions = updateCondition(crd.Status.Conditions,
    apiextensionsv1.CustomResourceDefinitionCondition{
        Type:   apiextensionsv1.KubernetesAPIApprovalPolicyConformant,
        Status: approvalStatus,
        Reason: reason,
        Message: message,
    })
```

### 5. Finalizer Controller

**Location**: `pkg/controller/finalizer/crd_finalizer.go` (~400 lines)
**Purpose**: Handle CRD deletion and cleanup

#### Finalizer

```go
const CustomResourceCleanupFinalizer = "customresourcecleanup.apiextensions.k8s.io"
```

Added to all CRDs to ensure cleanup before deletion.

#### Deletion Workflow

```
User Deletes CRD (kubectl delete crd ...)
    ↓
DeletionTimestamp Set
    ↓
Finalizer Controller Notified
    ↓
List All Custom Resource Instances
    ↓
Delete Each Instance
    ↓
Wait for All Instances Deleted
    ↓
Remove Finalizer
    ↓
CRD Fully Deleted
```

#### Implementation

**File**: `pkg/controller/finalizer/crd_finalizer.go:180-250`

```go
func (c *CRDFinalizer) finalizeCRD(crd *apiextensionsv1.CustomResourceDefinition) error {
    // Get custom resource storage
    storage, err := c.getStorageForCRD(crd)
    if err != nil {
        return err
    }

    // Delete all instances
    for _, version := range crd.Spec.Versions {
        if crd.Spec.Scope == apiextensionsv1.ClusterScoped {
            // Delete cluster-scoped resources
            err = storage.DeleteCollection(ctx, deleteOptions, listOptions)
        } else {
            // Delete namespaced resources in all namespaces
            namespaces, _ := c.namespaceClient.List(ctx, metav1.ListOptions{})
            for _, ns := range namespaces.Items {
                err = storage.DeleteCollection(ctx, deleteOptions,
                    listOptions.Namespace(ns.Name))
            }
        }
    }

    // Wait for deletion
    err = wait.PollImmediate(1*time.Second, 60*time.Second, func() (bool, error) {
        remaining, _ := storage.List(ctx, listOptions)
        return len(remaining.Items) == 0, nil
    })

    // Remove finalizer
    crd.Finalizers = removeString(crd.Finalizers, CustomResourceCleanupFinalizer)
    _, err = c.crdClient.Update(ctx, crd, metav1.UpdateOptions{})

    return err
}
```

#### Safety Mechanisms

- Prevents orphaned custom resources
- Ensures complete cleanup before CRD removal
- Handles both namespace-scoped and cluster-scoped resources
- Respects graceful deletion periods

### 6. Discovery Controller

**Location**: `pkg/apiserver/customresource_discovery_controller.go` (391 lines)
**Purpose**: Synchronize CRD changes to API discovery documents

**See**: [discovery-openapi.md](./discovery-openapi.md) for detailed coverage.

### 7. OpenAPI Controllers

**Location**: `pkg/controller/openapi/controller.go` and `pkg/controller/openapiv3/controller.go`
**Purpose**: Maintain OpenAPI v2/v3 specifications

**See**: [discovery-openapi.md](./discovery-openapi.md) for detailed coverage.

## CRD Status Management

### Status Structure

**File**: `pkg/apis/apiextensions/v1/types.go:200-220`

```go
type CustomResourceDefinitionStatus struct {
    Conditions        []CustomResourceDefinitionCondition
    AcceptedNames     CustomResourceDefinitionNames
    StoredVersions    []string
}

type CustomResourceDefinitionCondition struct {
    Type               CustomResourceDefinitionConditionType
    Status             ConditionStatus  // True, False, Unknown
    Reason             string
    Message            string
    LastTransitionTime metav1.Time
}
```

### Status Conditions

#### 1. Established

```go
Type:    Established
Status:  True/False
Reason:  "InitialNamesAccepted" | "EstablishingController"
Message: "CRD is ready to serve requests"
```

**Controller**: Establishing Controller
**Meaning**: CRD storage created and ready
**Required**: Yes, before serving custom resources

#### 2. NamesAccepted

```go
Type:    NamesAccepted
Status:  True/False
Reason:  "NoConflicts" | "NamingConflict"
Message: Details about conflict (if false)
```

**Controller**: Naming Controller
**Meaning**: Names are valid and conflict-free
**Required**: Yes, before establishing

#### 3. NonStructuralSchema

```go
Type:    NonStructuralSchema
Status:  True/False
Reason:  "Violations" | "Compliant"
Message: Description of structural violations
```

**Controller**: NonStructuralSchema Controller
**Meaning**: Warning about non-structural schema
**Required**: No, but affects features (CEL, defaulting)

#### 4. KubernetesAPIApprovalPolicyConformant

```go
Type:    KubernetesAPIApprovalPolicyConformant
Status:  True/False
Reason:  "ApprovedAnnotation" | "MissingAnnotation" | "UnapprovedAnnotation"
Message: Details about approval status
```

**Controller**: API Approval Controller
**Meaning**: Compliance with API approval policy
**Required**: Yes for `*.k8s.io` and `*.kubernetes.io` groups

#### 5. Terminating

```go
Type:    Terminating
Status:  True
Reason:  "InstanceDeletionInProgress"
Message: "custom resource instances are being deleted"
```

**Controller**: Finalizer Controller
**Meaning**: CRD deletion in progress
**Required**: Transient during deletion

### AcceptedNames

**Purpose**: Mirror of `spec.names` when validated

```go
status:
  acceptedNames:
    plural: widgets
    singular: widget
    kind: Widget
    listKind: WidgetList
    shortNames: [wg]
    categories: [all]
```

**Usage**: Clients use acceptedNames to determine active names, preventing confusion during naming conflicts.

### StoredVersions

**Purpose**: Track versions ever used for storage

```go
status:
  storedVersions:
  - v1beta1
  - v1
```

**Rules**:
- List of all versions that have been storage version
- Required for version migration planning
- Cannot be removed without migrating data
- Updated automatically by API server

**Migration**: To remove old version:
1. Change storage version to new version
2. Run migration job (read all, write back)
3. Remove old version from `storedVersions`
4. Remove old version from CRD spec

## Versioning Strategy

### Storage Version

**Rule**: Exactly ONE version marked `storage: true`

```yaml
versions:
- name: v1
  served: true
  storage: true    # ← Storage version
  schema: {...}
- name: v1beta1
  served: true
  storage: false   # Must convert to v1 for storage
  schema: {...}
```

**Implications**:
- All resources stored in storage version in etcd
- Reads/writes to other versions require conversion
- Changing storage version requires data migration

**See**: [version-conversion.md](./version-conversion.md) for conversion details.

### Version Priority

**Algorithm**: `pkg/apiserver/helpers/helpers.go:30-80`

```
Sorting Order:
1. GA versions (no suffix) - v10, v2, v1
2. Beta versions - v11beta2, v10beta3, v3beta1
3. Alpha versions - v12alpha1, v11alpha2
4. Non-Kubernetes versions - foo1, foo10

Within each tier:
- Major version descending
- Minor version descending
```

**Example**: `v10, v2, v1, v11beta2, v10beta3, v3beta1, v12alpha1, v11alpha2, foo1, foo10`

**Preferred Version**: First version in sorted order (typically highest GA version)

### Version Transitions

#### Adding Versions

```yaml
# Before
versions:
- name: v1
  served: true
  storage: true

# After - Add v2
versions:
- name: v2        # New version
  served: true
  storage: false  # Not storage yet
  schema: {...}
- name: v1
  served: true
  storage: true   # Still storage version
```

**Process**:
1. Add new version with `served: true, storage: false`
2. Deploy and validate
3. Optionally change storage version later

#### Deprecating Versions

```yaml
versions:
- name: v1beta1
  served: true
  storage: false
  deprecated: true
  deprecationWarning: "v1beta1 is deprecated; use v1"
- name: v1
  served: true
  storage: true
```

**Client Behavior**:
- API returns `Warning` header
- kubectl displays warning message
- Version continues to work

#### Removing Versions

**Prerequisites**:
1. Version not in `status.storedVersions`
2. All clients migrated to newer version
3. Deprecation period elapsed

**Process**:
1. Ensure version not storage version
2. Run migration if in storedVersions
3. Remove from CRD spec
4. Update clients

## CRD Registration Flow

### Storage Strategy

**Location**: `pkg/registry/customresourcedefinition/strategy.go` (~400 lines)

#### PrepareForCreate

```go
func (crdStrategy) PrepareForCreate(ctx context.Context, obj runtime.Object) {
    crd := obj.(*apiextensions.CustomResourceDefinition)

    // Clear status (user cannot set)
    crd.Status = apiextensions.CustomResourceDefinitionStatus{}

    // Set generation to 1
    crd.Generation = 1

    // Initialize StoredVersions from storage version
    for _, v := range crd.Spec.Versions {
        if v.Storage {
            crd.Status.StoredVersions = []string{v.Name}
            break
        }
    }

    // Drop disabled fields based on feature gates
    dropDisabledFields(crd)
}
```

#### PrepareForUpdate

```go
func (crdStrategy) PrepareForUpdate(ctx context.Context, obj, old runtime.Object) {
    newCRD := obj.(*apiextensions.CustomResourceDefinition)
    oldCRD := old.(*apiextensions.CustomResourceDefinition)

    // Preserve status from old object
    newCRD.Status = oldCRD.Status

    // Increment generation if spec changed
    if !apiequality.Semantic.DeepEqual(newCRD.Spec, oldCRD.Spec) {
        newCRD.Generation = oldCRD.Generation + 1
    }

    // Manage StoredVersions array
    updateStoredVersions(newCRD, oldCRD)

    // Drop disabled fields
    dropDisabledFields(newCRD)
}
```

### Validation Strategy

**Location**: `pkg/apis/apiextensions/validation/validation.go`

**Validations**:
- Structural schema validation
- CEL validation rule checks
- Webhook configuration validation
- Version compatibility checks
- Name format validation
- Required fields presence

## Feature Gates

### PreserveUnknownFields (Deprecated)

```yaml
spec:
  preserveUnknownFields: true  # Deprecated in v1
```

**Effect**: Disables field pruning globally
**Alternative**: Use `x-kubernetes-preserve-unknown-fields` in schema per field
**Status**: Deprecated, prefer schema-level control

## Critical Files Reference

| File | Purpose | Lines | Importance |
|------|---------|-------|------------|
| `pkg/apis/apiextensions/v1/types.go` | CRD API definitions | ~1,500 | Critical |
| `pkg/controller/establish/establishing_controller.go` | Establishing logic | ~300 | High |
| `pkg/controller/status/naming_controller.go` | Naming validation | ~200 | High |
| `pkg/controller/finalizer/crd_finalizer.go` | Deletion cleanup | ~400 | High |
| `pkg/controller/apiapproval/api_approval_controller.go` | API approval | ~300 | Medium |
| `pkg/controller/nonstructuralschema/nonstructuralschema_controller.go` | Schema validation | ~200 | Medium |
| `pkg/registry/customresourcedefinition/strategy.go` | Storage strategy | ~400 | High |

## Design Strengths

### 1. Controller Pattern
- Clean separation of concerns across controllers
- Independent reconciliation loops
- Standard Kubernetes approach
- Easy to test and maintain

### 2. Condition-Based Status
- Clear visibility into CRD state
- Machine-readable conditions
- Human-friendly messages
- Standard Kubernetes pattern

### 3. Multi-Version Support
- First-class support for API evolution
- Seamless version transitions
- Backward compatibility
- Storage version flexibility

### 4. Safety First
- Finalizers prevent data loss
- Validation before establishment
- Approval policy enforcement
- Graceful deletion

### 5. Policy Enforcement
- API approval prevents namespace conflicts
- Structural schema warnings
- Name conflict detection
- Cluster-wide uniqueness

## Design Considerations

### 1. StoredVersions Complexity
**Issue**: Manual migration burden for storage version changes

**Impact**: No automation for migrating stored data

**Mitigation**: Clear documentation, migration tools recommended

### 2. Finalizer Blocking
**Issue**: Large CR counts can block deletion significantly

**Example**: 10,000 CRs × 100ms each = ~17 minutes

**Mitigation**: Progress reporting, batch deletions

### 3. HA Delay
**Issue**: Hardcoded 5-second establishing delay

**Impact**: May be insufficient or excessive depending on cluster

**Recommendation**: Make configurable via flag

### 4. Naming Conflicts
**Issue**: Race conditions possible during concurrent CRD creates

**Scenario**: Two CRDs created simultaneously with conflicting names

**Mitigation**: Naming controller detects and rejects conflicts

## Testing

### Controller Tests
**Location**: `pkg/controller/*/controller_test.go`

**Coverage**:
- Individual controller logic
- Condition updates
- Edge cases and error handling

### Integration Tests
**Location**: `test/integration/`

**Coverage**:
- Full CRD lifecycle
- Multi-controller interaction
- Status transitions
- Version management

## Related Documentation

- **API Server**: [api-server.md](./api-server.md) - Controller integration
- **Custom Resource Handler**: [custom-resource-handler.md](./custom-resource-handler.md) - Storage creation
- **Discovery**: [discovery-openapi.md](./discovery-openapi.md) - Discovery controller
- **Schema Validation**: [schema-validation.md](./schema-validation.md) - Structural schemas
- **Version Conversion**: [version-conversion.md](./version-conversion.md) - Multi-version support

## Recommendations

### 1. StoredVersions Automation
Provide tooling for automated storage version migration:
- Migration job generator
- Progress tracking
- Rollback support

### 2. Deletion Progress
Add status field for cleanup progress during deletion:
```yaml
status:
  deletion:
    phase: "DeletingInstances"
    remaining: 5000
    total: 10000
```

### 3. HA Configuration
Make establishing delay configurable:
```yaml
--establishing-delay-seconds=5
```

### 4. Metrics
Add controller-specific metrics:
- Condition transition duration
- Reconciliation latency
- Error rates per controller
- Queue depth

### 5. Validation Enhancements
- Client-side validation in kubectl
- Dry-run support for validation
- Validation webhook integration
- Schema migration helpers
