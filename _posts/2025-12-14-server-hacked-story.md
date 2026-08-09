---
title: "서버 해킹 당한 썰"
date: 2025-12-14 18:00:00 +0900
categories: [개발]
tags: [보안, 서버, 악성코드, 해킹, 코인 채굴]
author: L.J
---

### 해킹은 해킹인데 하는게 아니라 당한쪽

얼마 전 갑작스럽게 운영중인 서버가 아예 중단되는 사건이 있었다. 클라우드 서버를 호스팅하여 사용중이라 hostinger dashboard로 향했다. 그런데 악성코드가 확인되어 서버를 긴급 셧다운 했다는 것이었다.

### 우리집에 왜 왔니...

```bash
# 로그 조회
awk '/2025-11-18/,/2025-12-11/' /var/lib/docker/containers/*/*-json.log | grep -Ei "wget|curl|runnv"
# 파일 확인
file /var/lib/docker/overlay2/.../runnv/runnv
# 해시값 계산
sha256sum /var/lib/docker/overlay2/.../runnv/runnv
# 문자열 추출
strings /var/lib/docker/overlay2/.../runnv/runnv | less
# 실행 파일 헤더 확인
readelf -h /var/lib/docker/overlay2/.../runnv/runnv
```

악성코드 판정. 우선 격리했다.

```bash
chmod 000 파일 경로
mkdir -p /root/quarantine
mv /var/lib/docker/overlay2/.../runnv/runnv /root/quarantine/
mv /var/lib/docker/overlay2/.../runnv/config.json /root/quarantine/
```

내부 파일을 확인해보니 **코인 채굴 스크립트**였다.

해당 로그가 있었던 container는 테스트용으로 올려두고 내리는 걸 깜빡한 next.js 웹페이지였다. IP와 port가 그대로 노출된 상태에 Nginx조차 없었다.

### 정리

다음 증거들을 토대로, Jenkins에서 취약점이 드러난 plugin을 통해 pm2를 변조한 뒤, 가장 보안이 취약한 웹사이트를 통해 코인 채굴 스크립트를 실행하려 했던 것으로 보인다.

다행히 dockerfile로 빌드하는 애플리케이션에 대해 권한을 따로 두었기 때문에 실패로 끝났다. 하지만 이번 기회에 모든 포트들을 80과 443으로 제한하고, Jenkins와 Grafana를 VPN 뒤로 숨기는 작업이 필요해졌다.