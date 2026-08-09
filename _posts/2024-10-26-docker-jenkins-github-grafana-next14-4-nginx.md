---
title: "4. Docker + Jenkins + Github + Grafana + Next14"
date: 2024-10-26 09:00:00 +0900
categories: [개발, Next.js]
tags: [Next.js, Docker, Jenkins, Github, Nginx, Reverse Proxy, CI/CD]
author: L.J
---

4. Docker + Jenkins + Github + Grafana + Next14

### **시작하기...**

이번에는 Nginx를 도커로 말아 프록시 서버로 동작할 수 있게 할 계획이다. Nginx와 Next14를 1:1로 사용할 예정이기도 하고, 기존에 세팅되어 있는 Nginx는 더 이상 사용하지 않을 것이기 때문에 시스템 상에서 Nginx를 내리고 새로운 Docker Nginx 프록시 서버를 올릴 필요가 있다.

만약 처음부터 할 예정이라면 불필요하겠지만, 필자처럼 이미 Linux 자체에 Nginx를 설치해둔 상태라면 도커 컨테이너로 전환하기 위해 내려두자. Nginx의 상태를 확인하고 내리는 명령어는 아래와 같다.

```
sudo systemctl status nginx
// 끄기
sudo systemctl stop nginx
```

추가로 Jenkinsfile을 수정하기 전에 아래의 network 설정도 같이 진행해준다. 해당 설정을 통해 같은 서버의 도커 컨테이너끼리 네트워크 통신이 가능하게 설정할 수 있다.

```
docker network create my-network
```

### **Jenkinsfile 수정**

현재까지 작성된 jenkinsfile은 이미지 빌드를 수행하는 단계까지만이다. 원래라면 Docker hub에 이미지를 푸시하고 pull 받아서 사용하는 구조를 고려하였으나, 이미 같은 서버를 사용중인데 굳이 Hub에 올리는 단계를 거쳐야 하는가에 대해 고민을 했었고, 결과적으로 우선은 바로 이미지를 컨테이너로 실행시키는 구조를 가지도록 변경을 진행하였다.

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
stage('Run Docker Container') {
steps {
sh 'docker stop next-trello || true'
sh 'docker rm next-trello || true'
sh 'docker run -d --name next-trello --network my-network next14-trello'
}
}
// stage...
}
}
```

기존에 이미 돌아가는 컨테이너가 있을 경우, 이를 멈추고 컨테이너를 삭제한 뒤 새로운 이미지로 컨테이너를 실행시켜야 한다. 때문에 해당 두 가지 작업을 선행적으로 진행하고, 다시 next-trello 도커 이미지를 컨테이너로 실행시키는 코드를 작성한다. 이제 깃허브 main에 push를 실행하면 젠킨스는 빌드된 도커 이미지를 기반으로 컨테이너를 실행시킨다.

제대로 진행되었다면 위와 같이 성공한다!

하지만 당연하게도 이렇게 실행된 애플리케이션은 현재 접근이 불가하다. 도커는 기본적으로 맵핑된 포트 이외에는 접근을 허용하지 않기 때문이다. 그래서 next-trello 컨테이너 내부에서는 실제로 3000번 포트가 돌고 있지만, 외부에서는 접근이 되지 않는 상태이다.

다시 초기에 우리가 설정하려 했던 것을 생각해보면, 우리는 Nginx를 리버스 프록시로 두고 DNS와 디폴트로 설정되어 있는 80번 포트에 해당 3000번 next-trello 프로젝트를 서빙하게 하려 한다. 여기서 잊으면 안되는 점은, 외부 3000번 포트는 이미 grafana에서 먹고 있다는 사실이다. 자. 그럼 우린 외부 3000번 포트를 사용할 수 없으니 도커 컨테이너 내부의 3000번 포트를 Nginx 80번 포트에서 이를 리버스 프록시로 담당하게 해야 한다. 조금 복잡하지만 정리하면 다음과 같다.

### **Docker Nginx 세팅**

먼저 nginx가 next-trello 컨테이너에서 실행되고 있는 3000번 포트를 upstream 하여 80 포트로 서빙할 수 있게 하는 conf 스크립트를 작성해야 한다.

```
events {}
http {
upstream next-app {
server next-trello:3000;
}
server {
listen 80;
server_name soominlab.com;
location / {
proxy_pass http://next-app;
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
}
}
}
```

필자의 경우 구분을 위해 /etc/nginx/nginx-prod.conf라는 이름의 파일에 해당 스크립트를 작성하였다. 이때 server next-trello:3000 세팅을 통해 next-trello container의 3000번 포트를 next-app 이라는 하나의 그룹으로 만들어준다. 로드 밸런싱을 위해 사용하는 기능이지만, 현재 필자의 서버가 좋지 못한 상태임으로 우선은 하나만 올려두었다. 여유가 된다면 로드밸런싱까지 활용하여 두 개의 포트에서 운영하게 하는 것도 좋을 것 같다. proxy_pass에서는 80번 포트에서 이 next-app이라는 그룹으로 요청을 보낼 수 있게 해준다.

이제 아래의 명령어를 실행시켜보자.

```
docker run -d --name nginx-trello --network my-network -p 80:80 -v /etc/nginx/nginx-prod.conf:/etc/nginx/nginx.conf:ro nginx
```

제대로 next-trello 가 보이는 것이 확인된다. 물론 msw mocking으로 개발된 부분들에 대해서는 production 환경에서 제대로 동작하지 않을 것이다. 하지만 멀쩡하게 도메인 접속 시 위와 같이 next-trello 프로젝트가 제대로 나타나고 있으니 jenkins와 도커 이미지 빌드 및 실행에 대한 전체적인 환경을 구성하는 작업은 성공적으로 마무리가 되었다.

CI에 대한 자동화가 되었으니, 이제 Next14 프로젝트로 본격적인 웹사이트를 개발해보도록 하려 한다.
