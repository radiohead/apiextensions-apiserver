# Troubleshooting Guide

This guide covers known limitations, common issues, debugging strategies, and operational troubleshooting for the Kubernetes API Extensions API Server.

## Overview

Understanding common failure modes and debugging techniques is essential for operating the apiextensions-apiserver in production. This guide provides practical troubleshooting steps for the most common issues.

## Known Limitations

### 1. Schema Migration

**Limitation**: No automatic schema migration on breaking changes

**Impact**:
- Existing custom resources may become invalid after schema changes
- No automatic data transformation on schema updates
- Manual intervention required for incompatible changes

**Scenario**:
```yaml
# Original schema (v1)
spec:
  replicas: 3  # type: integer

# Updated schema (breaking change)
spec:
  replicas: "3"  # type: string (incompatible!)
```

**Workaround**:
1. Add new version with updated schema
2. Migrate resources manually
3. Deprecate old version
4. Eventually remove old version

**Example Migration**:
```bash
# 1. Add v2 version with new schema
kubectl apply -f crd-with-v2.yaml

# 2. Read all v1 resources and create v2 equivalents
kubectl get myresources.v1.mygroup.example.com -o json | \
  jq '.items[] | .apiVersion = "mygroup.example.com/v2" | .spec.replicas = (.spec.replicas | tostring)' | \
  kubectl create -f -

# 3. Delete v1 resources
kubectl delete myresources.v1.mygroup.example.com --all

# 4. Update CRD to deprecate v1
kubectl apply -f crd-v1-deprecated.yaml

# 5. Eventually remove v1 (after grace period)
kubectl apply -f crd-v2-only.yaml
```

**Best Practice**:
- Avoid breaking changes whenever possible
- Use new versions for incompatible changes
- Provide migration documentation for users
- Use CEL validation to enforce constraints during migration

### 2. Storage Version Migration

**Limitation**: Manual process to change storage version

**Impact**:
- Changing `storage: true` from one version to another requires re-writing all stored objects
- No automatic migration tool
- Can cause downtime if not done carefully

**Scenario**:
```yaml
# Current: v1beta1 is storage version
versions:
- name: v1beta1
  storage: true
- name: v1
  storage: false

# Desired: v1 becomes storage version
versions:
- name: v1beta1
  storage: false
- name: v1
  storage: true  # Change storage version
```

**Workaround**:
```bash
# 1. Ensure conversion webhook is working (if schemas differ)
kubectl get crd myresources.mygroup.example.com -o jsonpath='{.spec.conversion}'

# 2. Update CRD to change storage version
kubectl apply -f crd-new-storage-version.yaml

# 3. Read and re-write all resources (forces conversion to new storage version)
kubectl get myresources --all-namespaces -o json | \
  kubectl replace --force -f -

# 4. Verify stored version changed
kubectl get crd myresources.mygroup.example.com -o jsonpath='{.status.storedVersions}'
# Should now include only ["v1"]

# 5. Clean up old stored versions (after confirming migration)
# This requires patching the CRD status (usually done by controller)
```

**Best Practice**:
- Plan storage version changes carefully
- Test conversion webhook thoroughly before migration
- Perform migration during maintenance window
- Monitor for conversion errors during migration
- Keep old version in `storedVersions` until migration complete

### 3. Conversion Performance

**Limitation**: Every operation may require version conversion

**Impact**:
- Additional latency (+1-10ms for in-process, +10-50ms for webhook)
- Webhook becomes critical dependency
- Conversion errors can block operations

**Scenario**:
```
Client requests v1 → Storage is v1beta1 → Requires conversion
                              ↓
                         Webhook call
                              ↓
                      +10-50ms latency
```

**Workaround**:
1. Use `conversion: None` if schemas are identical
2. Co-locate conversion webhook in same cluster
3. Optimize webhook handler performance
4. Use caching in webhook if possible
5. Consider making popular version the storage version

