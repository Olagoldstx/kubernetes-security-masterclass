# LAB 1.2: Pod Security Contexts - Castle Guard Protocols

## 🎯 Lab Objectives
1. Understand Pod Security Standards
2. Implement security contexts for pods
3. Configure non-root execution

## 🏰 Analogy: Castle Guard Protocols
Just like castle guards have strict protocols, pods need security contexts:
- **Non-root execution** = Guards can't act as kings
- **Read-only filesystems** = Guards can't modify castle plans
- **Capability dropping** = Guards only have necessary tools

## 🔧 Implementation
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
  containers:
  - name: app
    image: nginx:alpine
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
```
