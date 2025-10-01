# LAB 3.1: E-Commerce RBAC Matrix
## 🎯 Implement Role-Based Access Control for E-Commerce

## 👥 Business Roles
- **Customer Service**: Read orders, update status
- **Inventory Manager**: Manage products, read sales
- **Payment Processor**: Process payments only
- **Security Auditor**: Read everything, no modifications

## 🔧 Implementation
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: ecommerce
  name: inventory-manager
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list"]
```
