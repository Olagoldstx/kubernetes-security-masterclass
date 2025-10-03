# LAB 7.1: Complete E-Commerce Platform Deployment
## Building the Digital Kingdom

## 🎯 Lab Objectives
1. Deploy complete microservices e-commerce architecture
2. Implement secure service-to-service communication
3. Configure external integrations and dependencies
4. Establish monitoring and observability

## 🏰 Analogy: Building a Complete Medieval City
- **Microservices** = Different city districts (markets, residences, guards)
- **API Gateway** = City gates and entry points
- **Database** = Royal archives and treasury
- **External Services** = Trade routes with other kingdoms

## 🛠️ Hands-On Implementation

### Step 1: Create Production Namespace Structure
```bash
# Create dedicated namespaces with security labels
kubectl create namespace ecommerce-production
kubectl create namespace ecommerce-payments
kubectl create namespace ecommerce-database
kubectl create namespace ecommerce-monitoring

# Apply security labels
kubectl label namespace ecommerce-production environment=production security-tier=restricted
kubectl label namespace ecommerce-payments environment=production security-tier=sensitive
kubectl label namespace ecommerce-database environment=production security-tier=critical
kubectl label namespace ecommerce-monitoring environment=production security-tier=monitoring

# Apply Pod Security Standards
kubectl label namespace ecommerce-production pod-security.kubernetes.io/enforce=restricted
kubectl label namespace ecommerce-payments pod-security.kubernetes.io/enforce=restricted
kubectl label namespace ecommerce-database pod-security.kubernetes.io/enforce=baseline
kubectl label namespace ecommerce-monitoring pod-security.kubernetes.io/enforce=baseline

echo "=== PRODUCTION NAMESPACES CREATED ==="
kubectl get namespaces -l environment=production --show-labels
```

### Step 2: Deploy Core E-Commerce Microservices
```yaml
# ecommerce-core-services.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ecommerce-config
  namespace: ecommerce-production
data:
  app-config.yaml: |
    database:
      host: postgresql.ecommerce-database.svc.cluster.local
      port: 5432
      name: ecommerce
    redis:
      host: redis.ecommerce-production.svc.cluster.local
      port: 6379
    payment:
      gateway_url: https://api.payment-gateway.com
      timeout: 30s
    email:
      service_url: https://api.email-service.com
      from: noreply@ecommerce-app.com
---
# Frontend Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: ecommerce-production
  labels:
    app: frontend
    tier: web
    security-level: public
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
      tier: web
  template:
    metadata:
      labels:
        app: frontend
        tier: web
        security-level: public
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3000"
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
      containers:
      - name: frontend
        image: ecommerce-frontend:1.0.0
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          value: "production"
        - name: API_URL
          value: "http://api-gateway.ecommerce-production.svc.cluster.local:8080"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: true
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: ecommerce-production
spec:
  selector:
    app: frontend
    tier: web
  ports:
  - port: 80
    targetPort: 3000
  type: ClusterIP
---
# API Gateway
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  namespace: ecommerce-production
  labels:
    app: api-gateway
    tier: api
    security-level: protected
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-gateway
      tier: api
  template:
    metadata:
      labels:
        app: api-gateway
        tier: api
        security-level: protected
    spec:
      serviceAccountName: api-gateway-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        runAsGroup: 1001
      containers:
      - name: api-gateway
        image: ecommerce-api-gateway:1.0.0
        ports:
        - containerPort: 8080
        env:
        - name: USERS_SERVICE_URL
          value: "http://users-service.ecommerce-production.svc.cluster.local:8081"
        - name: PRODUCTS_SERVICE_URL
          value: "http://products-service.ecommerce-production.svc.cluster.local:8082"
        - name: ORDERS_SERVICE_URL
          value: "http://orders-service.ecommerce-production.svc.cluster.local:8083"
        - name: PAYMENTS_SERVICE_URL
          value: "http://payments-service.ecommerce-payments.svc.cluster.local:8084"
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: api-secrets
              key: jwt-secret
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: true
---
apiVersion: v1
kind: Service
metadata:
  name: api-gateway
  namespace: ecommerce-production
spec:
  selector:
    app: api-gateway
    tier: api
  ports:
  - port: 8080
    targetPort: 8080
  type: ClusterIP
EOF

# Apply core services
kubectl apply -f ecommerce-core-services.yaml
```

