# Client Libraries and Code Generation

The client library system provides typed Go clients for CRD resources through code generation. It includes clientsets, listers, informers, deep copy functions, and apply configurations, following standard Kubernetes patterns for type-safe interaction with custom resources.

## Overview

**Primary Location**: `pkg/client/`
**Generation Script**: `hack/update-codegen.sh`
**Confidence**: 88%

The generated clients enable controllers and applications to interact with CRDs using strongly-typed Go code, following the same patterns as core Kubernetes resources.

## Code Generation System

### Generation Tools

**Script**: `hack/update-codegen.sh`

**Generators Used** (from k8s.io/code-generator):
1. **client-gen**: Generate typed clientsets
2. **lister-gen**: Generate listers for cached reads
3. **informer-gen**: Generate informers for watching
4. **deepcopy-gen**: Generate deep copy methods
5. **applyconfiguration-gen**: Generate apply configurations for SSA
6. **openapi-gen**: Generate OpenAPI definitions

### Generation Markers

**In Source Code** (`pkg/apis/apiextensions/v1/types.go`):

```go
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
// +genclient
// +genclient:nonNamespaced
type CustomResourceDefinition struct {
    metav1.TypeMeta
    metav1.ObjectMeta
    Spec   CustomResourceDefinitionSpec
    Status CustomResourceDefinitionStatus
}
```

**Markers**:
- `+k8s:deepcopy-gen:interfaces`: Generate DeepCopy methods implementing runtime.Object
- `+genclient`: Generate client for this type
- `+genclient:nonNamespaced`: Resource is cluster-scoped (no namespace)

### Running Code Generation

```bash
# Generate all client code
./hack/update-codegen.sh

# Verify generated code is up-to-date
./hack/verify-codegen.sh
```

**Output**: Generated files prefixed with `zz_generated.` (should not be manually edited)

## Generated Client Structure

### Directory Layout

```
pkg/client/
├── clientset/
│   └── clientset/
│       ├── typed/
│       │   └── apiextensions/
│       │       ├── v1/
│       │       │   ├── apiextensions_client.go
│       │       │   ├── customresourcedefinition.go
│       │       │   ├── generated_expansion.go
│       │       │   └── fake/
│       │       │       └── fake_customresourcedefinition.go
│       │       └── v1beta1/ (legacy)
│       ├── clientset.go
│       ├── doc.go
│       └── fake/
│           └── clientset_generated.go
├── informers/
│   └── externalversions/
│       ├── factory.go
│       ├── generic.go
│       └── apiextensions/
│           ├── v1/
│           │   ├── interface.go
│           │   └── customresourcedefinition.go
│           └── v1beta1/
├── listers/
│   └── apiextensions/
│       ├── v1/
│       │   ├── customresourcedefinition.go
│       │   └── expansion_generated.go
│       └── v1beta1/
└── applyconfiguration/
    └── apiextensions/
        └── v1/
            ├── customresourcedefinition.go
            └── customresourcedefinitionspec.go
```

## Clientsets

### Purpose

Provide typed, strongly-typed access to CRD resources with compile-time safety.

### Structure

**Main Clientset** (`pkg/client/clientset/clientset/clientset.go`):

```go
type Clientset struct {
    *discovery.DiscoveryClient
    apiextensionsV1       *apiextensionsv1.ApiextensionsV1Client
    apiextensionsV1beta1  *apiextensionsv1beta1.ApiextensionsV1beta1Client
}

func NewForConfig(c *rest.Config) (*Clientset, error) {
    // Create clientset from kubeconfig
}
```

**Versioned Client**:

```go
type ApiextensionsV1Client struct {
    restClient rest.Interface
}

func (c *ApiextensionsV1Client) CustomResourceDefinitions() CustomResourceDefinitionInterface {
    return newCustomResourceDefinitions(c)
}
```

**Resource Interface**:

