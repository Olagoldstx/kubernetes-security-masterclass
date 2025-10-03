# LAB 6.1: CIS Kubernetes Benchmark Implementation
## Royal Kingdom Regulations

## 🎯 Lab Objectives
1. Understand and implement CIS Kubernetes benchmarks
2. Configure automated compliance scanning
3. Establish baseline security controls
4. Generate compliance reports

## 🏰 Analogy: Kingdom Laws and Regulations
- **CIS Benchmarks** = Royal decrees and laws
- **Security Controls** = Castle defense requirements
- **Compliance Scans** = Royal inspections
- **Audit Reports** = Official compliance documents

## 🛠️ Hands-On Implementation

### Step 1: Install and Run kube-bench
```bash
# Install kube-bench
curl -L https://github.com/aquasecurity/kube-bench/releases/download/v0.6.17/kube-bench_0.6.17_linux_amd64.tar.gz -o kube-bench.tar.gz
tar -xzf kube-bench.tar.gz
sudo mv kube-bench /usr/local/bin/

# Run CIS benchmark for control plane
kube-bench run --targets master

# Run CIS benchmark for worker nodes
kube-bench run --targets node

# Run all benchmarks
kube-bench run --targets master,node,etcd,policies

# Generate JSON report
kube-bench run --targets master,node --json > cis-report.json
```

### Step 2: Analyze Benchmark Results
```bash
# Parse and analyze results
cat cis-report.json | jq '.Totals'
cat cis-report.json | jq '.Controls[] | select(.passed == false) | .id'

# Create compliance summary
echo "=== CIS BENCHMARK COMPLIANCE SUMMARY ==="
kube-bench run --targets master,node | grep -E "(PASS|FAIL|WARN)" | sort | uniq -c

# Identify critical failures
kube-bench run --targets master,node | grep "FAIL" | grep -E "(1.1|1.2|4.2)" || echo "No critical failures found"
```

### Step 3: Implement CIS Remediations
```yaml
# cis-remediations.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cis-remediation-guide
  namespace: kube-system
data:
  control-1.2.1.md: |
    # Control 1.2.1: Ensure that the --anonymous-auth argument is set to false
    **Remediation:**
    - Edit the API server pod specification file /etc/kubernetes/manifests/kube-apiserver.yaml
    - Set --anonymous-auth=false
    - Restart the API server

  control-4.2.1.md: |
    # Control 4.2.1: Ensure that kubelet service file permissions are set to 644 or more restrictive
    **Remediation:**
    - Run: chmod 644 /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
    - Verify: stat -c %a /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

  control-5.2.2.md: |
    # Control 5.2.2: Minimize the admission of containers wishing to share the host process ID namespace
    **Remediation:**
    - Create PodSecurityPolicy or use Pod Security Standards
    - Set hostPID: false in pod specifications
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cis-compliance-scan
  namespace: security
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: kube-bench
            image: aquasec/kube-bench:latest
            command:
            - kube-bench
            - run
            - --targets
            - master,node
            - --json
            - --outputfile
            - /reports/cis-scan-20251002.json
            volumeMounts:
            - name: reports
              mountPath: /reports
          volumes:
          - name: reports
            persistentVolumeClaim:
              claimName: compliance-reports
          restartPolicy: OnFailure
```

### Step 4: CIS-Compliant Cluster Configuration
```yaml
# cis-compliant-cluster.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
apiServer:
  extraArgs:
    anonymous-auth: "false"
    authorization-mode: "Node,RBAC"
    enable-admission-plugins: "NodeRestriction,PodSecurityPolicy"
    audit-log-path: /var/log/kubernetes/audit.log
    audit-log-maxage: "30"
    audit-log-maxbackup: "10"
    audit-log-maxsize: "100"
    service-account-lookup: "true"
controllerManager:
  extraArgs:
    use-service-account-credentials: "true"
    feature-gates: "RotateKubeletServerCertificate=true"
scheduler:
  extraArgs:
    feature-gates: "RotateKubeletServerCertificate=true"
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
serverTLSBootstrap: true
rotateCertificates: true
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook
readOnlyPort: 0
protectKernelDefaults: true
makeIPTablesUtilChains: true
eventRecordQPS: 0
```

### Step 5: Compliance Dashboard and Reporting
```yaml
# compliance-dashboard.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: compliance-dashboard
  namespace: monitoring
data:
  dashboard.json: |
    {
      "dashboard": {
        "title": "Kubernetes Compliance Dashboard",
        "panels": [
          {
            "title": "CIS Benchmark Compliance",
            "type": "singlestat",
            "targets": [
              {
                "expr": "compliance_cis_passed / compliance_cis_total * 100"
              }
            ],
            "format": "percent"
          },
          {
            "title": "Compliance Over Time",
            "type": "graph",
            "targets": [
              {
                "expr": "rate(compliance_scan_total[24h])"
              }
            ]
          },
          {
            "title": "Top Compliance Violations",
            "type": "table",
            "targets": [
              {
                "expr": "topk(10, compliance_violations_total)"
              }
            ]
          }
        ]
      }
    }
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: compliance-alerts
  namespace: monitoring
data:
  alerts.yaml: |
    groups:
    - name: compliance.alerts
      rules:
      - alert: CISComplianceDegraded
        expr: compliance_cis_score < 90
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "Kubernetes CIS compliance score below 90%"
          description: "Current CIS compliance score is {{  }}%. Review and fix compliance violations."
      
      - alert: CriticalControlFailure
        expr: compliance_critical_failures > 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "Critical CIS control failure detected"
          description: "{{  }} critical CIS control failures need immediate attention."
```

## 🔍 Compliance Validation Checklist
- [ ] CIS benchmarks implemented and validated
- [ ] Automated compliance scanning configured
- [ ] Critical controls passing
- [ ] Compliance dashboard operational
- [ ] Regular audit reports generated
- [ ] Remediation procedures documented

## 📝 Learning Outcomes
✅ Master CIS Kubernetes benchmark implementation  
✅ Configure automated compliance scanning  
✅ Establish compliance monitoring and reporting  
✅ Implement remediation procedures

## 🎯 Next Lab Preview
In Lab 6.2, we'll implement automated compliance scanning with continuous monitoring.

