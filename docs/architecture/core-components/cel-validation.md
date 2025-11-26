# CEL Validation Integration

The Common Expression Language (CEL) integration provides powerful, expressive validation rules for CustomResourceDefinitions without requiring external webhooks. The system compiles CEL expressions at CRD registration time, caches compiled programs, enforces cost limits, and supports transition rules with access to both current (`self`) and previous (`oldSelf`) values.

## Overview

**Primary Location**: `pkg/apiserver/schema/cel/` (12 files, ~400KB code+tests)
**Key Files**:
- `compilation.go` (361 lines): CEL compilation and caching
- `validation.go` (994 lines): Expression evaluation
**Confidence**: 94%

CEL enables sophisticated validation logic directly in CRD schemas, providing type-safe, cost-limited validation without the complexity and latency of admission webhooks.

## CEL Validation Rules

### Rule Definition in CRD

```yaml
openAPIV3Schema:
  type: object
  properties:
    spec:
      type: object
      properties:
        replicas:
          type: integer
          x-kubernetes-validations:
          - rule: "self >= 0"
            message: "replicas must be non-negative"
          - rule: "self <= 100"
            message: "replicas must not exceed 100"
          - rule: "!has(oldSelf) || self >= oldSelf"
            message: "replicas cannot decrease"
            optionalOldSelf: true
```

### Rule Structure

```go
type ValidationRule struct {
    Rule              string  // CEL expression (must evaluate to bool)
    Message           string  // Static error message
    MessageExpression string  // CEL expression for dynamic messages
    Reason            string  // Machine-readable failure reason
    FieldPath         string  // Relative field path for error reporting
    OptionalOldSelf   *bool   // Whether oldSelf is optional
}
```

**Fields**:
- **Rule**: CEL boolean expression that must evaluate to true
- **Message**: Static error message shown on failure
- **MessageExpression**: Dynamic message using CEL (evaluated only on failure)
- **Reason**: Machine-readable reason code
- **FieldPath**: Adjust error location (e.g., `.name` points to subfield)
- **OptionalOldSelf**: Allow oldSelf to be absent (for new fields)

## Compilation System

### Compilation Flow

```
ValidationRule
    ↓
Parse CEL Expression
    ↓
Type Check against Schema DeclType
    ↓
Compile to cel.Program
    ↓
Estimate Worst-Case Cost
    ↓
Cache CompilationResult
    ↓
Reuse for All Validations
```

### CompilationResult Structure

**File**: `pkg/apiserver/schema/cel/compilation.go:40-80`

```go
type CompilationResult struct {
    Program    cel.Program         // Compiled CEL program
    Error      *apiservercel.Error // Compilation error (if any)

    UsesOldSelf bool   // References "oldSelf" variable?
    MaxCost     uint64 // Worst-case cost estimate

    MessageExpression         cel.Program // Compiled message expression
    MessageExpressionMaxCost  uint64      // Message cost estimate

    MaxCardinality             uint64 // Max invocations under unbounded lists
    NormalizedRuleFieldPath    string // Normalized error field path
}
```

### Environment Setup

**File**: `pkg/apiserver/schema/cel/compilation.go:150-200`

```go
func prepareEnvSet(baseEnvSet *environment.EnvSet, declType *apiservercel.DeclType) {
    // Add 'self' variable with schema type
    env.Extend(
        environment.VersionedOptions{
            IntroducedVersion: version.MajorMinor(1, 25),
            EnvOptions: []cel.EnvOption{
                cel.Variable(ScopedVarName, scopedType.CelType()),
            },
        },
    )

    // Add 'oldSelf' variable for transition rules
    env.Extend(
        environment.VersionedOptions{
            IntroducedVersion: version.MajorMinor(1, 25),
            EnvOptions: []cel.EnvOption{
                cel.Variable(OldScopedVarName, scopedType.CelType()),
            },
        },
    )
}
```

**Variables Available**:
- `self`: Current field value being validated
- `oldSelf`: Previous field value (for updates)

### Type System Integration

**DeclType Conversion**:
- OpenAPI schema → CEL DeclType
- Structural schema ensures clear types
- Map field names to CEL object properties
- Array items → CEL list types
- Nested objects → CEL nested types

**Supported Types**:
- **Scalars**: `int`, `double`, `string`, `bool`, `bytes`
- **Complex**: `list`, `map`, `object`
- **Special**: `timestamp`, `duration`
- **Optional**: `optional(T)` for oldSelf with optionalOldSelf: true

**Example Mapping**:
```yaml
# OpenAPI Schema
type: object
properties:
  name: {type: string}
  count: {type: integer}

# CEL DeclType
object {
  name: string
  count: int
}
```

## Validation Execution

### Validation Flow

