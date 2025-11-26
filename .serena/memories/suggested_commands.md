# Suggested Commands

## Testing
```bash
# Run all tests
go test ./...

# Run tests for specific package
go test ./pkg/apiserver/...

# Run integration tests
go test ./test/integration/...

# Run tests with verbose output
go test -v ./...

# Run tests with race detection
go test -race ./...
```

## Building
```bash
# Build the binary
go build -o apiextensions-apiserver

# Build all packages
go build ./...

# Run the main binary
go run main.go
```

## Code Generation
```bash
# Update all generated code (clients, deepcopy, OpenAPI)
./hack/update-codegen.sh

# Verify generated code is up-to-date
./hack/verify-codegen.sh
```

## Docker
```bash
# Build Docker image (requires binary to be built first)
./hack/build-image.sh
```

## Code Quality
```bash
# Format code
go fmt ./...

# Vet code for issues
go vet ./...

# Run linter (if golangci-lint is installed)
golangci-lint run

# Tidy dependencies
go mod tidy

# Verify dependencies
go mod verify
```

## Git Commands (Darwin-specific)
Standard git commands work on Darwin (macOS):
```bash
git status
git diff
git add .
git commit
git push
```

## Useful Darwin Commands
```bash
# List files with details
ls -la

# Find files
find . -name "*.go"

# Search in files
grep -r "pattern" .

# Count lines
wc -l file.go

# Directory tree (if tree is installed via brew)
tree -L 2
```

## Module Management
```bash
# Download dependencies
go mod download

# View dependency graph
go mod graph

# Check for updates
go list -u -m all
```
