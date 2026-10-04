---
title: "Rebuilding My Homelab CI/CD — From nginx + Jenkins to k3s Traefik, Cloudflare Certificates, and GitHub Actions GitOps"
date: 2026-10-04 23:00:00 +0900
categories: [개발, DevOps]
tags: [K3s, Traefik, Cloudflare, GitHub Actions, ArgoCD, GitOps, CI/CD, Homelab]
author: L.J
lang: en
---

### **Introduction**

This post documents the full overhaul of my homelab's (soominlab.com) deployment and infrastructure stack. Compressed into one sentence:

> **Docker nginx + certbot + a Jenkins container → k3s Traefik + Cloudflare Origin certificates + GitHub Actions → GitOps (ArgoCD)**

There are two services: the portfolio site `soominlab.com` (a Next.js case-study) and `creative.soominlab.com` (Next.js + React Three Fiber). The goal was to bring both under a single pipeline system.

---

### **Before: Operations Scattered Across Three Paradigms**

The old setup's core problem was that infrastructure lived in three different paradigms.

**1. Reverse proxy: nginx + certbot containers**

- A Dockerized nginx took 80/443, and certbot renewed Let's Encrypt certificates on a 90-day cycle
- Every renewal was a nagging "did it expire?" checklist item, and the coupling between the nginx container's ports/config and the app containers kept growing

**2. CI: a Jenkins container**

- The pipeline `GitHub → Jenkins → GHCR → GitOps repo → ArgoCD → k3s` worked fine, but Jenkins itself was the burden
- A containerized Jenkins demands constant care: plugin updates, credential management, worker resources
- Worse, **Jenkins was effectively another "server" to operate** — the essential cost of running it on a homelab

**3. CD: ArgoCD already existed**

- The GitOps layer was already complete with k3s + ArgoCD, so it stayed as-is

So the targets for replacement were **TLS (nginx/certbot) and CI (Jenkins)**; k3s + ArgoCD stayed.

---

### **After: The Full Architecture**

```
                    ┌── Cloudflare (proxied, Full strict) ──┐
soominlab.com ──────┤                                       │
creative.soominlab.com ──┘                                  │
                         │ 443 (svclb hostport)
                         ▼
              Traefik (HelmChart, kube-system)
                         │ TLSStore default → soominlab-origin-tls
                         ▼
        case-study:3000 / next-r3f:3000 (k3s Deployment)

CI:  push(main) → GitHub Actions → GHCR push → bump image tag in ops repo → push
CD:  ArgoCD watches ops repo → auto sync → k3s deploy
```

#### **1. Reverse Proxy: Unified on k3s Traefik**

The bundled k3s Traefik was disabled at install time (`--disable traefik`), and the official Helm chart is deployed directly via a HelmChart manifest:

```yaml
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: traefik-custom
  namespace: kube-system
spec:
  chart: traefik
  repo: https://traefik.github.io/charts
  valuesContent: |-
    ports:
      web: { port: 80 }
      websecure: { port: 443 }
    service: { type: LoadBalancer }
    additionalArguments:
      - --entrypoints.web.http.redirections.entrypoint.to=websecure
      - --entrypoints.web.http.redirections.entrypoint.scheme=https
      - --entrypoints.web.http.redirections.entrypoint.permanent=true
```

svclb now occupies the host's 80/443, and the nginx container retired (stopped; kept around before removal). Routing is declared with IngressRoute CRDs:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: case-study
  namespace: default
spec:
  entryPoints: [websecure]
  routes:
    - match: Host(`soominlab.com`)
      kind: Rule
      services:
        - name: case-study
          port: 3000
```

Fine-grained controls like cache headers are also declarative via Middleware CRDs — for example, a 1-day cache for RDKit.js static assets that lack content hashes in their filenames.

#### **2. TLS: Let's Encrypt → Cloudflare Origin Certificate**

The highest-satisfaction choice of the whole overhaul.

- A **Cloudflare Origin CA certificate** covers the entire `*.soominlab.com` with a single cert
- This certificate is valid **only on the Cloudflare↔origin leg**. Internet users see the certificate on Cloudflare's edge; CF uses the Origin certificate only when connecting back to the server
- In k3s it's stored as the `soominlab-origin-tls` secret and applied automatically to every IngressRoute via a **default TLSStore**:

```yaml
apiVersion: traefik.io/v1alpha1
kind: TLSStore
metadata:
  name: default
  namespace: kube-system
