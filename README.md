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

## 📂 Repository Structure

```
.
├── .github/workflows/
│   └── build.yaml          # CI: builds and pushes Docker image to GHCR
├── applications/           # ArgoCD Application CRDs (managed by app-of-apps)
│   └── guestbook.yaml      # ArgoCD app + Image Updater annotations
├── apps/                   # Application manifests (Kustomize)
│   └── guestbook/
│       ├── kustomization.yaml  # Image tag managed by Image Updater
│       ├── deployment.yaml
│       ├── service.yaml
│       └── ingress.yaml
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
- [x] Argo CD Image Updater
- [x] HTTPS with cert-manager
- [ ] Monitoring
- [ ] Multi-application deployment

## 🎯 Goal

Build a production-style GitOps platform for Kubernetes.