**Monitoring**:
```promql
# Conversion webhook latency
histogram_quantile(0.99,
  sum(rate(apiserver_webhook_duration_seconds_bucket{name=~".*conversion.*"}[5m])) by (le)
)

# Conversion failures
sum(rate(apiserver_webhook_rejection_count{name=~".*conversion.*"}[5m]))
```

### 4. CRD Count Limits

**Limitation**: Discovery performance degrades with many CRDs

**Impact**:
- kubectl startup slows down
- Discovery document size increases
- API server memory usage grows
- OpenAPI spec generation takes longer

**Degradation Pattern**:
- 0-100 CRDs: No noticeable impact
- 100-200 CRDs: kubectl startup ~1-2 seconds
- 200-500 CRDs: kubectl startup ~2-5 seconds
- 500+ CRDs: kubectl startup >5 seconds, discovery timeouts

**Workaround**:
1. Use aggregated discovery (enabled by default in recent kubectl)
2. Increase discovery cache TTL in kubectl
3. Consider splitting into multiple clusters
4. Remove unused CRDs

**Monitoring**:
```bash
# Count CRDs
kubectl get crd --no-headers | wc -l

# Check discovery document size
kubectl get --raw /apis | jq '. | length'

# Measure kubectl startup time
time kubectl get pods
```

**Best Practice**:
- Regularly audit and remove unused CRDs
- Use namespaced resources instead of cluster-scoped when possible
- Group related resources into single CRD with type field if appropriate

### 5. CEL Cost Estimation

**Limitation**: Conservative estimates may reject valid rules

**Impact**:
- Some valid CEL rules rejected due to cost limit
- Complex validation rules may hit 1M cost unit limit
- Nested list validation particularly expensive

**Scenario**:
```yaml
# May be rejected even if actual cost < 1M
x-kubernetes-validations:
- rule: "self.items.map(i, i.subitems.map(s, s.validate())).flatten().all(v, v)"
  # Estimated cost: >1M (conservative), actual cost: ~500K
```

**Workaround**:
1. Simplify validation rules (break into multiple rules)
2. Use separate rules per field instead of complex nested validation
3. Move complex validation to admission webhooks
4. Use `optionalOldSelf: true` only when necessary (reduces cost)

**Example Refactoring**:
```yaml
# BEFORE: Complex rule (may exceed cost limit)
x-kubernetes-validations:
- rule: "self.items.all(i, i.name != '' && i.value >= 0 && i.subitems.size() < 10)"

# AFTER: Split into multiple rules (lower cost each)
x-kubernetes-validations:
- rule: "self.items.all(i, i.name != '')"
- rule: "self.items.all(i, i.value >= 0)"
- rule: "self.items.all(i, i.subitems.size() < 10)"
```

**Best Practice**:
- Keep CEL rules simple and focused
- Test CEL rules before deploying to production
- Monitor CEL validation cost in metrics
- Use admission webhooks for very complex validation

### 6. Webhook Dependency

**Limitation**: Webhook downtime blocks operations

**Impact**:
- Conversion webhook down → can't read/write CRs
- Admission webhook down → can't create/update CRs (if `failurePolicy: Fail`)
- Creates single point of failure

**Scenario**:
```
Webhook pod crashes → All CR operations fail → Cluster impact
```

**Workaround**:
1. Run multiple webhook replicas (HA)
2. Use `failurePolicy: Ignore` for non-critical webhooks
3. Set reasonable timeouts (10-30s)
4. Monitor webhook health proactively
5. Have runbook for webhook recovery

**HA Webhook Deployment**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webhook
spec:
  replicas: 3  # HA for availability
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1  # Always keep 2 replicas available
  selector:
    matchLabels:
      app: webhook
  template:
    metadata:
      labels:
        app: webhook
    spec:
      affinity:
        podAntiAffinity:  # Spread across nodes
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: webhook
              topologyKey: kubernetes.io/hostname
      containers:
      - name: webhook
        image: mywebhook:v1.0.0
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8443
            scheme: HTTPS
        readinessProbe:
          httpGet:
            path: /readyz
            port: 8443
            scheme: HTTPS
