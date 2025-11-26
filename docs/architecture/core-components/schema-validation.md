# Schema Validation System

The schema validation system implements a comprehensive JSON Schema v3-based validation framework with Kubernetes-specific extensions. It enforces "structural schemas" - a stricter schema format required for advanced features like CEL validation, defaulting, and server-side apply.

## Overview

**Primary Location**: `pkg/apiserver/schema/` (21 files)
**Key File**: `pkg/apiserver/schema/validation.go` (15,303 lines)
**Confidence**: 91%

The system validates all custom resource instances against their CRD schemas, applying multiple validation layers to ensure type safety, constraint compliance, and Kubernetes metadata correctness.

## Structural Schema Concept

### Definition

A **structural schema** is a schema that:
1. Specifies explicit types for all values
2. Has no type ambiguity (no polymorphism)
3. Properly specifies object properties
4. Defines clear array item schemas
5. Contains no oneOf/anyOf/allOf patterns (with exceptions)

### Why Structural?

**Required For**:
- CEL validation rules (need clear types for type checking)
- Defaulting behavior (need predictable types)
- Server-side apply / SSA (need field identification)
- Pruning of unknown fields (need schema boundaries)
- Efficient validation (optimization opportunities)

**Benefits**:
- Deterministic validation behavior
- Performance optimization via caching
- Clear semantics for tooling
- Type-safe CEL evaluation

### Structural vs Non-Structural

**Structural (Good)**:
```yaml
type: object
properties:
  name:
    type: string
  count:
    type: integer
    minimum: 0
```

**Non-Structural (Bad)**:
```yaml
# Missing types
properties:
  name: {}  # No type specified

# Polymorphic
oneOf:
- type: string
- type: integer
```

### Validation

**File**: `pkg/apiserver/schema/structural.go:50-200`

```go
func NewStructural(schema *JSONSchemaProps) (*Structural, error) {
    // Check all values have types
    // Check no conflicting types
    // Check proper property definitions
    // Check valid array item schemas
    // Check no unresolved references
}
```

## Validation Pipeline

### Complete Flow

```
Custom Resource Instance
    ↓
Parse JSON/YAML
    ↓
Type Validation (string, int, bool, array, object)
    ↓
Format Validation (date-time, email, ipv4, uuid, etc.)
    ↓
Constraint Validation (min/max, pattern, enum, required)
    ↓
Object Metadata Validation (Kubernetes-specific)
    ↓
List Type Validation (atomic, set, map)
    ↓
CEL Validation (custom expression rules)
    ↓
Admission Webhooks
    ↓
Store or Return Errors
```

### 1. Type Validation

**Supported Types**:
- `string`: Text values
- `integer`: Whole numbers (int32, int64 formats)
- `number`: Floating point (float, double formats)
- `boolean`: true/false
- `array`: Ordered lists
- `object`: Key-value maps
- `null`: Absence of value

**Example**:
```yaml
type: integer
format: int32  # Optional format specifier
```

**Validation**: Values must match declared type

### 2. Format Validation

**Standard Formats**:
- `date-time`: RFC3339 timestamps (e.g., "2023-01-15T10:30:00Z")
- `date`: Date only (e.g., "2023-01-15")
- `time`: Time only (e.g., "10:30:00")
- `duration`: Go duration strings (e.g., "1h30m")
- `email`: Email addresses
- `hostname`: DNS hostnames
- `ipv4`, `ipv6`: IP addresses
- `cidr`: CIDR notation (e.g., "10.0.0.0/8")
- `uri`, `uri-reference`: URLs
- `uuid`: UUID strings
- `byte`: Base64-encoded data

**Kubernetes Extensions**:
- `int-or-string`: IntOrString type (e.g., "80" or 80 for ports)
- `quantity`: Resource quantities (e.g., "100m", "2Gi")

### 3. Constraint Validation

**Numeric Constraints**:
```yaml
type: integer
minimum: 0
maximum: 100
exclusiveMinimum: true  # Value must be > minimum
multipleOf: 5           # Value must be multiple of 5
```

**String Constraints**:
```yaml
type: string
minLength: 1
maxLength: 253
pattern: "^[a-z0-9]([-a-z0-9]*[a-z0-9])?$"  # DNS label format
```