```go
type CustomResourceDefinitionInterface interface {
    Create(ctx context.Context, crd *v1.CustomResourceDefinition, opts metav1.CreateOptions) (*v1.CustomResourceDefinition, error)
    Update(ctx context.Context, crd *v1.CustomResourceDefinition, opts metav1.UpdateOptions) (*v1.CustomResourceDefinition, error)
    UpdateStatus(ctx context.Context, crd *v1.CustomResourceDefinition, opts metav1.UpdateOptions) (*v1.CustomResourceDefinition, error)
    Delete(ctx context.Context, name string, opts metav1.DeleteOptions) error
    DeleteCollection(ctx context.Context, opts metav1.DeleteOptions, listOpts metav1.ListOptions) error
    Get(ctx context.Context, name string, opts metav1.GetOptions) (*v1.CustomResourceDefinition, error)
    List(ctx context.Context, opts metav1.ListOptions) (*v1.CustomResourceDefinitionList, error)
    Watch(ctx context.Context, opts metav1.ListOptions) (watch.Interface, error)
    Patch(ctx context.Context, name string, pt types.PatchType, data []byte, opts metav1.PatchOptions, subresources ...string) (*v1.CustomResourceDefinition, error)
    Apply(ctx context.Context, crd *apiextensionsv1.CustomResourceDefinitionApplyConfiguration, opts metav1.ApplyOptions) (*v1.CustomResourceDefinition, error)
    ApplyStatus(ctx context.Context, crd *apiextensionsv1.CustomResourceDefinitionApplyConfiguration, opts metav1.ApplyOptions) (*v1.CustomResourceDefinition, error)
}
```

### Usage Example

```go
package main

import (
    "context"
    apiextensionsv1 "k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/v1"
    clientset "k8s.io/apiextensions-apiserver/pkg/client/clientset/clientset"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/rest"
)

func main() {
    // Create client from kubeconfig
    config, _ := rest.InClusterConfig()
    client, _ := clientset.NewForConfig(config)

    ctx := context.Background()

    // Create CRD
    crd := &apiextensionsv1.CustomResourceDefinition{
        ObjectMeta: metav1.ObjectMeta{
            Name: "widgets.example.com",
        },
        Spec: apiextensionsv1.CustomResourceDefinitionSpec{
            Group: "example.com",
            Names: apiextensionsv1.CustomResourceDefinitionNames{
                Plural:   "widgets",
                Singular: "widget",
                Kind:     "Widget",
            },
            Scope: apiextensionsv1.NamespaceScoped,
            Versions: []apiextensionsv1.CustomResourceDefinitionVersion{...},
        },
    }
    result, err := client.ApiextensionsV1().CustomResourceDefinitions().Create(ctx, crd, metav1.CreateOptions{})

    // List CRDs
    list, err := client.ApiextensionsV1().CustomResourceDefinitions().List(ctx, metav1.ListOptions{})
    for _, crd := range list.Items {
        fmt.Println(crd.Name)
    }

    // Watch CRDs
    watcher, err := client.ApiextensionsV1().CustomResourceDefinitions().Watch(ctx, metav1.ListOptions{})
    for event := range watcher.ResultChan() {
        crd := event.Object.(*apiextensionsv1.CustomResourceDefinition)
        fmt.Printf("Event: %s, CRD: %s\n", event.Type, crd.Name)
    }
}
```

## Listers

### Purpose

Provide cached, read-only access to resources without API server round-trips.

### Structure

**Lister Interface**:

```go
type CustomResourceDefinitionLister interface {
    // List lists all CustomResourceDefinitions in the indexer matching the selector
    List(selector labels.Selector) ([]*v1.CustomResourceDefinition, error)
    // Get retrieves the CustomResourceDefinition from the index for a given name
    Get(name string) (*v1.CustomResourceDefinition, error)
}
```

**Implementation**:

