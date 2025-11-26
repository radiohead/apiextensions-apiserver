# Testing Guide

This guide covers the testing strategy, integration test framework, test patterns, and best practices for the Kubernetes API Extensions API Server.

## Overview

The testing strategy emphasizes comprehensive integration tests that start a real API server, create CRDs, and verify behavior end-to-end. This approach provides high confidence in production behavior while also including focused unit tests for specific components.

**Test Statistics**:
- Total integration test lines: ~15,582
- Number of test files: ~30
- Test coverage: All major features
- Test pattern: Table-driven tests (Kubernetes standard)

## Test Architecture

### Directory Structure

```
test/
└── integration/
    ├── fixtures/          # Test CRD definitions and helpers
    │   ├── crd.go         # CRD creation helpers
    │   ├── schema.go      # Schema fixtures
    │   └── resources.go   # CR creation helpers
    ├── testserver/        # Test harness
    │   ├── server.go      # API server setup
    │   └── client.go      # Client creation
    ├── *_test.go          # Integration tests
    │   ├── registration_test.go      # CRD lifecycle
    │   ├── defaulting_test.go        # Defaulting behavior
    │   ├── apply_test.go             # Server-side apply
    │   ├── subresources_test.go      # Status/scale subresources
    │   ├── conversion_test.go        # Version conversion
    │   ├── table_test.go             # Table output format
    │   └── ...
```

### Test Components

**1. Test Server**: In-memory API server with etcd
**2. Fixtures**: Reusable CRD and CR definitions
**3. Clients**: Generated (typed) and dynamic (unstructured) clients
**4. Assertions**: Table-driven test patterns with clear expectations
**5. Utilities**: Polling, waiting, and helper functions

## Integration Test Framework

### Test Server Setup

**Basic Integration Test Structure**:
```go
package integration

import (
    "context"
    "testing"

    apiextensionsv1 "k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/v1"
    "k8s.io/apiextensions-apiserver/test/integration/fixtures"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

func TestCRDCreation(t *testing.T) {
    // 1. Start test server
    server := framework.StartTestServer(t, framework.TestServerOptions{})
    defer server.TearDownFn()

    // 2. Get client
    client := server.ApiExtensionsClient

    // 3. Create CRD
    crd := fixtures.NewCRD("myresource", "mygroup.example.com")
    result, err := client.ApiextensionsV1().CustomResourceDefinitions().
        Create(context.TODO(), crd, metav1.CreateOptions{})

    // 4. Verify
    if err != nil {
        t.Fatalf("Failed to create CRD: %v", err)
    }

    // 5. Wait for established
    waitForEstablished(t, client, result.Name)

    // 6. Verify conditions
    crd, _ = client.ApiextensionsV1().CustomResourceDefinitions().
        Get(context.TODO(), result.Name, metav1.GetOptions{})
    assertCondition(t, crd, "Established", "True")
    assertCondition(t, crd, "NamesAccepted", "True")
}
```

### Test Server Components

**Server Configuration**:
```go
type TestServerOptions struct {
    // etcd options
    EtcdURL string  // Optional: use external etcd

    // Admission options
    AdmissionPlugins []string  // Optional: enable admission plugins

    // Feature gates
    FeatureGates map[string]bool
}

func StartTestServer(t *testing.T, opts TestServerOptions) *TestServer {
    // 1. Start etcd (in-memory or temporary directory)
    etcdServer := startEtcd(t)

    // 2. Create API server config
    config := createServerConfig(etcdServer.URL, opts)

    // 3. Start API server
    apiServer := startAPIServer(t, config)

    // 4. Wait for server ready
    waitForServerReady(t, apiServer)

    // 5. Create clients
    apiExtensionsClient := createAPIExtensionsClient(apiServer.URL)
    dynamicClient := createDynamicClient(apiServer.URL)

    return &TestServer{
        ApiExtensionsClient: apiExtensionsClient,
        DynamicClient:       dynamicClient,
        EtcdServer:          etcdServer,
        TearDownFn: func() {
            apiServer.Stop()
            etcdServer.Stop()
        },
    }
}
```

**What's Running**:
- etcd: In-memory or temporary directory
- API Server: Full apiextensions-apiserver
- Controllers: All standard controllers (Establishing, Naming, Discovery, etc.)
- Admission: Configured admission plugins (if specified)

## Major Test Categories

### 1. CRD Lifecycle Tests

**File**: `test/integration/registration_test.go`, `test/integration/change_test.go`

**Coverage**:
- CRD creation and validation
- CRD updates (spec changes, version additions)
- CRD deletion and finalizers
- Status condition transitions
- Naming conflicts
- Controller reconciliation

