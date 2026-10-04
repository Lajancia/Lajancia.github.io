---
title: "홈랩 CI/CD 전면 개편기 — nginx+Jenkins에서 k3s Traefik + Cloudflare 인증서 + GitHub Actions GitOps까지"
date: 2026-10-04 23:00:00 +0900
categories: [개발, DevOps]
tags: [K3s, Traefik, Cloudflare, GitHub Actions, ArgoCD, GitOps, CI/CD, Homelab]
author: L.J
---

### **들어가며**

이 글은 운영 중인 홈랩(soominlab.com)의 배포·인프라 구조를 전면 개편한 과정을 정리한 글이다. 바꾸기 전과 후를 한 문장으로 압축하면:

> **Docker nginx + certbot + Jenkins 컨테이너 → k3s Traefik + Cloudflare Origin 인증서 + GitHub Actions → GitOps(ArgeCD)**

서비스는 두 개다. 포트폴리오 사이트 `soominlab.com`(Next.js case-study)과 창작 데모 `creative.soominlab.com`(Next.js + React Three Fiber). 이 둘을 하나의 파이프라인 체계로 묶는 것이 목표였다.

---

### **바꾸기 전: 세 조각으로 흩어진 운영**

기존 구조의 문제는 "인프라가 세 가지 패러다임에 흩어져 있었다"는 점이다.

**1. 리버스 프록시: nginx + certbot 컨테이너**

- nginx Docker 컨테이너가 80/443을 받고, certbot이 Let's Encrypt 인증서를 90일 주기로 갱신
- 갱신 타이밍마다 "인증서 만료 전 알림 → 수동 확인"이 마음에 걸렸고, nginx 컨테이너와 앱 컨테이너의 포트·설정 결합이 복잡해졌다

**2. CI: Jenkins 컨테이너**

- `GitHub → Jenkins → GHCR → GitOps 리포 → ArgoCD → k3s` 파이프라인이 잘 돌아가고 있었지만, Jenkins 자체의 운영이 부담이었다
- 컨테이너 기반 Jenkins는 플러그인 업데이트, 크레던셜 관리, 워커 리소스를 계속 손봐야 한다
- 게다가 **서버 안에서 Jenkins가 또 하나의 "서버" 역할**을 했다 — 홈랩에서 운영하는 것의 본질적인 비용

**3. 배포: ArgoCD는 이미 있었음**

- GitOps 단계는 이미 k3s + ArgoCD로 완성돼 있었고, 이 부분은 유지하기로 했다

즉, 바꿀 대상은 **TLS(nginx/certbot)와 CI(Jenkins)**였고, 유지할 대상은 **k3s + ArgoCD**였다.

---

### **바꾼 후: 전체 아키텍처**

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