### Step 3: Deploy Business Logic Microservices
```yaml
# business-microservices.yaml
# Users Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: users-service
  namespace: ecommerce-production
  labels:
    app: users-service
    tier: backend
    security-level: protected
spec:
  replicas: 2
  selector:
    matchLabels:
      app: users-service
      tier: backend
  template:
    metadata:
      labels:
        app: users-service
        tier: backend
        security-level: protected
    spec:
      serviceAccountName: users-service-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1002
        runAsGroup: 1002
      containers:
      - name: users-service
        image: ecommerce-users-service:1.0.0
        ports:
        - containerPort: 8081
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: database-secrets
              key: connection-string
        - name: REDIS_URL
          value: "redis://redis.ecommerce-production.svc.cluster.local:6379"
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: api-secrets
              key: jwt-secret
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: true
---
apiVersion: v1
kind: Service
metadata:
  name: users-service
  namespace: ecommerce-production
spec:
  selector:
    app: users-service
    tier: backend
  ports:
  - port: 8081
    targetPort: 8081
  type: ClusterIP
---
# Products Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: products-service
  namespace: ecommerce-production
  labels:
    app: products-service
    tier: backend
    security-level: protected
spec:
  replicas: 2
  selector:
    matchLabels:
      app: products-service
      tier: backend
  template:
    metadata:
      labels:
        app: products-service
        tier: backend
        security-level: protected
    spec:
      serviceAccountName: products-service-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1003
        runAsGroup: 1003
      containers:
      - name: products-service
        image: ecommerce-products-service:1.0.0
        ports:
        - containerPort: 8082
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: database-secrets
              key: connection-string
        - name: CACHE_URL
          value: "redis://redis.ecommerce-production.svc.cluster.local:6379"
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: true
---
apiVersion: v1
kind: Service
metadata:
  name: products-service
  namespace: ecommerce-production
spec:
  selector:
    app: products-service
    tier: backend
  ports:
  - port: 8082
    targetPort: 8082
  type: ClusterIP
---
# Orders Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-service
  namespace: ecommerce-production
  labels:
    app: orders-service
    tier: backend
    security-level: protected
spec:
  replicas: 2
  selector:
    matchLabels:
      app: orders-service
      tier: backend
  template:
    metadata:
      labels:
        app: orders-service
        tier: backend
        security-level: protected
    spec:
      serviceAccountName: orders-service-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1004
        runAsGroup: 1004
      containers:
      - name: orders-service
        image: ecommerce-orders-service:1.0.0
        ports:
        - containerPort: 8083
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: database-secrets
              key: connection-string
        - name: PAYMENTS_SERVICE_URL
          value: "http://payments-service.ecommerce-payments.svc.cluster.local:8084"
        - name: EMAIL_SERVICE_URL
          value: "http://email-service.ecommerce-production.svc.cluster.local:8085"
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: true
---
apiVersion: v1
kind: Service
metadata:
  name: orders-service
  namespace: ecommerce-production
spec:
  selector:
    app: orders-service
    tier: backend
  ports:
  - port: 8083
    targetPort: 8083
  type: ClusterIP
EOF

# Apply business microservices
kubectl apply -f business-microservices.yaml
```