**Example Test**:
```go
func TestCRDEstablishing(t *testing.T) {
    server := framework.StartTestServer(t, framework.TestServerOptions{})
    defer server.TearDownFn()

    client := server.ApiExtensionsClient

    // Create CRD
    crd := fixtures.NewRandomNameCRD()
    created, err := client.ApiextensionsV1().CustomResourceDefinitions().
        Create(context.TODO(), crd, metav1.CreateOptions{})
    if err != nil {
        t.Fatalf("Failed to create CRD: %v", err)
    }

    // Wait for Established condition
    err = wait.Poll(100*time.Millisecond, 10*time.Second, func() (bool, error) {
        crd, err := client.ApiextensionsV1().CustomResourceDefinitions().
            Get(context.TODO(), created.Name, metav1.GetOptions{})
        if err != nil {
            return false, err
        }
        return isEstablished(crd), nil
    })

    if err != nil {
        t.Errorf("CRD never became established: %v", err)
    }

    // Verify all expected conditions
    crd, _ = client.ApiextensionsV1().CustomResourceDefinitions().
        Get(context.TODO(), created.Name, metav1.GetOptions{})
    assertCondition(t, crd, "Established", "True")
    assertCondition(t, crd, "NamesAccepted", "True")
    assertCondition(t, crd, "NonStructuralSchema", "False")
}
```

**Table-Driven Lifecycle Test**:
```go
func TestCRDNamingConflicts(t *testing.T) {
    tests := []struct {
        name      string
        crd1      *apiextensionsv1.CustomResourceDefinition
        crd2      *apiextensionsv1.CustomResourceDefinition
        expectErr bool
        errorMsg  string
    }{
        {
            name: "no conflict - different groups",
            crd1: fixtures.NewCRD("myresources", "group1.example.com"),
            crd2: fixtures.NewCRD("myresources", "group2.example.com"),
            expectErr: false,
        },
        {
            name: "conflict - same plural and group",
            crd1: fixtures.NewCRD("myresources", "mygroup.example.com"),
            crd2: fixtures.NewCRD("myresources", "mygroup.example.com"),
            expectErr: true,
            errorMsg: "already exists",
        },
        {
            name: "conflict - same singular name",
            crd1: fixtures.NewCRDWithNames("myresources", "myresource", "mygroup.example.com"),
            crd2: fixtures.NewCRDWithNames("otherresources", "myresource", "mygroup.example.com"),
            expectErr: true,
            errorMsg: "singular name conflict",
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            server := framework.StartTestServer(t, framework.TestServerOptions{})
            defer server.TearDownFn()

            client := server.ApiExtensionsClient

            // Create first CRD
            _, err := client.ApiextensionsV1().CustomResourceDefinitions().
                Create(context.TODO(), tt.crd1, metav1.CreateOptions{})
            if err != nil {
                t.Fatalf("Failed to create first CRD: %v", err)
            }
            waitForEstablished(t, client, tt.crd1.Name)

            // Try to create second CRD
            _, err = client.ApiextensionsV1().CustomResourceDefinitions().
                Create(context.TODO(), tt.crd2, metav1.CreateOptions{})

            if tt.expectErr && err == nil {
                t.Error("Expected error, got none")
            }
            if !tt.expectErr && err != nil {
                t.Errorf("Expected no error, got: %v", err)
            }
            if tt.expectErr && err != nil && !strings.Contains(err.Error(), tt.errorMsg) {
                t.Errorf("Expected error containing %q, got: %v", tt.errorMsg, err)
            }
        })
    }
}
```

### 2. Validation Tests

**File**: `test/integration/defaulting_test.go`, `test/integration/fieldselector_test.go`

**Coverage**:
- Schema validation (types, formats, constraints)
- CEL validation rules
- Default value application
- Validation error messages
- Field selector validation
- Pruning behavior

