# Codebase Structure

## Root Directory
```
/
├── main.go                 # Entry point for the API server binary
├── go.mod                  # Go module definition
├── go.sum                  # Go module checksums
├── README.md               # Project overview
├── CLAUDE.md              # Development guidance for Claude Code
├── CONTRIBUTING.md         # Contributing guidelines (points to main k8s repo)
├── LICENSE                 # Apache 2.0 license
├── SECURITY_CONTACTS       # Security contact information
├── code-of-conduct.md      # Code of conduct
└── OWNERS                  # OWNERS file for code review
```

## Main Directories

### `/pkg` - Main Package Directory
Core implementation of the API server:

- **`pkg/apiserver/`** - Main API server implementation
  - `apiserver.go` - Core API server setup and configuration
  - `customresource_handler.go` - Handles CRUD operations for custom resources
  - `customresource_discovery_controller.go` - API discovery management
  - `schema/` - Schema validation, CEL expressions, OpenAPI handling
    - `cel/` - Common Expression Language validation
    - `defaulting/` - Field defaulting algorithms

- **`pkg/apis/apiextensions/`** - API type definitions
  - `v1/` - v1 API version (stable)
  - `v1beta1/` - v1beta1 API version
  - Type definitions for CustomResourceDefinitions

- **`pkg/client/`** - Generated client libraries
  - Clientset for interacting with the API
  - Listers and informers

- **`pkg/cmd/`** - Command-line interface
  - `server/` - Server command implementation using Cobra

- **`pkg/controller/`** - Controllers
  - Background controllers for managing CRD lifecycle

- **`pkg/registry/`** - Registry implementations
  - REST storage implementations for API types

- **`pkg/generated/`** - Generated code
  - OpenAPI specifications
  - Other generated artifacts

- **`pkg/features/`** - Feature gates
- **`pkg/crdserverscheme/`** - CRD server scheme registration
- **`pkg/apihelpers/`** - API helper utilities
- **`pkg/test/`** - Test utilities

### `/test` - Testing
```
test/
└── integration/    # Integration tests for the API server
```

### `/hack` - Build and Development Scripts
```
hack/
├── update-codegen.sh       # Updates generated code (clients, deepcopy, OpenAPI)
├── verify-codegen.sh       # Verifies generated code is up-to-date
├── build-image.sh          # Builds Docker image
└── boilerplate.go.txt      # License header template for generated files
```

### `/examples` - Example Resources
Example CRD definitions and usage

### `/.github` - GitHub Configuration
CI/CD workflows and GitHub-specific configuration

### `/artifacts` - Build Artifacts
Output directory for build artifacts

## Key Files

- **`main.go:22`** - Entry point that creates and runs the server command
- **`pkg/apiserver/apiserver.go`** - Main API server configuration and initialization
- **`pkg/cmd/server/server.go`** - Cobra command setup for the server

## Code Organization Patterns

### Generated vs Manual Code
- Files starting with `zz_generated.` are generated and should not be edited manually
- Run `./hack/update-codegen.sh` to regenerate after API changes

### Versioned APIs
API types follow Kubernetes versioning:
- Internal types (unversioned)
- v1beta1 (older, may be deprecated)
- v1 (stable)

### Standard Patterns
- Uses Kubernetes API machinery patterns
- REST storage implementation
- Scheme and codec management
- Delegated API server model