```go
type customResourceDefinitionLister struct {
    indexer cache.Indexer
}

func (s *customResourceDefinitionLister) List(selector labels.Selector) ([]*v1.CustomResourceDefinition, error) {
    var ret []*v1.CustomResourceDefinition
    err := cache.ListAll(s.indexer, selector, func(m interface{}) {
        ret = append(ret, m.(*v1.CustomResourceDefinition))
    })
    return ret, err
}

func (s *customResourceDefinitionLister) Get(name string) (*v1.CustomResourceDefinition, error) {
    obj, exists, err := s.indexer.GetByKey(name)
    if err != nil {
        return nil, err
    }
    if !exists {
        return nil, errors.NewNotFound(v1.Resource("customresourcedefinition"), name)
    }
    return obj.(*v1.CustomResourceDefinition), nil
}
```

### Usage Example

```go
import (
    listers "k8s.io/apiextensions-apiserver/pkg/client/listers/apiextensions/v1"
    "k8s.io/apimachinery/pkg/labels"
)

// Get lister from informer
lister := crdInformer.Lister()

// List all CRDs
crds, err := lister.List(labels.Everything())

// Get specific CRD
crd, err := lister.Get("widgets.example.com")

// List with selector
selector := labels.SelectorFromSet(labels.Set{"app": "myapp"})
crds, err := lister.List(selector)
```

**Benefits**:
- No API server load
- Fast local reads
- Consistent with informer cache

## Informers

### Purpose

Watch resources and maintain local cache with event notifications.

### Structure

**Informer Factory**:

```go
type SharedInformerFactory interface {
    // Start starts informers that have been added
    Start(stopCh <-chan struct{})
    // WaitForCacheSync waits for all started informers to sync
    WaitForCacheSync(stopCh <-chan struct{}) map[reflect.Type]bool

    // Apiextensions returns a group interface for apiextensions
    Apiextensions() apiextensions.Interface
}
```

**Group Interface**:

```go
type Interface interface {
    V1() v1.Interface
    V1beta1() v1beta1.Interface
}
```

**Resource Informer**:

```go
type CustomResourceDefinitionInformer interface {
    Informer() cache.SharedIndexInformer
    Lister() v1.CustomResourceDefinitionLister
}
```

### Informer Architecture

```
API Server (Watch)
    ↓
Reflector (fetch and sync)
    ↓
DeltaFIFO Queue
    ↓
Event Handlers (Add/Update/Delete)
    ↓
Local Cache (Indexer)
    ↓
Listers (fast reads)
    ↓
Controllers/Reconcilers
```

### Usage Example

```go
package main

import (
    "time"
    informers "k8s.io/apiextensions-apiserver/pkg/client/informers/externalversions"
    "k8s.io/client-go/tools/cache"
)

func main() {
    // Create informer factory
    factory := informers.NewSharedInformerFactory(client, 5*time.Minute)

    // Get CRD informer
    crdInformer := factory.Apiextensions().V1().CustomResourceDefinitions()

    // Add event handler
    crdInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: func(obj interface{}) {
            crd := obj.(*apiextensionsv1.CustomResourceDefinition)
            fmt.Printf("CRD Added: %s\n", crd.Name)
        },
        UpdateFunc: func(old, new interface{}) {
            oldCRD := old.(*apiextensionsv1.CustomResourceDefinition)
            newCRD := new.(*apiextensionsv1.CustomResourceDefinition)
            fmt.Printf("CRD Updated: %s\n", newCRD.Name)
        },
        DeleteFunc: func(obj interface{}) {
            crd := obj.(*apiextensionsv1.CustomResourceDefinition)
            fmt.Printf("CRD Deleted: %s\n", crd.Name)
        },
    })

    // Start informers
    stopCh := make(chan struct{})
    defer close(stopCh)
    factory.Start(stopCh)

    // Wait for cache sync
    synced := factory.WaitForCacheSync(stopCh)
    if !synced[reflect.TypeOf(&apiextensionsv1.CustomResourceDefinition{})] {
        panic("informer not synced")
    }

    // Use lister
    lister := crdInformer.Lister()
    crds, _ := lister.List(labels.Everything())
    fmt.Printf("Found %d CRDs\n", len(crds))

    <-stopCh
}
```

### Informer Features

**Resync Period**: Periodic full list to detect missed events
**Work Queues**: Rate limiting and retry support
**Index Functions**: Custom indexing for fast lookups
**Event Handlers**: Multiple handlers per informer