**Example CEL Validation Test**:
```go
func TestCELValidation(t *testing.T) {
    tests := []struct {
        name      string
        rule      string
        obj       map[string]interface{}
        expectErr bool
        errorMsg  string
    }{
        {
            name: "valid - simple comparison",
            rule: "self.spec.replicas >= 0",
            obj: map[string]interface{}{
                "spec": map[string]interface{}{"replicas": 3},
            },
            expectErr: false,
        },
        {
            name: "invalid - simple comparison",
            rule: "self.spec.replicas >= 0",
            obj: map[string]interface{}{
                "spec": map[string]interface{}{"replicas": -1},
            },
            expectErr: true,
            errorMsg: "failed validation",
        },
        {
            name: "valid - list validation",
            rule: "self.spec.items.all(item, item.name != '')",
            obj: map[string]interface{}{
                "spec": map[string]interface{}{
                    "items": []interface{}{
                        map[string]interface{}{"name": "item1"},
                        map[string]interface{}{"name": "item2"},
                    },
                },
            },
            expectErr: false,
        },
        {
            name: "invalid - list validation",
            rule: "self.spec.items.all(item, item.name != '')",
            obj: map[string]interface{}{
                "spec": map[string]interface{}{
                    "items": []interface{}{
                        map[string]interface{}{"name": "item1"},
                        map[string]interface{}{"name": ""},  // Empty name
                    },
                },
            },
            expectErr: true,
            errorMsg: "failed validation",
        },
        {
            name: "valid - transition rule with oldSelf",
            rule: "!has(oldSelf) || self >= oldSelf",
            obj: map[string]interface{}{
                "spec": map[string]interface{}{"replicas": 5},
            },
            expectErr: false,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            server := framework.StartTestServer(t, framework.TestServerOptions{})
            defer server.TearDownFn()

            // Create CRD with CEL validation rule
            crd := fixtures.NewCRDWithCELValidation(tt.rule)
            _, err := server.ApiExtensionsClient.ApiextensionsV1().
                CustomResourceDefinitions().
                Create(context.TODO(), crd, metav1.CreateOptions{})
            if err != nil {
                t.Fatalf("Failed to create CRD: %v", err)
            }
            waitForEstablished(t, server.ApiExtensionsClient, crd.Name)

            // Try to create CR
            gvr := schema.GroupVersionResource{
                Group:    crd.Spec.Group,
                Version:  crd.Spec.Versions[0].Name,
                Resource: crd.Spec.Names.Plural,
            }
            obj := &unstructured.Unstructured{Object: tt.obj}
            obj.SetName("test")
            obj.SetAPIVersion(fmt.Sprintf("%s/%s", gvr.Group, gvr.Version))
            obj.SetKind(crd.Spec.Names.Kind)

            _, err = server.DynamicClient.Resource(gvr).Namespace("default").
                Create(context.TODO(), obj, metav1.CreateOptions{})

            if tt.expectErr && err == nil {
                t.Error("Expected error, got none")
            }
            if !tt.expectErr && err != nil {
                t.Errorf("Expected no error, got: %v", err)
            }
            if tt.expectErr && err != nil && !strings.Contains(err.Error(), tt.errorMsg) {
                t.Errorf("Expected error containing %q, got: %v", tt.errorMsg, err)
            }
        })
    }
}
```

**Defaulting Test**:
```go
func TestDefaulting(t *testing.T) {
    server := framework.StartTestServer(t, framework.TestServerOptions{})
    defer server.TearDownFn()

    // Create CRD with default values
    crd := fixtures.NewCRDWithDefaults(map[string]interface{}{
        "replicas": 1,
        "enabled":  true,
    })
    _, err := server.ApiExtensionsClient.ApiextensionsV1().
        CustomResourceDefinitions().
        Create(context.TODO(), crd, metav1.CreateOptions{})
    if err != nil {
        t.Fatalf("Failed to create CRD: %v", err)
    }
    waitForEstablished(t, server.ApiExtensionsClient, crd.Name)

    // Create CR without specifying default fields
    gvr := getGVR(crd)
    obj := &unstructured.Unstructured{
        Object: map[string]interface{}{
            "spec": map[string]interface{}{
                "name": "test",
                // replicas and enabled not specified
            },
        },
    }
    obj.SetName("test")
    setGVK(obj, crd)

    result, err := server.DynamicClient.Resource(gvr).Namespace("default").
        Create(context.TODO(), obj, metav1.CreateOptions{})
    if err != nil {
        t.Fatalf("Failed to create CR: %v", err)
    }

    // Verify defaults applied
    spec := result.Object["spec"].(map[string]interface{})
    if replicas := spec["replicas"]; replicas != int64(1) {
        t.Errorf("Expected replicas=1, got %v", replicas)
    }
    if enabled := spec["enabled"]; enabled != true {
        t.Errorf("Expected enabled=true, got %v", enabled)
    }
}
```

### 3. Conversion Tests

**Coverage**:
- None converter behavior (apiVersion-only change)
- Webhook converter integration
- Version-to-version conversion
- Storage version handling
- Conversion error handling

