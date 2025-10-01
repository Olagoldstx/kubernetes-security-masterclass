# LAB 1.1: Secure Cluster Setup
## 🎯 Objectives: Deploy hardened Kubernetes cluster

### Steps:
1. Create Kind cluster
2. Configure security settings
3. Verify security posture

## 🛡️ Security Principles
- Least Privilege
- Defense in Depth
- Zero Trust

## 🔧 Hands-On Steps
```bash
# Create secure cluster
kind create cluster --name security-lab --config kind-config.yaml

# Verify security
kubectl get nodes -o wide
kubectl cluster-info
```
