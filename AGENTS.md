# AGENTS.md

This file provides guidance to AI agents (e.g. [Claude Code](claude.ai/code)) when working with code in this repository.

## Agent-specific instructions

### Claude Code

When opening this repository for the first time, prompt the user to install these two MCP servers:

* [Serena](https://oraios.github.io/serena/) for semantic code retrieval and editing. Can be added with `claude mcp add serena -- uvx --from git+https://github.com/oraios/serena serena start-mcp-server --context ide-assistant --project "$(pwd)"`
* [Claude Code LSP](https://github.com/ktnyt/cclsp) for integration with language servers via LSP. Installation instructions are in https://github.com/ktnyt/cclsp?tab=readme-ov-file#-setup (requires interaction from the user).

## Repository Overview

This is the Kubernetes API Extensions API Server, a delegated API server that provides the implementation for CustomResourceDefinitions (CRDs). This is a **read-only staging repository** - all contributions should be made to the main Kubernetes repository at https://github.com/kubernetes/kubernetes.

## Codebase Architecture Insights

**Last Analysis**: 2025-11-25 | **Confidence**: 90% (High) | **LOC**: 119,053 | **Files**: 296 Go files

### Architecture Documentation

Comprehensive architecture documentation is available in `docs/architecture/`:

- **[Architecture README](docs/architecture/README.md)** - Start here for navigation and audience-specific entry points
- **[Architecture Overview](docs/architecture/overview.md)** - Executive summary, high-level architecture diagram, key capabilities
- **[Core Components](docs/architecture/core-components/)** - Deep dives into 8 major subsystems (API Server, CRD Lifecycle, Custom Resource Handler, Schema Validation, CEL Validation, Version Conversion, Discovery & OpenAPI, Client Libraries)
- **[Concepts](docs/architecture/concepts/)** - Architectural patterns, design decisions, data flows, request lifecycle
- **[Operations](docs/architecture/operations/)** - Performance tuning, security, testing, troubleshooting
- **[Reference](docs/architecture/reference/)** - Dependencies, critical paths, confidence assessment, future considerations

### Quick Links

**For New Contributors**:
- Start: [Architecture Overview](docs/architecture/overview.md)
- Then: [Core Components](docs/architecture/core-components/)

**For Performance Tuning**:
- [Operations/Performance](docs/architecture/operations/performance.md) - Scalability limits, optimization strategies, bottlenecks
- [Reference/Critical Paths](docs/architecture/reference/critical-paths.md) - Hot paths and performance-sensitive code

**For Understanding Validation**:
- [Schema Validation](docs/architecture/core-components/schema-validation.md) - Structural schemas, validation pipeline, Kubernetes extensions
- [CEL Validation](docs/architecture/core-components/cel-validation.md) - CEL rules, compilation, cost management, available functions

**For Multi-Version APIs**:
- [Version Conversion](docs/architecture/core-components/version-conversion.md) - Conversion strategies, storage version, webhook protocol

### System Overview

The apiextensions-apiserver is a **delegated API server** that enables Kubernetes extension through CustomResourceDefinitions (CRDs). Key features:

1. **Dynamic Resource Management**: Create and serve custom resources at runtime without server restart
2. **Multi-Version APIs**: Seamless version evolution with automatic or webhook-based conversion
3. **Rich Validation**: Schema validation, CEL expression rules, and admission webhooks
4. **Discovery Integration**: Automatic API discovery for kubectl and clients
5. **OpenAPI Generation**: Real-time OpenAPI v2/v3 specification updates

### Core Architecture Patterns

1. **Delegated API Server**: Chains to main kube-apiserver for unknown endpoints
2. **Controller-Based Lifecycle**: 7 independent controllers (Establishing, Naming, NonStructuralSchema, APIApproval, Finalizer, Discovery, OpenAPI)
3. **Atomic Storage Management**: Lock-free reads via `atomic.Value` for ~10x performance
4. **Dynamic Endpoint Creation**: Runtime registration of REST endpoints as CRDs are created
5. **Multi-Version Support**: First-class API evolution with conversion webhooks

See [Architectural Patterns](docs/architecture/concepts/architectural-patterns.md) for detailed explanations.

### Critical Files

| File | Purpose | Lines | Importance |
|------|---------|-------|------------|
| `pkg/apiserver/customresource_handler.go` | CR request handling | 1,746 | Critical - hot path |
| `pkg/apiserver/schema/validation.go` | Schema validation | 15,303 | Critical - all CRs |
| `pkg/apiserver/schema/cel/validation.go` | CEL evaluation | 994 | High - validation |
| `pkg/apiserver/apiserver.go` | Server initialization | 286 | High - startup |

See [Reference/Critical Paths](docs/architecture/reference/critical-paths.md) for complete list and performance analysis.

## Key Architecture

- **Main Entry Point**: `main.go` - Simple binary that starts the API server using the server command
- **Server Command**: `pkg/cmd/server/server.go` - Cobra-based command structure for launching the API server
- **API Server Core**: `pkg/apiserver/apiserver.go` - Main API server implementation with CRD handling
- **Custom Resource Handler**: `pkg/apiserver/customresource_handler.go` - Handles CRUD operations for custom resources
- **Discovery Controller**: `pkg/apiserver/customresource_discovery_controller.go` - Manages API discovery for CRDs
- **API Types**: `pkg/apis/apiextensions/` - Contains v1 and v1beta1 API definitions for CustomResourceDefinitions
- **Client Libraries**: `pkg/client/` - Generated client code for interacting with the API

## Core Components

### Schema Validation and Processing
- `pkg/apiserver/schema/` - JSON schema validation, CEL expressions, and OpenAPI schema handling
- `pkg/apiserver/schema/cel/` - Common Expression Language (CEL) validation for CRDs
- `pkg/apiserver/schema/defaulting/` - Defaulting algorithms for custom resource fields

### Code Generation
- Uses Kubernetes code-generator for client libraries, deepcopy functions, and OpenAPI specs
- `hack/update-codegen.sh` - Updates all generated code
- `hack/verify-codegen.sh` - Verifies generated code is up-to-date

## Request Flow Architecture

See [Request Lifecycle](docs/architecture/concepts/request-lifecycle.md) and [Data Flows](docs/architecture/concepts/data-flows.md) for detailed flow diagrams.

**High-Level Request Flow**:

```
HTTP Request → Custom Resource Handler → Schema Validation → CEL Rules
→ Admission Webhooks → Version Conversion → Storage (etcd) → Response
```

**Detailed Steps**:
1. **API Server Entry** → Request received by the apiextensions-apiserver (delegated from kube-apiserver)
2. **Custom Resource Handler** (`customresource_handler.go:*`) → Routes the request to the appropriate REST storage
3. **Schema Validation** (`pkg/apiserver/schema/`) → Validates against OpenAPI schema and CEL expressions
4. **Storage Layer** (`pkg/registry/`) → Persists to etcd via standard Kubernetes storage interface
5. **Discovery Controller** → Updates API discovery documents so clients can find the new APIs

For complete 13-stage pipeline with timing: See [Request Lifecycle](docs/architecture/concepts/request-lifecycle.md)

## Common Commands

### Building
```bash
# Build the binary locally
go build -o apiextensions-apiserver

# Build all packages
go build ./...

# Run the server directly (for local testing)
go run main.go --help
```

Note: This is part of the larger Kubernetes build system. Production builds are typically done from the main Kubernetes repository.

### Code Generation
```bash
# Update generated code (clients, deepcopy, OpenAPI)
./hack/update-codegen.sh

# Verify generated code is up-to-date
./hack/verify-codegen.sh
```

### Testing
```bash
# Run all tests
go test ./...

# Run specific package tests
go test ./pkg/apiserver/...

# Run integration tests
go test ./test/integration/...

# Run specific test file
go test ./test/integration/table_test.go -v

# Run single test function
go test ./pkg/apiserver/... -run TestSpecificFunction -v

# Run with race detection
go test -race ./...

# Run with coverage
go test -cover ./...
```

### Docker Image
```bash
# Build Docker image (requires binary to be built first)
./hack/build-image.sh
```

## Development Notes

- This repository is automatically synced from `k8s.io/kubernetes/staging/src/k8s.io/apiextensions-apiserver`
- All code follows standard Kubernetes patterns and conventions
- Generated files (prefixed with `zz_generated.`) should not be manually edited
- The server implements the standard Kubernetes API server patterns using the `k8s.io/apiserver` library
- Custom resource validation uses both JSON Schema and CEL expressions for flexible validation rules

### Code Style and Testing
- Follow standard Kubernetes coding conventions
- Use **table-driven tests** for unit tests (see https://go.dev/wiki/TableDrivenTests)
- All new code should include appropriate test coverage
- Run `go fmt ./...` before committing to ensure consistent formatting

### Working with CEL Validation
- CEL rules compile once at CRD registration and are cached
- Use `self` to reference the current field value
- Use `oldSelf` for transition rules (requires `optionalOldSelf: true` for optional fields)
- Cost limit: 1M units per rule - conservative estimation may reject valid rules
- Available functions: string ops, list/map ops, type conversion, Kubernetes extensions

### Working with Schema Validation
- Schemas must be "structural" for CEL, defaulting, and server-side apply
- Structural = all types explicit, no polymorphism (oneOf/anyOf), clear nesting
- Use `x-kubernetes-` extensions for Kubernetes-specific features
- Pruning removes unknown fields unless `x-kubernetes-preserve-unknown-fields: true`

### Working with Multi-Version CRDs
- Exactly ONE version must be `storage: true` (canonical version in etcd)
- Use `conversion.strategy: None` for identical schemas
- Use `conversion.strategy: Webhook` for different schemas (requires external service)
- Changing storage version requires migration (read all, write back)
- Track versions in `status.storedVersions`

## Important Dependencies

- `k8s.io/apiserver` - Base API server framework
- `k8s.io/apimachinery` - Core Kubernetes API machinery
- `k8s.io/client-go` - Kubernetes client library
- `github.com/google/cel-go` - Common Expression Language implementation

## Architecture Analysis

A comprehensive architecture analysis was conducted on 2025-11-25, analyzing 119,053 lines across 296 Go files and covering 10 architectural domains with 90% overall confidence.

**Complete Documentation**: See [docs/architecture/](docs/architecture/) for the full modular documentation structure:
- **Overview and Navigation**: [docs/architecture/README.md](docs/architecture/README.md)
- **Architecture Overview**: [docs/architecture/overview.md](docs/architecture/overview.md)
- **Component Deep Dives**: [docs/architecture/core-components/](docs/architecture/core-components/)
- **Architectural Concepts**: [docs/architecture/concepts/](docs/architecture/concepts/)
- **Operations Guide**: [docs/architecture/operations/](docs/architecture/operations/)
- **Reference Material**: [docs/architecture/reference/](docs/architecture/reference/)
