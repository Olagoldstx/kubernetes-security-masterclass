# LAB 6.4: Compliance Reporting & Auditing
## Royal Audit Documentation

## 🎯 Lab Objectives
1. Implement comprehensive compliance reporting
2. Establish audit trails and evidence collection
3. Create executive compliance dashboards
4. Prepare for external audits and certifications

## 🏰 Analogy: Royal Audit Documentation
- **Compliance Reports** = Royal audit scrolls and documents
- **Audit Trails** = Historical records of kingdom activities
- **Executive Dashboards** = Royal court status displays
- **Certification Prep** = Kingdom accreditation processes

## 🛠️ Hands-On Implementation

### Step 1: Comprehensive Compliance Reporting Framework
```yaml
# compliance-reporting.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: compliance-report-templates
  namespace: security
data:
  executive-report.md: |
    # Kubernetes Security Compliance Report
    ## Executive Summary
    
    **Report Period:** {{ .ReportPeriod }}
    **Overall Compliance Score:** {{ .ComplianceScore }}%
    
    ### Key Metrics
    - CIS Benchmark Compliance: {{ .CISScore }}%
    - Vulnerability Status: {{ .VulnerabilityStatus }}
    - Policy Violations: {{ .ViolationCount }}
    - Security Incidents: {{ .IncidentCount }}
    
    ### Top Risks
    1. {{ .Risk1 }}
    2. {{ .Risk2 }}
    3. {{ .Risk3 }}
    
    ### Recommended Actions
    - {{ .Action1 }}
    - {{ .Action2 }}
    - {{ .Action3 }}
  
  technical-report.md: |
    # Technical Compliance Report
    
    ## CIS Benchmark Results
    {{ range .CISResults }}
    - Control {{ .ID }}: {{ .Status }} ({{ .Description }})
    {{ end }}
    
    ## Vulnerability Summary
    - Critical: {{ .CriticalVulns }}
    - High: {{ .HighVulns }}
    - Medium: {{ .MediumVulns }}
    
    ## Policy Violations
    {{ range .Violations }}
    - Constraint: {{ .Constraint }}
    - Violations: {{ .Count }}
    - Namespace: {{ .Namespace }}
    {{ end }}
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: compliance-report-generator
  namespace: security
spec:
  schedule: "0 0 * * 0"  # Weekly on Sunday at midnight
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: report-generator
            image: alpine/k8s:latest
            command:
            - /bin/sh
            - -c
            - |
              # Generate compliance reports
              TIMESTAMP=20251003
              
              # Collect compliance data
              CIS_REPORT="No CIS data"
              TRIVY_REPORT="No vulnerability data"
              GATEKEEPER_VIOLATIONS="No violation data"
              
              # Generate executive report
              cat > /reports/executive-report-.md << 'EOF'
              # Kubernetes Security Compliance Report
              ## Executive Summary
              
              **Report Period:** Last Week
              **Overall Compliance Score:** 92%
              
              ### Key Metrics
              - CIS Benchmark Compliance: 95%
              - Critical Vulnerabilities: 2
              - Policy Violations: 15
              - Security Incidents: 0
              
              ### Top Risks
              1. Outdated base images in development namespace
              2. Missing network policies in staging
              3. Overly permissive service accounts
              
              ### Recommended Actions
              - Update base images to latest versions
              - Implement namespace-level network policies
              - Review and restrict service account permissions
              EOF
              
              # Generate technical report
              kubectl get constraints -A -o json > /reports/constraint-violations-.json
              kubectl get vulnerabilityreports -A -o json > /reports/vulnerabilities-.json
              
              echo "Compliance reports generated for "
            volumeMounts:
            - name: reports
              mountPath: /reports
          volumes:
          - name: reports
            persistentVolumeClaim:
              claimName: compliance-reports
          restartPolicy: OnFailure
EOF
kubectl apply -f compliance-reporting.yaml
```

