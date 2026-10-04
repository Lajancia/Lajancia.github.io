---
title: "아웃바운드 TCP가 전멸한 날 — 방화벽 룰 한 줄이 만든 하루 종일 장애 추적기"
date: 2026-10-04 21:00:00 +0900
categories: [DevOps]
tags: [K3s, ArgoCD, GitOps, iptables, conntrack, tcpdump, Troubleshooting, Homelab]
author: L.J
---

### **들어가며**

이 글은 "GitHub 접근이 안 된다"는 사소한 증상에서 시작해, 하루 종일을 걸쳐 **iptables 테이블 선택 실수 하나**에 도달한 문제 추적기를 정리한 글이다. 결론부터 말하면:

> **`-t raw` PREROUTING에 걸린 방화벽 룰이 아웃바운드 연결의 응답 패킷까지 드롭하고 있었다. 그리고 raw 테이블은 conntrack보다 먼저 실행되기 때문에, ESTABLISHED 예외 룰은 아무리 추가해도 매칭되지 않았다.**

---

### **증상**

Homelab 서버(Hostinger VPS, Ubuntu 24.04, k3s + ArgoCD)에서 이런 현상이 반복됐다.

- `github.com` 접근 — 134초 대기 후 타임아웃
- IPv4 아웃바운드 TCP **전부** 실패 (8.8.8.8:53 TCP, 1.1.1.1:80/443, github.com:443 …)
- ICMP는 정상 (`ping 8.8.8.8` 성공, RTT 1ms)
- **IPv6는 정상** (Cloudflare 엣지 도달 가능)
- Discord 데스크톱 앱 게이트웨이(TCP 443) 연결 끊김
- k3s NodePort 하드닝용 `nodeport-guard.service` 설치 직후부터 시작

"외부 egress가 막혔다"는 가설로 하루를 보냈다. 서버가 Cloudflare에 IPv4로 못 나가니까, GitHub을 직접 접근 불가한 채로 두고 우회를 설계하는 방향으로 진행했었다.

---

### **잠깐, 우회부터: IPv6 + Cloudflare 프록시**

GitHub은 IPv6를 지원하지 않는다. 그런데 서버에서 IPv6로는 Cloudflare 엣지에 도달할 수 있었다. 그래서 임시 우회로 이런 구조를 만들었다:

```
서버(ArgeCD) --IPv6--> Cloudflare 엣지 (gh.example.com 프록시)
                            ↓ CF 내부 네트워크
                         github.com (CF가 대신 접속)
```

DNS만 보면 `gh.example.com`은 Cloudflare IP(2606:4700::/32)로 리졸브되고, IPv6로 접속하면 0.3초 안에 응답이 왔다. ArgoCD의 repoURL을 이 프록시로 바꾸니 sync가 바로 통했다. 당시에는 "GitHub을 못 쓰니 우회한다"는 이해로 진행했다. **그런데 이 우회의 원인이 우리가 몇 시간 전에 만든 방화벽 룰이라는 사실은 그때 몰랐다.**

---

### **원인 규명: 패킷이 도착하고 있었다**

전환점은 `tcpdump`였다. 8.8.8.8:443으로 연결을 시도하면서 eth0을 캡처했더니:

```
SYN  →  147.93.98.23.44886 > 1.1.1.1.443: Flags [S]
SYN-ACK ←  1.1.1.1.443 > 147.93.98.23.44886: Flags [S.]
SYN  →  (재전송, 계속 반복)
SYN-ACK ←  (계속 도착)
```

**SYN-ACK는 매번 도착하고 있었다.** 그런데 커널은 이를 수신하지 못하는 것처럼 SYN을 계속 재전송했다. 즉, 패킷은 NIC에 도착했지만 네트워크 스택 어딘가에서 사라지고 있었다. 그 "어딘가"는 netfilter, 그리고 카운터가 답을 줬다.

```
$ iptables -t raw -L PREROUTING -nv --line-numbers
1  RETURN  ctstate RELATED,ESTABLISHED   pkts: 0     ← 한 번도 매칭 안 됨
5  DROP   tcp dports !80,443            pkts: 7,793 ↑ ← 계속 증가
```

문제의 룰은 이거였다:

```
iptables -t raw -A PREROUTING -i eth0 -p tcp -m multiport ! --dports 80,443 -j DROP
```

의도는 k3s NodePort(30000+)로 인터넷에서 직접 들어오는 것을 막는 것이었다. 그런데 **PREROUTING은 아웃바운드 연결의 응답 패킷도 통과하는 지점**이다. 서버가 `github.com:443`으로 SYN을 보내면, 응답은 eth0으로 들어오면서 dport가 443이 아니라 **로컬 임시 포트**(예: 44886)가 된다. `! --dports 80,443`에 걸려 DROP. 핸드셰이크 실패. 타임아웃.

왜 ICMP와 IPv6는 살아 있었는지도 이제 설명된다:

