---
title: "서버를 코드로 — Ansible로 홈랩 서버 재현·보안검사 자동화하기"
date: 2026-11-19 22:00:00 +0900
categories: [개발, DevOps]
tags: [Ansible, K3s, GitOps, IaC, 보안, Homelab]
author: L.J
---

### **들어가며**

이번 글은 홈랩(soominlab.com)의 서버 상태 전체를 Ansible playbook으로 코드화한 과정을 정리했다. 목표는 단순하다:

1. 언젠가 서버를 옮기게 되면(더 저렴한 호스팅, AWS 등) `ansible-playbook site.yml` 한 번으로 현재 구성 — k3s, nodeport-guard, tailscale serve, kube-prometheus-stack, 주간 보안검사 — 이 동일하게 올라간다.
2. 과거 두 번 겪은 "도커 컨테이너가 사라진 뒤 포트가 노출됐다" 사고를 **서버가 스스로 매주 검사하고, 문제 있을 때만 알려주는** 구조로 막는다.

---

## 1. Ansible이란 무엇인가

Ansible은 **서버의 상태를 코드(YAML)로 선언하고, 현재 상태를 그 선언과 일치시켜주는 도구**다.

- Chef/Puppet처럼 에이전트를 서버에 설치할 필요가 없다. SSH만 연결되면 동작한다.
- 내가 "apt로 curl, git을 설치해라"라고 **절차를** 쓰는 게 아니라, "curl과 git이 **설치되어 있어야 한다**"고 **상태**를 쓴다.
- 같은 playbook을 몇 번을 돌려도 결과가 같다 — **멱등성(idempotency)**. 이미 원하는 상태면 아무것도 하지 않는다. 이 특성이 이 글의 핵심이다.

구성 요소는 세 개면 충분하다:

| 개념 | 역할 |
|---|---|
| **Inventory** | 관리 대상 서버 목록 (`inventory/hosts.yml`) |
| **Playbook** | "이 서버에 이 role들을 적용해라" 선언 (`site.yml`) |
| **Role** | 재사용 가능한 상태 묶음 (k3s, 방화벽, 모니터링 등) |

---

## 2. 왜 Ansible인가 — 두 번의 사고와 하나의 계획

### 사고: 호스팅 업체 셧다운 뒤에 남은 것

과거에 호스팅 업체가 서버를 강제로 셧다운/재부팅한 적이 있었다. 그때 일부 서비스가 누락되어 내려간 상태로 도커가 살아나면서 **의도하지 않은 포트가 공개 인터넷에 노출**됐었다. 서비스가 죽어 있는데 포트만 열려 있는 상태는 최악의 조합이다.

이 서버에서는 이후 두 층의 방어를 직접 구축했다 (앞선 글의 사건):

- **nodeport-guard**: k3s NodePort는 mangle 테이블 iptables 룰로 eth0에서 80/443만 허용하고 나머지는 DROP. Tailscale(100.64.0.0/10)은 예외 RETURN.
- **이중 영속화**: 라이브 룰 + `/etc/iptables/rules.v4`에 모두 존재해야 한다. 호스팅 업체의 hypervisor 유지보수 재부팅은 룰을 날리기 때문이다.

그런데 방어가 있어도 "그 방어가 지금 살아 있는지"는 별개 문제다. 결국 **검사도 자동화**해야 했다.

### 계획: 더 저렴한 서버로의 이사

두 번째 이유는 앞으로의 이사 계획이다. 지금 구성은 k3s, iptables, systemd timer, tailscale, helm까지 손으로 세팅된 상태다. 새 서버(AWS든 더 저렴한 VPS든)가 생겼을 때 이걸 기억에 의존해 다시 세팅하면 **반드시 누락이 생긴다** — 위 사고가 정확히 그 패턴이었다. 서버 상태를 코드로 보관하면 이사는 "git clone → secrets 주입 → playbook 실행"이 된다.

> IaC(Infrastructure as Code)는 대규모 조직의 도구가 아니라, **혼자 서버를 운영하는 사람에게 더 절실한 도구**다. 기억이 코드보다 빨리 풍화되기 때문이다.

---

## 3. 사용 방법 — 실제 구성

리포 구성은 이렇다 (실제 리포는 프라이빗으로 운영 중이라 아래는 예시 트리):

```
server-ops/
├── site.yml              # 적용 순서 선언
├── inventory/hosts.yml   # 대상 서버
└── roles/
    ├── common/           # 베이스 패키지 (도커는 의도적으로 설치 안 함)
    ├── k3s/              # k3s server 설치 + node Ready 대기
    ├── nodeport_guard/   # mangle 방화벽 유닛 + rules.v4 이중 영속화
    ├── tailscale/        # 설치 + serve (ArgoCD :8443, Grafana :8444)
    ├── monitoring/       # kube-prometheus-stack (helm)
    └── weekly_audit/     # 주간 보안검사 (systemd timer)
```

### Secrets은 절대 리포에 넣지 않는다

kube-prometheus-stack values 파일에는 Grafana 관리자 비밀번호가 있다. 이런 파일은 리포 밖의 `/root/server-ops-secrets/`(chmod 700)에 두고 playbook 실행 시 경로로 참조한다. Discord webhook URL도 마찬가지다. **PAT 절대 저장 금지 규칙과 같은 맥락**으로, secrets은 수동 주입이 원칙이다.