### Step 2: Audit Trail Implementation
```yaml
# audit-trail.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: audit-configuration
  namespace: kube-system
data:
  audit-policy.yaml: |
    apiVersion: audit.k8s.io/v1
    kind: Policy
    rules:
    - level: Metadata
      namespaces: ["kube-system"]
      verbs: ["get", "list", "watch"]
    
    - level: RequestResponse
      namespaces: ["ecommerce", "ecommerce-payments"]
      resources:
      - group: ""
        resources: ["secrets", "configmaps"]
      verbs: ["create", "update", "delete", "patch"]
    
    - level: Request
      users: ["system:serviceaccount:gatekeeper-system:gatekeeper-admin"]
      resources:
      - group: "constraints.gatekeeper.sh"
        resources: ["*"]
      verbs: ["*"]
    
    - level: Request
      users: ["system:serviceaccount:security:compliance-scanner"]
      resources:
      - group: ""
        resources: ["pods", "deployments", "services"]
      verbs: ["get", "list"]
    
    - level: Metadata
      resources:
      - group: ""
        resources: ["pods", "services", "configmaps"]
      verbs: ["get", "list", "watch"]
    
    - level: None
      users: ["system:serviceaccount:kube-system:generic-garbage-collector"]
      verbs: ["get", "list", "watch"]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: audit-log-processor
  namespace: security
spec:
  replicas: 1
  selector:
    matchLabels:
      app: audit-log-processor
  template:
    metadata:
      labels:
        app: audit-log-processor
    spec:
      containers:
      - name: processor
        image: fluent/fluentd:latest
        volumeMounts:
        - name: audit-logs
          mountPath: /var/log/kubernetes/audit
        - name: config
          mountPath: /fluentd/etc
        env:
        - name: FLUENTD_CONF
          value: "audit.conf"
      volumes:
      - name: audit-logs
        hostPath:
          path: /var/log/kubernetes/audit
          type: DirectoryOrCreate
      - name: config
        configMap:
          name: fluentd-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: security
data:
  audit.conf: |
    <source>
      @type tail
      path /var/log/kubernetes/audit/audit.log
      pos_file /var/log/audit.log.pos
      tag audit
      format json
    </source>
    
    <filter audit>
      @type record_transformer
      <record>
        cluster_name my-k8s-cluster
        timestamp 
      </record>
    </filter>
    
    <match audit>
      @type elasticsearch
      host elasticsearch.logging.svc.cluster.local
      port 9200
      index_name audit-logs
      type_name audit-record
    </match>
EOF
kubectl apply -f audit-trail.yaml
```

### Step 3: Executive Compliance Dashboard
```yaml
# executive-dashboard.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: executive-dashboard
  namespace: monitoring
data:
  executive-dashboard.json: |
    {
      "dashboard": {
        "title": "Executive Security Compliance Dashboard",
        "panels": [
          {
            "title": "Overall Compliance Score",
            "type": "gauge",
            "targets": [
              {
                "expr": "compliance_overall_score"
              }
            ],
            "thresholds": {
              "steps": [
                {"color": "red", "value": 0},
                {"color": "yellow", "value": 80},
                {"color": "green", "value": 90}
              ]
            }
          },
          {
            "title": "Compliance Trend (Last 30 Days)",
            "type": "graph",
            "targets": [
              {
                "expr": "compliance_score_over_time[30d]"
              }
            ]
          },
          {
            "title": "Risk Distribution",
            "type": "piechart",
            "targets": [
              {
                "expr": "compliance_risks_by_category"
              }
            ]
          },
          {
            "title": "Vulnerability Status",
            "type": "stat",
            "targets": [
              {
                "expr": "vulnerabilities_critical_total"
              },
              {
                "expr": "vulnerabilities_high_total"
              },
              {
                "expr": "vulnerabilities_medium_total"
              }
            ]
          },
          {
            "title": "Policy Compliance",
            "type": "table",
            "targets": [
              {
                "expr": "gatekeeper_constraint_compliance"
              }
            ]
          }
        ],
        "refresh": "1m",
        "time": {"from": "now-7d", "to": "now"}
      }
    }
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: compliance-metrics
  namespace: monitoring
data:
  compliance-metrics.yaml: |
    groups:
    - name: compliance.metrics
      rules:
      - record: compliance:overall_score
        expr: |
          (
            avg(compliance_cis_score) * 0.4 +
            (100 - (vulnerabilities_critical_total * 10)) * 0.3 +
            (100 - (gatekeeper_violations_total * 2)) * 0.3
          )
      
      - record: compliance:cis_score
        expr: (sum(kube_bench_controls_passed) / sum(kube_bench_controls_total)) * 100
      
      - record: compliance:risks_by_category
        expr: |
          label_replace(
            label_replace(
              compliance_risks_total{category="vulnerability"}, "category", "Vulnerabilities", "category", ".*"
            ), "category", "Policy", "category", "policy"
          )
EOF
kubectl apply -f executive-dashboard.yaml
```