spec:
  defaultCertificate:
    secretName: soominlab-origin-tls
```

- Cloudflare runs in **Full (strict)** mode — CF validates the origin certificate, guaranteeing end-to-end TLS without a man in the middle
- The certbot renewal loop disappeared entirely. The Origin certificate is valid for **15 years**, so "TLS operations" essentially ceased to exist as a task

There is a trade-off: the Origin certificate is only valid behind Cloudflare, making **the CF proxy effectively mandatory**. For a homelab, that's actually a bonus — the origin IP is hidden behind CF.

#### **3. CI: Jenkins → GitHub Actions**

The Jenkins container was removed (the jenkins_home volume was preserved), and the pipeline moved to GitHub Actions:

```yaml
name: Build & Deploy (GitOps)
on:
  push:
    branches: [main]
jobs:
  build-push:
    runs-on: ubuntu-latest
    steps:
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: g3941813-svg
          password: ${{ secrets.GHCR_GITOPS_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          push: true
          tags: |
            ghcr.io/g3941813-svg/case-study:${{ github.sha }}
            ghcr.io/g3941813-svg/case-study:latest
      - name: Update GitOps repo
        uses: actions/checkout@v4
        with:
          repository: g3941813-svg/case-study-ops
          token: ${{ secrets.GHCR_GITOPS_TOKEN }}
      - run: |
          cd gitops
          sed -i "s|image: ghcr.io/g3941813-svg/case-study:.*|image: ghcr.io/g3941813-svg/case-study:${GITHUB_SHA::7}|" case-study.yaml
          git commit -am "chore: update image tag" && git push
```

- Build, GHCR push, and the image-tag bump in the ops repo are handled in **one workflow**
- Tokens live only in GitHub Secrets; no credentials are stored on the server
- Jenkins-as-another-server is gone, and the pipeline definition became a single yaml file inside the repo

#### **4. CD: ArgoCD, Unchanged**

ArgoCD continues watching the GitOps repos (case-study-ops, next-r3f-ops). Even though CI moved from Jenkins to Actions, **as long as image-tag commits land in the ops repo**, ArgoCD handles the rest. Drawing this boundary clearly is why the overhaul went smoothly:

> **CI ends at "source → image → ops repo commit"; CD belongs entirely to ArgoCD.**

---

### **Troubles Encountered During the Migration**

**GitOps self-reference and state consistency**

In a structure where an Application is defined by its own source repo, fields like repoURL revert to old values via self-heal if **the yaml HEAD and the cluster state** ever disagree. When changing a repoURL, the yaml, the Application, and the credential secret must be made consistent **within the same commit cycle**.

---

### **Summary: What Got Better**

| Area | Before | After |
|---|---|---|
| TLS | nginx + certbot containers, 90-day renewals | Cloudflare Origin cert, one TLSStore, effectively zero maintenance |
| Proxy | nginx Docker | Traefik (k3s HelmChart), all routing in Git |
| CI | Jenkins container | GitHub Actions (zero server resources) |
| CD | ArgoCD | ArgoCD (unchanged) |
| Infra definition | Scattered (docker-compose, Jenkins jobs) | k3s manifests + GitHub workflow = all code |

The deployment flow simplified too: **one git push, and Actions builds the image, updates the ops repo, and ArgoCD deploys.** The only human step is a commit.

Some of this simplification is only possible because it's a homelab — a single-node k3s, two services. But at exactly this scale, "choosing to operate fewer things" pays off the most. Three containers (nginx, certbot, Jenkins) are gone, and what remains is k3s manifests and workflow yaml.

The next step I'm considering: templating the two applications with an ArgoCD ApplicationSet, so adding a new service becomes "copy a yaml."
