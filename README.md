# SSL Certificate Expiry Checker

A Python-based tool for automated monitoring of SSL/TLS certificate expiration dates across multiple domains. Designed for deployment in Kubernetes environments with support for local development and Docker containerization.

[![Python 3.9+](https://img.shields.io/badge/python-3.9+-blue.svg)](https://www.python.org/downloads/)
[![Docker](https://img.shields.io/badge/docker-ready-brightgreen.svg)](https://www.docker.com/)
[![Kubernetes](https://img.shields.io/badge/kubernetes-compatible-326CE5.svg)](https://kubernetes.io/)

---

##  Table of Contents

- [Overview](#overview)
- [Features](#features)
- [How It Works](#how-it-works)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Quick Start](#quick-start)
  - [Local Execution](#local-execution)
  - [Docker Deployment](#docker-deployment)
  - [Kubernetes/Minikube Deployment](#kubernetesminikube-deployment)
- [Configuration](#configuration)
- [Monitoring & Alerts](#monitoring--alerts)
- [Troubleshooting](#troubleshooting)
- [Best Practices](#best-practices)

---

##  Overview

This project provides an automated SSL/TLS certificate monitoring solution that can be deployed in multiple environments:

- **Locally**: Run as a standalone Python script
- **Docker**: Containerized execution for consistency across platforms
- **Kubernetes**: Scheduled CronJob or continuous Deployment for production monitoring

The tool retrieves SSL certificates from configured domains, analyzes expiration dates, and provides clear status reporting with optional alerting capabilities.

---

##  Features

### Core Functionality
-  **Multi-domain checking**: Monitor hundreds of sites concurrently
-  **High performance**: Threaded execution with configurable worker pools (default: 16 workers)
-  **Clear reporting**: Human-readable summary table with color-coded status
-  **Flexible alerting**: Slack webhook integration (extensible to Teams/email)
-  **Robust error handling**: Gracefully handles network errors, expired certificates, and self-signed certificates
-  **Cross-version compatibility**: Supports Python 3.9–3.13 across platforms

### Certificate Analysis
-  **Status classification**: OK, WARNING, CRITICAL, EXPIRED
-  **Configurable thresholds**: Custom warning (default: 30 days) and critical (default: 7 days) periods
-  **Comprehensive parsing**: Dual-mode certificate parsing using both `ssl` module and `cryptography` library for reliability
-  **Flexible port support**: Custom port specification (default: 443)

### Deployment Options
-  **Kubernetes-native**: ConfigMap integration for dynamic site lists
-  **Scheduled execution**: CronJob for periodic checks (e.g., daily at 06:00)
-  **Always-on monitoring**: Deployment mode for continuous availability
-  **Docker-optimized**: Lightweight Alpine-based image

---

## How It Works

### Workflow

```
┌─────────────────┐
│  Load Site List │ ← sites.txt or ConfigMap
└────────┬────────┘
         │
         ▼
┌─────────────────────────────┐
│  Parallel TLS Connections   │
│  (ThreadPoolExecutor)        │
└────────┬────────────────────┘
         │
         ▼
┌─────────────────────────────┐
│  Certificate Retrieval      │
│  • Connect via TLS          │
│  • Parse notAfter field     │
│  • Extract subject/issuer   │
└────────┬────────────────────┘
         │
         ▼
┌─────────────────────────────┐
│  Status Classification      │
│  • Calculate days remaining │
│  • Apply thresholds         │
│  • Assign status            │
└────────┬────────────────────┘
         │
         ▼
┌─────────────────────────────┐
│  Output & Alerting          │
│  • Print summary table      │
│  • Send Slack notifications │
│  • Set exit code            │
└─────────────────────────────┘
```

### Certificate Parsing Strategy

The tool employs a **dual-mode parsing approach** for maximum compatibility:

1. **Primary Method**: Standard `ssl.getpeercert()` parsing
   - Fast and lightweight
   - Works on most systems

2. **Fallback Method**: `cryptography` library DER decoding
   - Handles edge cases and malformed certificates
   - Provides `not_valid_after_utc` for timezone-aware expiry

### Status Thresholds

| Status      | Condition                  | Exit Code |
|-------------|----------------------------|-----------|
| ✅ **OK**       | Days remaining > WARN_DAYS | 0         |
| ⚠️ **WARNING**  | Days remaining ≤ WARN_DAYS | 1         |
| 🚨 **CRITICAL** | Days remaining ≤ CRIT_DAYS | 2         |
| ❌ **EXPIRED**  | Days remaining < 0         | 2         |
| ❗ **ERROR**    | Connection/SSL failure     | 2         |

---

## Prerequisites

### For Local Execution
- **Python 3.9+** (tested on 3.9–3.13)
- **Python packages**:
  ```bash
  pip install cryptography
  ```

### For Docker Deployment
- **Docker** 20.10+ or compatible runtime
- Basic familiarity with Docker CLI

### For Kubernetes Deployment
- **kubectl** CLI tool
- **Minikube** (for local testing) or access to a Kubernetes cluster
- Basic understanding of Kubernetes resources (ConfigMaps, CronJobs, Pods)

---

## Project Structure

```
bss-platform-infrastructure/
│
├── Scripts/
│   ├── check_ssl.py          # Main Python script
│   └── sites.txt              # Default site list for local runs
│
├── Infrastructure/
│   ├── checker.Dockerfile     # Multi-stage Docker build
│   └── k8s/
│       ├── configmap-sites.yaml       # Kubernetes ConfigMap for site list
│       ├── cronjob-ssl-check.yaml     # CronJob manifest (scheduled)
│       └── deployment.yaml            # Deployment manifest (always-on)
│
└── README.md                  # This file
```

### Key Files

- **`check_ssl.py`**: Core SSL checking logic with concurrent execution and alerting
- **`sites.txt`**: Plain text file with one domain per line (supports comments with `#`)
- **`checker.Dockerfile`**: Optimized container image definition
- **`configmap-sites.yaml`**: Kubernetes-native configuration for domain list
- **`cronjob-ssl-check.yaml`**: Automated scheduling configuration

---

## Quick Start

### Local Execution

#### Basic Run

```bash
# Navigate to Scripts directory
cd Scripts/

# Install dependencies
pip install cryptography

# Run the checker
python3 check_ssl.py
```

**Output Example:**
```
----- SSL Certificate Expiry Summary -----
google.com | CN=*.google.com | Expires=2025-03-15 12:00:00 UTC | DaysLeft=127 | OK
expired.badssl.com | CN=*.badssl.com | Expires=2023-04-12 23:59:59 UTC | DaysLeft=-589 | EXPIRED
openai.com | CN=openai.com | Expires=2025-02-10 12:00:00 UTC | DaysLeft=94 | OK
------------------------------------------
```

#### Custom Configuration

```bash
# Override thresholds and site file
SITES_FILE=custom_sites.txt \
WARN_DAYS=14 \
CRIT_DAYS=3 \
PARALLEL_WORKERS=8 \
python3 check_ssl.py
```

#### With Slack Alerting

```bash
# Set Slack webhook URL
export SLACK_WEBHOOK="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
python3 check_ssl.py
```

---

### Docker Deployment

#### 1. Build the Image

From the project root directory:

```bash
docker build -f Infrastructure/checker.Dockerfile -t ssl-checker:latest .
```

**Verify build:**
```bash
docker images | grep ssl-checker
```

#### 2. Run the Container

**Basic execution:**
```bash
docker run --rm ssl-checker:latest
```

**With custom configuration:**
```bash
docker run --rm \
  -v $(pwd)/Scripts/sites.txt:/config/sites.txt \
  -e SITES_FILE=/config/sites.txt \
  -e WARN_DAYS=15 \
  -e CRIT_DAYS=5 \
  -e SLACK_WEBHOOK="https://hooks.slack.com/services/..." \
  ssl-checker:latest
```

**Interactive debugging:**
```bash
docker run -it --rm ssl-checker:latest bash
```

---

### Kubernetes/Minikube Deployment

#### Step 1: Start Minikube

```bash
minikube start
```

#### Step 2: Build Image in Minikube Context

```bash
# Point Docker CLI to Minikube's Docker daemon
eval $(minikube -p minikube docker-env)

# Build image inside Minikube
docker build -f Infrastructure/checker.Dockerfile -t ssl-checker:latest .

# Verify image is available in Minikube
minikube image ls | grep ssl-checker
```

**Alternative**: Load pre-built image
```bash
minikube image load ssl-checker:latest
```

#### Step 3: Create ConfigMap

**Option A: From file**
```bash
kubectl create configmap ssl-check-sites \
  --from-file=sites.txt=./Scripts/sites.txt \
  --dry-run=client -o yaml | kubectl apply -f -
```

**Option B: From manifest**
```bash
kubectl apply -f Infrastructure/k8s/configmap-sites.yaml
```

**Verify ConfigMap:**
```bash
kubectl get configmap ssl-check-sites -o yaml
```

#### Step 4: Deploy CronJob (Recommended for Production)

```bash
kubectl apply -f Infrastructure/k8s/cronjob-ssl-check.yaml
```

**Schedule**: Daily at 06:00 UTC (configurable in `spec.schedule`)

**Monitor CronJob:**
```bash
# View CronJob configuration
kubectl get cronjob ssl-expiry-check

# Watch for job executions
kubectl get jobs --watch

# Check job history
kubectl get jobs --sort-by=.metadata.creationTimestamp
```

#### Step 5: View Results

```bash
# Get latest job name
kubectl get jobs (-n namespace-name  --if deploting in a namespace)

# View logs
kubectl logs job job-name
```

---

### Alternative: Deployment Mode (Always-On)

For continuous monitoring dashboards or API endpoints:

```bash
kubectl apply -f Infrastructure/k8s/deployment.yaml
```

**Access deployment:**
```bash
# Get pod name
kubectl get pods (-n namespace-name  --if deploting in a namespace)

# View logs
kubectl logs POD_NAME 

# Execute commands inside pod
kubectl exec -it POD_NAME -- python3 /app/check_ssl.py
```

---

## Configuration

### Environment Variables

All configuration is driven by environment variables for maximum flexibility:

| Variable           | Description                          | Default         | Example                          |
|--------------------|--------------------------------------|-----------------|----------------------------------|
| `SITES_FILE`       | Path to site list file               | `sites.txt`     | `/config/sites.txt`              |
| `WARN_DAYS`        | Days threshold for warning           | `30`            | `14`                             |
| `CRIT_DAYS`        | Days threshold for critical          | `7`             | `3`                              |
| `SLACK_WEBHOOK`    | Slack webhook URL for alerts         | *(empty)*       | `https://hooks.slack.com/...`    |
| `PARALLEL_WORKERS` | Number of concurrent check threads   | `16`            | `8` (lower for resource-constrained envs) |
| `CONNECT_TIMEOUT`  | TCP connection timeout (seconds)     | `6.0`           | `10.0`                           |

### Sites File Format

The `sites.txt` file supports:
- ✅ One domain per line
- ✅ Optional port specification (e.g., `example.com:8443`)
- ✅ Comments starting with `#`
- ✅ Blank lines (ignored)

**Example:**
```text
# Production sites
google.com
openai.com
example.org:443

# Development environments
dev.example.com:8443

# Third-party APIs
api.github.com
letsencrypt.org

# Test cases
expired.badssl.com
self-signed.badssl.com
wrong.host.badssl.com
```

### Kubernetes ConfigMap Configuration

Edit the ConfigMap to update the site list dynamically:

```bash
kubectl edit configmap ssl-check-sites
```

After editing, **apply changes**:
```bash
kubectl apply -f Infrastructure/k8s/configmap-sites.yaml
```

**Note**: Running jobs won't pick up changes automatically. For CronJobs, wait for the next scheduled run, or manually trigger a new job (see [Troubleshooting](#troubleshooting)).

---

## Monitoring & Alerts

### Slack Integration

#### Setup

1. Create a Slack App with Incoming Webhooks enabled
2. Copy the webhook URL (e.g., `https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXX`)
3. Set the environment variable:

**Local/Docker:**
```bash
export SLACK_WEBHOOK="https://hooks.slack.com/services/..."
```

**Kubernetes:**
```yaml
# In cronjob-ssl-check.yaml or deployment.yaml
env:
  - name: SLACK_WEBHOOK
    value: "https://hooks.slack.com/services/..."
```

**Secure approach (recommended for production):**
```bash
# Create secret
kubectl create secret generic slack-webhook \
  --from-literal=url="https://hooks.slack.com/services/..."

# Reference in manifest
env:
  - name: SLACK_WEBHOOK
    valueFrom:
      secretKeyRef:
        name: slack-webhook
        key: url
```

#### Alert Format

Alerts are sent only when certificates meet WARNING, CRITICAL, or EXPIRED thresholds:

```
*CRITICAL*:
• expired.badssl.com - EXPIRED 2023-04-12 (-589 days)
• urgent.example.com - CRITICAL 2025-11-10 (3 days)

*Warnings*:
• soon-expiring.example.com - WARNING 2025-12-05 (28 days)
```

---

## Troubleshooting

### Common Issues

#### 1. **"Sites file not found" Error**

**Symptom:**
```
[ERROR] Sites file not found: sites.txt
```

**Solutions:**
- Verify file path: `ls -la Scripts/sites.txt`
- Check `SITES_FILE` environment variable: `echo $SITES_FILE`
- Use absolute path: `SITES_FILE=/full/path/to/sites.txt python3 check_ssl.py`

---

#### 2. **Docker Image Not Found in Minikube**

**Symptom:**
```
Failed to pull image "ssl-checker:latest": rpc error: code = Unknown desc = Error response from daemon: pull access denied
```

**Solutions:**
```bash
# Option 1: Rebuild in Minikube context
eval $(minikube docker-env)
docker build -f Infrastructure/checker.Dockerfile -t ssl-checker:latest .

# Option 2: Load pre-built image
docker save ssl-checker:latest | (eval $(minikube docker-env) && docker load)

# Option 3: Use imagePullPolicy
# In YAML manifests, set:
imagePullPolicy: Never  # or IfNotPresent
```

---

#### 3. **ConfigMap Changes Not Reflected**

**Symptom:** Updated site list isn't used by running pods

**Solutions:**
```bash
# Apply ConfigMap changes
kubectl apply -f Infrastructure/k8s/configmap-sites.yaml

# For CronJobs: Manually trigger new job
kubectl create job --from=cronjob/ssl-expiry-check ssl-expiry-manual-$(date +%s)

# For Deployments: Restart pods
kubectl rollout restart deployment ssl-checker-deployment

# Verify ConfigMap content
kubectl get configmap ssl-check-sites -o jsonpath='{.data.sites\.txt}'
```

---

#### 4. **SSL Connection Timeouts**

**Symptom:**
```
example.com - ERROR - Connection timeout
```

**Solutions:**
```bash
# Increase timeout
CONNECT_TIMEOUT=10.0 python3 check_ssl.py

# Reduce parallel workers (lower network load)
PARALLEL_WORKERS=4 python3 check_ssl.py

# Check network connectivity
curl -I https://example.com
openssl s_client -connect example.com:443 -servername example.com
```

---

#### 5. **Certificate Parsing Errors (Python < 3.10)**

**Symptom:**
```
ValueError: Could not retrieve certificate in binary form
```

**Solutions:**
```bash
# Ensure cryptography library is installed
pip install --upgrade cryptography

# Verify Python version
python3 --version  # Should be 3.9+

# Test specific domain
python3 -c "from check_ssl import get_cert_expiry; print(get_cert_expiry('google.com'))"
```

---

### Debugging Commands

#### Kubernetes Debugging

```bash
# View pod logs
kubectl logs <pod-name>

# Stream logs in real-time
kubectl logs -f <pod-name>

# Check pod status
kubectl describe pod <pod-name>

# View events (useful for image pull errors)
kubectl get events --sort-by=.metadata.creationTimestamp

# Execute command inside pod
kubectl exec -it <pod-name> -- cat /config/sites.txt
kubectl exec -it <pod-name> -- python3 /app/check_ssl.py

# Check CronJob schedule
kubectl get cronjob ssl-expiry-check -o yaml | grep schedule

# Delete failed jobs
kubectl delete job --field-selector status.successful=0
```

#### Docker Debugging

```bash
# Run container interactively
docker run -it --rm ssl-checker:latest bash

# Check container logs
docker logs <container-id>

# Inspect container filesystem
docker run --rm ssl-checker:latest ls -la /app

# Verify environment variables
docker run --rm ssl-checker:latest env
```

---

### Manual Job Triggering (Kubernetes)

To test immediately without waiting for the CronJob schedule:

```bash
# Create one-off job from CronJob
kubectl create job --from=cronjob/ssl-expiry-check ssl-expiry-manual-test

# Monitor job execution
kubectl get jobs --watch

# View logs
kubectl logs job/ssl-expiry-manual-test

# Clean up
kubectl delete job ssl-expiry-manual-test
```

---

## Best Practices

### Security

1. **Secrets Management**: Never commit webhook URLs or credentials
   ```bash
   # Use Kubernetes secrets
   kubectl create secret generic alerting-secrets \
     --from-literal=slack-webhook="https://hooks.slack.com/..."
   ```

2. **RBAC**: Apply least-privilege principles for service accounts
   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: ssl-checker-sa
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: Role
   metadata:
     name: ssl-checker-role
   rules:
   - apiGroups: [""]
     resources: ["configmaps"]
     verbs: ["get", "list"]
   ```

3. **Network Policies**: Restrict egress to only required domains

### Performance

1. **Resource Limits**: Define CPU/memory constraints in Kubernetes
   ```yaml
   resources:
     requests:
       memory: "128Mi"
       cpu: "100m"
     limits:
       memory: "256Mi"
       cpu: "500m"
   ```

2. **Tune Worker Count**: Balance between speed and resource usage
   - **Small lists (<20 sites)**: `PARALLEL_WORKERS=4`
   - **Medium lists (20-100 sites)**: `PARALLEL_WORKERS=16`
   - **Large lists (>100 sites)**: `PARALLEL_WORKERS=32`

3. **Optimize Timeouts**: Adjust based on network latency
   - **Fast networks**: `CONNECT_TIMEOUT=5.0`
   - **Slow/international networks**: `CONNECT_TIMEOUT=10.0`

### Operational

1. **Job History**: Configure retention in CronJob
   ```yaml
   spec:
     successfulJobsHistoryLimit: 3
     failedJobsHistoryLimit: 3
   ```

2. **Monitoring**: Set up alerts for job failures
   ```bash
   # Example: Check for failed jobs
   kubectl get jobs --field-selector status.successful=0
   ```

3. **Log Aggregation**: Integrate with centralized logging (ELK, Splunk, etc.)

4. **Regular Updates**: Keep dependencies current
   ```bash
   pip install --upgrade cryptography
   docker build --no-cache -f Infrastructure/checker.Dockerfile -t ssl-checker:latest .
   ```

---

##  Development Workflow

### Making Changes

1. **Edit Script**: Modify `Scripts/check_ssl.py`
2. **Test Locally**: `python3 Scripts/check_ssl.py`
3. **Rebuild Docker**: `docker build -f Infrastructure/checker.Dockerfile -t ssl-checker:latest .`
4. **Test in Minikube**:
   ```bash
   eval $(minikube docker-env)
   docker build -f Infrastructure/checker.Dockerfile -t ssl-checker:latest .
   kubectl create job --from=cronjob/ssl-expiry-check ssl-test-$(date +%s)
   ```

### Testing with Edge Cases

Use the pre-configured test domains in `Scripts/sites.txt`:

| Domain                        | Purpose                           |
|-------------------------------|-----------------------------------|
| `expired.badssl.com`          | Test EXPIRED status               |
| `self-signed.badssl.com`      | Test error handling               |
| `wrong.host.badssl.com`       | Test hostname mismatch            |
| `incomplete-chain.badssl.com` | Test chain validation issues      |

---

##  Acknowledgments

- **BadSSL.com**: Provides excellent test certificates for validation
- **Python cryptography**: Robust X.509 certificate parsing
- **Kubernetes community**: Documentation and best practices

---