- ICMP는 TCP만 드롭하는 룰의 대상이 아님 (참고로 RTT 1ms는 실제 왕복이 아니라 업스트림의 ICMP 인터셉션 응답이었던 것)
- `-t raw` (iptables)는 **IPv4 전용**이라 IPv6는 룰의 영향을 받지 않음

---

### **더 깊은 함정: raw 테이블에서 ctstate 매칭은 안 된다**

처음 수정 시도는 자연스러웠다. DROP 앞에 established 예외를 넣자:

```
iptables -t raw -I PREROUTING -i eth0 -m conntrack --ctstate ESTABLISHED,RELATED -j RETURN
iptables -t raw -A PREROUTING -i eth0 -p tcp -m multiport ! --dports 80,443 -j DROP
```

그런데 여전히 실패. established RETURN의 매칭 카운터는 **영(0)에서 움직이지 않았다.**

이유는 netfilter의 실행 순서에 있다. 패킷은 테이블·우선순위 순서로 처리된다:

```
raw (-300) → conntrack (-200) → mangle (-150) → nat → filter (0)
```

`-t raw`는 **conntrack보다 먼저** 실행된다. SYN-ACK 패킷이 도착한 시점에 raw 테이블이 평가될 때는 커넥션 트래킹이 이 패킷의 상태를 아직 확정하지 못했고, `--ctstate ESTABLISHED` 매칭은 실패했다. 패킷은 예외 룰을 통과하지 못하고 곧장 DROP으로 떨어졌다.

> **교훈 1: conntrack 상태 매칭은 raw 테이블에서 쓰지 마라. mangle 또는 filter에서 쓸 것.**

---

### **최종 수정**

가드 룰 전체를 mangle 테이블로 옮겼다. mangle PREROUTING은 우선순위 -150, 즉 conntrack(-200) **이후**라 ctstate 매칭이 정상 동작한다:

```
# /etc/systemd/system/nodeport-guard.service
ExecStart=... iptables -t mangle -A PREROUTING -i eth0 -s 100.64.0.0/10 -j RETURN
ExecStart=... iptables -t mangle -A PREROUTING -i eth0 -m conntrack --ctstate ESTABLISHED,RELATED -j RETURN
ExecStart=... iptables -t mangle -A PREROUTING -i eth0 -p tcp -m multiport ! --dports 80,443 -j DROP
```

동작 검증:

```
github.com: 200 in 0.073s          ← 몇 주 만에 직접 접근 복구
Discord gateway: 연결 정상          ← 같은 원인이었던 증상도 해소
ArgoCD case-study / next-r3f: Synced
```

mangle RETURN 카운터는 즉시 수백 회 매칭을 기록했고, DROP은 신규 연결에 한해 (5회) 드롭 카운터를 유지했다.

> **교훈 2: 아웃바운드 회신을 드롭해도 "인바운드 차단"의 보안 효과는 유지된다.**
> 보호 대상은 NEW 연결이고, ESTABLISHED 회신 통과는 공격 표면을 넓히지 않는다. SSH 22, k3s API 6443, kubelet 10250 모두 NEW만 들어오면 막힌 채로 유지된다.

---

### **우회 철수**

github.com 직접 접근이 복구되자, Cloudflare 프록시 우회는 원본으로 되돌렸다. ArgoCD Application의 repoURL, repo credential secret, gitops 리포의 yaml — 세 곳을 github.com 기준으로 일치시켰고 Synced + Healthy를 확인했다.

여기서 한 번 더 험했는데, auto-sync가 복구 커밋 **이전**의 옛 yaml을 앱에 재적용해서 앱이 옛 repoURL에 갇혔다가, HEAD·Application·secret을 전부 일치시키고 나서야 self-heal이 멈췄다. GitOps의 자기 참조 구조에서는 "yaml ↔ 클러스터 상태"의 일치가 곧 안정성이다.

---

### **정리: 이번 장애에서 배운 것**

1. **"외부가 막혔다"고 결론 내리기 전에 tcpdump부터** — 패킷이 NIC에 도착하고 있으면 문제는 로컬 netfilter다. SYN-ACK가 캡처에 보이는데 연결이 안 되면 100% 내부 문제다.
2. **iptables 카운터는 답을 들고 있다** — 매칭 카운터 0과 드롭 카운터 증가만으로도 범인 룰을 특정할 수 있다.
3. **테이블 선택은 우선순위 선택이다** — raw(-300)에서 conntrack 매칭은 실패한다. 상태 기반 룰은 mangle(-150)/filter(0)에.
4. **우회를 만들기 전에 원인이 정말 외부인지 확인하라** — 이번 우회(Cloudflare 프록시)는 나쁜 해법은 아니었지만, 원인이 우리가 3시간 전에 설치한 룰이었다. 즉 **우회의 존재 이유 자체가 우리가 만든 버그였던 셈.**
5. **하나의 방화벽 룰이 여러 증상을 만든다** — GitHub 타임아웃, Discord 연결 끊김, CI 실패, ArgoCD sync 실패. 전부 같은 룰에서 나왔다. 증상을 묶어서 보면 원인은 하나다.