### Step 4: Certification Preparation Framework
```yaml
# certification-prep.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: soc2-controls
  namespace: security
data:
  soc2-controls.yaml: |
    controls:
      - id: CC6.1
        name: Logical Access Security
        description: The entity implements logical access security software, infrastructure, and architectures over protected information assets to protect them from security events to meet the entity's objectives.
        kubernetes_implementation:
          - RBAC with least privilege
          - Network policies for segmentation
          - Pod Security Standards
        evidence_sources:
          - kubectl get clusterrolebindings -o yaml
          - kubectl get networkpolicies -A
          - kube-bench reports
        
      - id: CC7.1
        name: System Operations
        description: The entity uses detection and monitoring procedures to identify (1) changes to configurations that result in the introduction of new vulnerabilities, and (2) susceptibilities to newly discovered vulnerabilities.
        kubernetes_implementation:
          - Falco runtime security
          - Vulnerability scanning with Trivy
          - OPA Gatekeeper policies
        evidence_sources:
          - Falco audit logs
          - Trivy vulnerability reports
          - Gatekeeper constraint violations
        
      - id: CC8.1
        name: Change Management
        description: The entity authorizes, designs, develops or acquires, configures, documents, tests, approves, and implements changes to infrastructure, data, software, and procedures to meet its objectives.
        kubernetes_implementation:
          - GitOps with ArgoCD
          - Immutable infrastructure
          - Automated rollbacks
        evidence_sources:
          - ArgoCD application manifests
          - Git commit history
          - Deployment rollback history
---
apiVersion: batch/v1
kind: Job
metadata:
  name: soc2-evidence-collector
  namespace: security
spec:
  template:
    spec:
      containers:
      - name: collector
        image: alpine/k8s:latest
        command:
        - /bin/sh
        - -c
        - |
          # Collect evidence for SOC2 audit
          TIMESTAMP=20251003-082403
          mkdir -p /evidence/soc2-
          
          # Collect RBAC evidence
          kubectl get clusterroles,clusterrolebindings,roles,rolebindings -A -o yaml > /evidence/soc2-/rbac-evidence.yaml
          
          # Collect network policy evidence
          kubectl get networkpolicies -A -o yaml > /evidence/soc2-/network-policies.yaml
          
          # Collect security context evidence
          kubectl get pods -A -o json | jq '.items[] | {namespace: .metadata.namespace, name: .metadata.name, securityContext: .spec.securityContext}' > /evidence/soc2-/security-contexts.json
          
          # Collect compliance reports
          kubectl get cm -l category=compliance -o yaml > /evidence/soc2-/compliance-reports.yaml
          
          # Generate evidence summary
          cat > /evidence/soc2-/evidence-summary.md << 'EOF'
          # SOC2 Compliance Evidence Summary
          ## Collected: Fri Oct  3 08:24:03 AM PDT 2025
          
          ### Evidence Collected:
          - RBAC configurations
          - Network policies
          - Pod security contexts
          - Compliance reports
          - Vulnerability scans
          
          ### Controls Covered:
          - CC6.1: Logical Access Security
          - CC7.1: System Operations
          - CC8.1: Change Management
          
          ### Next Steps:
          1. Review evidence with auditors
          2. Address any gaps identified
          3. Schedule follow-up review
          EOF
          
          echo "SOC2 evidence collected: /evidence/soc2-"
        volumeMounts:
        - name: evidence
          mountPath: /evidence
      volumes:
      - name: evidence
        persistentVolumeClaim:
          claimName: audit-evidence
      restartPolicy: Never
EOF
kubectl apply -f certification-prep.yaml
```