**Example Conversion Test**:
```go
func TestConversionWebhook(t *testing.T) {
    // Start conversion webhook server
    webhookServer := startConversionWebhook(t)
    defer webhookServer.Stop()

    // Start test server
    server := framework.StartTestServer(t, framework.TestServerOptions{})
    defer server.TearDownFn()

    // Create CRD with conversion webhook
    crd := fixtures.NewMultiVersionCRD()
    crd.Spec.Conversion = &apiextensionsv1.CustomResourceConversion{
        Strategy: apiextensionsv1.WebhookConverter,
        Webhook: &apiextensionsv1.WebhookConversion{
            ClientConfig: &apiextensionsv1.WebhookClientConfig{
                URL: pointer.String(webhookServer.URL),
                CABundle: webhookServer.CABundle,
            },
            ConversionReviewVersions: []string{"v1"},
        },
    }
    _, err := server.ApiExtensionsClient.ApiextensionsV1().
        CustomResourceDefinitions().
        Create(context.TODO(), crd, metav1.CreateOptions{})
    if err != nil {
        t.Fatalf("Failed to create CRD: %v", err)
    }
    waitForEstablished(t, server.ApiExtensionsClient, crd.Name)

    // Create resource in v1beta1 (non-storage version)
    gvrV1Beta1 := schema.GroupVersionResource{
        Group:    crd.Spec.Group,
        Version:  "v1beta1",
        Resource: crd.Spec.Names.Plural,
    }
    objV1Beta1 := fixtures.NewCR("test", "v1beta1")
    created, err := server.DynamicClient.Resource(gvrV1Beta1).Namespace("default").
        Create(context.TODO(), objV1Beta1, metav1.CreateOptions{})
    if err != nil {
        t.Fatalf("Failed to create CR: %v", err)
    }

    // Read back in v1 (storage version) - should trigger conversion
    gvrV1 := schema.GroupVersionResource{
        Group:    crd.Spec.Group,
        Version:  "v1",
        Resource: crd.Spec.Names.Plural,
    }
    retrieved, err := server.DynamicClient.Resource(gvrV1).Namespace("default").
        Get(context.TODO(), created.GetName(), metav1.GetOptions{})
    if err != nil {
        t.Fatalf("Failed to get CR: %v", err)
    }

    // Verify conversion happened
    if retrieved.GetAPIVersion() != "mygroup.example.com/v1" {
        t.Errorf("Expected apiVersion=mygroup.example.com/v1, got %s", retrieved.GetAPIVersion())
    }
    // Verify webhook was called (check webhook server logs/metrics)
}
```

### 4. CRUD Operation Tests

**File**: `test/integration/apply_test.go`, `test/integration/subresources_test.go`

**Coverage**:
- Create, Read, Update, Delete operations
- Server-side apply
- Status subresource
- Scale subresource
- Patch operations (JSON, merge, strategic)
- Field ownership tracking

**Example Server-Side Apply Test**:
```go
func TestServerSideApply(t *testing.T) {
    server := framework.StartTestServer(t, framework.TestServerOptions{})
    defer server.TearDownFn()

    // Create CRD
    crd := fixtures.NewCRD("myresources", "mygroup.example.com")
    _, err := server.ApiExtensionsClient.ApiextensionsV1().
        CustomResourceDefinitions().
        Create(context.TODO(), crd, metav1.CreateOptions{})
    if err != nil {
        t.Fatalf("Failed to create CRD: %v", err)
    }
    waitForEstablished(t, server.ApiExtensionsClient, crd.Name)

    gvr := getGVR(crd)

    // First apply (field manager: user1)
    obj1 := &unstructured.Unstructured{
        Object: map[string]interface{}{
            "spec": map[string]interface{}{
                "field1": "value1",
            },
        },
    }
    obj1.SetName("test")
    setGVK(obj1, crd)

    result1, err := server.DynamicClient.Resource(gvr).Namespace("default").
        Apply(context.TODO(), "test", obj1, metav1.ApplyOptions{
            FieldManager: "user1",
        })
    if err != nil {
        t.Fatalf("First apply failed: %v", err)
    }

    // Verify field1 is owned by user1
    managedFields := result1.GetManagedFields()
    assertFieldOwnership(t, managedFields, "user1", "spec.field1")

    // Second apply (field manager: user2, different field)
    obj2 := &unstructured.Unstructured{
        Object: map[string]interface{}{
            "spec": map[string]interface{}{
                "field2": "value2",
            },
        },
    }
    obj2.SetName("test")
    obj2.SetResourceVersion(result1.GetResourceVersion())
    setGVK(obj2, crd)

    result2, err := server.DynamicClient.Resource(gvr).Namespace("default").
        Apply(context.TODO(), "test", obj2, metav1.ApplyOptions{
            FieldManager: "user2",
        })
    if err != nil {
        t.Fatalf("Second apply failed: %v", err)
    }

    // Verify both fields present, owned by different managers
    spec := result2.Object["spec"].(map[string]interface{})
    if spec["field1"] != "value1" {
        t.Errorf("Expected field1=value1, got %v", spec["field1"])
    }
    if spec["field2"] != "value2" {
        t.Errorf("Expected field2=value2, got %v", spec["field2"])
    }

    managedFields = result2.GetManagedFields()
    assertFieldOwnership(t, managedFields, "user1", "spec.field1")
    assertFieldOwnership(t, managedFields, "user2", "spec.field2")

    // Third apply (user2 tries to modify user1's field) - should conflict
    obj3 := &unstructured.Unstructured{
        Object: map[string]interface{}{
            "spec": map[string]interface{}{
                "field1": "modified",  // Conflict!
            },
        },
    }
    obj3.SetName("test")
    setGVK(obj3, crd)

    _, err = server.DynamicClient.Resource(gvr).Namespace("default").
        Apply(context.TODO(), "test", obj3, metav1.ApplyOptions{
            FieldManager: "user2",
        })

    // Verify conflict error
    if err == nil {
        t.Error("Expected conflict error, got none")
    }
    if !strings.Contains(err.Error(), "field managed by") {
        t.Errorf("Expected field conflict error, got: %v", err)
    }
}
```