```
CR Create/Update Request
    ↓
Load Compiled Programs (cached)
    ↓
For each validation rule:
    ↓
  Prepare Activation (bind self, oldSelf)
    ↓
  Evaluate CEL Expression
    ↓
  Track Cost Budget
    ↓
  If result == false: Generate Error
    ↓
Aggregate All Errors
    ↓
Return Validation Result
```

### Variable Binding

**Create Operation** (no oldSelf):
```go
activation := map[string]interface{}{
    "self": currentValue,
}
```

**Update Operation** (with oldSelf):
```go
activation := map[string]interface{}{
    "self":    newValue,
    "oldSelf": oldValue,
}
```

### Evaluation Example

**Rule**: `self.replicas >= 0 && self.replicas <= 100`

**Evaluation**:
```go
result, details, err := program.Eval(activation)
if err != nil {
    return fmt.Errorf("CEL evaluation error: %w", err)
}

boolResult, ok := result.Value().(bool)
if !ok || !boolResult {
    return field.Invalid(fieldPath, self, rule.Message)
}
```

## Cost Management System

### Cost Model

**CEL Cost Units**: Abstract units representing computational expense

**Cost Sources**:
- Function calls: 1-10 units per call
- Field access: 1 unit
- List iteration: Linear in size
- String operations: Based on string length
- Map lookups: Constant time

### Cost Limits

**Per-Call Limit**: 1,000,000 cost units

```go
const PerCallLimit = 1_000_000  // 1 million cost units
```

**Rationale**: Prevents DoS attacks via expensive expressions

**Total Budget**:
```go
MaxRequestSizeBytes / minSize * maxCostPerRule
```

**Example**: For 3MB max request size, ~3000 objects of 1KB each, total budget = 3B cost units

### Cost Estimation

**File**: `pkg/apiserver/schema/cel/compilation.go:250-350`

```go
func (c *sizeEstimator) EstimateSize(element checker.AstNode) *checker.SizeEstimate {
    // Estimate based on schema size constraints
    switch element.Type() {
    case "string":
        return &checker.SizeEstimate{
            Min: 0,
            Max: uint64(schema.MaxLength),
        }
    case "array":
        itemCost := c.EstimateSize(schema.Items)
        return &checker.SizeEstimate{
            Min: 0,
            Max: uint64(schema.MaxItems) * itemCost.Max,
        }
    // ... other types
    }
}
```

**Cardinality Calculation**:
```go
func maxCardinality(minSize int64) uint64 {
    sz := minSize + 1  // Assume one comma between elements
    return uint64(MaxRequestSizeBytes / sz)
}
```

### Cost Tracking

**At Compilation**:
```go
costEst, err := ruleEnv.EstimateCost(ast, estimator)
if costEst.Max > PerCallLimit {
    return nil, fmt.Errorf("estimated cost %d exceeds limit %d", costEst.Max, PerCallLimit)
}
compilationResult.MaxCost = costEst.Max
```

**At Runtime**:
```go
prog, err := ruleEnv.Program(ast,
    cel.CostLimit(PerCallLimit),
    cel.CostTracking(estimator),
    cel.InterruptCheckFrequency(100),
)
```

**Cost Exceeded**: Evaluation aborted, validation fails

## Transition Rules

### Purpose

Validate changes during updates: `oldSelf → self`

### Use Cases

**Immutability**:
```yaml
rule: "self == oldSelf"
message: "field is immutable"
```

**Monotonic Increase**:
```yaml
rule: "self >= oldSelf"
message: "value cannot decrease"
```

**Conditional Immutability**:
```yaml
rule: "self == oldSelf || self.phase == 'Pending'"
message: "field immutable after phase is no longer Pending"
```

**State Transitions**:
```yaml
rule: |
  (oldSelf.state == 'Pending' && self.state in ['Active', 'Failed']) ||
  (oldSelf.state == 'Active' && self.state in ['Completed', 'Failed']) ||
  (oldSelf.state == 'Failed' && self.state == 'Failed')
message: "invalid state transition"
```

### Optional OldSelf

**Problem**: oldSelf not available when field is newly added during update

**Solution**:
```yaml
x-kubernetes-validations:
- rule: "!has(oldSelf) || self >= oldSelf"
  message: "replicas cannot decrease"
  optionalOldSelf: true
```

**Behavior**:
- oldSelf type becomes `optional(T)`
- Use `has(oldSelf)` to check existence
- Guards against newly-added fields
- Enables safe transition rules for optional fields

## Message Expressions

### Dynamic Error Messages

**Static Message** (simple):
```yaml
message: "replicas must be between 0 and 100"
```

**Dynamic Message Expression**:
```yaml
messageExpression: "'replicas value ' + string(self) + ' must be between 0 and 100'"
```

