# API Extensions API Server - Architecture Documentation

**Last Updated**: 2025-11-25
**Analysis Confidence**: 90% (High)
**Codebase**: 119,053 LOC across 296 Go files

## What is apiextensions-apiserver?

The Kubernetes API Extensions API Server is a **delegated API server** that extends Kubernetes with CustomResourceDefinition (CRD) support. It enables dynamic registration of new resource types without recompiling or restarting the API server, providing the foundation for Kubernetes' extensibility model.

This is a **staging repository** - all contributions should be made to the main Kubernetes repository at https://github.com/kubernetes/kubernetes.

## Start Here

Choose your path based on your role:

### 🆕 New Contributors
Start with [Architecture Overview](overview.md) to understand the high-level system, then explore [Core Components](core-components/) to dive into specific areas.

### ⚙️ Operators & SREs
Begin with [Architecture Overview](overview.md), then focus on [Operations](operations/) for performance, security, and troubleshooting guidance.

### 👨‍💻 Feature Developers
Review [Core Components](core-components/) to understand implementation details, then study [Concepts](concepts/) for patterns and design decisions.

### 🚀 Performance Engineers
Jump to [Operations/Performance](operations/performance.md) for scalability limits and optimization strategies, then review [Critical Paths](reference/critical-paths.md).

## Documentation Structure

### 📋 Overview
- **[Architecture Overview](overview.md)** - Executive summary, high-level architecture diagram, key capabilities

### 🔧 Core Components
Detailed deep dives into the 8 major subsystems:

- **[README](core-components/README.md)** - Components overview and dependency graph
- **[API Server](core-components/api-server.md)** - Server initialization, configuration, delegation pattern
- **[CRD Lifecycle](core-components/crd-lifecycle.md)** - CRD registration, 7 lifecycle controllers, status management
- **[Custom Resource Handler](core-components/custom-resource-handler.md)** - CR CRUD operations, atomic storage, request routing
- **[Schema Validation](core-components/schema-validation.md)** - Structural schemas, validation pipeline, Kubernetes extensions
- **[CEL Validation](core-components/cel-validation.md)** - Common Expression Language integration, compilation, cost management
- **[Version Conversion](core-components/version-conversion.md)** - Multi-version APIs, webhook protocol, storage migration
- **[Discovery & OpenAPI](core-components/discovery-openapi.md)** - API discovery, aggregated discovery, OpenAPI v2/v3 generation
- **[Client Libraries](core-components/client-libraries.md)** - Code generation, clientsets, informers, listers

### 💡 Concepts
Architectural patterns and design principles:

- **[README](concepts/README.md)** - Concepts overview
- **[Architectural Patterns](concepts/architectural-patterns.md)** - 5 core patterns (Controller, Delegated API Server, Dynamic Registration, Multi-Version, Atomic Updates)
- **[Design Decisions](concepts/design-decisions.md)** - 7 key decisions with rationale and trade-offs
- **[Data Flows](concepts/data-flows.md)** - Step-by-step flow diagrams (CRD creation, CR creation, CR watch)
- **[Request Lifecycle](concepts/request-lifecycle.md)** - Complete HTTP request pipeline from entry to response

### 🔬 Operations
Running and maintaining the system:

- **[README](operations/README.md)** - Operations overview
- **[Performance](operations/performance.md)** - Scalability limits, latency targets, hot paths, optimization strategies
- **[Security](operations/security.md)** - RBAC, admission control, API approval policy, webhook security
- **[Testing](operations/testing.md)** - Integration tests, table-driven patterns, test infrastructure
- **[Troubleshooting](operations/troubleshooting.md)** - Known limitations, common issues, debugging strategies

### 📚 Reference
Quick lookups and metadata:

- **[README](reference/README.md)** - Reference overview
- **[Dependencies](reference/dependencies.md)** - Core dependencies, integration points, version constraints
- **[Critical Paths](reference/critical-paths.md)** - Hot paths, critical files table, performance-sensitive code
- **[Confidence Assessment](reference/confidence-assessment.md)** - Per-domain confidence scores, analysis methodology
- **[Future Considerations](reference/future-considerations.md)** - Potential improvements, API evolution, research directions

## Quick Reference

### Critical Files

| File | Purpose | Lines | Hot Path |
|------|---------|-------|----------|
| `pkg/apiserver/customresource_handler.go` | CR request handling | 1,746 | ✅ Critical |
| `pkg/apiserver/schema/validation.go` | Schema validation | 15,303 | ✅ Critical |
| `pkg/apiserver/schema/cel/validation.go` | CEL evaluation | 994 | ✅ Critical |
| `pkg/apiserver/apiserver.go` | Server initialization | 286 | Startup only |

See [Critical Paths](reference/critical-paths.md) for complete list.

### Request Flow (High-Level)

```
HTTP Request → Custom Resource Handler → Schema Validation → CEL Rules
→ Admission Webhooks → Version Conversion → Storage (etcd) → Response
```

See [Request Lifecycle](concepts/request-lifecycle.md) and [Data Flows](concepts/data-flows.md) for detailed diagrams.

### Performance Targets

| Operation | Target Latency | Notes |
|-----------|---------------|-------|
| CR Get/List | < 50ms | Without webhooks |
| CR Create/Update | < 100ms | Includes validation |
| With Webhooks | +10-50ms | Network overhead |

See [Operations/Performance](operations/performance.md) for complete characteristics.

## Analysis Metadata

- **Session ID**: `codebase-dedfc2ae-20251125-171241`
- **Analysis Strategy**: Comprehensive (10 domains)
- **Analysis Date**: 2025-11-25
- **Overall Confidence**: 90% (High)
- **Codebase Size**: 119,053 LOC across 296 Go files
- **Primary Language**: Go 1.25.0+

**Original Analysis Artifacts**: `.claude/codebase/codebase-dedfc2ae-20251125-171241/`
- Raw findings for each domain
- Complete synthesis report
- Statistics and metadata

## Contributing to Documentation

This documentation is organized to make it easy to find and update information:

### Adding New Content

- **New component docs**: Add to `core-components/` and update the index
- **New patterns**: Add to `concepts/`
- **Operational guides**: Add to `operations/`
- **Reference material**: Add to `reference/`

### Consistency Guidelines

- Use relative links for cross-references
- Include code file references with line numbers (e.g., `file.go:123`)
- Keep individual files focused (150-700 lines)
- Update README indexes when adding new files
- Reference original analysis artifacts when applicable

### Documentation Principles

1. **Audience-Driven**: Consider who needs this information
2. **Navigation First**: Make it easy to find information
3. **No Duplication**: Single source of truth for each topic
4. **Cross-Reference**: Link related concepts
5. **Maintainable**: Clear organization makes updates easier

## Related Resources

- **[CLAUDE.md](../../CLAUDE.md)** - Development guidance for Claude Code
- **[README.md](../../README.md)** - Project overview
- **[CONTRIBUTING.md](../../CONTRIBUTING.md)** - Contribution guidelines
- **Original Kubernetes Design Proposal**: [thirdpartyresources.md](https://github.com/kubernetes/design-proposals-archive/blob/main/api-machinery/thirdpartyresources.md)
