---
title: "Docker+Jenkins 동작 방식"
date: 2024-08-11 12:00:00 +0900
categories: [스터디]
tags: [Docker, Jenkins, CI/CD, Ubuntu, DevOps]
author: L.J
---

### **젠킨스에서 도커? 도커에서 젠킨스?**

도커라이징을 하기 위해서는 우선 도커 이미지가 어떻게 배포까지 이어지는지를 이해해야 한다. 해당 동작 과정을 가장 시각적으로 잘 표현한 것 같은 이미지를 참고로 가져왔다.

![](https://blog.kakaocdn.net/dna/bSXmWW/btsI1mrGG18/AAAAAAAAAAAAAAAAAAAAAK8YsFFs1omZybdMod4TmiMAox7P7igPkzDutdDOEPMV/img.png)

도커와 젠킨스의 동작 과정 예시

지금까지는 젠킨스가 직접 빌드와 테스트를 진행하고 빌드된 결과를 AWS에 배포하는 과정으로 동작했다. 하지만 이제 docker를 사용하게 되면서, 해당 배포를 위해 젠킨스가 docker 이미지 빌드 후 hub에 push 하면, push 된 이미지를 도커 허브에서 감지하여 aws에 이미지를 pull 받고 실행시키는 과정으로 변경된 것이다.

물론 jenkins 또한 docker에서 이미지로 pull 받아 실행시킬 수 있다. 이때 기존의 jenkins는 백업을 해두는 것이 중요하다.

필자는 현재 aws를 사용하지 않고 hostinger에서 구입한 VPS 서버를 이용중에 있다. 아마 AWS EC2 우분투 환경을 사용하는 것과 크게 다르지 않을 것이다.

### **linux Ubuntu에서 docker 설치하기**

우리가 로컬에서 docker를 설치해줬던 것 처럼, 이번에는 리눅스에 docker를 설치해줘야 한다. 방법은 간단하다. 아래의 명령어를 통해 설치할 수 있다.

```bash
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg lsb-release
// GPG키
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
//저장소 설정
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
//docker 엔진 설치
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
//docker 동작 확인
sudo docker run hello-world
```

만약 docker 엔진 설치 중 제대로 설치가 되지 않을 경우 다음을 시도해볼 수 있다.

```bash
apt install docker-ce-cli:amd64
docker --version
sudo systemctl status docker
```

여기까지 정상적으로 수행했다면, docker 설치는 끝났다. 이제 docker에서 jenkins 이미지를 pull 받고 설치해줘야 한다.

### **Docker jenkins 설치하기**

jenkins는 기본적으로 8080 포트에서 동작한다. 하지만 필자는 이미 8080 포트를 사용중에 있다. 그렇다면 8000 포트로 접근했을 떄, docker의 8080 포트와 맵핑될 필요가 있다.

```bash
docker run -d -p 8000:8080 -p 50000:50000 --name my-jenkins jenkins/jenkins:lts
```

이재 ip:8000번 포트를 실행하면 jenkins 초기 화면을 확인할 수 있다. 이때 키세팅을 위해 다음을 진행해야 한다.

```bash
sudo docker exec -it my-jenkins
cat /var/jenkins_home/secrets/initialAdminPassword
```

my-jenkins로 jenkins 설치를 진행했기 때문에 동일하게 my-jenkins bash를 실행한다.

이후 아래의 명령어를 실행하면 jenkins 초기 세팅을 위한 키를 알 수 있다. 해당 키를 복사하여 ip:8000에 접속해 키를 붙여넣는다. 제대로 진행되었다면 아래와 같이 결과 화면이 나타난다.

![](https://blog.kakaocdn.net/dna/dbEYm8/btsI04EPuim/AAAAAAAAAAAAAAAAAAAAAGj6VXb1kvPOJL8EGRBUcwmseDQnD-_Ys2QD8pV5rvaA/img.png)

install suggested plugins로 간단한 젠킨스 설치를 진행하면 docker에서 jenkins 설치까지 모두 완료되었다!

이제 우리는 이 jenkins에서 github webhook과 docker로 자동으로 docker를 빌드하고 이미지를 push 하게 설정하면 된다.
