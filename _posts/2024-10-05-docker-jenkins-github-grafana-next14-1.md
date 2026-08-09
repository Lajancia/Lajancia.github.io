---
title: "1. Docker + Jenkins + Github + Grafana + Next14"
date: 2024-10-05 09:00:00 +0900
categories: [개발, Next.js]
tags: [Next.js, Docker, Jenkins, Github, Grafana, CI/CD, Standalone, pm2]
author: L.J
---

1. Docker + Jenkins + Github + Grafana + Next14

### **시작하기...**

Next14로 간단한 프로젝트를 만들어 보는 것도 끝났고, 슬슬 기존에 만들었던 Three.js 웹사이트를 새로 리뉴얼할 때가 되었다고 생각이 들었다. 지난 시간에 Docker를 VPS에 설치하여 내부에 Jenkins를 설치하는 것 까지 시도를 했었다. 여태까지 VPS에 직접적으로 Jenkins를 설치해 관리했지만, 이제는 Docker로 운영하게 된 것이다.

그렇다면 기존의 Jenkins에서 새로운 Docker+jenkins로 교체한 만큼 Jenkins 세팅도 변경이 필요해졌다. 깃허브 main 브랜치에 merge가 일어나면 자동으로 Jenkins에서 이를 감지하고 docker image를 빌드하여 push하고, 새로 만들어진 docker 이미지를 pull 해서 운영하는 구조로 설계할 예정이다.

가장 먼저, 이번 세팅은 어디까지나 자동 배포와 모니터링 시스템을 구축하는 것이 목표이기에 Next14는 일전에 진행했던 Next-Trello 클론코딩 프로젝트를 대신 올려서 진행할 예정이다.

### **진행 과정 계획**

#### **Next14+Docker 세팅**

Dockerfile 작성을 통해 기존 next14에서 docker image 생성 시 정상적으로 동작하도록 작성해보도록 한다.

#### **Docker+Jenkins+Github 세팅**

앞서 만든 Next14+Dockerfile 을 기반으로 jenkins에서 Docker 이미지가 빌드될 수 있도록 한다. 해당 Docker 이미지는 docker hub에 push되고, 새로 push된 이미지를 감지해 이미지를 pull 한 뒤 컨테이너를 실행하는 구조로 동작하게 할 것이다.

#### **DNS 접속 시 기본포트 변경**

nginx 프록시 서버를 활용해 soominlab.com에 접속할 경우 next14가 실행되는 port에 디폴트로 연결될 수 있도록 해야 한다. 추후 로드밸런싱을 고려해보는 것도 괜찮을 듯 하다.

### **Next14 + Standalone 적용**

```
// next.config.js
output: "standalone",
```

standalone의 기능을 활용하기 위해서는 node_modules를 사용해야 한다. 때문에 yarn을 사용할 경우 zero-install과 함께 yarn cache를 사용하는 옵션을 활용하기는 어렵다. yarn node_modules 아니면 pnpm으로 대체하는 것도 좋은 방법이다. 지금은 기존에 계속 사용하던 npm을 사용하고, 추후 새로운 프로젝트에서는 pnpm을 사용하려 한다.

npm run build시 생성된다

위의 코드를 삽입하는 것 만으로도 standalone은 모두 세팅이 완료되었다. 실제로 잘 동작하는지 확인하고 싶다면, standalone 폴더에 접근해 아래의 명령어를 실행한다.

```
node server.js
```

standalone 폴더 하위의 server.js를 실행하면 간단하게 동작을 확인할 수 있다. 하지만 당연하게도 이미지가 보이지 않을 것이다. 이는 Standalone이 static 폴더와 public 폴더를 제대로 포함시키지 못해 발생하는 문제다. 이 부분은 Dockerfile을 작성하면서 이미지를 생성할 때 직접 폴더를 해당 위치에 주입할 수 있도록 수정하여 고칠 예정이다.

수동으로 우선 정상적으로 standalone이 동작하는지 확인하고 싶다면, standalone 폴더 하위에 public을 복사해서 넣고, standalone/.next 하위에 static 폴더를 복사해 넣으면 정상적으로 보이는 것을 확인할 수 있다.

#### **Dockerfile**

Docker를 설치하는 방법과 Linux에서 세팅하는 건 일전에 올렸던 블로그에 적어두었다.

<https://soomins.tistory.com/29>

이번에는 Dockerfile의 이미지를 생성할 때 Standalone을 기반으로 이미지를 생성함과 동시에, 안전한 운영을 위해 pm2로 서비스가 실행될 수 있도록 세팅을 해볼 계획이다.

Next-Trello에서 작성했던 Dockerfile을 다음과 같이 변경한다.

```
FROM node:18-alpine as base
RUN apk add --no-cache libc6-compat
FROM base as builder
WORKDIR /app
COPY . .
RUN npm run build
FROM builder as production
WORKDIR /app
RUN npm install --global pm2
RUN addgroup --system --gid 1002 nextgroup
RUN adduser --system --uid 1002 nextuser
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextuser:nextgroup /app/.next/standalone ./
COPY --from=builder --chown=nextuser:nextgroup /app/.next/static ./.next/static
COPY --from=builder --chown=nextuser:nextgroup /app/ecosystem.config.js ./
USER nextuser
ENTRYPOINT ["pm2-runtime", "start","ecosystem.config.js"]
```

ecosystem.config.js는 root 하위에 파일을 생성하여 추가한다.

```
module.exports = {
apps: [
{
name: "next-trello",
script: "npm",
args: "run serve",
exec_mode: "fork",
watch: false,
},
],
};
```

추가로 docker-compose도 현재 3000번 포트를 사용중에 있음으로 3005번에서 서비스 되도록 수정한다.

```
version: "3.8"
services:
app:
image: nextjs-trello-app
build:
context: ./
target: production
dockerfile: dockerfile
ports:
- "3005:3000"
```

이제 docker-compose up --build를 통해 실행하면 next-trello 프로젝트가 localhost:3005번에서 동작하는 것을 확인할 수 있다.

다음 시간에는 세팅한 dockerfile을 기반으로 jenkins와 배포 연결을 해보도록 하겠다.
