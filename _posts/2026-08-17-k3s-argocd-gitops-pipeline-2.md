---
title: "K3s + ArgoCD + Jenkins로 구축하는 GitOps CI/CD 파이프라인 (2) — 실제 구축하기"
date: 2026-08-17 12:00:00 +0900
categories: [스터디]
tags: [K3s, ArgoCD, Jenkins, GitOps, CI/CD, Kubernetes, DevOps]
author: L.J
---

지난 편에서는 왜 K3s와 GitOps인지, 전체 아키텍처가 어떤 모양인지 살펴봤다. 이번 편에서는 실제로 클러스터를 구축하고 ArgoCD와 Jenkins를 연결해 **GitHub Push 하나로 배포가 끝나는 파이프라인**을 완성해보겠다.

```
GitHub Push (main)
    ↓ (webhook)
Jenkins Pipeline
    ├─ Docker Build (Node 20 + pnpm)
    ├─ GHCR Push (ghcr.io/<USER>/next14-r3f:커밋SHA)
    └─ GitOps 리포 이미지 태그 업데이트
        ↓ (3분 감지)
ArgoCD Auto-Sync → K3s 배포
```

---

### **1. K3s 설치**

서버는 Ubuntu 24.04 VPS (싱글 노드)를 사용했다. K3s 설치는 단일 명령어로 끝난다.

```bash
curl -sfL https://get.k3s.io | sh -
```

다만 나는 기본으로 설치되는 **Traefik Ingress Controller를 꺼줬다**. 이미 Nginx를 리버스 프록시로 운영하고 있었기 때문이다.

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik" sh -
```

설치가 끝나면 노드 상태를 확인한다.

```bash
kubectl get nodes
# NAME        STATUS   ROLES           VERSION
# srv695066   Ready    control-plane   v1.35.5+k3s1
```

싱글 노드여서 `control-plane` 역할 하나뿐이다. K3s를 올리고 나서도 전체 시스템 리소스 사용량이 크게 늘지 않았다. 경량 분포의 장점이 체감되는 순간이다.

> **설치 후 체크리스트**
> - `kubectl get nodes` → `Ready` 상태
> - `kubectl get pods -A` → `kube-system` 파드들이 Running인지
> - `/etc/rancher/k3s/k3s.yaml` → kubeconfig로 로컬에서 접속 가능한지

---

### **2. ArgoCD 설치**

ArgoCD는 공식 매니페스트를 그대로 사용했다.

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

초기 admin 비밀번호는 자동 생성되어 Secret에 저장된다.

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```

운영이 아니라 포트폴리오/스터디 목적이므로 접근을 위해 NodePort로 서비스를 노출했다.

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'

kubectl get svc argocd-server -n argocd
# NAME            TYPE       CLUSTER-IP    PORT(S)                       AGE
# argocd-server   NodePort   10.43.x.x     80:32085/TCP,443:31495/TCP   64d
```

이후 Nginx 리버스 프록시로 특정 서브도메인을 이 NodePort에 연결하면 웹 UI로 접근할 수 있다. **admin 비밀번호는 반드시 첫 로그인 후 변경**하자.

> **보안 팁**: 실서비스라면 NodePort 대신 Ingress + TLS + SSO를 구성하는 게 맞다. 스터디 환경에서는 접근성을 우선했다.

---

### **3. GitOps 리포 구성**

GitOps의 핵심은 **"매니페스트도 코드"** 라는 점이다. 배포에 필요한 모든 YAML을 별도 리포(`next-r3f-ops`)에 모아두고, ArgoCD가 이 리포를 바라보게 한다.

```
next-r3f-ops/
├── argocd-app.yaml   # ArgoCD Application 정의
├── next-r3f.yaml     # Deployment
└── service.yaml      # Service
```

Deployment는 배포할 앱의 이미지와 리소스 한도를 정의한다.

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
          image: ghcr.io/<USER>/next14-r3f:커밋SHA
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

Service는 NodePort로 외부에 노출한다.

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

ArgoCD Application도 마찬가지로 Git에서 관리한다. 이게 이 구조의 묘미인데, **ArgoCD 자기 자신조차 GitOps로 관리**하게 된다.

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
      prune: true      # Git에 없는 리소스는 삭제
      selfHeal: true   # 수동 변경 시 Git 상태로 되돌림
    syncOptions:
      - CreateNamespace=true
```

핵심 옵션 두 가지:

- **`selfHeal: true`** — 누군가 `kubectl`로 Deployment를 지우거나 이미지를 바꿔도 ArgoCD가 주기적으로 Git 상태와 비교해 **자동으로 원복**한다.
- **`prune: true`** — Git 리포에서 리소스를 삭제하면 클러스터에서도 같이 삭제된다.

---

### **4. Jenkins 준비**

Jenkins도 Docker 컨테이너로 띄웠다. 이때 중요한 결정이 있는데, 바로 **호스트의 docker.sock을 마운트**하는 것이다.

```bash
docker run -d \
  --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /usr/bin/docker:/usr/bin/docker \
  jenkins/jenkins:lts
```

이 방식은 **Docker-outside-of-Docker (DooD)** 라고 부른다. Jenkins 컨테이너 안에서 `docker build` 명령을 실행하면 호스트의 Docker 데몬이 실제 빌드를 수행한다.

- 장점: 컨테이너 안에서 또 Docker를 설치할 필요 없음, 빌드 속도와 캐시 공유
- 단점: Jenkins 컨테이너가 호스트 Docker에 **전권**을 갖게 됨 (보안 민감). 실서비스라면 DinD(docker-in-docker)나 Kaniko/containerd 빌더를 고려하자.

GitHub와의 연동은 두 가지가 필요하다.

