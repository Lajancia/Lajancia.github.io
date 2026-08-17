---
title: "Building a GitOps CI/CD Pipeline with K3s, ArgoCD & Jenkins (2) — Hands-On Setup"
date: 2026-08-17 12:00:00 +0900
categories: [Study]
tags: [K3s, ArgoCD, Jenkins, GitOps, CI/CD, Kubernetes, DevOps]
author: L.J
lang: en
---

In the previous post, we looked at why we chose K3s and GitOps, and what the overall architecture looks like. In this post, we'll actually build the cluster, install ArgoCD, and write a Jenkins pipeline so that **a single `git push` triggers an entire deployment automatically**.

```
GitHub Push (main branch)
    ↓ (webhook)
Jenkins Pipeline
    ├─ Docker Build (Node 20 + pnpm)
    ├─ GHCR Push (ghcr.io/<USER>/next14-r3f:COMMIT_SHA)
    └─ GitOps repo image tag update
        ↓ (3-min polling)
ArgoCD Auto-Sync → K3s deployment
```

---

### **1. Installing K3s**

The server runs Ubuntu 24.04 LTS VPS (single node). K3s installation is a single command:

```bash
curl -sfL https://get.k3s.io | sh -
```

However, I disabled the **Traefik Ingress Controller** that comes bundled with K3s, since I was already running Nginx as a reverse proxy.

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik" sh -
```

Once installed, verify the node status:

```bash
kubectl get nodes
# NAME        STATUS   ROLES           VERSION
# srv695066   Ready    control-plane   v1.35.5+k3s1
```

Since it's a single-node cluster, only the `control-plane` role shows up. After deploying K3s, the overall system resource usage barely increased — living up to the lightweight promise.

> **Post-install checklist**
> - `kubectl get nodes` → `Ready` status
> - `kubectl get pods -A` → all `kube-system` pods are Running
> - `/etc/rancher/k3s/k3s.yaml` → kubeconfig is accessible

---

### **2. Installing ArgoCD**

ArgoCD is installed using the official manifests:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

The initial admin password is auto-generated and stored in a Secret:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```

Since this is a portfolio/study project (not production), I exposed the service via NodePort for easy access:

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'

kubectl get svc argocd-server -n argocd
# NAME            TYPE       CLUSTER-IP    PORT(S)                       AGE
# argocd-server   NodePort   10.43.x.x     80:32085/TCP,443:31495/TCP   64d
```

Then, configure Nginx reverse proxy to point a subdomain to this NodePort. **Make sure to change the admin password on first login.**

> **Security note**: For production, use Ingress + TLS + SSO instead of NodePort. For a study environment, accessibility was the priority.

---

### **3. Setting Up the GitOps Repository**

The core idea of GitOps is that **manifests are code too**. All deployment YAML files live in a separate repository (`next-r3f-ops`), and ArgoCD watches this repo.

```
next-r3f-ops/
├── argocd-app.yaml   # ArgoCD Application definition
├── next-r3f.yaml     # Deployment
└── service.yaml      # Service
```

The Deployment defines the app's image and resource limits:

```yaml
# next-r3f.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: next-r3f
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: next-r3f
  template:
    metadata:
      labels:
        app: next-r3f
    spec:
      containers:
        - name: next-r3f
          image: ghcr.io/<USER>/next14-r3f:COMMIT_SHA
          ports:
            - containerPort: 3000
          resources:
            requests:
              memory: 256Mi
              cpu: 250m
            limits:
              memory: 512Mi
              cpu: 500m
```

The Service exposes the app via NodePort:

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: next-r3f
  namespace: default
spec:
  selector:
    app: next-r3f
  ports:
    - port: 3000
      targetPort: 3000
      name: http
```

The ArgoCD Application itself is also managed in Git. This is the beauty of this approach — **ArgoCD manages itself via GitOps**.

