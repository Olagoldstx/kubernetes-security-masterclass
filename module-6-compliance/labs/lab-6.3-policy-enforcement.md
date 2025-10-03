# LAB 6.3: Policy Enforcement with OPA/Gatekeeper
## Royal Decree Enforcement

## 🎯 Lab Objectives
1. Implement Open Policy Agent (OPA) Gatekeeper
2. Create and enforce custom security policies
3. Establish policy-as-code practices
4. Monitor policy violations and compliance

## 🏰 Analogy: Royal Decree Enforcement
- **OPA Gatekeeper** = Royal decree enforcers
- **Constraints** = Specific laws and regulations
- **Constraint Templates** = Law frameworks and templates
- **Policy Violations** = Law-breaking incidents

## 🛠️ Hands-On Implementation

### Step 1: Install OPA Gatekeeper
```bash
# Add Gatekeeper Helm repository
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm repo update

# Install Gatekeeper
helm install gatekeeper gatekeeper/gatekeeper \
  --namespace gatekeeper-system \
  --create-namespace \
  --set auditInterval=60 \
  --set constraintViolationsLimit=100 \
  --set disableValidatingWebhook=false

# Wait for Gatekeeper to be ready
kubectl wait --namespace gatekeeper-system --for=condition=ready pod -l control-plane=controller-manager --timeout=300s

# Verify installation
kubectl get pods -n gatekeeper-system
kubectl get validatingwebhookconfigurations | grep gatekeeper
```

### Step 2: Create Constraint Templates (Law Frameworks)
```yaml
# constraint-templates.yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels
        
        violation[{"msg": msg, "details": {"missing_labels": missing}}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("you must provide labels: %v", [missing])
        }
---
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8scontainerlimits
spec:
  crd:
    spec:
      names:
        kind: K8sContainerLimits
      validation:
        openAPIV3Schema:
          type: object
          properties:
            cpu_request:
              type: string
            memory_request:
              type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8scontainerlimits
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.requests.cpu
          msg := sprintf("Container %v must have CPU request set", [container.name])
        }
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.requests.memory
          msg := sprintf("Container %v must have memory request set", [container.name])
        }
---
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8sblockhostnamespace
spec:
  crd:
    spec:
      names:
        kind: K8sBlockHostNamespace
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sblockhostnamespace
        
        violation[{"msg": msg}] {
          has_field(input.review.object.spec, "hostPID")
          input.review.object.spec.hostPID
          msg := "Sharing the host PID namespace is not allowed"
        }
        
        violation[{"msg": msg}] {
          has_field(input.review.object.spec, "hostIPC")
          input.review.object.spec.hostIPC
          msg := "Sharing the host IPC namespace is not allowed"
        }
        
        violation[{"msg": msg}] {
          has_field(input.review.object.spec, "hostNetwork")
          input.review.object.spec.hostNetwork
          msg := "Sharing the host network namespace is not allowed"
        }
        
        has_field(obj, field) = true {
          obj[field]
        }
EOF
kubectl apply -f constraint-templates.yaml
```

### Step 3: Create Constraints (Specific Laws)
```yaml
# security-constraints.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: required-labels
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod", "Deployment", "StatefulSet", "DaemonSet"]
    namespaces:
    - "production"
    - "ecommerce"
  parameters:
    labels: ["app", "version", "environment", "owner"]
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sContainerLimits
metadata:
  name: container-limits
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    namespaces:
    - "production"
    - "ecommerce"
  parameters:
    cpu_request: "10m"
    memory_request: "32Mi"
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sBlockHostNamespace
metadata:
  name: block-host-namespace
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sNoPrivilegedContainers
metadata:
  name: no-privileged-containers
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
  parameters:
    privileged: false
    message: "Privileged containers are not allowed"
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredProbes
metadata:
  name: required-probes
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    namespaces:
    - "production"
  parameters:
    liveliness_probe_required: true
    readiness_probe_required: true
EOF
kubectl apply -f security-constraints.yaml
```

### Step 4: Create Custom Security Policies
```yaml
# custom-security-policies.yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8ssecureimages
spec:
  crd:
    spec:
      names:
        kind: K8sSecureImages
      validation:
        openAPIV3Schema:
          type: object
          properties:
            allowed_registries:
              type: array
              items:
                type: string
            blocked_images:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8ssecureimages
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not valid_registry(container.image)
          msg := sprintf("Image %v comes from unauthorized registry. Allowed registries: %v", [container.image, input.parameters.allowed_registries])
        }
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          blocked_image(container.image)
          msg := sprintf("Image %v is blocked. Blocked images: %v", [container.image, input.parameters.blocked_images])
        }
        
        valid_registry(image) {
          startswith(image, reg)
          reg := input.parameters.allowed_registries[_]
        }
        
        blocked_image(image) {
          blocked_pat := input.parameters.blocked_images[_]
          contains(image, blocked_pat)
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sSecureImages
metadata:
  name: secure-images
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
  parameters:
    allowed_registries:
    - "docker.io/library/"
    - "gcr.io/"
    - "registry.k8s.io/"
    - "quay.io/"
    blocked_images:
    - ":latest"
    - ":edge"
    - "alpine:3.9"  # Known vulnerabilities
---
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8snetworkpolicyrequired
spec:
  crd:
    spec:
      names:
        kind: K8sNetworkPolicyRequired
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8snetworkpolicyrequired
        
        violation[{"msg": msg}] {
          namespace := input.review.object.metadata.namespace
          not has_network_policy(namespace)
          msg := sprintf("Namespace %v must have at least one NetworkPolicy", [namespace])
        }
        
        has_network_policy(ns) {
          data.inventory.namespace[ns][_][_][_]["kind"] == "NetworkPolicy"
        }
EOF
kubectl apply -f custom-security-policies.yaml
```

