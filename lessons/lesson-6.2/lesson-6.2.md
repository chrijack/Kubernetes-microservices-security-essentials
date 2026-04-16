# Lesson 6.2 — Container and Kubernetes Vulnerability Scanning with Trivy

> From the [Certified Kubernetes Security Specialist (CKS) Video Course](https://www.pearsonitcertification.com/store/certified-kubernetes-security-specialist-cks-video-9780138296476)

## Objective

Learn to scan container images and Kubernetes clusters for vulnerabilities using Trivy, a simple and comprehensive vulnerability scanner suitable for CI/CD pipelines.

## Prerequisites

- Docker installed and running
- Access to a Kubernetes cluster (for Kubernetes scanning examples)
- apt-get package manager (Debian/Ubuntu-based system)

## Steps

### Step 1: Install Trivy

Add the Aqua Security repository and install Trivy:

```bash
apt-get update
apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
apt-get update
apt-get install trivy -y
```

If you encounter a "Unable to locate package trivy" error, download and install the binary directly:

```bash
wget https://github.com/aquasecurity/trivy/releases/download/v0.54.1/trivy_0.54.1_Linux-64bit.deb
sudo dpkg -i trivy_0.54.1_Linux-64bit.deb
```

### Step 2: Scan Kubernetes Cluster

Run Trivy scans against your Kubernetes cluster using various options:

```bash
# Scan for secrets only
trivy k8s --disable-node-collector --scanners secret --report summary

# General cluster scan
trivy k8s --disable-node-collector --report summary

# Scan for critical vulnerabilities
trivy k8s --disable-node-collector --severity CRITICAL --report all

# Scan specific namespace
trivy k8s --disable-node-collector --include-namespaces kube-system --report summary

# Scan with compliance check
trivy k8s --disable-node-collector --compliance k8s-cis-1.23 --report summary
```

### Step 3: Scan Container Images for Critical Vulnerabilities

Scan the following images and check for CRITICAL issues:

```bash
docker pull nginx:1.21.4
trivy image --severity CRITICAL nginx:1.21.4
# Expected output:
# nginx:1.21.4 (debian 11.1)
# ==========================
# Total: 7 (CRITICAL: 7)

docker pull nginx:1.21.4-alpine
trivy image --severity CRITICAL nginx:1.21.4-alpine
# Expected output:
# nginx:1.21.4-alpine (alpine 3.14.3)
# ===================================
# Total: 0 (CRITICAL: 0)

docker pull nginx:1.26.2-alpine
trivy image --severity CRITICAL nginx:1.26.2-alpine
# Expected output:
# nginx:1.26.2-alpine (alpine 3.14.3)
# ===================================
# Total: 0 (CRITICAL: 0)
```

### Step 4: Scan Container Images with Custom Output Format

Scan nginx:1.21.4 for HIGH severity vulnerabilities and output results in JSON format:

```bash
docker pull nginx:1.21.4
trivy image --severity HIGH --format json --output ./nginx.json nginx:1.21.4
```

### Step 5: Generate Default Trivy Configuration

Generate a default configuration file for image scanning:

```bash
trivy image --generate-default-config
```

## Verification

- Confirm Trivy is installed: `trivy --version`
- Review scan results in the terminal output or JSON file
- Verify that Alpine-based images typically show fewer critical vulnerabilities than Debian-based images
- Check that the JSON output file contains structured vulnerability data

## Cleanup

Remove downloaded images if needed:

```bash
docker rmi nginx:1.21.4 nginx:1.21.4-alpine nginx:1.26.2-alpine amazonlinux:2.0.20211201.0
```

Remove generated output files:

```bash
rm -f ./nginx.json
```