**Array Constraints**:
```yaml
type: array
minItems: 1
maxItems: 10
uniqueItems: true  # No duplicates
```

**Object Constraints**:
```yaml
type: object
required:
- name
- namespace
minProperties: 1
maxProperties: 10
additionalProperties: false  # No unknown properties
```

**Enum Validation**:
```yaml
type: string
enum:
- Active
- Pending
- Failed
```

### 4. Kubernetes-Specific Validation

#### x-kubernetes-embedded-resource

```yaml
type: object
x-kubernetes-embedded-resource: true
# Allows embedding full Kubernetes objects
# Adds apiVersion, kind, metadata fields
```

**Use Case**: Templates, pod specs within workloads

#### x-kubernetes-int-or-string

```yaml
x-kubernetes-int-or-string: true
# Special union type for ports, etc.
# Accepts: 80, "80", "http"
```

**Use Case**: Container ports, service targets

#### x-kubernetes-preserve-unknown-fields

```yaml
type: object
x-kubernetes-preserve-unknown-fields: true
# Disables pruning for this field and descendants
```

**Use Case**: Free-form configuration, legacy compatibility

#### x-kubernetes-list-type

```yaml
type: array
x-kubernetes-list-type: atomic  # or set, map
```

**Types**:
- `atomic`: Treat as single unit (full replacement)
- `set`: No duplicates, order not significant
- `map`: Items identified by keys (fine-grained updates)