**Example with Work Queue**:

```go
queue := workqueue.NewRateLimitingQueue(workqueue.DefaultControllerRateLimiter())

crdInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc: func(obj interface{}) {
        key, _ := cache.MetaNamespaceKeyFunc(obj)
        queue.Add(key)
    },
    UpdateFunc: func(old, new interface{}) {
        key, _ := cache.MetaNamespaceKeyFunc(new)
        queue.Add(key)
    },
    DeleteFunc: func(obj interface{}) {
        key, _ := cache.DeletionHandlingMetaNamespaceKeyFunc(obj)
        queue.Add(key)
    },
})

// Process queue in worker
for processNextItem(queue, lister) {
}
```

## Deep Copy Methods

### Purpose

Create independent copies of objects to prevent accidental mutations.

### Generated Methods

**For Each Type**:

```go
// DeepCopy returns a deep copy
func (in *CustomResourceDefinition) DeepCopy() *CustomResourceDefinition {
    if in == nil {
        return nil
    }
    out := new(CustomResourceDefinition)
    in.DeepCopyInto(out)
    return out
}

// DeepCopyInto copies value into out
func (in *CustomResourceDefinition) DeepCopyInto(out *CustomResourceDefinition) {
    *out = *in
    out.TypeMeta = in.TypeMeta
    in.ObjectMeta.DeepCopyInto(&out.ObjectMeta)
    in.Spec.DeepCopyInto(&out.Spec)
    in.Status.DeepCopyInto(&out.Status)
}

// DeepCopyObject returns a deep copy as runtime.Object
func (in *CustomResourceDefinition) DeepCopyObject() runtime.Object {
    if c := in.DeepCopy(); c != nil {
        return c
    }
    return nil
}
```

### Usage

**Manual Copy**:

```go
original := &v1.CustomResourceDefinition{...}
copy := original.DeepCopy()

// Modify copy without affecting original
copy.Spec.Versions = append(copy.Spec.Versions, newVersion)
```

**Informer Safety** (IMPORTANT):

```go
// Objects from listers are SHARED - must copy before modifying
crd, _ := lister.Get("widgets.example.com")

// WRONG: Modifies shared cache object!
// crd.Spec.Versions[0].Served = false

// CORRECT: Copy first
crdCopy := crd.DeepCopy()
crdCopy.Spec.Versions[0].Served = false
client.Update(ctx, crdCopy, metav1.UpdateOptions{})
```

## Fake Clients

### Purpose

Testing without real API server (unit tests).

### Structure

**Fake Clientset**:

```go
type Clientset struct {
    *testing.Fake
    tracker testing.ObjectTracker
    apiextensionsV1 *fakeapiextensionsv1.FakeApiextensionsV1
}

func NewSimpleClientset(objects ...runtime.Object) *Clientset {
    // Create fake clientset with pre-populated objects
}
```

**Fake Resource Client**:

```go
type FakeCustomResourceDefinitions struct {
    Fake *FakeApiextensionsV1
}

func (c *FakeCustomResourceDefinitions) Create(ctx context.Context, crd *v1.CustomResourceDefinition, opts metav1.CreateOptions) (*v1.CustomResourceDefinition, error) {
    obj, err := c.Fake.Invokes(testing.NewRootCreateAction(customresourcedefinitionsResource, crd), &v1.CustomResourceDefinition{})
    return obj.(*v1.CustomResourceDefinition), err
}
```

### Testing Example

