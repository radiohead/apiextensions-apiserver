# Version Conversion System

The version conversion system enables multi-version CRDs through webhook-based conversion or simple apiVersion changes. It supports seamless API evolution, allowing CRDs to serve multiple versions simultaneously while storing in a single canonical version.

## Overview

**Primary Location**: `pkg/apiserver/conversion/`
**Key Files**:
- `converter.go` (~300 lines): Converter factory
- `webhook_converter.go` (~200 lines): Webhook client
**Confidence**: 87%

The conversion system allows CRDs to evolve their APIs while maintaining backward compatibility, automatically converting between versions on read and write operations.

## Conversion Strategies

### Strategy Types

**File**: `pkg/apis/apiextensions/v1/types.go:400-420`

```go
type ConversionStrategyType string

const (
    NoneConverter    ConversionStrategyType = "None"     // Only changes apiVersion
    WebhookConverter ConversionStrategyType = "Webhook"  // Calls external webhook
)
```

### 1. None Converter

**Behavior**:
- Changes only `apiVersion` field
- Leaves all other fields unchanged
- Assumes versions are semantically compatible
- No external dependencies
- Zero conversion latency

**Use Case**: Identical or compatible schemas across versions

**Configuration**:
```yaml
conversion:
  strategy: None
```

**Example**:
```yaml
# v1beta1 → v1 conversion (None strategy)
Input (v1beta1):
  apiVersion: example.com/v1beta1
  kind: Widget
  spec: {replicas: 3}

Output (v1):
  apiVersion: example.com/v1
  kind: Widget
  spec: {replicas: 3}  # Unchanged
```

### 2. Webhook Converter

**Behavior**:
- Calls external webhook for conversion
- Supports bidirectional conversion (any version → any version)
- Enables schema differences between versions
- Requires external service availability
- Adds network latency

**Use Case**: Different schemas, field renames, restructuring

**Configuration**:
```yaml
conversion:
  strategy: Webhook
  webhook:
    clientConfig:
      service:
        name: widget-webhook
        namespace: default
        path: /convert
        port: 443
      caBundle: <base64-encoded-ca-cert>
    conversionReviewVersions: ["v1", "v1beta1"]
```

## Conversion Architecture

### Conversion Flow

```
API Request (v1beta1 object)
    ↓
Handler determines storage version (v1)
    ↓
Request version != storage version?
    ↓ Yes
Lookup conversion strategy from CRD
    ↓
If None: Change apiVersion only
If Webhook: Call conversion webhook
    ↓
Convert to storage version (v1)
    ↓
Store in etcd
    ↓
On Read:
    ↓
Read from etcd (storage version)
    ↓
Requested version != storage version?
    ↓ Yes
Convert storage → requested version
    ↓
Return response
```

### CRConverterFactory

**File**: `pkg/apiserver/conversion/converter.go:50-150`

```go
type CRConverterFactory struct {
    // Webhook configuration
    serviceResolver      webhook.ServiceResolver
    authResolverWrapper  webhook.AuthenticationInfoResolverWrapper

    // Converter cache (keyed by CRD UID)
    converters map[types.UID]*crConverter
    lock       sync.RWMutex
}

func (f *CRConverterFactory) NewConverter(crd *apiextensionsv1.CustomResourceDefinition) (runtime.ObjectConvertor, error) {
    // Check cache
    f.lock.RLock()
    converter, exists := f.converters[crd.UID]
    f.lock.RUnlock()
    if exists {
        return converter, nil
    }

    // Create new converter
    converter = f.createConverter(crd)

    // Cache it
    f.lock.Lock()
    f.converters[crd.UID] = converter
    f.lock.Unlock()

    return converter, nil
}
```

**Caching**: Per-CRD converter cached for performance

### Conversion Request/Response Protocol

**ConversionReview Request**:
```json
{
  "apiVersion": "apiextensions.k8s.io/v1",
  "kind": "ConversionReview",
  "request": {
    "uid": "705ab4f5-6393-11e8-b7cc-42010a800002",
    "desiredAPIVersion": "example.com/v1",
    "objects": [
      {
        "apiVersion": "example.com/v1beta1",
        "kind": "Widget",
        "metadata": {"name": "widget1"},
        "spec": {"count": 5}
      }
    ]
  }
}
```

**ConversionReview Response**:
```json
{
  "apiVersion": "apiextensions.k8s.io/v1",
  "kind": "ConversionReview",
  "response": {
    "uid": "705ab4f5-6393-11e8-b7cc-42010a800002",
    "result": {
      "status": "Success"
    },
    "convertedObjects": [
      {
        "apiVersion": "example.com/v1",
        "kind": "Widget",
        "metadata": {"name": "widget1"},
        "spec": {"replicas": 5}  # Field renamed: count → replicas
      }
    ]
  }
}
```