```yaml
# argocd-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: next-r3f
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<USER>/next-r3f-ops.git
    targetRevision: master
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true      # Delete resources not in Git
      selfHeal: true   # Revert manual changes to Git state
    syncOptions:
      - CreateNamespace=true
```

Two critical options:

- **`selfHeal: true`** — Even if someone manually changes the Deployment via `kubectl`, ArgoCD periodically compares the cluster state with Git and **automatically restores it**.
- **`prune: true`** — If you delete a resource from the Git repo, it gets deleted from the cluster too.

---

### **4. Preparing Jenkins**

Jenkins runs as a Docker container. The key decision here is **mounting the host's docker.sock**:

```bash
docker run -d \
  --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /usr/bin/docker:/usr/bin/docker \
  jenkins/jenkins:lts
```

This is called **Docker-outside-of-Docker (DooD)**. When Jenkins runs `docker build` inside the container, the host's Docker daemon performs the actual build.

- Pros: No need to install Docker inside the container, shared build cache, fast builds
- Cons: Jenkins has **full access** to the host's Docker daemon (security concern). For production, consider DinD or Kaniko.

Two things are needed for GitHub integration:

1. **GitHub Webhook** — Delivers Push events to Jenkins. Go to repo Settings → Webhooks → add Jenkins URL `/github-webhook/`
2. **Credentials** — Jenkins → Manage Jenkins → Credentials → add GitHub PAT. Used by the Jenkinsfile to push to the GitOps repo

---

### **5. Dockerfile — Multi-Stage Optimization**

The Next.js app's Dockerfile uses 4 stages:

```dockerfile
FROM node:20-alpine AS base
RUN apk add --no-cache libc6-compat

# Install pnpm
RUN corepack enable && corepack prepare pnpm@9.11.0 --activate

FROM base AS deps
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --no-frozen-lockfile

FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN pnpm panda && pnpm build

FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1

RUN addgroup --system --gid 1002 nextgroup
RUN adduser --system --uid 1002 nextuser

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextuser:nextgroup /app/.next/standalone ./
COPY --from=builder --chown=nextuser:nextgroup /app/.next/static ./.next/static

USER nextuser
EXPOSE 3000
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"
CMD ["node", "server.js"]
```

Stage breakdown:

| Stage | Role | Description |
|-------|------|-------------|
| `base` | Common base | Node 20 + pnpm activation |
| `deps` | Dependencies | Lock-file-based reproducible install |
| `builder` | Build | `pnpm build` produces `.next/standalone` |
| `runner` | Runtime | Standalone server + **non-root user** |

Two key takeaways:

- **`.next/standalone`** — Next.js's `output: 'standalone'` produces a minimal runtime. It only includes the `node_modules` that are actually used, **significantly reducing the image size**.
- **Non-root execution** — Running as root in a container is risky. A dedicated user (`nextuser`) minimizes the blast radius of a container escape.

---

### **6. Jenkinsfile — Pipeline as Code**

Now the core of the pipeline: the Jenkinsfile. In GitOps, **Jenkins does not deploy directly.** It builds the image, pushes it to the registry, and updates the image tag in the GitOps repo. ArgoCD handles the rest.

```groovy
pipeline {
    agent any

    environment {
        GHCR_IMAGE = 'ghcr.io/<USER>/next14-r3f'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    def shortCommit = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    echo "Building image: ${GHCR_IMAGE}:${shortCommit}"
                    sh "docker build -f dockerfile -t ${GHCR_IMAGE}:${shortCommit} ."
                    sh "docker tag ${GHCR_IMAGE}:${shortCommit} ${GHCR_IMAGE}:latest"
                    sh "docker push ${GHCR_IMAGE}:${shortCommit}"
                    sh "docker push ${GHCR_IMAGE}:latest"
                }
            }
        }

        stage('Update GitOps Repo') {
            steps {
                script {
                    def shortCommit = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    sh """
                        export GIT_TERMINAL_PROMPT=0
                        rm -rf gitops-tmp
                        git clone https://github.com/<USER>/next-r3f-ops.git gitops-tmp
                        cd gitops-tmp
                        sed -i 's|image: ghcr.io/<USER>/next14-r3f:.*|image: ${GHCR_IMAGE}:${shortCommit}|' next-r3f.yaml
                        git config user.name "Jenkins CI"
                        git config user.email "ci@example.com"
                        git add next-r3f.yaml
                        git commit -m "chore: update image tag to ${shortCommit}"
                        git push
                        cd .. && rm -rf gitops-tmp
                    """
                }
            }
        }

        stage('Cleanup') {
            steps {
                sh 'docker image prune -f || true'
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully! Image pushed to GHCR and GitOps repo updated.'
        }
        failure {
            echo 'Pipeline failed. Check Jenkins logs for details.'
        }
    }
}
```