### Step 5: Policy Testing and Validation
```bash
# Test policy enforcement with compliant pod
cat > compliant-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: compliant-test
  namespace: production
  labels:
    app: test-app
    version: "1.0"
    environment: production
    owner: security-team
spec:
  containers:
  - name: nginx
    image: nginx:1.25-alpine
    resources:
      requests:
        memory: "64Mi"
        cpu: "50m"
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 30
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
EOF
kubectl apply -f compliant-pod.yaml && echo "✅ Compliant pod created successfully"

# Test policy enforcement with non-compliant pod
cat > non-compliant-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: non-compliant-test
  namespace: production
  labels:
    app: test-app
    # Missing required labels: version, environment, owner
spec:
  containers:
  - name: nginx
    image: nginx:latest  # Blocked image tag
    # Missing resource requests
EOF
kubectl apply -f non-compliant-pod.yaml 2>&1 | grep -E "(denied|error)" && echo "✅ Policy enforcement working - non-compliant pod blocked"

# Check constraint violations
kubectl get K8sRequiredLabels required-labels -o yaml | grep -A 10 "status:"
kubectl get K8sSecureImages secure-images -o yaml | grep -A 10 "status:"
```

### Step 6: Policy Monitoring and Reporting
```yaml
# policy-monitoring.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gatekeeper-dashboard
  namespace: monitoring
data:
  dashboard.json: |
    {
      "dashboard": {
        "title": "OPA Gatekeeper Policy Enforcement",
        "panels": [
          {
            "title": "Policy Violations by Constraint",
            "type": "barchart",
            "targets": [
              {
                "expr": "gatekeeper_violations"
              }
            ]
          },
          {
            "title": "Constraint Audit Status",
            "type": "singlestat",
            "targets": [
              {
                "expr": "sum(gatekeeper_constraint_audit_last_run_status)"
              }
            ]
          },
          {
            "title": "Violations Over Time",
            "type": "graph",
            "targets": [
              {
                "expr": "rate(gatekeeper_violations[5m])"
              }
            ]
          }
        ]
      }
    }
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: policy-compliance-report
  namespace: gatekeeper-system
spec:
  schedule: "0 6 * * *"  # Daily at 6 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: reporter
            image: curlimages/curl:latest
            command:
            - /bin/sh
            - -c
            - |
              # Generate policy compliance report
              echo "=== OPA GATEKEEPER COMPLIANCE REPORT ===" > /reports/policy-report-20251003.txt
              echo "Generated: Fri Oct  3 08:14:22 AM PDT 2025" >> /reports/policy-report-20251003.txt
              echo "" >> /reports/policy-report-20251003.txt
              
              # Get constraint violations
              echo "CONSTRAINT VIOLATIONS:" >> /reports/policy-report-20251003.txt
              kubectl get constraints --all-namespaces -o json | jq -r '.items[] | "\(.metadata.name): \(.status.totalViolations) violations"' >> /reports/policy-report-20251003.txt
              
              # Get audit status
              echo "" >> /reports/policy-report-20251003.txt
              echo "AUDIT STATUS:" >> /reports/policy-report-20251003.txt
              kubectl get constraints --all-namespaces -o json | jq -r '.items[] | select(.status.auditTimestamp) | "\(.metadata.name): last audit \(.status.auditTimestamp)"' >> /reports/policy-report-20251003.txt
              
              echo "Report generated: /reports/policy-report-20251003.txt"
            volumeMounts:
            - name: reports
              mountPath: /reports
          volumes:
          - name: reports
            persistentVolumeClaim:
              claimName: policy-reports
          restartPolicy: OnFailure
EOF
kubectl apply -f policy-monitoring.yaml
```

## 🔍 Policy Enforcement Validation Checklist
- [ ] OPA Gatekeeper installed and operational
- [ ] Constraint templates created for security policies
- [ ] Constraints enforcing security requirements
- [ ] Policy violations being detected and reported
- [ ] Custom security policies implemented
- [ ] Policy monitoring and reporting configured
- [ ] Regular compliance reports generated

## 📝 Learning Outcomes
✅ Master OPA Gatekeeper policy enforcement  
✅ Create custom constraint templates and constraints  
✅ Implement policy-as-code practices  
✅ Establish comprehensive policy monitoring

## 🎯 Next Lab Preview
In Lab 6.4, we'll complete Module 6 with Compliance Reporting & Auditing for comprehensive governance.

