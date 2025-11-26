# Security Guide

This guide covers the security model, authentication, authorization, admission control, and security best practices for the Kubernetes API Extensions API Server.

## Overview

The apiextensions-apiserver inherits Kubernetes' security model and extends it for custom resources. Security is enforced through multiple layers: authentication, RBAC authorization, admission control, API approval policy, webhook security, and storage encryption.

## Security Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Security Layers                           │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  1. Authentication (TLS client certificates, tokens)         │
│                          ↓                                    │
│  2. RBAC Authorization (ClusterRole, Role bindings)          │
│                          ↓                                    │
│  3. Admission Control (mutating + validating webhooks)       │
│                          ↓                                    │
│  4. API Approval Policy (k8s.io namespace protection)        │
│                          ↓                                    │
│  5. Storage (etcd encryption at rest, TLS in transit)        │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

## RBAC Integration

### Custom Resource RBAC

Custom resources respect standard Kubernetes RBAC rules. CRDs automatically integrate with the RBAC system, allowing fine-grained access control.

**Standard Verbs**:
- `get`: Retrieve single resource
- `list`: List resources
- `watch`: Watch for changes
- `create`: Create new resource
- `update`: Update entire resource
- `patch`: Partial update
- `delete`: Delete resource
- `deletecollection`: Delete multiple resources

**Subresource Permissions**:
- `get`, `update`, `patch` on `<resource>/status` - Status subresource
- `get`, `update`, `patch` on `<resource>/scale` - Scale subresource

### RBAC Examples

