---
title: "Hermes 원격 agent를 로컬 맥북에 remote gateway로 연결하기"
date: 2026-07-11 18:00:00 +0900
categories: [개발]
tags: [Hermes Agent, Tailscale, Remote Gateway, 보안, 네트워크]
author: L.J
---

## 목차

### 이 가이드가 필요한 이유

Hermes Agent는 백엔드 서버로서 지속적으로 실행되도록 설계되었습니다. `hermes serve` 명령어는 Desktop App과 원격 백엔드에 WebSocket/JSON-RPC 방식으로 전원을 공급합니다.

가장 쉬운 방법은 `0.0.0.0`(모든 인터페이스)에 바인드하는 것이지만, 이는 **공인 인터넷, 로컬 LAN, 접속 중인 모든 VPN**에 백엔드를 노출시킵니다. 방화벽 규칙 하나 잘못 설정하거나 백엔드에 제로데이 취약점이 발견되면 전체 Hermes 세션이 노출될 수 있습니다.

이 가이드는 Tailscale(제로 설정 WireGuard 메시 VPN)과 직접 IP 바인딩 및 인증을 결합한 **심층 방어(defense-in-depth)** 접근법을 제시합니다.

### 아키텍처 개요

```
┌─────────────────────────────────────────────────────────────────────┐
│ Tailnet (사설 메시 네트워크)                                        │
│ ┌──────────────────────┐ ┌──────────────────────────────┐          │
│ │ MacBook Pro (클라이언트)│ │ Mac Mini (서버)           │          │
│ │                      │ │                              │          │
│ │ Hermes Desktop App   │ │ hermes serve --host          │          │
│ │                      │ │ <Tailscale IP>:9119          │          │
│ │         ▼            │◄────────────►                  │          │
│ │ Gateway 설정         │   암호화 터널                  │          │
│ │ URL: http://host-    │ │                  ▼           │          │
│ │ name.ts.net:         │ │              Dashboard 인증  │          │
│ │ 9119                 │ │         username: admin      │          │
│ └──────────────────────┘ │         password: (해싱됨)   │          │
│                          └──────────────────────────────┘          │
└─────────────────────────────────────────────────────────────────────┘
❌ 공인 인터넷 ❌ 로컬 LAN (192.168.x.x)
❌ 카페 WiFi ❌ 기타 모든 네트워크
```

**핵심 속성**: Tailscale IP(`100.x.x.x`)는 **Tailscale WireGuard 메시 내에서만** 존재합니다. 다른 어떤 네트워크도 이 IP로 라우팅할 수 없습니다. Hermes가 이 IP에 바인드되면 Tailnet 외부에서는 물리적으로 접근이 불가능합니다.

### 사전 준비사항

| 요구사항 | 비고 |
| --- | --- |
| Tailscale 계정 | 무료 플랜: 최대 3명의 사용자, 100대의 기기 |
| 양쪽 기기에 Tailscale 설치 | macOS: `brew install --cask tailscale` / Linux: curl 스크립트 |
| MagicDNS 활성화 | 설정 → DNS → Enable MagicDNS (Tailscale 관리 콘솔) |
| 두 기기가 같은 tailnet에 로그인 | 각각 `tailscale up` 실행, 같은 계정으로 인증 |
| 서버에 Hermes Agent 설치 | `pip install hermes-agent` 또는 인스톨러 사용 |
| 클라이언트에 Hermes Desktop App | Nous Research에서 제공 |

연결 확인:

```bash
# 양쪽 기기에서 실행
tailscale status
# 예상 출력 - 두 기기가 같은 계정 아래 표시되어야 함
100.x.x.x my-server user@ macOS -
100.y.y.y my-laptop user@ macOS -
```

### 1단계: Tailscale 설치 및 설정

**Tailscale이 아직 설정되지 않은 경우:**

```bash
# macOS (Homebrew)
brew install --cask tailscale
# 앱 실행 (Applications 폴더에서 열고 브라우저로 로그인)
open -a Tailscale
# 또는 CLI 데몬 실행 (헤드리스)
sudo tailscaled --tun=userspace-networking &
tailscale up
```

**MagicDNS 활성화** (호스트명 기반 URL에 필수): `tailnet1234.ts.net`

**DNS 확인:**

```bash
dig <서버-호스트명>.<tailnet-이름>.ts.net +short
# 예상: 100.x.x.x (서버의 Tailscale IP)
```

### 2단계: 현재 상태 파악

변경 전에 현재 실행 중인 상태를 확인합니다:

```bash
# 모든 Hermes 프로세스 찾기
ps aux | grep hermes | grep -v grep
# 모든 인터페이스에서 리스닝 중인 포트 확인
lsof -iTCP -sTCP:LISTEN -P -n | grep "0.0.0.0"
```

다음과 같은 출력이 보일 수 있습니다:

