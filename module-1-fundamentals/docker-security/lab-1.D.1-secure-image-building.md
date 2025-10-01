# LAB 1.D.1: Secure Docker Image Building
## Building Fortress-Strong Containers

## 🎯 Lab Objectives
1. Create minimal, secure Docker images
2. Implement security best practices
3. Scan images for vulnerabilities
4. Understand multi-stage builds

## 🏰 Analogy: Castle Construction
- **Multi-stage builds** = Separate workshops for different materials
- **Minimal base images** = Lightweight but strong foundation stones
- **Non-root users** = Workers with limited privileges
- **Image layers** = Organized construction phases

## 🛠️ Hands-On Implementation

### Step 1: Create a Vulnerable Dockerfile (What NOT to do)
```dockerfile
# INSECURE - Educational example only!
FROM ubuntu:latest
RUN apt-get update && apt-get install -y python3
COPY . /app
WORKDIR /app
CMD ["python3", "app.py"]
```

### Step 2: Create a Secure Dockerfile
```dockerfile
# SECURE - Production ready
FROM python:3.9-slim as builder

# Install build dependencies
RUN apt-get update && apt-get install -y     build-essential     && rm -rf /var/lib/apt/lists/*

# Copy requirements and install
COPY requirements.txt .
RUN pip install --user -r requirements.txt

# Final stage
FROM python:3.9-slim

# Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Copy from builder stage
COPY --from=builder /root/.local /home/appuser/.local
COPY --chown=appuser:appuser . /app

WORKDIR /app
USER appuser

# Security settings
ENV PATH=/home/appuser/.local/bin:/home/cloudlab/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/snap/bin
ENV PYTHONUNBUFFERED=1

CMD ["python", "app.py"]
```

### Step 3: Build and Scan Images
```bash
# Build both images
docker build -t insecure-app -f Dockerfile.insecure .
docker build -t secure-app -f Dockerfile.secure .

# Scan for vulnerabilities (install trivy first)
trivy image insecure-app
trivy image secure-app

# Compare results
echo "=== VULNERABILITY COMPARISON ==="
echo "Insecure image vulnerabilities:"
trivy image insecure-app --format table | head -10
echo ""
echo "Secure image vulnerabilities:"  
trivy image secure-app --format table | head -10
```

## 🔍 Security Validation Checklist
- [ ] Uses minimal base image (slim/alpine)
- [ ] Runs as non-root user
- [ ] Multi-stage build reduces attack surface
- [ ] No sensitive data in image layers
- [ ] Regular dependency updates
- [ ] Image scanning in CI/CD pipeline

## 📝 Learning Outcomes
✅ Understand Docker security principles  
✅ Create production-ready secure images  
✅ Implement vulnerability scanning  
✅ Apply security best practices

## 🎯 Next Lab Preview
In Lab 1.D.2, we'll implement container runtime security and security profiles.