**With Multiple Values**:
```yaml
rule: "self.replicas <= self.maxReplicas"
messageExpression: |
  'replicas (' + string(self.replicas) + 
  ') exceeds maxReplicas (' + string(self.maxReplicas) + ')'
```

### Message Expression Compilation

**Separate Compilation**:
- Must evaluate to string type
- Has own cost limit (separate from rule cost)
- Compiled independently
- Evaluated ONLY on validation failure (for performance)

**Type Checking**: Expression must return `string`

**Example with Conditionals**:
```yaml
messageExpression: |
  self.replicas < 0 
    ? 'replicas must be non-negative, got ' + string(self.replicas)
    : 'replicas exceeds maximum of 100, got ' + string(self.replicas)
```

## CEL Standard Library

### String Functions

```cel
// Substring checks
"hello".contains("ell")           // true
"hello".startsWith("he")          // true
"hello".endsWith("lo")            // true

// Regex matching
"hello123".matches("^[a-z]+[0-9]+$")  // true

// Splitting and joining
"a,b,c".split(",")                // ["a", "b", "c"]
["a", "b"].join(",")              // "a,b"

// Case conversion
"Hello".lowerAscii()              // "hello"
"hello".upperAscii()              // "HELLO"
```

### List Functions

```cel
// Size
[1, 2, 3].size()                  // 3

// Map transformation
[1, 2, 3].map(x, x * 2)           // [2, 4, 6]

// Filter
[1, 2, 3, 4].filter(x, x % 2 == 0) // [2, 4]

// Existence checks
[1, 2, 3].exists(x, x > 2)        // true (at least one)
[1, 2, 3].exists_one(x, x == 2)   // true (exactly one)
[1, 2, 3].all(x, x > 0)           // true (all match)

// Field existence
has(object.field)                 // true if field exists
```

### Map Functions

```cel
// Size
{"a": 1, "b": 2}.size()           // 2

// Membership
"a" in {"a": 1, "b": 2}           // true

// Iteration
{"a": 1, "b": 2}.all(k, v, v > 0) // true
```

### Type Conversion

```cel
int("123")                        // 123
string(123)                       // "123"
double(5)                         // 5.0
type(123)                         // int
```

### Kubernetes Extensions

```cel
// Resource quantities
quantity("100m").compare(quantity("0.1")) == 0  // true

// Durations
duration("1h30m").getSeconds()    // 5400

// URLs
url("https://example.com:8080/path").getHost()  // "example.com"
```

### Best Practices

**Performance** (early exits):
```yaml
# GOOD: Short-circuit evaluation
rule: "self.enabled && self.replicas > 0"

# BAD: Expensive operation first
rule: "self.items.all(i, i.validate()) && self.enabled"
```

**Readability** (clarity over cleverness):
```yaml
# GOOD: Clear logic
rule: "self.state == 'Active' || self.state == 'Pending'"

# BAD: Obscure double negation
rule: "!(self.state != 'Active' && self.state != 'Pending')"
```

**Cost Awareness** (avoid expensive operations in loops):
```yaml
# GOOD: Pre-compute outside loop
rule: "self.items.all(i, i.value <= self.maxValue)"

# BAD: Repeated complex calculation
rule: "self.items.all(i, i.value <= self.computeMax())"
```

## Integration with Schema Validation

### Validation Order

```
1. Type validation (schema types)
2. Format validation (schema formats)
3. Constraint validation (min/max, pattern)
4. Object metadata validation
5. → CEL validation ← HERE
6. Admission webhooks
```

### CEL Requires Structural Schema

**Reason**: CEL needs clear types for:
- Type checking expressions at compile time
- Cost estimation (array sizes, object complexity)
- Variable binding (self, oldSelf)
- Error reporting (field paths)

**Non-Structural Schema**: CEL validation disabled with warning

## Error Reporting

### Validation Error Structure

```go
type Error struct {
    Type    ErrorType  // Invalid, Internal, etc.
    Field   string     // Field path
    Detail  string     // Human-readable message
}
```

### Example Errors

**Basic Validation**:
```
Rule: "self.replicas >= 0"
Value: -5
Error: spec.replicas: Invalid value: -5: replicas must be non-negative
```

**With FieldPath**:
```yaml
rule: "size(self.name) <= 253"
fieldPath: ".name"
```
Error reported at `spec.name` instead of `spec`

**Transition Rule**:
```
Rule: "self >= oldSelf"
Old: 10, New: 5
Error: spec.replicas: Invalid value: 5: replicas cannot decrease
```

## Performance Characteristics

### Compilation Cost

- **When**: CRD registration/update
- **Cost**: O(n) in expression size, O(m) in schema size
- **Cached**: Yes, per CRD version
- **Impact**: One-time cost, ~1-10ms per rule