CI:  push(main) → GitHub Actions → GHCR push → ops 리포 이미지 태그 bump → push
CD:  ArgoCD가 ops 리포 감시 → 자동 sync → k3s 배포
```

#### **1. 리버스 프록시: k3s Traefik으로 통일**

k3s 설치 시 번들 Traefik을 `--disable traefik`으로 끄고, 공식 Helm 차트를 HelmChart 매니페스트로 직접 배포했다:

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

svclb가 호스트의 80/443을 점유하면서 nginx 컨테이너는 퇴역했다(중지, 제거는 보존 후). 라우팅은 IngressRoute CRD로 선언한다:

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

캐시 헤더 같은 세부 제어도 Middleware CRD로 선언적으로 처리된다(RDKit.js처럼 파일명에 해시가 없는 정적 자산에 1일 캐시를 걸었다).

#### **2. TLS: Let's Encrypt → Cloudflare Origin 인증서**

이번 개편에서 가장 만족도가 높은 선택이다.

- **Cloudflare Origin CA 인증서**를 발급해 `*.soominlab.com` 전체를 한 장으로 커버한다
- 이 인증서는 **Cloudflare↔origin 구간 전용**이다. 인터넷 사용자는 Cloudflare 엣지의 인증서를 보고, CF가 origin에 접속할 때만 Origin 인증서를 쓴다
- k3s에는 `soominlab-origin-tls` 시크릿으로 넣고, **TLSStore `default`**로 지정해 모든 IngressRoute에 자동 적용된다:

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

- Cloudflare는 **Full (strict)** 모드 — CF가 origin 인증서를 검증하므로 중간자 공격 없이 end-to-end TLS가 보장된다
- certbot 갱신 루프가 통째로 사라졌다. Origin 인증서 유효기간은 **15년**이라, 실질적으로 "TLS 운영"이라는 작업 자체가 소멸했다

물론 트레이드오프는 있다. Origin 인증서는 Cloudflare를 통하지 않으면 유효하지 않으므로, **CF 프록시가 사실상 필수**가 된다. 홈랩 기준으로는 오히려 장점이다 — origin IP가 CF 뒤로 가려진다.

#### **3. CI: Jenkins → GitHub Actions**

Jenkins 컨테이너를 제거하고(jenkins_home 볼륨은 보존), 파이프라인을 GitHub Actions로 옮겼다:

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

- 빌드·GHCR 푸시·ops 리포의 이미지 태그 bump를 **하나의 workflow**에서 처리
- 토큰은 GitHub Secrets에만 존재하고, 서버에 어떤 자격증명도 저장되지 않는다
- Jenkins의 "또 하나의 서버"가 사라졌고, 파이프라인 정의가 리포 안의 yaml 파일 하나가 됐다

#### **4. CD: ArgoCD는 그대로**

GitOps 리포(case-study-ops, next-r3f-ops)를 감시하던 ArgoCD는 그대로 유지됐다. CI 도구가 Jenkins에서 Actions로 바뀌어도, **ops 리포에 이미지 태그 커밋만 착실하게 쌓이면** ArgoCD가 나머지를 처리한다. 이 경계를 명확히 나눈 게 개편이 순조로웠던 핵심이었다:

> **CI는 "소스 → 이미지 → ops 리포 커밋"까지만, CD는 ArgoCD가 전담.**

---

### **개편 중 만난 트러블**

**1. NodePort 하드닝 방화벽이 아웃바운드를 죽였다**

k3s NodePort(30000+)를 인터넷에 노출하지 않으려고 iptables raw 테이블 PREROUTING에 "80/443 외 TCP 드롭" 룰을 걸었다. 그런데 이 룰이 **서버 자신의 아웃바운드 연결 응답까지** 드롭했다 — PREROUTING은 아웃바운드 연결의 회신 패킷도 통과하는 지점이기 때문. 게다가 raw 테이블은 conntrack보다 먼저 실행되어 `--ctstate ESTABLISHED` 예외도 매칭이 안 됐다.

결과적으로 github.com 접근이 죽고, Discord가 끊기고, 이걸 "외부 네트워크 문제"로 오해해 Cloudflare 프록시 우회(gh 도메인)까지 만들었다. 최종 해결은 **룰을 mangle 테이블(conntrack 이후)로 옮기고 ESTABLISHED 회신을 통과**시키는 것. 자세한 추적기는 별도 포스트로 정리했다.

**2. GitOps 자기 참조와 상태 일치**

Application이 자신의 소스 리포에 의해 정의되는 구조에서는, repoURL 같은 필드가 **yaml HEAD ↔ 클러스터 상태** 중 어느 한쪽이라도 어긋나면 self-heal이 옛 값으로 되돌린다. repoURL을 바꿀 때는 yaml, Application, credential secret 세 곳을 **같은 커밋 사이클 안에서** 일치시켜야 한다.

---

### **정리: 뭐가 좋아졌나**

| 영역 | 이전 | 이후 |
|---|---|---|
| TLS | nginx + certbot 컨테이너, 90일 갱신 | Cloudflare Origin 인증서, TLSStore 1개, 사실상 유지보수 0 |
| 프록시 | nginx Docker | Traefik (k3s HelmChart), 라우팅은 모두 Git으로 |
| CI | Jenkins 컨테이너 | GitHub Actions (서버 리소스 0) |
| CD | ArgoCD | ArgoCD (변경 없음) |
| 인프라 정의 | 흩어짐 (docker-compose, jenkins job) | k3s 매니페스트 + GitHub workflow = 전부 코드 |

배포 흐름도 단순해졌다: **git push 한 번이면 Actions가 이미지를 만들고, ops 리포를 갱신하고, ArgoCD가 배포한다.** 사람이 하는 일은 커밋뿐이다.

홈랩이라서 가능했던 단순화도 있지만 — 단일 노드 k3s, 두 개의 서비스 — 오히려 이 규모에서 "운영 대상을 줄이는 선택"이 가장 효과가 컸다. nginx, certbot, Jenkins 세 개의 컨테이너가 사라지고, 남은 건 k3s 매니페스트와 workflow yaml뿐이다.

다음 단계로는 ArgoCD ApplicationSet으로 두 앱을 템플릿화하고, 새 서비스 추가를 "yaml 복사" 수준으로 만드는 것을 고민하고 있다.
