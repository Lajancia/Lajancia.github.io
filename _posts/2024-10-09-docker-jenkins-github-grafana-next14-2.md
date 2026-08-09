---
title: "2. Docker + Jenkins + Github + Grafana + Next14"
date: 2024-10-09 09:00:00 +0900
categories: [개발, Next.js]
tags: [Next.js, Docker, Jenkins, Github, CI/CD, Webhook, Grafana]
author: L.J
---

2. Docker + Jenkins + Github + Grafana + Next14

### **이어서...**

Dockerfile을 작성하고 로컬에서 정상적으로 동작하는 것을 확인했다. 해당 도커 이미지는 서버에 올릴 때도 동일한 컨테이너로 생성되어 동작하기 때문에 로컬과 서버에서 서로 다른 환경 문제로 오류가 발생하는지의 여부를 미리 파악할 수 있다.

이제 Docker를 기반으로 Jenkins가 동작하도록 세팅을 해보도록 하겠다.

### **Docker + Jenkins**

일전에 포스팅했던 Docker 세팅 과정에 Jenkins를 설치하는 방법에 대해서 다뤄두었다. 당시에 세팅해뒀던 Jenkins를 다시 systemctl로 활성화를 시켜보자.

활성화된 jenkins

이때 주의할 점은 jenkins 키를 찾을 때 해당 container에 접근해서 키를 찾아야 한다는 점이다.

<https://soomins.tistory.com/29>

Docker+Jenkins 동작 방식

soomins.tistory.com

이전 포스트에서 작성했던 내용을 참고하면 된다. 정상적으로 세팅이 완료되면 8000번 포트에서 jenkins를 확인할 수 있다. 혹시 이전에 실행되거나 종료된 container를 확인하고 싶다면 아래의 명령어를 통해 확인할 수 있다.

```
docker ps -a
```

Docker에서 jenkins를 실행하는 것 까지 되었으니, 이제 jenkins를 github와 docker hub로 연결해보자.

### **Jenkins + Github 연동**

<https://velog.io/@chrkb1569/CICD-GitHub-Jenkins-Docker%EB%A5%BC-%ED%99%9C%EC%9A%A9%ED%95%9C-CICD-%ED%99%98%EA%B2%BD-%EA%B5%AC%EC%B6%95-4>

<https://velog.io/@rungoat/CICD-Jenkins%EC%99%80-GitHub-%EC%97%B0%EB%8F%99%ED%95%98%EA%B8%B0>

#### **Github Credential**

가장 먼저 깃허브 settings > Developer Settings > personal access token > tokens > generate new token을 통해 아래 세가지를 체크하여 토큰을 생성한다.

생성된 토큰은 다시 볼 수 없으니 미리 저장해두자.

해당 토큰을 Jenkins에서 등록하기 위해서는 Jenkins 관리 > Credentials에 접근해 global을 클릭한 후 Add Credentials를 한다.

필자는 아래와 같이 github token을 세팅했다.

이제 저장해두면 jenkins에서 해당 credentials를 사용할 수 있다.

#### **Github webhook**

여기서 add webhooks를 통해 본인의 젠킨스 주소/github-webhook/을 추가해준다.

#### **Jenkins Pipeline 추가**

dashboard에서 새로운 item을 클릭한 후 pipeline script를 선택하여 url을 입력한다.

이때 Pipeline script from SCM은 깃허브 브랜치에 올라가 있는 Jenkinsfile을 통해 빌드를 수행하는 것을 세팅하는 방법이다. pipeline script를 사용하면 jenkins에 바로 jenkinsfile script를 작성해서 동작시킬 수 있다.

#### **Jenkinsfile 작성**

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
stage('Test') {
steps {
echo 'Testing..'
}
}
// stage...
}
}
```

next-trello 프로젝트 루트 경로에 Jenkinsfile을 생성하여 위와같이 빌드 대상 브랜치명과 경로를 작성한다.

이대로 github에 push를 하게 되면 webhook에서 이를 인지하고 자동으로 빌드를 수행한다. 여기까지 하면 jenkins와 github를 연결하여 자동으로 빌드될 수 있도록 하는 세팅은 완료되었다. 하지만 아마 처음 젠킨스를 세팅하는 사람들은 대부분 익숙한 stage view가 보이지 않는 상태일 것이다.

해당 stage view는 별도로 세팅이 필요하다.

#### **Stage view 세팅**

jenkins 관리 > plugins > Available Plugin > stage view 검색을 통해 Pipeline: stage view를 설치한다. 해당 설치가 완료되면 stage view가 보이게 된다.

Jenkins 파일에 작성했던 stage들이 시각화되어 보이기 시작한다.