### Step 4: Deploy Sensitive Payment Services
```yaml
# payment-services.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payments-service
  namespace: ecommerce-payments
  labels:
    app: payments-service
    tier: backend
    security-level: sensitive
spec:
  replicas: 2
  selector:
    matchLabels:
      app: payments-service
      tier: backend
  template:
    metadata:
      labels:
        app: payments-service
        tier: backend
        security-level: sensitive
    spec:
      serviceAccountName: payments-service-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1005
        runAsGroup: 1005
      containers:
      - name: payments-service
        image: ecommerce-payments-service:1.0.0
        ports:
        - containerPort: 8084
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: payment-db-secrets
              key: connection-string
        - name: PAYMENT_GATEWAY_API_KEY
          valueFrom:
            secretKeyRef:
              name: payment-gateway-secrets
              key: api-key
        - name: PAYMENT_GATEWAY_SECRET
          valueFrom:
            secretKeyRef:
              name: payment-gateway-secrets
              key: secret-key
        - name: ENCRYPTION_KEY
          valueFrom:
            secretKeyRef:
              name: payment-encryption-secrets
              key: encryption-key
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: true
          seccompProfile:
            type: RuntimeDefault
---
apiVersion: v1
kind: Service
metadata:
  name: payments-service
  namespace: ecommerce-payments
spec:
  selector:
    app: payments-service
    tier: backend
  ports:
  - port: 8084
    targetPort: 8084
  type: ClusterIP
---
# Database (PostgreSQL)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql
  namespace: ecommerce-database
  labels:
    app: postgresql
    tier: database
    security-level: critical
spec:
  serviceName: postgresql
  replicas: 1
  selector:
    matchLabels:
      app: postgresql
      tier: database
  template:
    metadata:
      labels:
        app: postgresql
        tier: database
        security-level: critical
    spec:
      serviceAccountName: postgresql-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 999
        runAsGroup: 999
        fsGroup: 999
      containers:
      - name: postgresql
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          valueFrom:
            secretKeyRef:
              name: postgres-secrets
              key: database-name
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secrets
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secrets
              key: password
        volumeMounts:
        - name: postgresql-data
          mountPath: /var/lib/postgresql/data
          subPath: postgresql
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: false  # Database needs write access
          seccompProfile:
            type: RuntimeDefault
        livenessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - 
          initialDelaySeconds: 60
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - 
          initialDelaySeconds: 5
          periodSeconds: 5
  volumeClaimTemplates:
  - metadata:
      name: postgresql-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: postgresql
  namespace: ecommerce-database
spec:
  selector:
    app: postgresql
    tier: database
  ports:
  - port: 5432
    targetPort: 5432
  type: ClusterIP
EOF

# Apply payment and database services
kubectl apply -f payment-services.yaml
```

### Step 5: Create Required Secrets
```bash
# Create all required secrets for the e-commerce platform
kubectl create secret generic api-secrets   --namespace ecommerce-production   --from-literal=jwt-secret="super-secret-jwt-key-2024"   --from-literal=api-key="internal-api-key-123456"

kubectl create secret generic database-secrets   --namespace ecommerce-production   --from-literal=connection-string="postgresql://ecommerce_user:securepassword123@postgresql.ecommerce-database:5432/ecommerce"

kubectl create secret generic payment-db-secrets   --namespace ecommerce-payments   --from-literal=connection-string="postgresql://payment_user:paymentpass456@postgresql.ecommerce-database:5432/payments"

kubectl create secret generic payment-gateway-secrets   --namespace ecommerce-payments   --from-literal=api-key="pk_live_secureapikey789"   --from-literal=secret-key="sk_live_supersecretkey012"

kubectl create secret generic payment-encryption-secrets   --namespace ecommerce-payments   --from-literal=encryption-key="encryption-key-super-secret-2024"

kubectl create secret generic postgres-secrets   --namespace ecommerce-database   --from-literal=database-name="ecommerce"   --from-literal=username="postgres"   --from-literal=password="super-secure-postgres-password-2024"

echo "✅ ALL SECRETS CREATED FOR E-COMMERCE PLATFORM"
```

## 🔍 Deployment Validation Checklist
- [ ] All namespaces created with proper security labels
- [ ] Core microservices deployed and running
- [ ] Business logic services operational
- [ ] Payment services deployed in isolated namespace
- [ ] Database statefulset running with persistence
- [ ] All secrets properly created and mounted
- [ ] Security contexts applied to all pods
- [ ] Resource limits configured
- [ ] Health checks implemented

## 📝 Learning Outcomes
✅ Deploy complete microservices e-commerce architecture  
✅ Implement secure service-to-service communication  
✅ Configure proper security contexts and resource management  
✅ Establish production-ready application foundation

## 🎯 Next Lab Preview
In Lab 7.2, we'll implement comprehensive multi-layer security controls across the entire platform.

