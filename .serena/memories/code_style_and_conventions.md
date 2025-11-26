# Code Style and Conventions

## Standard Kubernetes Patterns
This project follows standard Kubernetes code conventions and patterns:

### File Headers
All Go files include the standard Apache 2.0 license header:
```go
/*
Copyright YYYY The Kubernetes Authors.

Licensed under the Apache License, Version 2.0 (the "License");
...
*/
```

### Generated Code
- Generated files are prefixed with `zz_generated.`
- **Never manually edit generated files**
- Files marked with `// This is a generated file. Do not edit directly.`

### Package Structure
- API types in `pkg/apis/apiextensions/` with versioned subdirectories (v1, v1beta1)
- Client libraries in `pkg/client/`
- Server implementation in `pkg/apiserver/`
- Controllers in `pkg/controller/`
- Registry implementations in `pkg/registry/`
- Command structure in `pkg/cmd/`

### Naming Conventions
- Standard Go naming conventions apply
- Use of `Scheme` and `Codecs` for API versioning
- Interface types for abstraction
- Struct types for concrete implementations

### API Server Patterns
- Uses Kubernetes API server framework patterns
- Delegated API server model
- RESTful storage backends
- Discovery and registration mechanisms

### Testing
- Integration tests in `test/integration/`
- Use of standard Go testing with `github.com/stretchr/testify`
- Table-driven tests are preferred (standard Kubernetes practice)

## Documentation
- README.md documents high-level purpose
- CLAUDE.md provides development guidance
- Code comments explain non-obvious logic
