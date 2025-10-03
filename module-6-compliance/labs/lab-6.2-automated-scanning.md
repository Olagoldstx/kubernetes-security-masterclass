# LAB 6.2: Automated Compliance Scanning
## Continuous Royal Inspections

## 🎯 Lab Objectives
1. Implement continuous compliance scanning
2. Configure automated security assessments
3. Establish real-time compliance monitoring
4. Create remediation workflows

## 🏰 Analogy: Continuous Royal Inspections
- **Automated Scanners** = Royal inspection squads
- **Continuous Monitoring** = Regular castle inspections
- **Real-time Alerts** = Emergency signal flares
- **Remediation Workflows** = Castle repair crews

## 🛠️ Hands-On Implementation

### Step 1: Install kube-hunter for Attack Simulation
```bash
# Install kube-hunter
pip install kube-hunter

# Run kube-hunter in passive mode (discovery only)
kube-hunter --remote <your-cluster-ip> --report json > kube-hunter-report.json

# Run kube-hunter in active mode (penetration testing)
kube-hunter --remote <your-cluster-ip> --active --report json > kube-hunter-active.json

# Analyze results
cat kube-hunter-report.json | jq '.vulnerabilities[] | {severity: .severity, category: .category, description: .description}'

# Create kube-hunter CronJob for regular scanning
cat > kube-hunter-scan.yaml << 'EOF'
apiVersion: batch/v1
kind: CronJob
metadata:
  name: kube-hunter-scan
  namespace: security
spec:
  schedule: "0 3 * * 0"  # Weekly on Sunday at 3 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: kube-hunter
            image: aquasec/kube-hunter:latest
            command:
            - kube-hunter
            - --pod
            - --report
            - json
            - --log
            - none
            env:
            - name: KUBE_HUNTER_ARGS
              value: "--pod --report json --log none"
          restartPolicy: OnFailure
EOF
kubectl apply -f kube-hunter-scan.yaml
```

### Step 2: Implement kube-bench Operator for Continuous CIS Scanning
```bash
# Install kube-bench operator
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml

# Create ConfigMap with custom benchmarks
cat > kube-bench-config.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-bench-config
  namespace: security
data:
  config.yaml: |
    targets:
      master:
        - master
      node:
        - node
      etcd:
        - etcd
      policies:
        - policies
    schedule: "0 2 * * *"  # Daily at 2 AM
    output:
      format: json
      file: /reports/kube-bench-20251003.json
EOF
kubectl apply -f kube-bench-config.yaml
```

### Step 3: Deploy Trivy for Vulnerability Scanning
```bash
# Install Trivy operator for Kubernetes
helm repo add aqua https://aquasecurity.github.io/helm-charts/
helm repo update

# Install Trivy operator
helm install trivy-operator aqua/trivy-operator \
  --namespace trivy-system \
  --create-namespace \
  --set trivy.ignoreUnfixed=true \
  --set trivy.severity=CRITICAL,HIGH

# Wait for operator to be ready
kubectl wait --namespace trivy-system --for=condition=ready pod -l app=trivy-operator --timeout=300s

# Create VulnerabilityScan for all namespaces
cat > vulnerability-scan.yaml << 'EOF'
apiVersion: aquasecurity.github.io/v1alpha1
kind: VulnerabilityReport
metadata:
  name: cluster-wide-scan
  namespace: default
spec:
  schedule: "0 4 * * *"  # Daily at 4 AM
  scanType: Cluster
  parameters:
    ignoreUnfixed: true
    severity: CRITICAL,HIGH
EOF
kubectl apply -f vulnerability-scan.yaml
```

### Step 4: Implement Compliance Dashboard with Prometheus
```yaml
# compliance-metrics.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: compliance-metrics
  namespace: monitoring
data:
  compliance-rules.yaml: |
    groups:
    - name: compliance.metrics
      rules:
      - record: compliance:cis_controls_passed:percentage
        expr: (sum(kube_bench_controls_passed) / sum(kube_bench_controls_total)) * 100
      
      - record: compliance:vulnerabilities:critical
        expr: sum(trivy_vulnerabilities{severity="CRITICAL"})
      
      - record: compliance:scan_duration:seconds
        expr: time() - kube_bench_scan_start_time
      
      - record: compliance:remediation_backlog:count
        expr: count(compliance_violations{status="open"})
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: compliance-exporter
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: compliance-exporter
  template:
    metadata:
      labels:
        app: compliance-exporter
    spec:
      containers:
      - name: exporter
        image: prometheus-community/compliance-exporter:latest
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: config
          mountPath: /etc/compliance-exporter
      volumes:
      - name: config
        configMap:
          name: compliance-metrics
---
apiVersion: v1
kind: Service
metadata:
  name: compliance-exporter
  namespace: monitoring
spec:
  selector:
    app: compliance-exporter
  ports:
  - port: 8080
    targetPort: 8080
EOF
kubectl apply -f compliance-metrics.yaml
```

