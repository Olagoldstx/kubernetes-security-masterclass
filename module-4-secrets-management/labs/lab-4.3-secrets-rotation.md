# LAB 4.3: Secrets Rotation Automation
## Royal Treasury Maintenance

## 🎯 Lab Objectives
1. Implement automated secret rotation
2. Configure secret expiration and renewal
3. Establish secret lifecycle management
4. Create rotation policies and procedures

## 🏰 Analogy: Treasury Maintenance Schedule
- **Secret Rotation** = Regular treasury audits and lock changes
- **Expiration Policies** = Time-limited access tokens
- **Lifecycle Management** = Treasury inventory and maintenance
- **Automation** = Scheduled maintenance crews

## 🛠️ Hands-On Implementation

### Step 1: Configure Vault Secret Leases and TTLs
```bash
# Configure database secret TTL (Time To Live)
vault write database/roles/ecommerce-role \
  db_name=ecommerce-postgres \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}';" \
  default_ttl="1h" \
  max_ttl="24h" \
  revocation_statements="DROP ROLE \"{{name}}\";"

# Create KV secret with lease
vault kv put -mount=secret ecommerce/payment \
  api-key=initial-key-123 \
  ttl=24h

# Check secret lease information
vault lease lookup <lease-id>
```

### Step 2: Implement Automatic Secret Rotation
```bash
# Create rotation policy for static secrets
cat > rotation-policy.hcl << 'EOF'
path "secret/data/ecommerce/*" {
  capabilities = ["read", "update"]
  
  # Auto-rotate every 24 hours
  min_wrapping_ttl = "24h"
  max_wrapping_ttl = "24h"
}
EOF

vault policy write secret-rotation rotation-policy.hcl

# Create scheduled rotation using Vault's built-in rotation
vault write sys/rotate/config \
  enabled=true \
  max_ttl=720h
```

### Step 3: Kubernetes CronJob for Secret Rotation
```yaml
# secret-rotation-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: secret-rotation
  namespace: ecommerce
spec:
  schedule: "0 */6 * * *"  # Every 6 hours
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: secret-rotator
          containers:
          - name: rotate-secrets
            image: vault:latest
            command:
            - /bin/sh
            - -c
            - |
              # Rotate database credentials
              vault read database/creds/ecommerce-role
              
              # Update static API keys
              vault kv patch secret/ecommerce/payment \
                api-key="new-key-1759452235"
              
              echo "Secrets rotated successfully at Thu Oct  2 05:43:55 PM PDT 2025"
            env:
            - name: VAULT_ADDR
              value: "http://vault.vault:8200"
            - name: VAULT_TOKEN
              valueFrom:
                secretKeyRef:
                  name: vault-token
                  key: token
          restartPolicy: OnFailure
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: secret-rotator
  namespace: ecommerce
```

### Step 4: External Secrets with Automatic Refresh
```yaml
# external-secret-with-refresh.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: payment-api-with-refresh
  namespace: ecommerce
spec:
  refreshInterval: "1h"  # Auto-refresh every hour
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: payment-secret
    creationPolicy: Owner
    # Force secret update in pods
    template:
      metadata:
        annotations:
          secret-update-timestamp: "{{ .Timestamp }}"
  data:
  - secretKey: api-key
    remoteRef:
      key: secret/data/ecommerce/payment
      property: api-key
      # Version for tracking changes
      version: "latest"
```

### Step 5: Secret Lifecycle Monitoring
```bash
# Create secret lifecycle dashboard
cat > secret-monitoring.sh << 'EOF'
echo "=== SECRET LIFECYCLE MONITORING ==="

# Check expiring secrets
vault list -format=json sys/leases/lookup/database/creds/ecommerce-role |   jq -r '.[]' | while read lease; do
    expiry=
    echo "Lease:  expires at: "
done

# Check secret versions
vault kv metadata get -format=json secret/ecommerce/payment |   jq -r '.data.versions | to_entries[] | "Version: \(.key) Created: \(.value.created_time)"'

# Monitor ExternalSecret status
kubectl get externalsecrets -A -o wide
EOF

chmod +x secret-monitoring.sh
```

### Step 6: Emergency Secret Rotation Procedure
```yaml
# emergency-rotation.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: emergency-secret-rotation
  namespace: ecommerce
  annotations:
    emergency: "true"
    rotation-reason: "security-breach-suspected"
spec:
  template:
    spec:
      serviceAccountName: emergency-rotator
      containers:
      - name: emergency-rotate
        image: vault:latest
        command:
        - /bin/sh
        - -c
        - |
          set -e
          echo "🚨 EMERGENCY SECRET ROTATION INITIATED"
          
          # Revoke all existing database credentials
          vault lease revoke -prefix database/creds/ecommerce-role
          echo "✅ All database credentials revoked"
          
          # Generate new root database password
          new_password="qxvAOl9QYZ70sqC7eIgmznZ5yJA+afYIVluR3G6JTEs="
          vault kv put secret/ecommerce/database-root \
            password="" \
            rotated_at="2025-10-03T00:43:55Z"
          
          # Rotate all API keys
          vault kv patch secret/ecommerce/payment \
            api-key="emergency-key-1759452235"
          vault kv patch secret/ecommerce/email \
            api-key="emergency-key-1759452235"
          
          echo "🚨 EMERGENCY ROTATION COMPLETE"
          echo "All secrets have been rotated due to security concerns"
        env:
        - name: VAULT_ADDR
          value: "http://vault.vault:8200"
        - name: VAULT_TOKEN
          valueFrom:
            secretKeyRef:
              name: vault-emergency-token
              key: token
      restartPolicy: Never
```

## 🔍 Security Validation Checklist
- [ ] Automated rotation schedules configured
- [ ] Secret TTLs and expiration policies set
- [ ] Emergency rotation procedures documented
- [ ] Lifecycle monitoring implemented
- [ ] Rollback procedures established
- [ ] Compliance with security policies

## 📝 Learning Outcomes
✅ Master secret rotation automation  
✅ Implement secret lifecycle management  
✅ Configure emergency rotation procedures  
✅ Establish comprehensive secret monitoring

## 🎯 Secrets Management Module Nearly Complete!
Only one lab remaining in Module 4!
- ✅ Lab 4.1: Kubernetes Native Secrets
- ✅ Lab 4.2: External Secrets with Vault  
- ✅ Lab 4.3: Secrets Rotation Automation

**Your secrets are now self-maintaining and secure* 🔄