**Status Subresource Test**:
```go
func TestStatusSubresource(t *testing.T) {
    server := framework.StartTestServer(t, framework.TestServerOptions{})
    defer server.TearDownFn()

    // Create CRD with status subresource
    crd := fixtures.NewCRD("myresources", "mygroup.example.com")
    crd.Spec.Versions[0].Subresources = &apiextensionsv1.CustomResourceSubresources{
        Status: &apiextensionsv1.CustomResourceSubresourceStatus{},
    }
    _, err := server.ApiExtensionsClient.ApiextensionsV1().
        CustomResourceDefinitions().
        Create(context.TODO(), crd, metav1.CreateOptions{})
    if err != nil {
        t.Fatalf("Failed to create CRD: %v", err)
    }
    waitForEstablished(t, server.ApiExtensionsClient, crd.Name)

    gvr := getGVR(crd)

    // Create resource
    obj := fixtures.NewCR("test", "v1")
    created, err := server.DynamicClient.Resource(gvr).Namespace("default").
        Create(context.TODO(), obj, metav1.CreateOptions{})
    if err != nil {
        t.Fatalf("Failed to create CR: %v", err)
    }

    // Update status
    created.Object["status"] = map[string]interface{}{
        "state": "Ready",
        "observedGeneration": 1,
    }
    updated, err := server.DynamicClient.Resource(gvr).Namespace("default").
        UpdateStatus(context.TODO(), created, metav1.UpdateOptions{})
    if err != nil {
        t.Fatalf("Failed to update status: %v", err)
    }

    // Verify status updated
    status := updated.Object["status"].(map[string]interface{})
    if status["state"] != "Ready" {
        t.Errorf("Expected state=Ready, got %v", status["state"])
    }

    // Verify spec not changed (generation same)
    if updated.GetGeneration() != created.GetGeneration() {
        t.Error("Generation changed on status update")
    }
}
```

### 5. Discovery Tests

**Coverage**:
- API group discovery
- API version discovery
- Resource discovery
- Aggregated discovery
- Discovery updates after CRD changes

**Example Discovery Test**:
```go
func TestDiscovery(t *testing.T) {
    server := framework.StartTestServer(t, framework.TestServerOptions{})
    defer server.TearDownFn()

    // Create CRD
    crd := fixtures.NewCRD("myresources", "mygroup.example.com")
    _, err := server.ApiExtensionsClient.ApiextensionsV1().
        CustomResourceDefinitions().
        Create(context.TODO(), crd, metav1.CreateOptions{})
    if err != nil {
        t.Fatalf("Failed to create CRD: %v", err)
    }
    waitForEstablished(t, server.ApiExtensionsClient, crd.Name)

    // Wait for discovery to update
    time.Sleep(1 * time.Second)

    // Check API group discovery
    groups, err := server.DiscoveryClient.ServerGroups()
    if err != nil {
        t.Fatalf("Failed to get server groups: %v", err)
    }

    found := false
    for _, group := range groups.Groups {
        if group.Name == "mygroup.example.com" {
            found = true
            // Verify versions
            if len(group.Versions) != 1 || group.Versions[0].Version != "v1" {
                t.Errorf("Expected version v1, got %v", group.Versions)
            }
        }
    }
    if !found {
        t.Error("Group mygroup.example.com not found in discovery")
    }

    // Check resource discovery
    resources, err := server.DiscoveryClient.ServerResourcesForGroupVersion("mygroup.example.com/v1")
    if err != nil {
        t.Fatalf("Failed to get resources: %v", err)
    }

    found = false
    for _, resource := range resources.APIResources {
        if resource.Name == "myresources" {
            found = true
            // Verify verbs
            expectedVerbs := []string{"create", "delete", "deletecollection", "get", "list", "patch", "update", "watch"}
            if !sliceEqual(resource.Verbs, expectedVerbs) {
                t.Errorf("Expected verbs %v, got %v", expectedVerbs, resource.Verbs)
            }
            // Verify scope
            if resource.Namespaced != true {
                t.Error("Expected namespaced=true")
            }
        }
    }
    if !found {
        t.Error("Resource myresources not found in discovery")
    }
}
```