### 주간 보안검사 — "서버가 스스로, 문제 있을 때만"

`weekly-audit.timer`(매주 일요일 03:00 UTC, `Persistent=true`라 재부팅 후 놓친 실행을 보충한다)가 스크립트를 돌린다. 검사 항목은 과거 사고의 재현을 정확히 노린다:

1. **도커 컨테이너 실행 감지** — 이 서버의 모든 서비스는 k3s(자체 containerd) 위에서 운영되므로, Docker 컨테이너의 정상 개수는 **0개**다. Docker는 사실상 비어 있는 유령 상태이고, 하나라도 살아 있으면 의도치 않게 올라온 것 — 과거 사고의 패턴이다.
2. **guard 룰 이중 확인** — mangle 라이브 룰과 `rules.v4` 영속화 둘 다 점검. 재부팅 후 어느 한쪽이 비면 잡힌다.
3. **NodePort 외부 스캔** — 30801(그라파나)/32085(ArgoCD)를 서버에서 curl하는 것은 거짓 보고를 한다(자기 자신에게의 연결은 `lo`/hairpin을 지나서). 그래서 **check-host.net** 같은 외부 스캐너에게 "공용 인터넷에서 이 포트에 연결되는가?"를 물어본다.
4. **컨트롤 플레인** — k3s/nodeport-guard 서비스 active, monitoring 파드 Running 여부.

정상이면 로그에 `OK: all checks passed` 한 줄만 남기고 조용하다. **문제가 있을 때만 Discord로 알림 1건**을 보낸다. 매일 정상 리포트가 쌓이는 알림은 결국 읽지 않게 되는 법 — 알림 피로를 설계 단계에서 차단한 것이 포인트다.

### 멱등성 검증 — 가장 중요한 실측

playbook이 "코드로 된 상태"라는 말은 **재실행해도 아무것도 바뀌지 않아야** 증명된다. 실제로 두 번 연속 실행했다:

```
$ ansible-playbook site.yml --skip-tags manual-step   # 1차
PLAY RECAP ... changed=0 failed=0
$ ansible-playbook site.yml --skip-tags manual-step   # 2차
PLAY RECAP ... changed=0 failed=0
```

`changed=0`이 나오려면 몇 가지 함정을 고쳐야 했다. 실전에서 배운 것들:

- **`changed_when: true`를 습관적으로 붙이지 마라.** iptables 룰 영속화 태스크에 붙였다가 매 실행마다 "변경됨"으로 판정해 k3s 재시작까지 유발했다. Ansible은 이미 정확한 판정기를 내장하고 있다.
- **helm upgrade는 상태 확인 후에.** 매번 `helm upgrade`를 돌면 항상 changed가 된다. `helm get metadata -o json`으로 릴리스 버전을 읽어, 누락되었거나 버전이 다를 때만 업그레이드한다.
- **"처음 한 번만" 로직에는 실행 흔적을 스탬프로.** 검사 첫 실행은 `/var/log/weekly-audit.log` 존재 여부로 판단한다.
- **`--check` 모드의 한계**: 유닛 파일을 같은 플레이가 만드는 경우, check 모드는 파일을 쓰지 않으므로 `systemd` enable 태스크가 실패한다. 이런 플레이는 실제 실행으로 검증한다.

이 "주기적 재실행으로 멱등성이 깨지지 않았는지 확인"하는 습관은, 재부팅 후 설정이 다시 적용되지 않는 사건들과 세트로 다룰 가치가 있다. 설정은 저장되지 않고, 검증은 반복되지 않는다면 — 그 사이에 운영자는 기만당하고 있는 것이다.

---

## 4. 앞으로의 활용 기대

- **서버 이사 = 명령 3개**: git clone → secrets 주입 → `ansible-playbook site.yml --skip-tags manual-step`. 새 서버에서 tailnet 인증(`manual-step`)만 대화형으로 하면 나머지는 전부 코드가 결정한다.
- **AWS 확장 시**: inventory에 호스트를 추가하고 role을 그대로 적용하면 동일 구성이 복제된다. multi-node k3s가 필요해지면 role에 worker 조인 로직만 추가하면 된다.
- **CI와의 결합**: 지금은 주간검사가 서버 내부 systemd timer다. 다음 단계로는 외부에서(예: GitHub Actions) playbook을 주기 실행해 서버 밖에서도 drift를 확인하는 이중화를 생각 중이다 — 이 글의 idempotency 실측이 그때의 안전망이 된다.
- **보안 검사의 확장**: 현재는 포트·컨테이너·파드지만, 같은 틀에 인증 로그 감사(외부 로그인 시도 0건 검증) 같은 항목도 쌓을 수 있다.

운영 중인 서버에 IaC를 도입하는 작업의 대부분은 코딩이 아니라 **"지금 서버가 실제로 어떤 상태인지를 정직하게 코드로 옮기는 것"**이었다. 그리고 옮겨진 코드가 정말 아무것도 바꾸지 않는지(`changed=0`) 두 번 확인하는 것. 이 두 가지가 끝나면, 서버는 더 이상 기억이 아니라 코드로 존재하게 된다.

