# Kubernetes GitOps Lab

A production-style Kubernetes GitOps laboratory built for learning modern DevOps practices.

## 🚀 Technology Stack

- Kubernetes (K3s)
- Argo CD
- Docker
- GitHub Actions
- GitHub Container Registry (GHCR)
- NGINX Ingress
- cert-manager
- Let's Encrypt
- Kustomize

## 📂 Repository Structure

```
.
├── .github/workflows/
│   └── build.yaml          # CI: builds and pushes Docker image to GHCR
├── applications/           # ArgoCD Application CRDs (managed by app-of-apps)
│   └── guestbook.yaml      # ArgoCD Application for guestbook
├── apps/                   # Application manifests (Kustomize)
│   └── guestbook/
│       ├── kustomization.yaml  # Kustomize config with image tag
│       ├── deployment.yaml     # K8s Deployment with security context
│       ├── service.yaml        # K8s Service
│       └── ingress.yaml        # Ingress with TLS
├── bootstrap/              # One-time manual apply to start GitOps loop
│   ├── project.yaml        # ArgoCD AppProject (RBAC boundaries)
│   ├── app-of-apps.yaml    # Root Application that manages applications/
│   ├── cluster-issuer.yaml # Let's Encrypt ClusterIssuers (staging + production)
│   └── argocd-ingress.yaml # ArgoCD UI ingress (optional)
└── README.md
```

## 📌 Project Roadmap

- [x] Repository structure
- [x] Custom Docker image
- [x] GitHub Container Registry (GHCR)
- [x] GitHub Actions CI
- [x] Argo CD Application manifest
- [x] Argo CD Bootstrap (App of Apps)
- [x] HTTPS with cert-manager
- [x] Production-hardened manifests (security context, resources, probes)
- [ ] Monitoring (Prometheus + Grafana)
- [ ] Multi-application deployment

## 🎯 Goal

Build a production-style GitOps platform for Kubernetes.

## 🚀 How to Deploy

### Prerequisites

- K3s cluster running
- ArgoCD installed
- ingress-nginx installed
- cert-manager installed

### Bootstrap (one-time setup)

```bash
# Clone the repo
git clone https://github.com/aungyephyo2215/k8s-gitops-lab.git
cd k8s-gitops-lab

# Apply bootstrap resources
kubectl apply -f bootstrap/project.yaml
kubectl apply -f bootstrap/cluster-issuer.yaml
kubectl apply -f bootstrap/app-of-apps.yaml

# Get ArgoCD admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### Deploy a New Version

```bash
# 1. Edit your app
nano apps/guestbook/index.html

# 2. Commit and push (triggers CI to build new image)
git add apps/guestbook/
git commit -m "feat: update app"
git push origin main

# 3. Wait for CI to finish (check GitHub Actions)

# 4. Update image tag in kustomization.yaml
# Get the commit SHA from GitHub Actions
nano apps/guestbook/kustomization.yaml
# Change: newTag: <new-sha>

# 5. Commit and push (ArgoCD deploys new image)
git add apps/guestbook/kustomization.yaml
git commit -m "deploy: update image tag to <sha>"
git push origin main
```

## 📝 Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     YOUR WORKFLOW                            │
│                                                             │
│   1. Edit files (index.html, deployment.yaml, etc.)        │
│   2. git commit + git push                                  │
│   3. GitHub Actions builds Docker image                     │
│   4. Image pushed to GHCR                                   │
│   5. Update kustomization.yaml with new tag                 │
│   6. git commit + git push                                  │
│   7. ArgoCD syncs to cluster                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 🔒 Security Features

- Non-root container (UID 101)
- Read-only root filesystem
- Dropped Linux capabilities
- Resource requests and limits
- Health probes (liveness + readiness)
- RBAC via AppProject
- TLS via Let's Encrypt

## 📚 Learn More

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Kustomize Documentation](https://kustomize.io/)
- [cert-manager Documentation](https://cert-manager.io/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