**See**: [List Type Validation](#list-type-validation) section below

## Defaulting System

### Location

`pkg/apiserver/schema/defaulting/algorithm.go`

### Algorithm

**Features**:
1. Applies default values from schema
2. Respects schema structure
3. Recursive application through nested objects
4. Type-safe defaults
5. Conditional defaults based on schema

**Timing**: Applied BEFORE validation during Create/Update

### Default Value Types

**Scalar Defaults**:
```yaml
type: string
default: "default-value"
```

**Object Defaults**:
```yaml
type: object
default:
  key: value
  nested:
    field: data
```

**Array Defaults**:
```yaml
type: array
default: []
items:
  type: string
```

### Behavior

- Defaults applied on both Create and Update
- Only fills missing fields
- Does NOT override user-provided values (even if user provides empty/zero values)
- Applied recursively through schema tree

### Example

**Schema**:
```yaml
type: object
properties:
  replicas:
    type: integer
    default: 1
  strategy:
    type: string
    default: "RollingUpdate"
    enum: ["RollingUpdate", "Recreate"]
```

**Input**: `{}`
**After Defaulting**: `{replicas: 1, strategy: "RollingUpdate"}`

**Input**: `{replicas: 0}`
**After Defaulting**: `{replicas: 0, strategy: "RollingUpdate"}` (respects explicit 0)

## Pruning System

### Location

`pkg/apiserver/schema/pruning/prune.go`

### Purpose

Remove fields not specified in schema (unknown fields) to prevent:
- Confusion about what fields are valid
- Storage bloat from unused fields
- Security issues from hidden fields

### Pruning Modes

#### 1. Enabled (Default)

```yaml
# Schema
type: object
properties:
  name: { type: string }

# Input
{ "name": "test", "unknown": "field" }

# Output (pruned)
{ "name": "test" }
```

#### 2. Disabled Globally (Deprecated)

```yaml
spec:
  preserveUnknownFields: true
```

**Note**: Deprecated in v1, use per-field control instead

#### 3. Disabled Per-Field

```yaml
properties:
  metadata:
    type: object
    x-kubernetes-preserve-unknown-fields: true
```

### Pruning Algorithm

**Process**:
1. Walk object structure recursively
2. Compare against schema at each level
3. Remove fields not in schema properties
4. Preserve fields with `x-kubernetes-preserve-unknown-fields: true`
5. Always preserve: `metadata`, `apiVersion`, `kind`

### Special Cases

**Metadata**: Pruned according to objectmeta schema (strict)
**IntOrString**: Preserved as-is regardless of schema
**Embedded Resources**: Follow embedded schema if `x-kubernetes-embedded-resource: true`

## Object Metadata Validation

### Location

`pkg/apiserver/schema/objectmeta/validation.go`

### Purpose

Validate Kubernetes metadata fields according to Kubernetes conventions

### Validated Fields

**ObjectMeta Structure**:
```go
metadata:
  name:        string (DNS subdomain, max 253 chars)
  namespace:   string (DNS label, max 63 chars)
  labels:      map[string]string (qualified names)
  annotations: map[string]string (any string keys)
  finalizers:  []string (qualified names)
  ownerReferences: []OwnerReference
  generateName: string (DNS subdomain prefix)
```

### Validation Rules

**Name Validation**:
- DNS subdomain format (RFC 1123)
- Max 253 characters
- Lowercase alphanumeric, `-`, `.`
- Start and end with alphanumeric
- Regex: `^[a-z0-9]([-a-z0-9]*[a-z0-9])?(\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*$`

**Label Validation**:
- **Key**: optional prefix + name
  - Prefix: DNS subdomain format (e.g., `example.com/`)
  - Name: DNS label format (max 63 chars)
- **Value**: DNS label or empty (max 63 chars)
- Total label size limits enforced

**Annotation Validation**:
- Keys: Any string (best practice: use prefixes)
- Values: Any string
- Total size limits apply (256 KB total)

**Finalizer Validation**:
- Qualified names (with optional domain prefix)
- Used to prevent deletion until cleanup complete

### Metadata Pruning

**Allowed Top-Level Fields** (strictly enforced):
- Standard ObjectMeta fields only
- No unknown fields allowed
- Prevents confusion and ensures consistency

**Rationale**: Tight control over metadata prevents namespace pollution and security issues

## List Type Validation

### Location

`pkg/apiserver/schema/listtype/validation.go`

### List Type Overview

List types define merge semantics for server-side apply and updates.

### 1. Atomic Lists

```yaml
type: array
x-kubernetes-list-type: atomic
items:
  type: string
```

**Behavior**:
- Treated as single unit
- Full replacement on update (no granular merging)
- Order significant
- No uniqueness requirement

**Use Case**: Command arguments, environment variables

### 2. Set Lists

```yaml
type: array
x-kubernetes-list-type: set
items:
  type: string
uniqueItems: true
```

**Behavior**:
- No duplicates allowed (enforced)
- Order NOT significant
- Merge by value equality
- Items compared for uniqueness

**Use Case**: Tags, categories, unique identifiers

### 3. Map Lists

```yaml
type: array
x-kubernetes-list-type: map
x-kubernetes-list-map-keys:
- name
items:
  type: object
  properties:
    name: { type: string }
    value: { type: string }
  required: [name]
```

**Behavior**:
- Items identified by key field(s)
- Merge by key equality
- Enables fine-grained updates
- Key fields must be required

**Use Case**: Container lists, port lists, volume mounts

### Server-Side Apply Integration

List types critical for SSA field management:
- **Atomic**: Field manager owns entire list
- **Set**: Field manager owns individual items
- **Map**: Field manager owns items by key

## Validation Errors

### Error Structure

**Field Path**:
```
spec.template.spec.containers[0].ports[1].containerPort
```

Shows exact location of validation failure.

**Error Types**:
- `Required`: Missing required field
- `Invalid`: Value doesn't match schema
- `Forbidden`: Field not allowed
- `TooLong`: String/array exceeds max
- `TooShort`: Below minimum
- `TooMany`: Too many items/properties
- `Duplicate`: Duplicate in set
- `NotSupported`: Invalid enum value

### Example Validation Error

```go
field.Invalid(
    field.NewPath("spec", "replicas"),
    -1,
    "must be greater than or equal to 0"
)
```

**Output**:
```
spec.replicas: Invalid value: -1: must be greater than or equal to 0
```

## Performance Considerations

### Structural Schema Benefits

1. **Early Type Checking**: Fast-fail for invalid types
2. **No Recursive Resolution**: Schema fully resolved upfront
3. **Optimized Traversal**: Clear structure enables efficient walking
4. **Cached Compilation**: Structural form cached per CRD version

### Validation Caching

**CEL Compilation**: Compiled once at CRD registration, reused for all instances
**Schema Parsing**: Parsed at CRD registration, cached
**ObjectMeta Schema**: Shared across all CRDs

### Performance Profile

**Schema Validation**: O(n) in object size
**Deep Nesting**: Can be expensive, recommend max depth of 10-15 levels
**Large Arrays**: O(n*m) where n = items, m = item complexity

## Critical Files Reference

| File | Purpose | Lines | Importance |
|------|---------|-------|------------|
| `pkg/apiserver/schema/structural.go` | Structural schema core | ~700 | Critical |
| `pkg/apiserver/schema/validation.go` | Validation logic | 15,303 | Critical |
| `pkg/apiserver/schema/defaulting/algorithm.go` | Defaulting | ~300 | High |
| `pkg/apiserver/schema/pruning/prune.go` | Pruning | ~200 | High |
| `pkg/apiserver/schema/objectmeta/validation.go` | Metadata validation | ~400 | High |
| `pkg/apiserver/schema/listtype/validation.go` | List types | ~200 | Medium |

## Design Strengths

### 1. Structural Requirement
Enables optimization and clear semantics:
- Type-safe CEL evaluation
- Predictable defaulting
- Efficient validation

### 2. Comprehensive Validation
Multiple layers catch errors early:
- Type mismatches
- Constraint violations
- Metadata errors
- Format issues

### 3. Defaulting Integration
Seamless default value application:
- Reduces boilerplate in user input
- Ensures complete objects
- Type-safe defaults

### 4. Pruning Safety
Prevents confusion from unknown fields:
- Clear schema boundaries
- Security benefits
- Storage efficiency

### 5. Metadata Enforcement
Ensures Kubernetes compliance:
- Name format validation
- Label/annotation rules
- Finalizer management

## Design Considerations

### 1. Complexity
**Issue**: Structural schema rules can be non-intuitive

**Example**: oneOf restrictions, type requirements

**Mitigation**: Clear error messages, documentation, examples

### 2. Error Messages
**Issue**: Validation errors can be verbose with deep nesting

**Example**: `spec.template.spec.containers[0].env[5].valueFrom.configMapKeyRef.name: Required value`

**Mitigation**: Path information helps locate issues

### 3. Migration
**Issue**: Converting non-structural to structural schemas

**Challenge**: Existing CRDs may have non-structural schemas

**Recommendation**: Provide migration tooling and guides

### 4. Performance
**Issue**: Deep nesting can slow validation

**Impact**: Validation time increases with object complexity

**Mitigation**: Recommend schema depth limits, caching

## Integration Points

### Dependencies
- OpenAPI v3 spec parser for schema definitions
- CEL validation (structural schemas required)
- Admission webhooks for external validation

### Consumers
- CRD handler for CR validation (every create/update)
- kubectl for client-side validation (optional)
- API server for schema storage and compilation

## Testing

### Unit Tests
**Location**: `pkg/apiserver/schema/*_test.go`

**Coverage**:
- Structural schema validation
- Validation pipeline
- Defaulting algorithm
- Pruning behavior

### Integration Tests
**Location**: `test/integration/`

**Coverage**:
- End-to-end CR validation
- Schema evolution scenarios
- Error message accuracy

## Related Documentation

- **CRD Lifecycle**: [crd-lifecycle.md](./crd-lifecycle.md) - NonStructuralSchema controller
- **CEL Validation**: [cel-validation.md](./cel-validation.md) - CEL requires structural schemas
- **Custom Resource Handler**: [custom-resource-handler.md](./custom-resource-handler.md) - Validation in request flow

## Recommendations

### 1. Documentation
Improve structural schema error messages:
- Explain what makes schema non-structural
- Provide concrete examples
- Suggest fixes

### 2. Tooling
Provide schema conversion tools:
- Non-structural → structural converter
- Schema validator CLI
- Migration assistant

### 3. Validation Levels
Add strict/permissive modes:
- Strict: Reject non-structural schemas
- Permissive: Warn but allow (current behavior)

### 4. Caching
Enhance validation result caching:
- Cache results for identical objects
- LRU cache for recent validations
- Reduce redundant validation work

### 5. Performance Monitoring
Add validation metrics:
- Validation latency by CRD
- Schema complexity metrics
- Cache hit rates
- Error rates by type