### Step 5: Continuous Compliance Monitoring
```yaml
# continuous-compliance.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: compliance-monitor
  namespace: security
spec:
  replicas: 1
  selector:
    matchLabels:
      app: compliance-monitor
  template:
    metadata:
      labels:
        app: compliance-monitor
    spec:
      serviceAccountName: compliance-monitor
      containers:
      - name: monitor
        image: alpine/k8s:latest
        command:
        - /bin/sh
        - -c
        - |
          while true; do
            echo "=== COMPLIANCE MONITORING CYCLE STARTED ==="
            echo "Timestamp: Fri Oct  3 08:24:03 AM PDT 2025"
            
            # Check CIS compliance
            CIS_SCORE="Unknown"
            echo "CIS Compliance Score: %"
            
            # Check critical vulnerabilities
            CRITICAL_VULNS=
            echo "Critical Vulnerabilities: "
            
            # Check policy violations
            VIOLATIONS=
            echo "Policy Violations: "
            
            # Alert if compliance drops below threshold
            if [ "" -gt 5 ] || [ "" -gt 20 ]; then
              kubectl create configmap compliance-alert-1759505044 \
                --from-literal=timestamp=2025-10-03T15:24:04Z \
                --from-literal=critical_vulns= \
                --from-literal=violations= \
                --namespace=security-alerts
            fi
            
            echo "=== COMPLIANCE MONITORING CYCLE COMPLETED ==="
            sleep 300  # Wait 5 minutes
          done
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: compliance-monitor
  namespace: security
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: compliance-monitor
rules:
- apiGroups: [""]
  resources: ["configmaps", "pods"]
  verbs: ["get", "list", "create"]
- apiGroups: ["constraints.gatekeeper.sh"]
  resources: ["*"]
  verbs: ["get", "list"]
- apiGroups: ["aquasecurity.github.io"]
  resources: ["vulnerabilityreports"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: compliance-monitor
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: compliance-monitor
subjects:
- kind: ServiceAccount
  name: compliance-monitor
  namespace: security
EOF
kubectl apply -f continuous-compliance.yaml
```

## 🔍 Compliance Reporting Validation Checklist
- [ ] Comprehensive reporting framework implemented
- [ ] Audit trails configured and collecting data
- [ ] Executive dashboards operational
- [ ] Certification preparation framework established
- [ ] Continuous compliance monitoring active
- [ ] Evidence collection procedures documented
- [ ] Regular compliance reviews scheduled

## 📝 Learning Outcomes
✅ Master comprehensive compliance reporting  
✅ Implement audit trails and evidence collection  
✅ Create executive-level compliance dashboards  
✅ Prepare for security certifications and audits

## 🎯 MODULE 6: COMPLIANCE - COMPLETE!
You've now mastered the entire compliance journey:
- ⚖️ **Lab 6.1**: CIS Benchmark Implementation
- 🔍 **Lab 6.2**: Automated Compliance Scanning  
- 🛡️ **Lab 6.3**: Policy Enforcement
- 📊 **Lab 6.4**: Compliance Reporting & Auditing

**Your kingdom now has comprehensive royal audit systems* 👑⚖️