1. **GitHub Webhook** — Jenkins에 Push 이벤트를 전달. 리포 설정 → Webhooks → Jenkins URL `/github-webhook/` 등록
2. **Credentials** — Jenkins → Manage Jenkins → Credentials에서 GitHub PAT 등록. Jenkinsfile이 GitOps 리포에 push할 때 사용

---

### **5. Dockerfile — 멀티 스테이지 최적화**

Next.js 앱의 Dockerfile은 4단계 멀티 스테이지로 구성했다.

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

포인트를 정리하면:

| 스테이지 | 역할 | 설명 |
|---------|------|------|
| `base` | 공용 베이스 | Node 20 + pnpm 활성화 |
| `deps` | 의존성 설치 | lock 파일 기반 재현 가능한 설치 |
| `builder` | 빌드 | `pnpm build`로 `.next/standalone` 산출 |
| `runner` | 런타임 | standalone 서버 + **non-root 사용자** |

특히 마지막 두 가지가 중요하다.

- **`.next/standalone`** — Next.js의 output 설정으로 생성되는 최소 실행 단위. `node_modules` 중 실제 사용하는 것만 포함되어 **이미지 크기가 크게 줄어든다**.
- **non-root 실행** — 컨테이너에서 root로 실행하면 보안 사고 시 컨테이너 탈출 위험이 커진다. 전용 유저(`nextuser`)를 만들어 실행한다.

---

### **6. Jenkinsfile — 파이프라인 코드**

이제 핵심인 Jenkinsfile이다. GitOps 방식에서는 **Jenkins가 직접 배포하지 않는다.** 이미지를 빌드해서 레지스트리에 올리고, GitOps 리포의 이미지 태그만 바꿔주면 나머지는 ArgoCD가 한다.

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

각 스테이지의 역할:

| 스테이지 | 하는 일 |
|---------|--------|
| **Checkout** | GitHub 소스 클론 |
| **Docker Build & Push** | 커밋 SHA 태그 + `latest` 태그로 GHCR에 push |
| **Update GitOps Repo** | GitOps 리포를 클론 → `sed`로 이미지 태그 교체 → commit & push |
| **Cleanup** | 빌드 캐시 정리 |

이미지 태그를 **커밋 SHA**로 쓰는 게 포인트다. 어느 커밋에서 만들어진 이미지인지가 태그만 봐도 명확해지고, 롤백할 때도 `git revert` 후 해당 SHA 이미지를 다시 참조하면 된다.

> **GHCR 인증**: `docker push` 전에 한 번만 로그인해두면 된다. `echo $PAT | docker login ghcr.io -u <USER> --password-stdin` — PAT는 Jenkins Credentials에 저장하고 Jenkinsfile에 평문으로 넣지 않는다.

---

### **7. 배포 테스트 — Push 한 번으로 끝까지**

이제 실제로 흐름을 검증해보자.

```bash
git add .
git commit -m "feat: add new feature"
git push origin main
```

이후 일어나는 일:

1. **GitHub Webhook**이 Jenkins에 Push 이벤트 전달
2. Jenkins 파이프라인 실행 — Docker build, GHCR push, GitOps 리포 이미지 태그 업데이트
3. **ArgoCD**가 3분 이내에 GitOps 리포 변경 감지 (기본 polling)
4. ArgoCD가 새 이미지로 **Rolling Update** 실행
5. Nginx → NodePort → Pod으로 트래픽 전달

검증 명령어:

```bash
# 1. ArgoCD Application 상태
kubectl get applications -n argocd
# NAME       SYNC STATUS   HEALTH STATUS
# next-r3f   Synced        Healthy

# 2. Pod이 새 이미지를 쓰는지
kubectl get pods -l app=next-r3f -o jsonpath='{.items[0].spec.containers[0].image}'

# 3. HTTP 응답
curl -sI https://example.com | head -1
# HTTP/2 200
```

실제로 이 과정에서 재미있는 걸 하나 발견했다. `kubectl`로 Deployment의 이미지를 수동으로 바꿔봤는데, 몇 분 뒤 ArgoCD가 **자동으로 Git에 있는 이미지로 되돌려놨다.** self-heal이 실제로 동작하는 걸 눈으로 확인한 순간이었다. 😄

---

### **8. 구축하면서 겪은 삽질 포인트**

1. **Google Fonts 로딩 타임아웃** — 앱이 폰트를 가져오다가 빌드가 멈추는 문제가 있었다. CSS 파일을 직접 연결하는 방식으로 전환해 해결했다.
2. **GitOps 리포 push 인증** — Jenkins가 GitOps 리포에 push할 때 401이 났다. Jenkins 컨테이너에 `.git-credentials`를 복사해 해결했다.
3. **Traefik 기본 설치** — K3s 기본 설치 시 Traefik이 80/443을 점유해 Nginx와 충돌한다. 처음부터 `--disable traefik`으로 설치하자.
4. **초기 admin 비밀번호** — ArgoCD 로그인에 헤매지 않도록 초기 비밀번호는 Secret에서 바로 꺼내 쓰자.

---

### **마치며**

이렇게 해서 **GitHub Push → Jenkins → GHCR → GitOps 리포 → ArgoCD → K3s** 로 이어지는 GitOps 파이프라인이 완성됐다.

가장 큰 변화는 **배포가 "예약된 동작"이 아니라 "선언된 상태의 추적"이 됐다는 것**이다. 배포 실패의 원인을 찾으려고 서버에 들어가 로그를 뒤질 필요 없이, ArgoCD 대시보드에서 Sync 상태를 보면 어디서 문제인지 한눈에 파악할 수 있다.

다음 편에서는 **롤백 전략과 ArgoCD의 Blue/Green 배포**, 그리고 **Secret 관리(External Secrets / Sealed Secrets)** 를 다뤄볼 예정이다.