```
python3.1 12345 Ghost 14u IPv4 0x... TCP *:9119 (LISTEN) ❌ 0.0.0.0
python3.1 12345 Ghost 26u IPv4 0x... TCP *:8080 (LISTEN) ❌ 0.0.0.0
```

**둘 다 위험 신호입니다.** 이제부터 고쳐봅시다.

### 3단계: 위험 — 0.0.0.0 바인드

프로세스가 `0.0.0.0`에 바인드되면 **사용 가능한 모든 네트워크 인터페이스**에서 연결을 수락합니다:

```
┌──────────────────────────────────────────┐
│ Mac Mini                                 │
│                                          │
│ ┌─────┐ en0 (WiFi) ─── 192.168.1.100    │
│ │Hermes├── en1 (유선) ──── 10.0.0.5     │
│ │:9119 ├── utun (Tailscale) ─ 100.x.x.x │
│ └─────┘ lo0 (루프백) ── 127.0.0.1       │
│                                          │
│ 바인드: 0.0.0.0 → 모든 인터페이스 ❌     │
└──────────────────────────────────────────┘
```

**위험**: 같은 WiFi(카페, 코워킹 스페이스)에 있는 사람, LAN에 있는 사람, 그리고 라우터에 포트 포워딩이 설정되어 있다면 공인 인터넷상의 누구든 연결을 시도할 수 있습니다.

### 4단계: Tailscale IP에 바인드 (해결)

**명령어:**

```bash
# 서버의 Tailscale IP 확인
TAILSCALE_IP=$(tailscale ip -4)
echo "서버 Tailscale IP: $TAILSCALE_IP"
# 실행 중인 Hermes serve 중지
pkill -f "hermes serve" 2>/dev/null
# Tailscale IP에만 바인드하여 Hermes serve 시작
hermes serve --host $TAILSCALE_IP --port 9119
```

**바인드 확인:**

```bash
lsof -i :9119 -P -n
# 예상: TCP 100.x.x.x:9119 (LISTEN) ✅
```

**launchd로 영구 설정:**

```bash
# launchd plist 생성
cat > ~/Library/LaunchAgents/hermes.serve.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
"http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
<key>Label</key>
<string>hermes.serve</string>
<key>ProgramArguments</key>
<array>
<string>/usr/local/bin/hermes</string>
<string>serve</string>
<string>--host</string>
<string>$TAILSCALE_IP</string>
<string>--port</string>
<string>9119</string>
</array>
<key>RunAtLoad</key>
<true/>
<key>KeepAlive</key>
<true/>
</dict>
</plist>
EOF
launchctl load ~/Library/LaunchAgents/hermes.serve.plist
```

### 5단계: 인증 설정

Tailscale IP에 바인드하는 것은 무단 네트워크 접근을 차단하지만, tailnet 내의 모든 기기는 여전히 포트에 접근할 수 있습니다. 두 번째 방어선으로 **사용자명/비밀번호 인증**을 추가합니다:

```yaml
# ~/.hermes/config.yaml
dashboard:
  basic_auth:
    username: admin
    password: <강력한_비밀번호>
```

**중요:** `password_hash`와 `password`가 모두 설정 파일에 있으면 `password_hash`가 우선시되어 `password`는 무시됩니다.

**강력한 비밀번호 생성:**

```bash
openssl rand -base64 24
```

### 6단계: Desktop App에서 연결

**클라이언트 머신**(MacBook Pro)에서:

| 필드 | 값 |
| --- | --- |
| Connection URL | `http://<서버-호스트명>.<tailnet>.ts.net:9119` |
| Username | `admin` |
| Password | 설정한 비밀번호 |

### 대안 비교

| 접근법 | 보안 | 복잡도 | URL 형식 |
| --- | --- | --- | --- |
| **Tailscale IP 직접** | ✅ 높음 | 낮음 | `http://host.ts.net:9119` |
| **127.0.0.1 + Tailscale Serve** | ✅ 높음 | 중간 | `https://host.ts.net` |
| **127.0.0.1 + SSH 터널** | ✅ 높음 | 높음 | `http://localhost:9119` |
| **0.0.0.0 + 방화벽 규칙** | ❌ 중간 | 높음 | 비추천 |
| **Cloudflare Tunnel** | ✅ 높음 | 중간 | `https://host.domain.com` |

### 결론

Hermes Serve를 `0.0.0.0` 대신 Tailscale IP에 바인드하면:
1. **표면적 공격 축소**: Hermes가 Tailnet 외부에서 완전히 보이지 않음
2. **암호화**: 모든 트래픽이 WireGuard로 암호화
3. **감사 가능성**: Tailscale 로그로 모든 연결 추적 가능
4. **간단함**: 방화벽 규칙이나 추가 소프트웨어 불필요

이 패턴은 데이터베이스, 개발 서버, API 등 Tailscale을 통해 안전하게 노출하려는 모든 서비스에 적용할 수 있습니다.

*Guide by tara. All IPs, passwords, and tokens shown are examples only. Replace with your own values.*