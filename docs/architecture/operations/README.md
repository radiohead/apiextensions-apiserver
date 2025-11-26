# Operations Documentation

This directory contains operational guides for running, monitoring, securing, testing, and troubleshooting the Kubernetes API Extensions API Server.

## Contents

### [Performance Guide](./performance.md)
Performance characteristics, scalability limits, latency targets, hot paths, optimization strategies, and bottleneck analysis.

**Topics covered**:
- Scalability limits (CRD count, CR count, watch connections)
- Memory usage per CRD and version
- Latency targets for operations
- Critical path analysis (hot paths)
- Optimization strategies (atomic storage, caching, compilation)
- Performance bottlenecks and mitigation

**When to read**: Before capacity planning, performance tuning, or investigating latency issues.

### [Security Guide](./security.md)
Security model, RBAC integration, admission control, API approval policy, webhook security, and storage encryption.

**Topics covered**:
- RBAC integration for custom resources
- Admission control (mutating/validating webhooks)
- API approval policy for k8s.io namespaces
- Webhook security (TLS, CA bundles, timeouts)
- Storage encryption (etcd encryption at rest)
- Audit logging support

**When to read**: Before deploying to production, implementing webhooks, or configuring RBAC policies.

### [Testing Guide](./testing.md)
Test strategy, integration tests, test coverage, table-driven patterns, test infrastructure, and test execution.

**Topics covered**:
- Integration test architecture
- Test server setup and harness
- Major test categories (lifecycle, validation, conversion, CRUD)
- Table-driven test patterns
- Test utilities and helpers
- Running tests (all, specific, verbose)

**When to read**: Before writing tests, investigating test failures, or understanding test coverage.

### [Troubleshooting Guide](./troubleshooting.md)
Known limitations, common issues, debugging strategies, workarounds, and operational tips.

**Topics covered**:
- 7 known limitations (schema migration, storage version, conversion performance, etc.)
- Common operational issues
- Debugging strategies and tools
- Workarounds and best practices
- Monitoring and observability

**When to read**: When experiencing issues, investigating unexpected behavior, or planning migrations.

## Quick Reference

### Performance Targets

| Operation | Target Latency | Notes |
|-----------|---------------|-------|
| CR Get/List | < 50ms | Without webhooks |
| CR Create/Update | < 100ms | Includes validation, no webhooks |
| With Webhooks | +10-50ms | Network + processing overhead |
| Discovery | < 10ms | Client-side cached (10 min kubectl) |
| Watch Event | < 100ms | Event propagation time |

### Scalability Limits

| Resource | Limit | Notes |
|----------|-------|-------|
| CRD Count | 100s | Tested, discovery degrades beyond |
| CR Count | Millions | Per CRD, etcd-limited |
| Watch Connections | Thousands | Concurrent watchers |
| Version Count | 10-20 | Per CRD, typical range |

### Security Checklist

- [ ] RBAC rules configured for custom resources
- [ ] Webhook TLS certificates valid and not expiring soon
- [ ] API approval annotation for k8s.io groups
- [ ] etcd encryption at rest enabled (if required)
- [ ] Admission policies configured appropriately
- [ ] Audit logging enabled for compliance

### Common Commands

**Check CRD Status**:
```bash
kubectl get crd <crd-name> -o yaml
kubectl describe crd <crd-name>
```

**Debug Validation Issues**:
```bash
kubectl create -f resource.yaml --dry-run=server -v=8
```

**Check Webhook Configuration**:
```bash
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations
```

**View Controller Status**:
```bash
kubectl logs -n kube-system <apiserver-pod> | grep controller
```

**Check Discovery**:
```bash
kubectl api-resources | grep <group>
kubectl api-versions | grep <group>
```

## Related Documentation

### Core Components
- [Custom Resource Handler](../core-components/custom-resource-handler.md) - Request handling and storage
- [Schema Validation](../core-components/schema-validation.md) - Validation pipeline
- [CEL Integration](../core-components/cel-integration.md) - CEL validation rules
- [Lifecycle Controllers](../core-components/lifecycle-controllers.md) - CRD lifecycle management

### Concepts
- [CRD Lifecycle](../concepts/crd-lifecycle.md) - CRD creation and status
- [Multi-Version Support](../concepts/multi-version-support.md) - Version evolution
- [Discovery System](../concepts/discovery-system.md) - API discovery

### Reference
- [Architecture Overview](../overview.md) - High-level architecture
- [Request Flow](../reference/request-flow.md) - End-to-end request processing

## Contributing

When updating operations documentation:

1. **Keep it actionable**: Focus on practical guidance, commands, and examples
2. **Include metrics**: Provide specific numbers (latency, memory, etc.)
3. **Add examples**: Show real-world scenarios and solutions
4. **Cross-reference**: Link to related core components
5. **Update regularly**: Keep performance numbers and best practices current

## Support

For operational issues:

1. Check the [Troubleshooting Guide](./troubleshooting.md)
2. Review [Known Limitations](./troubleshooting.md#known-limitations)
3. Search Kubernetes issues: https://github.com/kubernetes/kubernetes/issues
4. Ask in Kubernetes Slack: #sig-api-machinery

For security issues:
- Follow Kubernetes security reporting process
- Do not open public issues for vulnerabilities
- Contact security@kubernetes.io
