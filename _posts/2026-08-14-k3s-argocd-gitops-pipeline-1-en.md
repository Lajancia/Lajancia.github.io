---
title: "Building a GitOps CI/CD Pipeline with K3s, ArgoCD & Jenkins (1) — Overview & Motivation"
date: 2026-08-14 12:00:00 +0900
categories: [Study]
tags: [K3s, ArgoCD, Jenkins, GitOps, CI/CD, Kubernetes, DevOps]
author: L.J
lang: en
---

### **Introduction**

Until recently, my deployment environment was simple. I had a VPS server with Docker installed, and Jenkins would receive a GitHub webhook, build a Docker image, and run the container locally.

```
GitHub Push → Jenkins Build → Docker build → docker run (local)
```

This approach works, but it has some shortcomings:

- If a container crashes, **I have to restart it manually**
- **Rollbacks** are tedious
- Managing many services becomes painful
- There's **no dashboard** to see the deployment state at a glance

While thinking about these issues, I started studying Kubernetes. So I decided to build a **lightweight K3s cluster** and set up a **GitOps-style CI/CD pipeline** from scratch.

---

### **Why K3s?**

The full Kubernetes distribution is heavy and complex. Especially on a **single-node VPS** like mine, the overhead can be a burden. K3s, on the other hand:

- **Lightweight** — runs as a single binary (~60MB)
- **Low-spec friendly** — works with 1GB RAM and 1 CPU
- **kubeconfig compatible** — standard `kubectl` commands just work
- **Simple installation** — one command to bootstrap the cluster

```
curl -sfL https://get.k3s.io | sh -
```

After deploying K3s on my server, the overall system resource usage barely increased.

---

### **Why GitOps?**

GitOps is an operating model that uses **Git as the single source of truth**.

The difference between traditional deployment and GitOps:

| Aspect | Traditional | GitOps |
|--------|-------------|--------|
| Deployment trigger | Jenkins deploys directly | Git repo change → auto sync |
| State management | Manual or scripts | Git repo = desired state |
| Rollback | Redeploy old version | Git revert → auto recovery |
| Visibility | Check logs | Real-time sync status in dashboard |

Adopting GitOps gives you:

1. **Declarative infrastructure** — K8s manifests live in Git, so you define *desired state*, not *current state*
2. **Self-healing** — even if someone deletes a Pod, ArgoCD restores it to the state defined in Git
3. **Audit trail** — every change is recorded as a Git commit

---

### **Overall Architecture**

```
GitHub Push (main branch)
    ↓ (webhook)
Jenkins Pipeline
    ├─ Docker Build (Node 20 + pnpm)
    ├─ GHCR Push (ghcr.io/.../next14-r3f:COMMIT_SHA)
    └─ GitOps repo image tag update
        ↓ (3-min polling)
ArgoCD Auto-Sync
    ↓
K3s cluster deployment
```

---

### **Tech Stack**

| Tool | Role | Notes |
|------|------|-------|
| **K3s** | Lightweight Kubernetes | Single node, `--disable traefik` |
| **ArgoCD** | GitOps Operator | Auto-sync, Self-Heal |
| **Jenkins** | CI pipeline | GitHub Webhook, Pipeline as Code |
| **GHCR** | Container registry | GitHub Container Registry |
| **Nginx** | Reverse proxy | SSL + domain routing |
| **Next.js** | Deployed application | R3F-based portfolio site |

---

In the next post, I'll walk through the actual setup step by step: installing K3s, configuring ArgoCD, and writing the Jenkinsfile.