### 6. Edge Case Tests

**File**: `test/integration/scope_test.go`, `test/integration/yaml_test.go`, `test/integration/table_test.go`

**Coverage**:
- Namespace vs cluster scope
- YAML input handling
- Table output format
- Large object handling
- Concurrent CRD operations
- Deprecation warnings
- Version preference

**Example Deprecation Test**:
```go
func TestDeprecationWarning(t *testing.T) {
    server := framework.StartTestServer(t, framework.TestServerOptions{})
    defer server.TearDownFn()

    // Create CRD with deprecated version
    crd := fixtures.NewCRD("myresources", "mygroup.example.com")
    crd.Spec.Versions = []apiextensionsv1.CustomResourceDefinitionVersion{
        {
            Name:    "v1beta1",
            Served:  true,
            Storage: true,
            Deprecated: true,
            DeprecationWarning: pointer.String("v1beta1 is deprecated, use v1 instead"),
            Schema:  fixtures.AllowAllSchema(),
        },
        {
            Name:    "v1",
            Served:  true,
            Storage: false,
            Schema:  fixtures.AllowAllSchema(),
        },
    }
    _, err := server.ApiExtensionsClient.ApiextensionsV1().
        CustomResourceDefinitions().
        Create(context.TODO(), crd, metav1.CreateOptions{})
    if err != nil {
        t.Fatalf("Failed to create CRD: %v", err)
    }
    waitForEstablished(t, server.ApiExtensionsClient, crd.Name)

    // Create CR using deprecated version
    gvr := schema.GroupVersionResource{
        Group:    "mygroup.example.com",
        Version:  "v1beta1",
        Resource: "myresources",
    }
    obj := fixtures.NewCR("test", "v1beta1")

    // Capture warnings
    warnings := &warningRecorder{}
    ctx := warning.WithWarningRecorder(context.TODO(), warnings)

    _, err = server.DynamicClient.Resource(gvr).Namespace("default").
        Create(ctx, obj, metav1.CreateOptions{})
    if err != nil {
        t.Fatalf("Failed to create CR: %v", err)
    }

    // Verify warning received
    if len(warnings.Warnings) != 1 {
        t.Errorf("Expected 1 warning, got %d", len(warnings.Warnings))
    }
    if !strings.Contains(warnings.Warnings[0], "v1beta1 is deprecated") {
        t.Errorf("Expected deprecation warning, got: %v", warnings.Warnings[0])
    }
}
```

## Test Patterns

### Table-Driven Tests (Kubernetes Standard)

**Pattern**:
```go
func TestFeature(t *testing.T) {
    tests := []struct {
        name      string
        input     interface{}
        expected  interface{}
        expectErr bool
        errorMsg  string
    }{
        {
            name:      "valid case",
            input:     validInput,
            expected:  expectedOutput,
            expectErr: false,
        },
        {
            name:      "invalid case",
            input:     invalidInput,
            expectErr: true,
            errorMsg:  "expected error message",
        },
        // More test cases...
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Setup (if needed)
            server := framework.StartTestServer(t, framework.TestServerOptions{})
            defer server.TearDownFn()

            // Execute
            result, err := someFunction(tt.input)

            // Verify
            if tt.expectErr && err == nil {
                t.Error("Expected error, got none")
            }
            if !tt.expectErr && err != nil {
                t.Errorf("Expected no error, got: %v", err)
            }
            if tt.expectErr && err != nil && !strings.Contains(err.Error(), tt.errorMsg) {
                t.Errorf("Expected error containing %q, got: %v", tt.errorMsg, err)
            }
            if !tt.expectErr && !reflect.DeepEqual(result, tt.expected) {
                t.Errorf("Expected %v, got %v", tt.expected, result)
            }
        })
    }
}
```

**Benefits**:
- Easy to add new test cases
- Clear test case descriptions
- Isolated test execution (t.Run)
- Standard Kubernetes approach

### Polling for Conditions

**Pattern**:
```go
func waitForCondition(t *testing.T, client clientset.Interface, crdName string, conditionType string) {
    err := wait.Poll(100*time.Millisecond, 10*time.Second, func() (bool, error) {
        crd, err := client.ApiextensionsV1().CustomResourceDefinitions().
            Get(context.TODO(), crdName, metav1.GetOptions{})
        if err != nil {
            return false, err
        }

        for _, cond := range crd.Status.Conditions {
            if cond.Type == apiextensionsv1.CustomResourceDefinitionConditionType(conditionType) &&
               cond.Status == apiextensionsv1.ConditionTrue {
                return true, nil
            }
        }
        return false, nil
    })

    if err != nil {
        t.Fatalf("Condition %s never became true: %v", conditionType, err)
    }
}
```

