# GitOps Lab - Working Process

## Overview

This document explains how the GitOps Lab project works, step by step.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        DEVELOPER                                 │
│                                                                  │
│   1. Edit source code (index.html, nginx.conf)                 │
│   2. git commit + git push                                      │
│                                                                  │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                        GITHUB                                    │
│                                                                  │
│   Two things happen automatically:                              │
│                                                                  │
│   ┌─────────────────────┐    ┌─────────────────────────────┐   │
│   │   GitHub Actions    │    │         ArgoCD               │   │
│   │                     │    │                              │   │
│   │  - Detects change   │    │  - Detects Git change        │   │
│   │    in apps/guestbook│    │  - Reads manifests            │   │
│   │  - Builds Docker    │    │  - Syncs to cluster          │   │
│   │    image            │    │                              │   │
│   │  - Pushes to GHCR   │    │                              │   │
│   └─────────────────────┘    └─────────────────────────────┘   │
│                                                                  │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      K3S CLUSTER                                 │
│                                                                  │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│   │ ArgoCD   │  │ guestbook│  │ cert-    │  │ ingress- │       │
│   │ (GitOps) │  │ (your    │  │ manager  │  │ nginx    │       │
│   │          │  │  app)    │  │ (HTTPS)  │  │ (router) │       │
│   └──────────┘  └──────────┘  └──────────┘  └──────────┘       │
│                                                                  │
│   Your app is accessible at: https://guestbook.k8s.cmtmm.online │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Step-by-Step Process

### Step 1: Repository Structure

```
k8s-gitops-lab/
├── .github/workflows/
│   └── build.yaml              # CI pipeline
├── applications/
│   └── guestbook.yaml          # ArgoCD Application
├── apps/
│   └── guestbook/
│       ├── kustomization.yaml  # Kustomize config
│       ├── deployment.yaml     # K8s Deployment
│       ├── service.yaml        # K8s Service
│       ├── ingress.yaml        # K8s Ingress
│       ├── Dockerfile          # Container image
│       ├── nginx.conf          # Nginx config
│       └── index.html          # App content
├── bootstrap/
│   ├── project.yaml            # ArgoCD AppProject
│   ├── app-of-apps.yaml        # Root Application
│   ├── cluster-issuer.yaml     # Let's Encrypt
│   └── argocd-ingress.yaml     # ArgoCD UI
└── README.md
```

### Step 2: Bootstrap (One-Time Setup)

When setting up for the first time:

```bash
# Clone the repository
git clone https://github.com/aungyephyo2215/k8s-gitops-lab.git
cd k8s-gitops-lab

# Apply bootstrap resources (in order)
kubectl apply -f bootstrap/project.yaml          # Create ArgoCD project
kubectl apply -f bootstrap/cluster-issuer.yaml   # Create Let's Encrypt issuers
kubectl apply -f bootstrap/app-of-apps.yaml      # Start GitOps loop
```

**What happens:**
1. `project.yaml` → Creates ArgoCD AppProject with RBAC rules
2. `cluster-issuer.yaml` → Tells cert-manager how to get HTTPS certificates
3. `app-of-apps.yaml` → Tells ArgoCD to watch the `applications/` directory

**After this, everything is Git-managed. You never run `kubectl apply` again.**

### Step 3: How ArgoCD Works

```
┌─────────────────────────────────────────────────────────────┐
│                     ArgoCD Process                           │
│                                                              │
│   1. app-of-apps watches applications/ directory            │
│   2. Finds guestbook.yaml                                   │
│   3. Creates guestbook Application                          │
│   4. Reads apps/guestbook/ directory                        │
│   5. Runs kustomize build                                   │
│   6. Applies Deployment, Service, Ingress                   │
│   7. Watches for changes (auto-sync)                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Step 4: CI Pipeline (GitHub Actions)

When you push changes to `apps/guestbook/`:

```yaml
# .github/workflows/build.yaml
name: Build and Push Guestbook Image