```

**Monitoring**:
```promql
# Webhook availability
avg_over_time(up{job="webhook"}[5m]) < 1

# Webhook error rate
sum(rate(apiserver_admission_webhook_rejection_count[5m])) by (name)
```

### 7. StoredVersions Management

**Limitation**: Requires manual cleanup after version removal

**Impact**:
- `status.storedVersions` keeps old versions even after removal
- Prevents true version deprecation
- Can confuse operators

**Scenario**:
```yaml
# status.storedVersions: ["v1beta1", "v1"]
# Even after removing v1beta1 from spec, storedVersions still shows it
```

**Workaround**:
```bash
# 1. Ensure no resources stored in old version (migrate first)
kubectl get myresources --all-namespaces -o json | \
  kubectl replace --force -f -

# 2. Check stored versions
kubectl get crd myresources.mygroup.example.com -o jsonpath='{.status.storedVersions}'

# 3. Manually patch to remove old version (requires API access)
kubectl patch crd myresources.mygroup.example.com \
  --subresource=status \
  --type=json \
  -p='[{"op": "remove", "path": "/status/storedVersions/0"}]'  # Remove v1beta1
```

**Best Practice**:
- Document version lifecycle clearly
- Migrate and clean up stored versions before removing from spec
- Monitor `storedVersions` in production

## Common Issues

### CRD Won't Become Established

**Symptoms**:
- CRD created but `Established` condition never becomes `True`
- Custom resources can't be created
- `kubectl get <resource>` returns "resource not found"

**Possible Causes**:

**1. Naming Conflicts**:
```bash
# Check NamesAccepted condition
kubectl get crd myresources.mygroup.example.com -o jsonpath='{.status.conditions[?(@.type=="NamesAccepted")]}'

# If False, check message
kubectl describe crd myresources.mygroup.example.com
```

**Fix**: Resolve naming conflict or use different name

**2. Non-Structural Schema**:
```bash
# Check NonStructuralSchema condition
kubectl get crd myresources.mygroup.example.com -o jsonpath='{.status.conditions[?(@.type=="NonStructuralSchema")]}'
```

**Fix**: Make schema structural (all types explicit, no oneOf/anyOf)

**3. API Approval Missing**:
```bash
# Check KubernetesAPIApprovalPolicyConformant condition
kubectl get crd myresources.mygroup.example.com -o jsonpath='{.status.conditions[?(@.type=="KubernetesAPIApprovalPolicyConformant")]}'
```

**Fix**: Add `api-approved.kubernetes.io` annotation for `*.k8s.io` groups

**4. Controller Not Running**:
```bash
# Check API server logs
kubectl logs -n kube-system <apiserver-pod> | grep -i "establishing"

# Verify controllers started
kubectl logs -n kube-system <apiserver-pod> | grep -i "post-start-hook"
```

**Fix**: Ensure API server is running correctly, check for errors

### Custom Resource Validation Failures

**Symptoms**:
- `kubectl create -f resource.yaml` returns validation error
- Unclear error message
- CEL validation rejecting valid objects

**Debugging**:

**1. Use Dry-Run to See Detailed Errors**:
```bash
kubectl create -f resource.yaml --dry-run=server -v=8
```

**2. Check Schema Validation**:
```bash
# Get CRD schema
kubectl get crd myresources.mygroup.example.com -o jsonpath='{.spec.versions[0].schema}'

# Validate JSON against schema manually
# Use online JSON schema validator or jsonschema tool
```

**3. Test CEL Rules Separately**:
```yaml
# Temporarily disable CEL rules to isolate issue
x-kubernetes-validations: []
```

**4. Check for Type Mismatches**:
```bash
# Common issue: integer vs string
spec:
  replicas: "3"  # String, but schema expects integer