**Usage**:
```go
waitForCondition(t, client, "myresources.mygroup.example.com", "Established")
```

### Fixture Management

**Pattern**:
```go
package fixtures

// CRD creation helpers
func NewCRD(plural, group string) *apiextensionsv1.CustomResourceDefinition {
    return &apiextensionsv1.CustomResourceDefinition{
        ObjectMeta: metav1.ObjectMeta{
            Name: plural + "." + group,
        },
        Spec: apiextensionsv1.CustomResourceDefinitionSpec{
            Group: group,
            Names: apiextensionsv1.CustomResourceDefinitionNames{
                Plural:   plural,
                Singular: strings.TrimSuffix(plural, "s"),
                Kind:     strings.Title(strings.TrimSuffix(plural, "s")),
                ListKind: strings.Title(strings.TrimSuffix(plural, "s")) + "List",
            },
            Scope: apiextensionsv1.NamespaceScoped,
            Versions: []apiextensionsv1.CustomResourceDefinitionVersion{
                {
                    Name:    "v1",
                    Served:  true,
                    Storage: true,
                    Schema:  AllowAllSchema(),
                },
            },
        },
    }
}

func NewCRDWithCELValidation(rule string) *apiextensionsv1.CustomResourceDefinition {
    crd := NewCRD("myresources", "mygroup.example.com")
    crd.Spec.Versions[0].Schema = &apiextensionsv1.CustomResourceValidation{
        OpenAPIV3Schema: &apiextensionsv1.JSONSchemaProps{
            Type: "object",
            Properties: map[string]apiextensionsv1.JSONSchemaProps{
                "spec": {
                    Type: "object",
                    XValidations: []apiextensionsv1.ValidationRule{
                        {
                            Rule: rule,
                        },
                    },
                },
            },
        },
    }
    return crd
}

func AllowAllSchema() *apiextensionsv1.CustomResourceValidation {
    return &apiextensionsv1.CustomResourceValidation{
        OpenAPIV3Schema: &apiextensionsv1.JSONSchemaProps{
            Type:                   "object",
            XPreserveUnknownFields: pointer.Bool(true),
        },
    }
}
```

## Unit Tests

### Unit Test Coverage

**Areas**:
- Schema validation logic (`pkg/apiserver/schema/`)
- CEL compilation (`pkg/apiserver/schema/cel/`)
- Conversion helpers (`pkg/apiserver/conversion/`)
- Utility functions (name validation, etc.)
- Type conversions

**Example Unit Test**:
```go
func TestStructuralSchemaValidation(t *testing.T) {
    tests := []struct {
        name     string
        schema   *apiextensionsv1.JSONSchemaProps
        isValid  bool
        errorMsg string
    }{
        {
            name: "valid structural schema",
            schema: &apiextensionsv1.JSONSchemaProps{
                Type: "object",
                Properties: map[string]apiextensionsv1.JSONSchemaProps{
                    "field": {Type: "string"},
                },
            },
            isValid: true,
        },
        {
            name: "non-structural - missing type",
            schema: &apiextensionsv1.JSONSchemaProps{
                Properties: map[string]apiextensionsv1.JSONSchemaProps{
                    "field": {},  // No type specified
                },
            },
            isValid: false,
            errorMsg: "type is required",
        },
        {
            name: "non-structural - oneOf",
            schema: &apiextensionsv1.JSONSchemaProps{
                Type: "object",
                OneOf: []apiextensionsv1.JSONSchemaProps{
                    {Type: "string"},
                    {Type: "integer"},
                },
            },
            isValid: false,
            errorMsg: "oneOf is not allowed",
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            _, err := schema.NewStructural(tt.schema)
            if tt.isValid && err != nil {
                t.Errorf("Expected valid schema, got error: %v", err)
            }
            if !tt.isValid && err == nil {
                t.Error("Expected error for invalid schema, got none")
            }
            if !tt.isValid && err != nil && !strings.Contains(err.Error(), tt.errorMsg) {
                t.Errorf("Expected error containing %q, got: %v", tt.errorMsg, err)
            }
        })
    }
}
```

## Test Utilities

### Common Helper Functions

