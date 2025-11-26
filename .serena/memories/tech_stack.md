# Tech Stack

## Language
- **Go**: Version 1.25.0+ (current environment: go1.25.4 darwin/arm64)
- **godebug**: default=go1.25

## Core Dependencies
- `k8s.io/apiserver` - Base API server framework
- `k8s.io/apimachinery` - Core Kubernetes API machinery
- `k8s.io/client-go` - Kubernetes client library
- `k8s.io/code-generator` - Code generation tools
- `k8s.io/component-base` - Kubernetes component utilities
- `k8s.io/kube-openapi` - OpenAPI specification tools

## Key Libraries
- **CLI**: `github.com/spf13/cobra` and `github.com/spf13/pflag`
- **CEL (Common Expression Language)**: `github.com/google/cel-go` v0.26.0 - For CRD validation
- **REST API**: `github.com/emicklei/go-restful/v3`
- **Data Storage**: `go.etcd.io/etcd/client/v3` - etcd client
- **Telemetry**: `go.opentelemetry.io/otel` - OpenTelemetry for tracing
- **Serialization**: 
  - `github.com/fxamacker/cbor/v2` - CBOR encoding
  - `gopkg.in/evanphx/json-patch.v4` - JSON patching
  - `sigs.k8s.io/yaml` - YAML handling
- **Testing**: `github.com/stretchr/testify`

## Build System
- Standard Go toolchain (`go build`, `go test`)
- Shell scripts in `hack/` directory for code generation and verification
- No Makefile present - uses direct Go commands and shell scripts