**ClusterRole for Full Access**:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: myresource-admin
rules:
- apiGroups: ["mygroup.example.com"]
  resources: ["myresources"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["mygroup.example.com"]
  resources: ["myresources/status"]
  verbs: ["get", "update", "patch"]
- apiGroups: ["mygroup.example.com"]
  resources: ["myresources/scale"]
  verbs: ["get", "update", "patch"]
```

**ClusterRole for Read-Only Access**:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: myresource-viewer
rules:
- apiGroups: ["mygroup.example.com"]
  resources: ["myresources"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["mygroup.example.com"]
  resources: ["myresources/status"]
  verbs: ["get"]
```

**Role for Namespace-Scoped Access**:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: myresource-editor
  namespace: production
rules:
- apiGroups: ["mygroup.example.com"]
  resources: ["myresources"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

**ClusterRoleBinding**:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: myresource-admin-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: myresource-admin
subjects:
- kind: ServiceAccount
  name: myapp-controller
  namespace: default
- kind: User
  name: admin@example.com
  apiGroup: rbac.authorization.k8s.io
```

### Namespace-Scoped vs Cluster-Scoped

**Namespace-Scoped CRD**:
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: myresources.mygroup.example.com
spec:
  scope: Namespaced  # Resources exist within namespaces
  # ...
```

**RBAC**: Use `Role` and `RoleBinding` for namespace-specific access

**Cluster-Scoped CRD**:
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: myglobalresources.mygroup.example.com
spec:
  scope: Cluster  # Resources are cluster-wide
  # ...
```

**RBAC**: Must use `ClusterRole` and `ClusterRoleBinding`

### Testing RBAC Rules

**Check User Permissions**:
```bash
# Can I create a resource?
kubectl auth can-i create myresources.mygroup.example.com

# Can I update status?
kubectl auth can-i update myresources.mygroup.example.com/status

# Check as specific user
kubectl auth can-i list myresources.mygroup.example.com --as=user@example.com

# Check as service account
kubectl auth can-i delete myresources.mygroup.example.com \
  --as=system:serviceaccount:default:myapp
```

**Impersonate User**:
```bash
# Test as specific user
kubectl get myresources --as=user@example.com

# Test as service account
kubectl get myresources \
  --as=system:serviceaccount:default:myapp
```

### Common RBAC Patterns

**Controller Service Account**:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-controller
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: myapp-controller-role
rules:
# Read CRD definitions
- apiGroups: ["apiextensions.k8s.io"]
  resources: ["customresourcedefinitions"]
  verbs: ["get", "list", "watch"]
# Manage custom resources
- apiGroups: ["mygroup.example.com"]
  resources: ["myresources"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
# Update status
- apiGroups: ["mygroup.example.com"]
  resources: ["myresources/status"]
  verbs: ["get", "update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: myapp-controller-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: myapp-controller-role
subjects:
- kind: ServiceAccount
  name: myapp-controller
  namespace: default
```

**Human Operator Access**:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: operator-role
rules:
# View CRDs
- apiGroups: ["apiextensions.k8s.io"]
  resources: ["customresourcedefinitions"]
  verbs: ["get", "list"]
# Manage custom resources
- apiGroups: ["mygroup.example.com"]
  resources: ["myresources"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
# View status (read-only)
- apiGroups: ["mygroup.example.com"]
  resources: ["myresources/status"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: operators-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: operator-role
subjects:
- kind: Group
  name: operators@example.com
  apiGroup: rbac.authorization.k8s.io
```

## Admission Control

### Admission Webhook Flow

```
Create/Update Request
    ↓
1. Schema Validation (built-in)
    ↓
2. Mutating Admission Webhooks (modify request)
    ↓
3. CEL Validation (built-in)
    ↓
4. Validating Admission Webhooks (accept/reject)
    ↓
5. Store in etcd (if approved)
```

### Mutating Webhooks

Mutating webhooks can modify resources before they are stored.

**Use Cases**:
- Set default values
- Inject sidecars
- Normalize field values
- Add labels/annotations

**MutatingWebhookConfiguration**:
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: myresource-mutating-webhook
webhooks:
- name: mutate.mygroup.example.com
  clientConfig:
    service:
      namespace: default
      name: webhook-service
      path: /mutate
    caBundle: <base64-encoded-ca-cert>
  rules:
  - apiGroups: ["mygroup.example.com"]
    apiVersions: ["v1"]
    operations: ["CREATE", "UPDATE"]
    resources: ["myresources"]
  admissionReviewVersions: ["v1"]
  sideEffects: None
  timeoutSeconds: 10
  failurePolicy: Fail  # or Ignore
```

**Webhook Handler Example**:
```go
func (h *MutatingHandler) Handle(ctx context.Context, req admission.Request) admission.Response {
    // Parse resource
    resource := &MyResource{}
    if err := json.Unmarshal(req.Object.Raw, resource); err != nil {
        return admission.Errored(http.StatusBadRequest, err)
    }

    // Apply mutations
    if resource.Spec.Replicas == nil {
        resource.Spec.Replicas = pointer.Int32(1)  // Default replicas
    }
    if resource.Labels == nil {
        resource.Labels = make(map[string]string)
    }
    resource.Labels["mutated-by"] = "webhook"

    // Return patched resource
    marshaledResource, err := json.Marshal(resource)
    if err != nil {
        return admission.Errored(http.StatusInternalServerError, err)
    }
    return admission.PatchResponseFromRaw(req.Object.Raw, marshaledResource)
}
```

### Validating Webhooks

Validating webhooks accept or reject resources based on custom logic.

**Use Cases**:
- Business logic validation
- Cross-field validation
- External system validation
- Quota enforcement

**ValidatingWebhookConfiguration**:
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: myresource-validating-webhook
webhooks:
- name: validate.mygroup.example.com
  clientConfig:
    service:
      namespace: default
      name: webhook-service
      path: /validate
    caBundle: <base64-encoded-ca-cert>
  rules:
  - apiGroups: ["mygroup.example.com"]
    apiVersions: ["v1"]
    operations: ["CREATE", "UPDATE"]
    resources: ["myresources"]
  admissionReviewVersions: ["v1"]
  sideEffects: None
  timeoutSeconds: 10
  failurePolicy: Fail
```

**Webhook Handler Example**:
```go
func (h *ValidatingHandler) Handle(ctx context.Context, req admission.Request) admission.Response {
    // Parse resource
    resource := &MyResource{}
    if err := json.Unmarshal(req.Object.Raw, resource); err != nil {
        return admission.Errored(http.StatusBadRequest, err)
    }

    // Validate business logic
    if resource.Spec.Replicas != nil && *resource.Spec.Replicas > 10 {
        return admission.Denied("replicas cannot exceed 10")
    }

    // Validate against external system
    if !isValidInExternalSystem(resource.Spec.Config) {
        return admission.Denied("config not found in external system")
    }

    return admission.Allowed("")
}
```

### Admission Webhook Security

**Best Practices**:

1. **Use TLS**: Always require TLS for webhook endpoints
2. **Validate CA Bundle**: Ensure CA certificate is correct
3. **Set Timeouts**: Configure reasonable timeouts (10-30s)
4. **Configure Failure Policy**: Choose `Fail` (safe) or `Ignore` (available)
5. **Minimize Scope**: Target specific resources/operations
6. **Avoid External Calls**: Keep webhooks fast (< 10ms)
7. **Use Service Accounts**: Webhook should use dedicated SA

**Generate TLS Certificates**:
```bash
# Generate CA
openssl genrsa -out ca.key 2048
openssl req -x509 -new -nodes -key ca.key -subj "/CN=Webhook CA" -days 365 -out ca.crt

# Generate webhook certificate
openssl genrsa -out webhook.key 2048
openssl req -new -key webhook.key -subj "/CN=webhook-service.default.svc" -out webhook.csr
openssl x509 -req -in webhook.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out webhook.crt -days 365

# Create secret
kubectl create secret tls webhook-cert --cert=webhook.crt --key=webhook.key

# Get CA bundle for webhook config
cat ca.crt | base64 | tr -d '\n'
```

**Webhook Service**:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: webhook-service
  namespace: default
spec:
  selector:
    app: webhook
  ports:
  - port: 443
    targetPort: 8443
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webhook
  namespace: default
spec:
  replicas: 2  # HA for availability
  selector:
    matchLabels:
      app: webhook
  template:
    metadata:
      labels:
        app: webhook
    spec:
      serviceAccountName: webhook-sa
      containers:
      - name: webhook
        image: mywebhook:v1.0.0
        ports:
        - containerPort: 8443
        volumeMounts:
        - name: webhook-cert
          mountPath: /etc/webhook/certs
          readOnly: true
      volumes:
      - name: webhook-cert
        secret:
          secretName: webhook-cert
```

### Admission Control Monitoring

**Check Webhook Status**:
```bash
# List webhook configurations
kubectl get mutatingwebhookconfigurations
kubectl get validatingwebhookconfigurations

# Check webhook details
kubectl describe mutatingwebhookconfiguration myresource-mutating-webhook

# View webhook logs
kubectl logs -n default deployment/webhook
```

**Metrics**:
```promql
# Webhook latency
histogram_quantile(0.99,
  sum(rate(apiserver_admission_webhook_admission_duration_seconds_bucket[5m])) by (le, name)
)

# Webhook rejection rate
sum(rate(apiserver_admission_webhook_rejection_count[5m])) by (name)
```

## API Approval Policy

### Protected Namespaces

Kubernetes reserves `*.k8s.io` and `*.kubernetes.io` API groups for official APIs. The API Approval Policy prevents unauthorized use of these namespaces.

**Protected Groups**:
- `*.k8s.io` (e.g., `myresource.k8s.io`)
- `*.kubernetes.io` (e.g., `myresource.kubernetes.io`)

**Requirement**: Must have `api-approved.kubernetes.io` annotation

### API Approval Annotation

**Valid Annotation Values**:
```yaml
# Links to PR that approved this API
api-approved.kubernetes.io: "https://github.com/kubernetes/kubernetes/pull/12345"

# For legacy/unapproved APIs (discouraged)
api-approved.kubernetes.io: "unapproved, request not yet approved"
```

**Example CRD**:
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: myresources.mygroup.k8s.io
  annotations:
    api-approved.kubernetes.io: "https://github.com/kubernetes/kubernetes/pull/12345"
spec:
  group: mygroup.k8s.io  # Protected namespace
  # ...
```

**Without Annotation** (will fail):
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: myresources.mygroup.k8s.io
  # Missing annotation!
spec:
  group: mygroup.k8s.io
  # ...
```

**Error Message**:
```
CustomResourceDefinition.apiextensions.k8s.io "myresources.mygroup.k8s.io" is invalid:
metadata.annotations: Required value: must have api-approved.kubernetes.io annotation
```

### Non-Protected Groups

Groups not ending in `.k8s.io` or `.kubernetes.io` do not require approval:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: myresources.mygroup.example.com
  # No annotation needed
spec:
  group: mygroup.example.com  # Not protected
  # ...
```

**Best Practice**: Use your organization's domain for custom API groups (e.g., `mygroup.mycompany.com`)

### Checking API Approval Status

```bash
# Check CRD condition
kubectl get crd myresources.mygroup.k8s.io -o jsonpath='{.status.conditions[?(@.type=="KubernetesAPIApprovalPolicyConformant")]}'

# Expected output for approved CRD:
{"type":"KubernetesAPIApprovalPolicyConformant","status":"True","reason":"ApprovedAnnotation"}
```

## Webhook Security

### TLS Requirements

**All webhooks MUST use TLS**:
- Client certificate validation
- Server certificate validation
- CA bundle verification

**Certificate Requirements**:
- Valid X.509 certificate
- CN matches service DNS name: `<service>.<namespace>.svc`
- CA bundle provided in webhook configuration

### Webhook Authentication

**Service Account Authentication**:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: webhook-sa
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: webhook-role
rules:
# Webhook may need to read CRDs
- apiGroups: ["apiextensions.k8s.io"]
  resources: ["customresourcedefinitions"]
  verbs: ["get", "list"]
# Webhook may need to read other resources for validation
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: webhook-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: webhook-role
subjects:
- kind: ServiceAccount
  name: webhook-sa
  namespace: default
```

### Webhook Timeouts

**Configure Reasonable Timeouts**:
```yaml
webhooks:
- name: validate.mygroup.example.com
  timeoutSeconds: 10  # Default: 30s, reduce for faster failures
```

**Behavior**:
- If webhook doesn't respond within timeout, `failurePolicy` applies
- `failurePolicy: Fail` → request rejected
- `failurePolicy: Ignore` → request accepted

### Webhook Failure Policy

**Fail Closed (Safe)**:
```yaml
failurePolicy: Fail  # Reject request on webhook failure
```

**Use for**:
- Critical security validations
- Compliance requirements
- Quota enforcement

**Fail Open (Available)**:
```yaml
failurePolicy: Ignore  # Accept request on webhook failure
```

**Use for**:
- Non-critical validations
- Informational webhooks
- High availability requirements

### Webhook Scope

**Minimize Webhook Scope**:
```yaml
rules:
- apiGroups: ["mygroup.example.com"]  # Specific group
  apiVersions: ["v1"]                 # Specific version
  operations: ["CREATE", "UPDATE"]    # Specific operations
  resources: ["myresources"]          # Specific resource
  scope: "Namespaced"                 # Namespace or Cluster
```

**Avoid Wildcards**:
```yaml
# BAD: Too broad
rules:
- apiGroups: ["*"]
  apiVersions: ["*"]
  operations: ["*"]
  resources: ["*"]
```

## Storage Encryption

### etcd Encryption at Rest

**Enable Encryption Config**:
```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - customresourcedefinitions.apiextensions.k8s.io
  - myresources.mygroup.example.com  # Your custom resource
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-32-byte-key>
  - identity: {}  # Fallback for unencrypted data
```

**API Server Flag**:
```bash
kube-apiserver \
  --encryption-provider-config=/etc/kubernetes/encryption-config.yaml \
  ...
```

**Verify Encryption**:
```bash
# Check if data is encrypted in etcd
ETCDCTL_API=3 etcdctl get /registry/mygroup.example.com/myresources/default/test | strings

# Encrypted output starts with: k8s:enc:aescbc:v1:key1:
# Unencrypted output: readable JSON
```

**Rotate Encryption Keys**:
```yaml
resources:
- resources:
  - myresources.mygroup.example.com
  providers:
  - aescbc:
      keys:
      - name: key2  # New key (will be used for writes)
        secret: <new-base64-encoded-key>
      - name: key1  # Old key (can still decrypt)
        secret: <old-base64-encoded-key>
  - identity: {}
```

**Re-encrypt All Data**:
```bash
# After adding new key, re-encrypt all resources
kubectl get myresources -o json | kubectl replace -f -
```

### etcd TLS

**Client-to-etcd TLS**:
```bash
kube-apiserver \
  --etcd-servers=https://etcd-1:2379,https://etcd-2:2379 \
  --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt \
  --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt \
  --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key \
  ...
```

### Audit Logging

**Enable Audit Logging**:
```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# Log custom resource access
- level: RequestResponse
  resources:
  - group: "mygroup.example.com"
    resources: ["myresources"]
# Log CRD changes
- level: RequestResponse
  resources:
  - group: "apiextensions.k8s.io"
    resources: ["customresourcedefinitions"]
```

**API Server Flags**:
```bash
kube-apiserver \
  --audit-policy-file=/etc/kubernetes/audit-policy.yaml \
  --audit-log-path=/var/log/kubernetes/audit.log \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=10 \
  --audit-log-maxsize=100 \
  ...
```

**Audit Log Example**:
```json
{
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1",
  "level": "RequestResponse",
  "auditID": "...",
  "stage": "ResponseComplete",
  "requestURI": "/apis/mygroup.example.com/v1/namespaces/default/myresources/test",
  "verb": "create",
  "user": {
    "username": "admin",
    "uid": "...",
    "groups": ["system:authenticated"]
  },
  "objectRef": {
    "resource": "myresources",
    "namespace": "default",
    "name": "test",
    "apiGroup": "mygroup.example.com",
    "apiVersion": "v1"
  },
  "responseStatus": {
    "code": 201
  },
  "requestObject": {...},
  "responseObject": {...}
}
```

## Security Best Practices

### CRD Design

**1. Minimize Privileges**:
```yaml
# Grant minimal RBAC permissions
rules:
- apiGroups: ["mygroup.example.com"]
  resources: ["myresources"]
  verbs: ["get", "list"]  # Read-only
```

**2. Use Status Subresource**:
```yaml
spec:
  versions:
  - name: v1
    subresources:
      status: {}  # Separate permissions for status
```

**3. Validate Sensitive Fields**:
```yaml
schema:
  properties:
    apiKey:
      type: string
      pattern: "^[a-zA-Z0-9]{32}$"  # Enforce format
      x-kubernetes-validations:
      - rule: "self != 'changeme'"  # Prevent default values
```

### Webhook Security

**1. Minimize External Calls**:
- Avoid calling external APIs (slow + security risk)
- Cache external data locally
- Use timeouts for external calls

**2. Validate Webhook Input**:
```go
func (h *Handler) Handle(ctx context.Context, req admission.Request) admission.Response {
    // Validate admission request
    if req.Operation != admissionv1.Create && req.Operation != admissionv1.Update {
        return admission.Allowed("")  // Only validate create/update
    }

    // Validate object is not nil
    if req.Object.Raw == nil {
        return admission.Errored(http.StatusBadRequest, errors.New("object is nil"))
    }

    // Parse and validate
    // ...
}
```

**3. Handle Panics**:
```go
func (h *Handler) Handle(ctx context.Context, req admission.Request) admission.Response {
    defer func() {
        if r := recover(); r != nil {
            log.Errorf("Panic in webhook handler: %v", r)
            // Return error instead of crashing
        }
    }()

    // Handler logic
}
```

### Operational Security

**1. Regular Security Audits**:
```bash
# Review RBAC rules
kubectl get clusterroles,clusterrolebindings | grep mygroup

# Check webhook configurations
kubectl get mutatingwebhookconfigurations,validatingwebhookconfigurations

# Review CRDs for api-approved annotation
kubectl get crd -o jsonpath='{range .items[*]}{.metadata.name}: {.metadata.annotations.api-approved\.kubernetes\.io}{"\n"}{end}'
```

**2. Monitor Audit Logs**:
```bash
# Search for failed authorization
grep "Forbidden" /var/log/kubernetes/audit.log

# Search for custom resource access
grep "mygroup.example.com" /var/log/kubernetes/audit.log

# Search for privilege escalation attempts
grep "impersonate" /var/log/kubernetes/audit.log
```

**3. Keep Certificates Valid**:
```bash
# Check webhook certificate expiration
kubectl get secret webhook-cert -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -dates

# Set up alerts for expiring certificates (< 30 days)
```

### Secrets Management

**Do Not Store Secrets in CRDs**:
```yaml
# BAD: Secret in CRD
spec:
  apiKey: "super-secret-key"  # Stored in etcd, visible in audit logs

# GOOD: Reference to Secret
spec:
  apiKeySecretRef:
    name: myapp-secret
    key: apiKey
```

**Use Kubernetes Secrets**:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secret
type: Opaque
data:
  apiKey: <base64-encoded-key>
```

**Controller Access**:
```go
// Read secret securely
secret, err := client.CoreV1().Secrets(namespace).Get(ctx, secretName, metav1.GetOptions{})
if err != nil {
    return err
}
apiKey := string(secret.Data["apiKey"])
```

## Security Checklist

### Before Deploying to Production

- [ ] RBAC rules configured with minimal privileges
- [ ] Admission webhooks use TLS with valid certificates
- [ ] Webhook failure policies configured appropriately
- [ ] API approval annotation set for k8s.io groups
- [ ] etcd encryption at rest enabled (if required)
- [ ] Audit logging configured for compliance
- [ ] Secrets stored in Kubernetes Secrets, not CRDs
- [ ] Service accounts use minimal required permissions
- [ ] Certificate expiration monitoring configured
- [ ] Security audit completed

### Regular Maintenance

- [ ] Review RBAC rules quarterly
- [ ] Rotate webhook certificates annually
- [ ] Review audit logs monthly
- [ ] Update webhook images for security patches
- [ ] Rotate etcd encryption keys annually
- [ ] Test disaster recovery procedures
- [ ] Review and update security policies

## Related Documentation

- [Lifecycle Controllers](../core-components/lifecycle-controllers.md) - API approval controller
- [Custom Resource Handler](../core-components/custom-resource-handler.md) - RBAC integration
- [Schema Validation](../core-components/schema-validation.md) - Validation security
- [Troubleshooting Guide](./troubleshooting.md) - Security issue debugging
