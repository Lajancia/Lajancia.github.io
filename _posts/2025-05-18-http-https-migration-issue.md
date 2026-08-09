---
title: "HTTP -> HTTPS 변경 후 이슈 정리"
date: 2025-05-18 08:00:00 +0900
categories: [개발]
tags: [HTTPS, Docker, Jenkins, SSL, 인증서]
author: L.J
---

## HTTP -> HTTPS 변경 후 이슈 정리

### **Docker Container가 전부 내려갔다**

정확히 말하면 인증서 때문이라고 할 수는 없다. apt 업데이트를 최신으로 하다 보니 벌어진 일이었다. 어쩔 수 없는 일이기는 했다 (언젠가는 해줘야 하는 일이었지만... 버전 업데이트는 늘 신중히...). nginx 쪽과 HTTPS 인증서를 위해 업데이트를 하는 과정에서 기존에 올려두었던 container들이 죄다 네트워크 에러와 함께 내려간 것이었다. 하지만 이 부분은 큰 문제가 아니다. 다시 켜주면 되는 것이니까.

다만 그간 이것 저것 올려둔 컨테이너들이 워낙 많아서 혹시 빠뜨린 것이 없는지가 걱정일 뿐이었다. 다음에는 jenkins로 docker container를 한꺼번에 살릴 수 있도록 파이프라인 하나를 만들어두는 것도 좋을 것 같다.

### **Jenkins에서 Docker container 실행 불가 이슈**

사실 치명적인 건 이 부분이었다. HTTPS에서 다시 next14를 올리는 건 문제가 아니었지만, jenkins에서 잡을 실행할 때 공통적으로 다음과 같은 이슈가 발생하여 컨테이너가 올라오지 않는 문제가 있었다.

```bash
+ docker build --no-cache -t next14-r3f .
ERROR: permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Head "http://%2Fvar%2Frun%2Fdocker.sock/_ping": dial unix /var/run/docker.sock: connect: permission denied
```

permission denied 관련 이슈인 모양이다. HTTPS 인증서 추가 후에 무언가 내부적으로 권한 설정이 리셋되거나 보안이 강화되어 별도의 그룹 설정 없이도 잘 되던 상황에서 막혀버린 모양이었다. 어떤 부분이 문제가 되는지 확인해보니, 다음과 같은 차이가 있었다.

```bash
# jenkins container 내부 그룹
getent group | grep docker # docker:x:1001:jenkins
# 서버 그룹
getent group | grep docker # docker:x:988:root
```

이처럼 jenkins가 group ID 988 그룹이 아닌 1001 그룹에 속해 있어서 발생하고 있었다.

가장 간단한 방법은 docker.sock 자체에 666 권한을 주어 모든 유저가 해당 파일을 마음대로 사용하는 방법이겠지만, 너무 권한을 열어두는 것도 좋지는 않을 것 같아, 다른 방법을 쓰기로 했다. 바로 docker 그룹에 해당 파일의 접근 권한을 주는 것이다. 이 방법을 사용하면 그룹 이외의 사용자는 접근할 수 없다.

```bash
sudo chown root:docker /var/run/docker.sock
sudo chmod 660 /var/run/docker.sock
```

위의 코드를 통해 docker 그룹이 해당 파일을 읽고 쓸 수 있도록 설정할 수 있다. 또한 660을 통해 그룹에 해당하는 경우에 파일을 사용할 수 있도록 권한을 설정하였다. 이후 다시 서버에서 docker 그룹을 확인하니, 정상적으로 1001로 jenkins 내의 docker 그룹 아이디와 일치되게 갱신한 것을 확인할 수 있었다.