```

**Fix**: Correct resource to match schema, or update schema if appropriate

### Webhook Failures

**Symptoms**:
- Custom resources can't be created/updated
- Error message mentions webhook timeout or rejection
- Intermittent failures

**Debugging**:

**1. Check Webhook Configuration**:
```bash
# List webhook configs
kubectl get mutatingwebhookconfigurations
kubectl get validatingwebhookconfigurations

# Describe webhook
kubectl describe validatingwebhookconfiguration myresource-webhook
```

**2. Verify Webhook Service**:
```bash
# Check webhook service exists
kubectl get svc webhook-service

# Check webhook pods
kubectl get pods -l app=webhook

# Check webhook logs
kubectl logs -l app=webhook
```

**3. Test Webhook Directly**:
```bash
# Port-forward to webhook
kubectl port-forward svc/webhook-service 8443:443

# Test webhook endpoint
curl -k https://localhost:8443/validate -H "Content-Type: application/json" -d @admission-review.json
```

**4. Check TLS Certificates**:
```bash
# Get CA bundle from webhook config
kubectl get validatingwebhookconfiguration myresource-webhook -o jsonpath='{.webhooks[0].clientConfig.caBundle}' | base64 -d | openssl x509 -noout -text

# Check webhook certificate in pod
kubectl exec -it <webhook-pod> -- openssl x509 -in /etc/webhook/certs/tls.crt -noout -text
```

**Common Fixes**:
- Renew expired certificates
- Fix webhook service selector
- Increase webhook timeout
- Change `failurePolicy` to `Ignore` temporarily
- Scale up webhook replicas

### Discovery Issues

**Symptoms**:
- `kubectl api-resources` doesn't show custom resource
- `kubectl get <resource>` returns "resource not found"
- Discovery seems cached/stale

**Debugging**:

**1. Check Discovery Documents**:
```bash
# Check API groups
kubectl get --raw /apis | jq .

# Check specific group
kubectl get --raw /apis/mygroup.example.com | jq .

# Check specific version
kubectl get --raw /apis/mygroup.example.com/v1 | jq .
```

**2. Clear kubectl Cache**:
```bash
# kubectl caches discovery for 10 minutes
# Force refresh by deleting cache
rm -rf ~/.kube/cache/discovery/

# Or wait 10 minutes
```

**3. Check CRD Status**:
```bash
# Verify CRD is established
kubectl get crd myresources.mygroup.example.com -o jsonpath='{.status.conditions[?(@.type=="Established")]}'
```

**4. Check API Server Logs**:
```bash
kubectl logs -n kube-system <apiserver-pod> | grep -i discovery
```

### Performance Issues

**Symptoms**:
- Slow custom resource operations
- High API server CPU/memory
- Timeouts on large lists

**Debugging**:

**1. Profile Request Latency**:
```bash
# Use verbose kubectl to see timing
kubectl get myresources -v=8 2>&1 | grep "Response Status"

# Check API server metrics
kubectl get --raw /metrics | grep apiserver_request_duration_seconds
```

**2. Check for Expensive CEL Rules**:
```bash
# Look for high CEL cost in logs
kubectl logs -n kube-system <apiserver-pod> | grep -i "cel.*cost"

# Review CEL rules in CRD
kubectl get crd myresources.mygroup.example.com -o jsonpath='{.spec.versions[0].schema.openAPIV3Schema.x-kubernetes-validations}'
```

**3. Monitor Webhook Latency**:
```promql
histogram_quantile(0.99,
  sum(rate(apiserver_admission_webhook_admission_duration_seconds_bucket[5m])) by (le, name)
)
```

**4. Check Resource Counts**:
```bash
# Count custom resources
kubectl get myresources --all-namespaces --no-headers | wc -l