**Batch Conversion**: Multiple objects in single request for efficiency

## Webhook Client Configuration

### Service Reference (In-Cluster)

```yaml
webhook:
  clientConfig:
    service:
      name: widget-webhook-svc
      namespace: webhook-namespace
      port: 443              # Optional, defaults to 443
      path: /convert         # Optional
```

**Resolution**: `service.namespace.svc.cluster.local:port/path`

**TLS**: HTTPS required, validated against caBundle

### URL Reference (External)

```yaml
webhook:
  clientConfig:
    url: "https://external-webhook.example.com:8443/convert"
```

**Constraints**:
- Must be HTTPS (TLS required)
- No localhost/127.0.0.1 (cluster portability)
- No user:password authentication
- No fragments (#) or query parameters (?)

### CA Bundle

**Purpose**: Validate webhook server certificate

```yaml
webhook:
  clientConfig:
    caBundle: <base64-encoded-PEM-certificate>
```

**Format**: Base64-encoded PEM certificate(s)
**Fallback**: System trust roots if unspecified
**Security**: Prevents man-in-the-middle attacks

## Storage Version Management

### Purpose

Single canonical version for storage in etcd, reducing conversion overhead and ensuring consistency.

### Selection Rules

1. **Exactly One**: Only one version can be `storage: true`
2. **Served Requirement**: Storage version must be served
3. **Migration Required**: Changing storage version requires data migration

### Configuration Example

```yaml
versions:
- name: v1
  served: true
  storage: true    # ← Storage version (canonical)
  schema: {...}
- name: v1beta1
  served: true
  storage: false   # Must convert to v1 for storage
  schema: {...}
- name: v1alpha1
  served: false    # No longer served
  storage: false
  schema: {...}
```

### Storage Version Transitions

**Process**:
1. Add new version with `storage: false, served: true`
2. Deploy and validate new version works
3. **Run storage migration job** (read all CRs, write back)
4. Update CRD: Set new version `storage: true`, old version `storage: false`
5. Old version now automatically converted

**Migration Job Example**:
```go
// Read all resources in old storage version
list, _ := client.Resource(gvr).List(ctx, metav1.ListOptions{})

// Write them back (triggers conversion to new storage version)
for _, item := range list.Items {
    _, _ = client.Resource(gvr).Update(ctx, &item, metav1.UpdateOptions{})
}
```

**No Downtime**: Migration can be done without service interruption

## Conversion Review Versions

### Purpose

Specify supported ConversionReview API versions for webhook protocol negotiation.

### Configuration

```yaml
webhook:
  conversionReviewVersions: ["v1", "v1beta1"]
```

**Behavior**:
- API server tries versions in specified order
- Uses first version webhook supports
- Falls back to next version if webhook doesn't support

**Version Differences**:
- **v1**: Stable, current API
- **v1beta1**: Legacy, deprecated, for backward compatibility

## Conversion Error Handling

### Webhook Errors

**Network Failures**:
- Connection refused
- Timeout (default 30s)
- TLS validation errors
- DNS resolution failures

**API Server Response**: `500 Internal Server Error`

**Validation Failures**:
- Webhook returns error status
- Invalid response format
- Missing required fields
- UID mismatch

**API Server Response**: `422 Unprocessable Entity`

### Error Reporting to Client

```json
{
  "kind": "Status",
  "status": "Failure",
  "message": "conversion webhook for example.com failed: Post https://webhook.default.svc:443/convert: connection refused",
  "reason": "InternalError",
  "code": 500
}
```

**User Experience**: Clear error messages indicating webhook issue

### Retry Behavior

**No Automatic Retry**: Single webhook call per conversion
**Client Retry**: Client can retry entire request
**Recommendation**: Implement retry logic in webhook for transient errors

## Performance Impact

### Conversion Overhead

**Read Operations**: Convert storage → requested version
**Write Operations**: Convert request → storage version
**Watch Operations**: Convert each event in stream

**Latency Impact**:
- None strategy: < 1ms (in-memory field change)
- Webhook strategy: +10-50ms (network + processing)

### Optimization Strategies

#### 1. Use Storage Version

```go
// FAST: Request storage version (no conversion)
dynamicClient.Resource(gvr).Get(ctx, name, metav1.GetOptions{})

// SLOW: Request non-storage version (requires conversion)
dynamicClient.Resource(otherGVR).Get(ctx, name, metav1.GetOptions{})
```

#### 2. Fast Webhooks

- Minimize conversion logic complexity
- Use efficient serialization (protobuf)
- Local in-cluster deployment
- Adequate resource limits

#### 3. Batch Conversions

Multiple objects in single webhook request reduces overhead:
- List operations: Batch convert all items
- Watch operations: Buffer and batch when possible

#### 4. Caching Considerations

**Current**: No result caching (every conversion calls webhook)
**Future**: Consider caching conversion results by (version, resourceVersion)

## Webhook Implementation Guide

### Webhook Requirements

1. **HTTPS Endpoint**: Must serve over TLS
2. **POST Handler**: Accept ConversionReview POST requests
3. **Bidirectional**: Support all version combinations
4. **Idempotent**: Same input → same output (deterministic)
5. **Fast**: Low latency (< 100ms recommended)
6. **Available**: High availability (impacts all CR operations)

### Example Webhook Server

```go
package main

import (
    "encoding/json"
    "net/http"
    apiextensionsv1 "k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/v1"
)

func handleConvert(w http.ResponseWriter, r *http.Request) {
    var review apiextensionsv1.ConversionReview
    if err := json.NewDecoder(r.Body).Decode(&review); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    convertedObjects := []runtime.RawExtension{}
    for _, obj := range review.Request.Objects {
        // Determine source and target versions
        sourceVersion := obj.Object.GetAPIVersion()
        targetVersion := review.Request.DesiredAPIVersion

        // Convert object
        converted, err := convertObject(obj, sourceVersion, targetVersion)
        if err != nil {
            // Return error response
            review.Response = &apiextensionsv1.ConversionResponse{
                UID: review.Request.UID,
                Result: metav1.Status{
                    Status:  "Failure",
                    Message: err.Error(),
                },
            }
            json.NewEncoder(w).Encode(review)
            return
        }

        convertedObjects = append(convertedObjects, converted)
    }

    // Success response
    review.Response = &apiextensionsv1.ConversionResponse{
        UID:              review.Request.UID,
        Result:           metav1.Status{Status: "Success"},
        ConvertedObjects: convertedObjects,
    }

    json.NewEncoder(w).Encode(review)
}

func main() {
    http.HandleFunc("/convert", handleConvert)
    http.ListenAndServeTLS(":443", "server.crt", "server.key", nil)
}
```

### Conversion Logic Patterns

**Field Renames**:
```go
if sourceVersion == "v1beta1" && targetVersion == "v1" {
    v1Obj.Spec.Replicas = v1beta1Obj.Spec.Count  // Rename: count → replicas
}
```

**Field Restructuring**:
```go
// v1beta1: flat structure
// v1beta1.spec.host = "example.com"
// v1beta1.spec.port = 8080

// v1: nested structure
// v1.spec.endpoint.host = "example.com"
// v1.spec.endpoint.port = 8080

if sourceVersion == "v1beta1" && targetVersion == "v1" {
    v1Obj.Spec.Endpoint = Endpoint{
        Host: v1beta1Obj.Spec.Host,
        Port: v1beta1Obj.Spec.Port,
    }
}
```

**Default Values**:
```go
if sourceVersion == "v1beta1" && targetVersion == "v1" {
    if v1Obj.Spec.Strategy == "" {
        v1Obj.Spec.Strategy = "RollingUpdate"  // Default added in v1
    }
}
```

## Critical Files Reference

| File | Purpose | Lines | Importance |
|------|---------|-------|------------|
| `pkg/apiserver/conversion/converter.go` | Converter factory | ~300 | Critical |
| `pkg/apiserver/conversion/webhook_converter.go` | Webhook client | ~200 | Critical |
| `pkg/apis/apiextensions/v1/types.go` | Conversion types | ~100 | High |

## Related Documentation

- **CRD Lifecycle**: [crd-lifecycle.md](./crd-lifecycle.md) - StoredVersions management
- **Custom Resource Handler**: [custom-resource-handler.md](./custom-resource-handler.md) - Conversion in request flow
- **API Server**: [api-server.md](./api-server.md) - Converter factory setup

## Recommendations

### 1. Result Caching
Cache recent conversion results to reduce webhook calls:
- LRU cache keyed by (sourceVersion, targetVersion, resourceVersion)
- Configurable cache size and TTL
- Significant performance improvement for read-heavy workloads

### 2. Webhook Monitoring
Add comprehensive metrics:
- Conversion latency by version pair
- Error rates by error type
- Webhook availability
- Cache hit rates

### 3. Fallback Strategy
Support graceful degradation:
- Retry with exponential backoff
- Circuit breaker for failing webhooks
- Fallback to cached results (if available)

### 4. Validation
Validate conversion correctness:
- Round-trip testing (v1 → v2 → v1)
- Schema compliance checking
- Metadata preservation validation

### 5. Migration Tooling
Provide storage version migration tools:
- Automated migration job generator
- Progress tracking and reporting
- Dry-run mode for validation
- Rollback support