**Waiting Functions**:
```go
// Wait for CRD to become established
func waitForEstablished(t *testing.T, client clientset.Interface, name string) {
    waitForCondition(t, client, name, "Established")
}

// Wait for CRD deletion
func waitForCRDDeletion(t *testing.T, client clientset.Interface, name string) {
    err := wait.Poll(100*time.Millisecond, 10*time.Second, func() (bool, error) {
        _, err := client.ApiextensionsV1().CustomResourceDefinitions().
            Get(context.TODO(), name, metav1.GetOptions{})
        if apierrors.IsNotFound(err) {
            return true, nil  // CRD deleted
        }
        return false, err
    })
    if err != nil {
        t.Fatalf("CRD %s was not deleted: %v", name, err)
    }
}

// Wait for specific CR count
func waitForCRCount(t *testing.T, dynamicClient dynamic.Interface,
    gvr schema.GroupVersionResource, namespace string, count int) {
    err := wait.Poll(100*time.Millisecond, 10*time.Second, func() (bool, error) {
        list, err := dynamicClient.Resource(gvr).Namespace(namespace).
            List(context.TODO(), metav1.ListOptions{})
        if err != nil {
            return false, err
        }
        return len(list.Items) == count, nil
    })
    if err != nil {
        t.Fatalf("CR count never reached %d: %v", count, err)
    }
}
```

**Assertion Helpers**:
```go
// Assert CRD condition
func assertCondition(t *testing.T, crd *apiextensionsv1.CustomResourceDefinition,
    condType string, status string) {
    for _, cond := range crd.Status.Conditions {
        if string(cond.Type) == condType {
            if string(cond.Status) != status {
                t.Errorf("Expected condition %s=%s, got %s", condType, status, cond.Status)
            }
            return
        }
    }
    t.Errorf("Condition %s not found", condType)
}

// Assert field exists in unstructured object
func assertFieldExists(t *testing.T, obj *unstructured.Unstructured, fieldPath string) {
    fields := strings.Split(fieldPath, ".")
    current := obj.Object
    for _, field := range fields {
        value, found := current[field]
        if !found {
            t.Errorf("Field %s not found in object", fieldPath)
            return
        }
        if nested, ok := value.(map[string]interface{}); ok {
            current = nested
        }
    }
}

// Assert error contains message
func assertError(t *testing.T, err error, expectedMsg string) {
    if err == nil {
        t.Errorf("Expected error containing %q, got none", expectedMsg)
        return
    }
    if !strings.Contains(err.Error(), expectedMsg) {
        t.Errorf("Expected error containing %q, got: %v", expectedMsg, err)
    }
}
```

## Test Execution

### Running Tests

**All Integration Tests**:
```bash
go test ./test/integration/...
```

**Specific Test File**:
```bash
go test ./test/integration/registration_test.go -v
```

**Specific Test Function**:
```bash
go test ./test/integration -run TestCRDEstablishing -v
```

**With Race Detection**:
```bash
go test -race ./test/integration/...
```

**With Coverage**:
```bash
go test -cover ./test/integration/...
go test -coverprofile=coverage.out ./test/integration/...
go tool cover -html=coverage.out
```

**Parallel Execution**:
```bash
go test -parallel 4 ./test/integration/...
```

### CI Integration

**Pre-Merge Checks** (typical Kubernetes workflow):
1. Unit tests: `go test ./pkg/...`
2. Integration tests: `go test ./test/integration/...`
3. Code generation verification: `./hack/verify-codegen.sh`
4. Linting: `golangci-lint run`
5. Format check: `go fmt ./...`

## Best Practices

### Writing New Tests

1. **Use table-driven tests** for multiple similar cases
2. **Isolate tests** - each test should be independent
3. **Use fixtures** for common CRD/CR creation
4. **Poll for conditions** instead of sleeping
5. **Clean up resources** with `defer server.TearDownFn()`
6. **Test error cases** in addition to happy path
7. **Use descriptive test names** that explain what's being tested

### Test Organization

1. **Group related tests** in same file
2. **Use subtests** (`t.Run`) for variants
3. **Share setup code** via helpers
4. **Document complex test logic** with comments
5. **Keep tests fast** (< 1 second each if possible)

### Debugging Tests

**Verbose Output**:
```bash
go test -v ./test/integration -run TestCRDEstablishing
```

**Print Debug Info**:
```go
t.Logf("CRD status: %+v", crd.Status)
t.Logf("CR object: %+v", obj.Object)
```

**Check Test Server Logs**:
```go
// In test setup, enable API server logging
framework.StartTestServer(t, framework.TestServerOptions{
    APIServerArgs: []string{
        "--v=8",  // Verbose logging
    },
})
```

## Related Documentation

- [Custom Resource Handler](../core-components/custom-resource-handler.md) - Handler implementation
- [Schema Validation](../core-components/schema-validation.md) - Validation logic
- [CEL Integration](../core-components/cel-integration.md) - CEL validation
- [Troubleshooting Guide](./troubleshooting.md) - Debugging test failures