### Step 5: Automated Remediation Workflows
```yaml
# automated-remediation.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: auto-remediation
  namespace: security
spec:
  schedule: "*/30 * * * *"  # Every 30 minutes
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: remediation-bot
          containers:
          - name: remediator
            image: kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              # Check for critical vulnerabilities
              CRITICAL_VULNS=0
              
              if [ "" -gt 0 ]; then
                echo "🚨 CRITICAL VULNERABILITIES DETECTED: "
                
                # Automated remediation actions
                # 1. Restart deployments with critical vulnerabilities
                kubectl get vulnerabilityreports -A -o json |                   jq -r '.items[] | select(.report.summary.criticalCount > 0) | .metadata.namespace + "/" + .metadata.name' |                   while read resource; do
                    ns=
                    name=
                    echo "Restarting deployment in : "
                    kubectl rollout restart deployment  -n 
                  done
                
                # 2. Create remediation ticket
                kubectl create configmap remediation-ticket-1759503914 \
                  --from-literal=timestamp=2025-10-03T15:05:14Z \
                  --from-literal=critical_vulnerabilities= \
                  --namespace=security-events
                
                echo "✅ Automated remediation executed for  critical vulnerabilities"
              else
                echo "✅ No critical vulnerabilities requiring immediate remediation"
              fi
          restartPolicy: OnFailure
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: remediation-bot
  namespace: security
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: remediation-bot
rules:
- apiGroups: [""]
  resources: ["configmaps", "pods", "deployments"]
  verbs: ["get", "list", "create", "patch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "patch"]
- apiGroups: ["aquasecurity.github.io"]
  resources: ["vulnerabilityreports"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: remediation-bot
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: remediation-bot
subjects:
- kind: ServiceAccount
  name: remediation-bot
  namespace: security
EOF
kubectl apply -f automated-remediation.yaml
```

### Step 6: Compliance-as-Code with GitOps
```yaml
# compliance-gitops.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: compliance-policies
  namespace: security
data:
  policy-1-no-privileged.yaml: |
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
  
  policy-2-required-labels.yaml: |
    apiVersion: constraints.gatekeeper.sh/v1beta1
    kind: K8sRequiredLabels
    metadata:
      name: required-labels
    spec:
      match:
        kinds:
        - apiGroups: [""]
          kinds: ["Pod"]
      parameters:
        labels: ["app", "version", "owner"]
  
  policy-3-resource-limits.yaml: |
    apiVersion: constraints.gatekeeper.sh/v1beta1
    kind: K8sContainerLimits
    metadata:
      name: container-limits
    spec:
      match:
        kinds:
        - apiGroups: [""]
          kinds: ["Pod"]
      parameters:
        cpu_request: "10m"
        memory_request: "32Mi"
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: compliance-policies
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/compliance-policies.git
    targetRevision: main
    path: policies
  destination:
    server: https://kubernetes.default.svc
    namespace: gatekeeper-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF
kubectl apply -f compliance-gitops.yaml
```

## 🔍 Compliance Validation Checklist
- [ ] Continuous vulnerability scanning implemented
- [ ] Automated CIS benchmark scanning configured
- [ ] Real-time compliance monitoring operational
- [ ] Automated remediation workflows established
- [ ] Compliance-as-code policies enforced
- [ ] Regular compliance reports generated

## 📝 Learning Outcomes
✅ Master automated compliance scanning tools  
✅ Implement continuous security monitoring  
✅ Establish automated remediation workflows  
✅ Enforce compliance-as-code practices

## 🎯 Next Lab Preview
In Lab 6.3, we'll implement Policy Enforcement with OPA/Gatekeeper for automated policy compliance.