# Count CRDs
kubectl get crd --no-headers | wc -l
```

**Fixes**:
- Simplify CEL rules
- Optimize webhook handlers
- Add pagination to list operations
- Consider increasing API server resources
- Remove unused CRDs

## Debugging Strategies

### Enable Verbose Logging

**API Server**:
```bash
# Increase verbosity level (--v flag)
kube-apiserver --v=5 ...  # Info level
kube-apiserver --v=8 ...  # Debug level (very verbose)
```

**kubectl**:
```bash
kubectl get myresources -v=8  # See all HTTP requests/responses
```

### Inspect API Server Logs

**Common Log Patterns**:
```bash
# Validation errors
kubectl logs -n kube-system <apiserver-pod> | grep -i "validation failed"

# Webhook errors
kubectl logs -n kube-system <apiserver-pod> | grep -i "webhook"

# CRD lifecycle events
kubectl logs -n kube-system <apiserver-pod> | grep -i "crd"

# Controller reconciliation
kubectl logs -n kube-system <apiserver-pod> | grep -i "controller"
```

### Use API Server Profiling

**Enable Profiling Endpoint**:
```bash
# Access profiling endpoint (if enabled)
kubectl proxy &
curl http://localhost:8001/debug/pprof/

# CPU profile
curl http://localhost:8001/debug/pprof/profile?seconds=30 > cpu.prof
go tool pprof cpu.prof

# Memory profile
curl http://localhost:8001/debug/pprof/heap > heap.prof
go tool pprof heap.prof
```

### Inspect etcd Directly

**Check Stored Data**:
```bash
# List CRDs in etcd
ETCDCTL_API=3 etcdctl get /registry/apiextensions.k8s.io/customresourcedefinitions --prefix --keys-only

# Get specific CRD
ETCDCTL_API=3 etcdctl get /registry/apiextensions.k8s.io/customresourcedefinitions/myresources.mygroup.example.com

# List custom resources
ETCDCTL_API=3 etcdctl get /registry/mygroup.example.com/myresources --prefix --keys-only
```

**Check Storage Version**:
```bash
# Verify storage version in etcd
ETCDCTL_API=3 etcdctl get /registry/mygroup.example.com/myresources/default/test | strings | grep apiVersion
```

### Test Admission Chain

**Bypass Webhooks Temporarily**:
```yaml
# Update webhook configuration to skip certain operations
rules:
- operations: ["CREATE"]  # Remove UPDATE temporarily
  apiGroups: ["mygroup.example.com"]
  apiVersions: ["v1"]
  resources: ["myresources"]
```

**Test Dry-Run**:
```bash
# Test without actually storing
kubectl create -f resource.yaml --dry-run=server -v=8
```

## Monitoring and Observability

### Key Metrics

**API Server Metrics**:
```promql
# Request rate
sum(rate(apiserver_request_total{resource="myresources"}[5m])) by (verb)

# Request latency (P99)
histogram_quantile(0.99,
  sum(rate(apiserver_request_duration_seconds_bucket{resource="myresources"}[5m])) by (le, verb)
)

# Error rate
sum(rate(apiserver_request_total{resource="myresources", code=~"5.."}[5m])) by (code)
```

**Webhook Metrics**:
```promql
# Webhook latency
histogram_quantile(0.99,
  sum(rate(apiserver_admission_webhook_admission_duration_seconds_bucket[5m])) by (le, name)
)

# Webhook rejection rate
sum(rate(apiserver_admission_webhook_rejection_count[5m])) by (name, reason)
```

**Resource Metrics**:
```promql
# API server memory
process_resident_memory_bytes{job="apiserver"}

# Watch connection count
apiserver_longrunning_requests{verb="WATCH"}

# etcd request latency
histogram_quantile(0.99,
  sum(rate(etcd_request_duration_seconds_bucket[5m])) by (le)
)
```

### Alerts

**Recommended Alerts**:
```yaml
# High error rate
- alert: CRHighErrorRate
  expr: |
    sum(rate(apiserver_request_total{resource="myresources", code=~"5.."}[5m]))
    /
    sum(rate(apiserver_request_total{resource="myresources"}[5m]))
    > 0.05
  annotations:
    summary: "High error rate for custom resource operations"

