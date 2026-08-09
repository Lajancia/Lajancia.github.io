---
title: "3. Docker + Jenkins + Github + Grafana + Next14"
date: 2024-10-22 09:00:00 +0900
categories: [개발, Next.js]
tags: [Next.js, Docker, Jenkins, Github, Grafana, Docker in Docker, CI/CD]
author: L.J
---

3. Docker + Jenkins + Github + Grafana + Next14

### **Jenkins에 Docker 설치 전에...**

Grafana 관련으로 약간의 수정사항이 발생했다. 아무래도 도커 컨테이너를 기반으로 운영을 할 예정이다 보니, Grafana 또한 Docker로 관리하는 것이 좋을 것 같다는 판단이 들었다. 때문에 기존의 Grafana를 제거하고 Docker Grafana를 설치하기로 진행했다.

#### **기존 Grafana 삭제**

```
sudo apt-get remove grafana
sudo apt-get purge grafana
sudo rm -rf /etc/grafana
sudo rm -rf /var/lib/grafana
sudo rm -rf /usr/share/grafana
```

#### **Docker Grafana 설치**

```
docker run -d -p 3000:3000 --name=grafana grafana/grafana-enterprise
```

간단하게 위의 명령어를 실행하는 것 만으로도 잘 동작한다. 나머지 로그인을 하는 방법은 초기에 admin/admin 으로 동일하고 비밀번호를 바꿔줘야 한다.

### **Docker in Docker**

다시 젠킨스로 돌아와보자. 현재 우리는 Docker 기반으로 Jenkins를 동작시키고 있는 상황인데, 문제는 Jenkins 또한 Docker image 빌드를 수행하기 위해 Docker를 사용해야 한다는 것이다. 공식 홈페이지에 따르면 Docker 내부에 다시 Docker를 설치하는 것은 그리 추천하지 않는 세팅이라고 한다. 그렇다면 어떻게 해야 할까?

Docker Image를 빌드할 때, 도커 안에 새로운 도커를 설치하는 대신, 기존에 이미 설치되어 있던 도커를 내부에서도 활용할 수 있게 연결해주는 방법으로 젠킨스에서도 도커를 사용 가능하게 할 수 있다.

먼저 기존의 도커 컨테이너를 멈춰야 한다

```
docker stop <container-id>
```

이제, 공식 젠킨스 이미지를 다시 설치하도록 한다.

```
docker run -p 8000:8080 -v /var/run/docker.sock:/var/run/docker.sock --name jenkins jenkins/jenkins:lts
```

여기서 중요한 것은 docker.sock 부분이다. 해당 경로의 파일에 접근 권한을 부여하고 젠킨스에서 사용할 수 있도록 할 예정이기 때문이다. 필자는 8080 포트를 이미 사용중에 있어 8000 포트로 매핑했다.

이미지를 통해 정상적으로 컨테이너가 실행되었다면, 이제 해당 컨테이너에 접속해야 한다.

```
docker exec -it -u root jenkins bash
// 공식 docker apt 및 docker ce 설치
apt-get update && apt-get -y install apt-transport-https ca-certificates curl gnupg2 software-properties-common && curl -fsSL https://download.docker.com/linux/$(. /etc/os-release; echo "$ID")/gpg > /tmp/dkey; apt-key add /tmp/dkey && add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") $(lsb_release -cs) stable" && apt-get update && apt-get -y install docker-ce
```

위의 명령어로 설치를 진행한 뒤 jenkins 컨테이너에서 권한을 사용할 수 있도록 설정한다.

```
groupadd -f docker
usermod -aG docker jenkins
chown root:docker /var/run/docker.sock
```

위의 세팅을 한 후 jenkins에 접속해서 세팅을 끝낸 후 freestyle project item을 하나 생성한다.

build steps 단계에서 execute shell을 선택하고 docker images 명령어 실행을 통해 jenkins가 정상적으로 docker 명령어를 실행하는지 확인할 수 있다.

저장한 뒤 지금 빌드를 눌러 빌드를 수행하고 성공하면 정상적으로 docker가 적용된 상태이다. 만약 실행 시 정상적으로 동작하지 않을 경우, 권한이 제대로 적용되지 않았을 가능성이 있다. 이럴 경우, 실행중인 컨테이너를 멈추고 다시 실행하면 정상적으로 동작한다.

```
docker stop <container-id>
docker start <container-id>
```

정상적으로 동작할 경우 console 창에서 docker images 명령어 실행 결과를 확인할 수 있다.

### **젠킨스에서 도커 이미지를 빌드해보기**

다시 이전 포스트에서 세팅했던 것과 동일하게 credentials와 item 파이프라인을 다시 생성하고 난 뒤, 이제 중요한 부분은 젠킨스에서 도커 이미지를 빌드하는 과정에서 npm이 정상적으로 설치가 진행되는지다. 먼저 Jenkinsfile을 아래와 같이 수정한다.

```
pipeline {
agent any
stages() {
stage('Checkout') {
steps {
checkout scmGit(branches: [[name: 'main']],
userRemoteConfigs: [[url: 'https://github.com/Lajancia/nextjs-trello.git']])
}
}
stage('Docker Image Build') {
steps {
echo 'Docker building..'
script {
sh 'docker build -t next14-trello .'
}
}
}
// stage...
}
}
```

위의 명령어를 통해 docker를 빌드할 수 있다. 하지만 빌드 실행 도중 아마 npm : not found 에러와 함께 jenkins 빌드 실행이 중단될 것이다. 이 경우 추가적으로 jenkins plugin이 필요하다.

jenkins관리 > plugins > Available Plugins로 들어가 Nodejs 를 검색하고 설치한 뒤 다시 빌드를 실행시켜보자.

빌드 성공!

빌드 자체는 성공했지만 필자의 턱없이 부족한 서버 메모리로 빌드 시간이 꽤 오래 걸리는 것을 확인할 수 있다. prometheus까지 실행시키는 것을 목표로 했지만, 지금의 성능으로는 어림도 없을 듯 싶다. 우선은 Next14 프로젝트를 빌드 및 실행까지 시킨 후 본격적으로 그라파나 대시보드를 작업해야 할 듯 하다.

### **다음 포스트에서는...**

현재는 Nginx의 루트 디렉토리에서 정적 페이지들을 80번 포트로 서빙하게 설정해두었다. 때문에 기존에 설정해뒀던 이 부분을 원래대로 복구하고 upstream을 통해 3000번 포트의 next 프로젝트를 80번으로 로드되도록 설정을 진행할 예정이다.