```go
package mycontroller_test

import (
    "testing"
    apiextensionsv1 "k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/v1"
    fakeclientset "k8s.io/apiextensions-apiserver/pkg/client/clientset/clientset/fake"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

func TestMyController(t *testing.T) {
    // Create fake client with initial objects
    crd := &apiextensionsv1.CustomResourceDefinition{
        ObjectMeta: metav1.ObjectMeta{Name: "widgets.example.com"},
        Spec:       apiextensionsv1.CustomResourceDefinitionSpec{...},
    }
    client := fakeclientset.NewSimpleClientset(crd)

    // Run controller
    controller := NewController(client)
    err := controller.Reconcile("widgets.example.com")
    if err != nil {
        t.Fatalf("reconcile failed: %v", err)
    }

    // Verify actions
    actions := client.Actions()
    if len(actions) != 2 {
        t.Errorf("expected 2 actions, got %d", len(actions))
    }

    // Verify update action
    updateAction := actions[1].(testing.UpdateAction)
    updatedCRD := updateAction.GetObject().(*apiextensionsv1.CustomResourceDefinition)
    if !updatedCRD.Status.Conditions[0].Status {
        t.Error("expected condition to be true")
    }
}
```

## Apply Configurations

### Purpose

Type-safe server-side apply (SSA) support.

### Structure

**Apply Configuration**:

```go
type CustomResourceDefinitionApplyConfiguration struct {
    *metav1apply.TypeMetaApplyConfiguration
    *metav1apply.ObjectMetaApplyConfiguration
    Spec   *CustomResourceDefinitionSpecApplyConfiguration
    Status *CustomResourceDefinitionStatusApplyConfiguration
}

func CustomResourceDefinition(name string) *CustomResourceDefinitionApplyConfiguration {
    return &CustomResourceDefinitionApplyConfiguration{
        ObjectMetaApplyConfiguration: &metav1apply.ObjectMetaApplyConfiguration{
            Name: &name,
        },
    }
}
```

### Usage

```go
import (
    apiextensionsv1 "k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/v1"
    apiextensionsapply "k8s.io/apiextensions-apiserver/pkg/client/applyconfiguration/apiextensions/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// Create apply configuration
crdApply := apiextensionsapply.CustomResourceDefinition("widgets.example.com").
    WithSpec(apiextensionsapply.CustomResourceDefinitionSpec().
        WithGroup("example.com").
        WithNames(apiextensionsapply.CustomResourceDefinitionNames().
            WithPlural("widgets").
            WithSingular("widget").
            WithKind("Widget")).
        WithScope(apiextensionsv1.NamespaceScoped))

// Apply with field manager
result, err := client.ApiextensionsV1().CustomResourceDefinitions().
    Apply(context.TODO(), crdApply, metav1.ApplyOptions{
        FieldManager: "my-controller",
        Force:        false,
    })
```

**Benefits**:
- Declarative configuration
- Field ownership tracking
- Conflict detection
- Partial updates

## Critical Files Reference

| File | Purpose | Lines | Importance |
|------|---------|-------|------------|
| `pkg/client/clientset/clientset/clientset.go` | Main clientset | ~100 | Critical |
| `pkg/client/clientset/clientset/typed/apiextensions/v1/customresourcedefinition.go` | CRD client | ~200 | Critical |
| `pkg/client/informers/externalversions/factory.go` | Informer factory | ~150 | High |
| `pkg/client/listers/apiextensions/v1/customresourcedefinition.go` | CRD lister | ~100 | High |
| `hack/update-codegen.sh` | Code generation script | ~100 | High |

## Related Documentation

- **API Server**: [api-server.md](./api-server.md) - Loopback client usage
- **CRD Lifecycle**: [crd-lifecycle.md](./crd-lifecycle.md) - Controllers using clients

## Recommendations

### 1. Generation Automation
Integrate code generation with CI/CD:
- Pre-commit hooks for verification
- CI checks for generated code freshness
- Automated regeneration on API changes

### 2. Documentation Generation
Generate API documentation from code:
- GoDoc comments on generated types
- OpenAPI specs from markers
- Usage examples in documentation

### 3. Validation
Verify generated code correctness:
- Unit tests for generated clients
- Integration tests with real API server
- Roundtrip tests (create → read → verify)

### 4. Caching Tuning
Optimize informer performance:
- Tune resync periods based on workload
- Use selective informers (field/label selectors)
- Monitor cache memory usage

### 5. Testing Best Practices
Improve test coverage:
- Use fake clients for unit tests
- Use real clients for integration tests
- Test error handling paths
- Verify proper deep copying