Stage breakdown:

| Stage | What it does |
|-------|-------------|
| **Checkout** | Clone the source code from GitHub |
| **Docker Build & Push** | Build with commit SHA tag + `latest` tag, push to GHCR |
| **Update GitOps Repo** | Clone GitOps repo → `sed` to replace the image tag → commit & push |
| **Cleanup** | Prune old build cache |

Using the **commit SHA** as the image tag is intentional. You can always trace which commit produced which image, and rolling back is as simple as `git revert` — ArgoCD picks up the old image tag automatically.

> **GHCR Authentication**: Login once before the pipeline runs: `echo $PAT | docker login ghcr.io -u <USER> --password-stdin`. Store the PAT in Jenkins Credentials, never in plain text in the Jenkinsfile.

---

### **7. Testing the Full Pipeline**

Now let's verify the entire flow:

```bash
git add .
git commit -m "feat: add new feature"
git push origin main
```

What happens next:

1. **GitHub Webhook** triggers the Jenkins pipeline
2. Jenkins runs: Docker build, GHCR push, GitOps repo image tag update
3. **ArgoCD** detects the change in the GitOps repo (within 3 minutes, default polling)
4. ArgoCD performs a **Rolling Update** with the new image
5. Nginx → NodePort → Pod delivers the traffic

Verification commands:

```bash
# 1. ArgoCD Application status
kubectl get applications -n argocd
# NAME       SYNC STATUS   HEALTH STATUS
# next-r3f   Synced        Healthy

# 2. Check if the pod is running the new image
kubectl get pods -l app=next-r3f -o jsonpath='{.items[0].spec.containers[0].image}'

# 3. HTTP response
curl -sI https://example.com | head -1
# HTTP/2 200
```

One thing I tested out of curiosity: I manually changed the Deployment's image tag using `kubectl edit`. A few minutes later, ArgoCD **automatically reverted it back** to the tag in Git. The self-heal mechanism actually works as advertised! 😄

---

### **8. Lessons Learned (Pain Points)**

1. **Google Fonts loading timeout** — The app would hang during build trying to fetch fonts. Solved by switching to a direct CSS link.
2. **GitOps repo push authentication** — Jenkins got 401 when pushing to the GitOps repo. Fixed by copying `.git-credentials` into the Jenkins container.
3. **Traefik port conflict** — K3s installs Traefik by default, which occupies ports 80/443 and conflicts with Nginx. Always use `--disable traefik` if you already have a reverse proxy.
4. **ArgoCD initial password** — Save yourself the headache: check the Secret right after installation.

---

### **Wrapping Up**

With this setup, the **GitHub Push → Jenkins → GHCR → GitOps Repo → ArgoCD → K3s** pipeline is complete.

The biggest change in mindset is that **deployment is no longer a "scheduled action" — it's the "tracking of a declared state"**. Instead of SSHing into the server to dig through logs when something goes wrong, you just look at the ArgoCD dashboard to see where the sync broke.

In the next post, I'll cover **rollback strategies, ArgoCD Blue/Green deployments, and Secret management (External Secrets / Sealed Secrets)**.