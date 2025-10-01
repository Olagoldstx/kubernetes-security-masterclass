# LAB 2.1: Network Segmentation
Isolate e-commerce services with network policies

## 🎯 E-Commerce Architecture
Frontend → API Gateway → Microservices → Database

## 🔧 Implementation Steps
```bash
# Create namespaces
kubectl create ns ecommerce-frontend
kubectl create ns ecommerce-backend
kubectl create ns ecommerce-payments

# Apply network policies
kubectl apply -f network-policies/
```
