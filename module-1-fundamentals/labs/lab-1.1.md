# LAB 1.1: Secure Cluster Setup

## Lab 1.1 — Create and Verify a Local Kubernetes Cluster (Kind)

### Objectives

- Create a local **Kind** cluster using Docker as the container runtime.
- Verify that the control plane and nodes are healthy with `kubectl`.
- Cleanly remove the cluster after verification.

---

### Prerequisites

- Docker Desktop installed and **running**.
- `kind` and `kubectl` installed and available on your PATH.
- Terminal: Linux/macOS shell or **Windows Git Bash**.

> **Quick version checks**

```bash
docker --version
kind --version
kubectl version --client
```

---

### 1) Create the cluster

Create a single-node control plane cluster with a clear name so teardown is unambiguous:

```bash
kind create cluster --name security-lab
```

Need custom ports or networking? Use a config file:

```bash
kind create cluster --name security-lab --config kind-config.yaml
```

**Why these flags**

- `create cluster` — tells Kind to provision a Kubernetes-in-Docker cluster.
- `--name security-lab` — lets you target this cluster for later commands.
- `--config` — optional YAML file for advanced settings (ports, pod/service CIDRs, node labels, extra mounts).

---

### 2) Verify cluster health

Confirm the control plane endpoint and node readiness:

```bash
kubectl cluster-info
kubectl get nodes -o wide
```

- `cluster-info` prints API server and CoreDNS endpoints so you know the control plane is reachable.
- `get nodes -o wide` shows readiness, internal IP, OS image, and container runtime.

You can also list system pods:

```bash
kubectl get pods -A
```

---

### 3) (Optional) Smoke test a Pod

Run a quick workload test to confirm scheduling works correctly:

```bash
kubectl run test-pod --image=nginx --restart=Never
kubectl get pods -o wide
```

---

### 4) Cleanup

Remove the test pod (if present) and delete the cluster cleanly:

```bash
kubectl delete pod test-pod --ignore-not-found
kind delete cluster --name security-lab
```

---

### Troubleshooting (Windows Git Bash)

- **Docker daemon not running** → Start Docker Desktop; wait until it says **Running**.
- **kubectl connection refused** → Cluster may still be initializing; wait 20–60 seconds and retry.
- **PATH issues** → Confirm executables with `which kind` and `which kubectl`.
- **WSL2 quirks** → Prefer Git Bash or PowerShell unless Docker/WSL integration is configured.

---