### Evaluation Cost

- **When**: Every CR create/update
- **Cost**: O(n) in object size × number of rules
- **Limited**: Yes, per-call and total budgets
- **Optimized**: Short-circuit evaluation, early exits

**Example**: 10 rules, 1KB object, 100 cost units per rule = 1000 units total

### Memory Impact

- **Compiled Programs**: Cached in memory
- **Per CRD Version**: ~10-100 KB per version
- **Total**: Scales with (CRD count × versions × rules)

**Example**: 100 CRDs, 3 versions each, 5 rules each = ~15MB

## Critical Files Reference

| File | Purpose | Lines | Importance |
|------|---------|-------|------------|
| `pkg/apiserver/schema/cel/compilation.go` | Compilation logic | 361 | Critical |
| `pkg/apiserver/schema/cel/validation.go` | Evaluation logic | 994 | Critical |
| `pkg/apiserver/schema/cel/model/model.go` | Type system bridge | ~200 | High |
| `pkg/apiserver/schema/cel/compilation_test.go` | Compilation tests | 1,554 | Medium |
| `pkg/apiserver/schema/cel/validation_test.go` | Validation tests | 5,073 | Medium |

## Design Strengths

### 1. Expressive
Rich validation logic without external webhooks:
- Complex boolean logic
- Cross-field validation
- State transition rules
- Dynamic error messages

### 2. Type-Safe
Compile-time type checking:
- Catches type errors early
- Prevents runtime type issues
- Clear error messages

### 3. Cost-Limited
Prevents DoS via expensive expressions:
- Compilation-time cost estimation
- Runtime cost tracking
- Per-call and total budgets
- Abort on cost exceeded

### 4. Cached
Compilation happens once:
- Compiled at CRD registration
- Reused for all validations
- Minimal per-request overhead

### 5. Transition Rules
oldSelf enables sophisticated update validation:
- Immutability enforcement
- State transition validation
- Monotonic constraints
- Conditional updates

## Design Considerations

### 1. Learning Curve
**Issue**: CEL syntax unfamiliar to many users

**Impact**: Requires learning new expression language

**Mitigation**: Documentation, examples, tooling support

### 2. Cost Estimation
**Issue**: Conservative estimates may reject valid rules

**Example**: Nested loops with actual small lists rejected due to worst-case estimate

**Recommendation**: Provide cost estimation tools, tuning options

### 3. Debugging
**Issue**: Limited error context for complex expressions

**Challenge**: Hard to debug why expression failed

**Recommendation**: Better debugging tools, expression testers

### 4. Memory
**Issue**: Cached programs consume memory

**Impact**: Scales with rule count

**Mitigation**: Monitor memory usage, limit rule count per CRD

## Integration Points

### Dependencies
- **github.com/google/cel-go**: CEL implementation
- Structural schemas for type information
- Cost estimator for budget enforcement
- Schema validation pipeline

### Consumers
- Schema validation system (validation pipeline)
- CRD handler for CR validation
- kubectl client-side validation (future)

## Testing

### Compilation Tests
**Location**: `pkg/apiserver/schema/cel/compilation_test.go`

**Coverage**:
- Type checking
- Cost estimation
- Error cases
- Variable binding

### Validation Tests
**Location**: `pkg/apiserver/schema/cel/validation_test.go`

**Coverage**:
- Rule evaluation
- Transition rules
- Error messages
- Cost tracking

### Cost Stability Tests
**Location**: `pkg/apiserver/schema/cel/celcoststability_test.go`

**Purpose**: Ensure cost estimates remain stable across CEL versions

## Related Documentation

- **Schema Validation**: [schema-validation.md](./schema-validation.md) - Structural schema requirements
- **Custom Resource Handler**: [custom-resource-handler.md](./custom-resource-handler.md) - Validation in request flow
- **CRD Lifecycle**: [crd-lifecycle.md](./crd-lifecycle.md) - CRD validation

## Recommendations

### 1. Tooling
Provide CEL expression tester/debugger:
- Online CEL evaluator
- Cost estimation tool
- Syntax highlighting
- Test data generator

### 2. Documentation
More examples and best practices:
- Common patterns library
- Anti-patterns to avoid
- Performance tuning guide
- Migration from webhooks

### 3. Error Messages
Enhance expression error reporting:
- Show sub-expression results
- Highlight failing part
- Suggest corrections
- Link to documentation

### 4. Cost Tuning
Make cost limits configurable:
- Per-CRD cost limits
- Cluster-wide budget
- Cost metrics and monitoring
- Dynamic adjustment based on cluster capacity

### 5. Client-Side Validation
Support CEL validation in kubectl:
- Validate before API call
- Faster feedback loop
- Reduce API server load
- Better user experience
