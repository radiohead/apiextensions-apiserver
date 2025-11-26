# Task Completion Checklist

When completing a development task in this project, follow these steps:

## 1. Code Generation Check
If you modified any API types or added new types:
```bash
# Update generated code
./hack/update-codegen.sh

# Verify it's up-to-date
./hack/verify-codegen.sh
```

## 2. Testing
Always run tests to ensure nothing broke:
```bash
# Run all tests
go test ./...

# For specific changes, run targeted tests
go test ./pkg/apiserver/...
go test ./test/integration/...
```

## 3. Code Quality
Ensure code quality standards:
```bash
# Format code
go fmt ./...

# Vet for common issues
go vet ./...

# Tidy dependencies if you added/removed any
go mod tidy
```

## 4. Verify Build
Make sure the project builds successfully:
```bash
go build ./...
```

## 5. Documentation
- Update comments for exported functions/types
- Update CLAUDE.md if you discovered new patterns or important information
- Do NOT create new documentation files unless explicitly requested

## Important Notes
- **This is a read-only staging repository**: Do not open PRs or issues here
- Any real contributions must go to https://github.com/kubernetes/kubernetes
- Generated files (prefixed with `zz_generated.`) should never be manually edited
- Always verify code generation is up-to-date before committing

## Pre-commit Checklist
- [ ] Tests pass (`go test ./...`)
- [ ] Code is formatted (`go fmt ./...`)
- [ ] Code is vetted (`go vet ./...`)
- [ ] Generated code is up-to-date (`./hack/verify-codegen.sh`)
- [ ] Build succeeds (`go build ./...`)
- [ ] Dependencies are tidy (`go mod tidy`)