on:
  push:
    branches: [main]
    paths:
      - apps/guestbook/**

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - Checkout code
      - Login to GHCR
      - Build Docker image
      - Push to GHCR with SHA tag
```

**What happens:**
1. GitHub detects change in `apps/guestbook/`
2. Builds Docker image from `Dockerfile`
3. Pushes to GHCR with commit SHA as tag (e.g., `abc1234`)

### Step 5: Deploying Updates

```bash
# 1. Edit your app
nano apps/guestbook/index.html

# 2. Commit and push (triggers CI)
git add apps/guestbook/
git commit -m "feat: update app content"
git push origin main

# 3. Wait for CI to finish (2-3 minutes)
# Check: https://github.com/aungyephyo2215/k8s-gitops-lab/actions

# 4. Update image tag in kustomization.yaml
# Get the SHA from GitHub Actions output
nano apps/guestbook/kustomization.yaml
# Change: newTag: <new-sha>

# 5. Commit and push (ArgoCD deploys)
git add apps/guestbook/kustomization.yaml
git commit -m "deploy: update image to <sha>"
git push origin main

# 6. ArgoCD detects change and syncs to cluster
# Pod restarts with new image
```

### Step 6: How HTTPS Works

```
User visits: https://guestbook.k8s.cmtmm.online
        │
        ▼
DNS resolves to cluster IP
        │
        ▼
┌─── ingress-nginx ────────────────────────────────┐
│                                                    │
│   Listens on port 443 (HTTPS)                     │
│   Has TLS certificate from Let's Encrypt          │
│   Reads Ingress rules                             │
│                                                    │
│   "guestbook.k8s.cmtmm.online?"                   │
│   "Yes, route to guestbook-ui service"            │
│                                                    │
└───────────────────┬────────────────────────────────┘
                    │
                    ▼
┌─── Service: guestbook-ui ────────────────────────┐
│                                                    │
│   Stable address inside the cluster               │
│   Routes to pods with label: app=guestbook        │
│                                                    │
└───────────────────┬────────────────────────────────┘
                    │
                    ▼
┌─── Pod: guestbook-ui ────────────────────────────┐
│                                                    │
│   Container: nginx serving index.html             │
│   Port 80                                         │
│                                                    │
└────────────────────────────────────────────────────┘
```

**Certificate renewal:**
- cert-manager automatically renews certificates before expiry
- No manual intervention needed

### Step 7: Security Features

| Feature | Description |
|---|---|
| Non-root container | Runs as UID 101 (nginx user) |
| Read-only filesystem | Cannot write to disk (except /tmp, /var/cache/nginx) |
| Dropped capabilities | All Linux capabilities removed |
| Resource limits | CPU: 50m-100m, Memory: 64Mi-128Mi |
| Health probes | Liveness + readiness checks |
| RBAC | AppProject restricts what ArgoCD can do |
| TLS | HTTPS via Let's Encrypt |

### Step 8: Self-Healing

ArgoCD continuously monitors the cluster:

```
Manual change in cluster (kubectl edit)
        │
        ▼
ArgoCD detects drift
        │
        ▼
ArgoCD reverts to match Git
        │
        ▼
Cluster matches Git again
```

**This is the #1 benefit of GitOps: the cluster always matches what's in Git.**

---

## Key Concepts

### What is GitOps?

GitOps = **Git is the source of truth for your infrastructure**

- Every change is a Git commit
- Git has full history of every change
- Rollback = `git revert`
- Collaboration = Pull Requests
- Audit trail = Git log

### What is ArgoCD?

ArgoCD is a tool that watches your Git repository and applies changes to your cluster automatically.

- Watches Git for changes
- Runs `kustomize build` on your manifests
- Applies the result to the cluster
- Monitors for drift (self-healing)

### What is Kustomize?

Kustomize lets you customize manifests without copying them.

```yaml
# kustomization.yaml
namespace: app-01           # Apply namespace to all resources
resources:
  - deployment.yaml
  - service.yaml
  - ingress.yaml
images:
  - name: ghcr.io/aungyephyo2215/guestbook
    newTag: abc1234         # Override image tag
```

### What is cert-manager?

cert-manager automatically gets and renews TLS certificates.

```yaml
# Ingress annotation
cert-manager.io/cluster-issuer: letsencrypt-production
```

**What happens:**
1. cert-manager sees the annotation
2. Contacts Let's Encrypt
3. Verifies domain ownership
4. Issues certificate
5. Stores in Kubernetes Secret
6. Auto-renews before expiry

---

## Troubleshooting

### Check ArgoCD status

```bash
kubectl get applications -n argocd
```

### Check pod status

```bash
kubectl get pods -n app-01
```

### Check ingress

```bash
kubectl get ingress -n app-01
```

### Check certificate

```bash
kubectl get certificate -n app-01
```

### Force ArgoCD sync

```bash
kubectl annotate application guestbook -n argocd argocd.argoproj.io/refresh=hard
```

### Check pod logs

```bash
kubectl logs -n app-01 -l app=guestbook-ui
```

---

## Summary

| Component | Purpose |
|---|---|
| Git | Source of truth for everything |
| GitHub Actions | Builds Docker images |
| GHCR | Stores container images |
| ArgoCD | Deploys to cluster |
| Kustomize | Customizes manifests |
| cert-manager | Manages TLS certificates |
| ingress-nginx | Routes external traffic |

**The key takeaway: You only edit files in Git. Everything else is automatic.**