# High latency
- alert: CRHighLatency
  expr: |
    histogram_quantile(0.99,
      sum(rate(apiserver_request_duration_seconds_bucket{resource="myresources"}[5m])) by (le)
    ) > 1.0
  annotations:
    summary: "P99 latency for CR operations > 1 second"

# Webhook down
- alert: WebhookDown
  expr: |
    up{job="webhook"} == 0
  annotations:
    summary: "Conversion/admission webhook is down"

# CRD not established
- alert: CRDNotEstablished
  expr: |
    crd_established{crd="myresources.mygroup.example.com"} == 0
  annotations:
    summary: "CRD is not in Established state"
```

## Operational Tips

### Pre-Production Checklist

- [ ] Test CRD creation in staging
- [ ] Verify all versions serve correctly
- [ ] Test conversion webhook (if applicable)
- [ ] Test admission webhooks (if applicable)
- [ ] Load test with expected CR count
- [ ] Verify RBAC rules
- [ ] Set up monitoring and alerts
- [ ] Document migration procedures
- [ ] Prepare rollback plan
- [ ] Test disaster recovery

### Regular Maintenance

**Weekly**:
- Review API server logs for errors
- Check webhook health and certificate expiry
- Monitor CR growth rate

**Monthly**:
- Audit CRD list for unused resources
- Review and optimize CEL rules
- Update webhook images for security patches
- Review RBAC rules

**Quarterly**:
- Plan version deprecations
- Migrate storage versions if needed
- Conduct performance testing
- Update operational documentation

### Disaster Recovery

**Backup CRDs**:
```bash
# Export all CRDs
kubectl get crd -o yaml > crds-backup.yaml

# Export specific CRD with all versions
kubectl get crd myresources.mygroup.example.com -o yaml > myresources-crd.yaml
```

**Backup Custom Resources**:
```bash
# Export all CRs of a specific type
kubectl get myresources --all-namespaces -o yaml > myresources-backup.yaml

# Export with pagination (for large result sets)
kubectl get myresources --all-namespaces --chunk-size=500 -o yaml > myresources-backup.yaml
```

**Restore Procedure**:
```bash
# 1. Restore CRDs first
kubectl apply -f crds-backup.yaml

# 2. Wait for all CRDs to become established
for crd in $(kubectl get crd -o name); do
  kubectl wait --for=condition=Established $crd --timeout=60s
done

# 3. Restore custom resources
kubectl apply -f myresources-backup.yaml
```

## Getting Help

### Information to Gather

When asking for help, provide:

1. **CRD Definition**:
   ```bash
   kubectl get crd myresources.mygroup.example.com -o yaml
   ```

2. **CRD Status**:
   ```bash
   kubectl describe crd myresources.mygroup.example.com
   ```

3. **API Server Logs**:
   ```bash
   kubectl logs -n kube-system <apiserver-pod> --tail=100
   ```

4. **Failing Resource**:
   ```bash
   kubectl get myresource test -o yaml
   ```

5. **Error Message**:
   ```bash
   kubectl create -f resource.yaml --dry-run=server -v=8 2>&1
   ```

6. **Version Information**:
   ```bash
   kubectl version
   kubectl get --raw /version
   ```

### Resources

**Kubernetes Documentation**:
- [Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
- [CustomResourceDefinitions](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)
- [Admission Webhooks](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)

**Community**:
- Kubernetes Slack: #sig-api-machinery
- Kubernetes GitHub: https://github.com/kubernetes/kubernetes/issues
- Stack Overflow: [kubernetes] tag

**Filing Issues**:
- Search existing issues first
- Provide reproducible test case
- Include version information
- Attach relevant logs (redact sensitive data)

## Related Documentation

- [Performance Guide](./performance.md) - Performance optimization strategies
- [Security Guide](./security.md) - Security best practices
- [Testing Guide](./testing.md) - Test debugging strategies
- [Lifecycle Controllers](../core-components/lifecycle-controllers.md) - Controller troubleshooting
- [Custom Resource Handler](../core-components/custom-resource-handler.md) - Request handling internals
