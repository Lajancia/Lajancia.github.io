---
title: "K3s + ArgoCD + Jenkins로 구축하는 GitOps CI/CD 파이프라인 (1) — 개요와 동기"
date: 2026-08-14 12:00:00 +0900
categories: [스터디]
tags: [K3s, ArgoCD, Jenkins, GitOps, CI/CD, Kubernetes, DevOps]
author: L.J
---

### **들어가며**

몇 년 전까지만 해도 나의 배포 환경은 단순했다. VPS 서버에 Docker를 설치하고, Jenkins가 GitHub Webhook을 받아 Docker 이미지를 빌드한 뒤 로컬에서 컨테이너를 실행하는 방식이었다.

```
GitHub Push → Jenkins 빌드 → Docker build → docker run (로컬)
```

이 방식은 간단하지만 몇 가지 아쉬운 점이 있었다.

- 컨테이너가 죽으면 **직접 다시 띄워야** 한다
- **롤백**이 번거롭다
- 여러 서비스가 늘어나면 관리가 어렵다
- 배포 상태를 한눈에 볼 수 있는 **대시보드가 없다**

이런 고민을 하던 중 Kubernetes를 공부하게 되었고, 이번 기회에 **경량 K3s 클러스터**를 직접 구축하고 **GitOps 방식의 CI/CD 파이프라인**을 만들어보기로 했다.

---

### **왜 K3s인가?**

Kubernetes의 풀 버전은 무겁고 복잡하다. 특히 나처럼 **싱글 노드 VPS**에서 운영할 경우 오버헤드가 부담스럽다. 반면 K3s는:

- **경량** — 바이너리 하나로 실행 가능 (약 60MB)
- **저사양 지원** — 1GB RAM, 1 CPU로도 동작
- **kubeconfig 호환** — 표준 kubectl 명령어 그대로 사용
- **설치가 간단** — 단일 명령어로 클러스터 구축 완료

```
curl -sfL https://get.k3s.io | sh -
```

실제로 서버 한 대에 K3s를 올렸더니, 전체 시스템 리소스 사용량이 크게 늘지 않았다.

---

### **왜 GitOps인가?**

GitOps는 **"Git을 단일 진실 공급원(Single Source of Truth)** 으로 사용하는 운영 방식이다.

기존 배포 방식과 GitOps의 차이:

| 구분 | 기존 방식 | GitOps |
|------|----------|--------|
| 배포 트리거 | Jenkins가 직접 배포 | Git 리포 변경 → 자동 Sync |
| 상태 관리 | 수동 또는 스크립트 | Git 리포 = 현재 상태 |
| 롤백 | 이전 버전 다시 배포 | Git revert → 자동 복구 |
| 가시성 | 로그 확인 | 대시보드에서 실시간 Sync 상태 확인 |

GitOps를 도입하면 다음과 같은 장점이 있다:

1. **선언적 인프라** — Kubernetes 매니페스트가 Git에 있으므로 "지금 어떤 상태인지"가 아닌 "원하는 상태"를 정의
2. **자동 복구** — 누군가 실수로 Pod을 지워도 ArgoCD가 Git에 정의된 상태로 되돌림
3. **감사 추적** — 모든 변경이 Git commit으로 기록됨

---

### **전체 아키텍처**

```
GitHub Push (main 브랜치)
    ↓ (webhook)
Jenkins Pipeline
    ├─ Docker Build (Node 20 + pnpm)
    ├─ GHCR Push (ghcr.io/.../next14-r3f:커밋SHA)
    └─ GitOps 리포 이미지 태그 업데이트
        ↓ (3분 감지)
ArgoCD Auto-Sync
    ↓
K3s 클러스터 배포
```

---

### **사용한 기술 스택**

| 도구 | 역할 | 비고 |
|------|------|------|
| **K3s** | 경량 Kubernetes | 싱글 노드, --disable traefik |
| **ArgoCD** | GitOps Operator | Auto-sync, Self-Heal |
| **Jenkins** | CI 파이프라인 | GitHub Webhook, Pipeline as Code |
| **GHCR** | 컨테이너 레지스트리 | GitHub Container Registry |
| **Nginx** | 리버스 프록시 | SSL + 도메인 라우팅 |
| **Next.js** | 배포 애플리케이션 | R3F 기반 포트폴리오 사이트 |

---

다음 편에서는 실제 K3s 설치부터 ArgoCD 설정, Jenkinsfile 작성까지 단계별로 다뤄보겠다.