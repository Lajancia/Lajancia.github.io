---
title: "Docker와 Standalone"
date: 2024-08-10 12:00:00 +0900
categories: [스터디]
tags: [Docker, Next.js, Standalone, DevOps, Dockerfile]
author: L.J
---

### **제 local에서는 돌아가는데 왜 서버에서는 안될까요....**

난감하기 그지없는 상황이다. 환경도 맞춰줬고, 버전도 맞춰줬고, 해달라는 건 다 해줬는데 대체 왜 배포만 하면 안되는 걸까. 아니, 배포가 잘 되면 왜 내 로컬에서는 또 안되는 걸까. 도대체 나한테 왜 이러는 걸까... 많은 생각이 든다. 컴퓨터는 거짓말을 하지는 않지만 너무 까다로운 동료다. 버전이 1 다르다고 파업을 선언하는 일이 너무 잦다는 생각이 들고, 프로젝트는 점점 복잡해져갈 무렵, 우리는 도커를 생각하지 않을 수가 없다.

### **도커라이제이션, 이제는 해야만 한다**

그랬다. 우리는 이제 피할 수 없는 단 하나의 선택을 해야만 했다. Docker를 사용하는 것. 사실 이전 회사에서 docker로 배포하는 것을 진행한 적이 있었다. 하지만 Docker 라이센스 문제에 직면하여 결국 테스트 적용까지만 하고 실사용까지 이어지지는 못했다. 거의 가물가물 흐물흐물해진 기억을 다시 되살리고, next14에 도커를 적용하기 전, 기본적인 방법에 대한 정리를 해두려 한다.

#### **docker? Standalone?**

도커에 대해서는 지난 시간에 AWS EC2 그리고 Docker에서 다룬 적이 있다.

그렇다면 Standalone은 무엇일까? 간단하게 설명하자면, nextjs에서 사용되는 아웃풋 포맷이며, 웹 어플리케이션을 실행하는 것에 대하여 최소한의 코드만 추출하여 사용하는 모드다. 현재 많은 프로젝트들의 배포 시간을 조금이라도 줄이는 것이 큰 과제인 만큼, Standalone은 이러한 문제를 해결하는 것에 도움이 될 것으로 전망하고 있다.

#### **어떻게 적용할 수 있을까?**

여기, 친절한 설명이 있다.

마침 지금 블로그에 포스팅중인 trello 클론 코딩이 Next14로 개발되고 있다. 해당 프로젝트를 토대로 블로그의 내용을 따라 한번 도커라이징을 해보도록 하겠다.

우선 docker가 설치되어 있지 않다는 가정 하에, mac 환경에서 도커를 세팅하는 것 부터 시작해보자.

<https://docs.docker.com/desktop/install/mac-install/>

먼저 도커 공식 홈페이지에서 Docker Desktop을 설치한다. 설치가 완료된 뒤 터미널에서 docker --version으로 도커가 설치된 것을 확인할 수 있다. 이제 next14를 세팅해보자.

```js
const nextConfig = {
output: "standalone",
reactStrictMode: false,
// other Next.js configurations
};
```

nextConfig에 standalone을 세팅한다. 다음으로는 루트에 dockerfile을 생성하여 기본적인 세팅을 해야 한다. 현재 trello는 npm을 사용하고 있기 때문에 npm install과 npm run dev를 할 수 있도록 한다.

```dockerfile
FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD npm run dev
```

```bash
docker build --tag dockerfile:0.1 .
```

이제 해당 명령어를 실행하면 빌드가 실행된다. 이 때 --tag로 태그를 지정할 수 있다. 마지막 . 은 현재 디렉토리를 가리킨다. 따라서 해당 명령어를 요약하면 현재 디렉토리에 존재하는 docker 파일을 기반으로 dockerfile:0.1 태그로 지정된 이미지를 빌드한다는 것이다. 생성된 이미지는 아래의 명령어로 확인할 수 있다.

```bash
docker images
```

![](https://blog.kakaocdn.net/dna/skBxC/btsI0glO0Ji/AAAAAAAAAAAAAAAAAAAAAAhg-6DM09vx-hPue5q-Kif2oAR_fu43-ZTD4Wk-CZ5U/img.png)

이제 도커로 생성한 이미지를 띄워보자.

```bash
docker run -p 3000:3000 dockerfile:0.1
```

이때 3000:3000은 3000번 포트로 접근 시 도커 컨테이너 3000번 포트로 접근한다는 뜻이다. 만약 3000:8000이라면 3000번 포트로 접근 시 컨테이너 8000번 포트로 매핑된다.

정상적으로 docker가 동작하는 것이 확인되었으니, dockerfile을 다시 수정해보자.

```dockerfile
FROM node:18-alpine as base
RUN apk add --no-cache g++ make py3-pip libc6-compat
WORKDIR /app
COPY package*.json ./
EXPOSE 3000

FROM base as builder
WORKDIR /app
COPY . .
RUN npm run build

FROM base as production
WORKDIR /app
ENV NODE_ENV=production
RUN npm ci
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nextjs -u 1001
USER nextjs
COPY --from=builder --chown=nextjs:nodejs /app/.next ./.next
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./package.json
COPY --from=builder /app/public ./public
CMD npm start

FROM base as dev
ENV NODE_ENV=development
RUN npm install
COPY . .
CMD npm run dev
```

해당 파일은 네가지 부분으로 나눠진다.

```dockerfile
FROM node:18-alpine as base
RUN apk add --no-cache g++ make py3-pip libc6-compat
WORKDIR /app
COPY package*.json ./
EXPOSE 3000
```

먼저 공통적으로 사용되는 세팅이다. linux node 18 버전을 사용하는 것과, 도커 이미지 사이즈를 작게 유지하는데 필요한 g++, make, py3-pip, libc6-compat를 설치한다. 나머지는 일전에 언급했던 것과 같은 세팅이다.

```dockerfile
FROM base as builder
WORKDIR /app
COPY . .
RUN npm run build
```

builder stage일 때 실행할 명령어들이다. 현재 파일들을 /app에 복사하고 build한다.

```dockerfile
FROM base as production
WORKDIR /app
ENV NODE_ENV=production
RUN npm ci
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nextjs -u 1001
USER nextjs
COPY --from=builder --chown=nextjs:nodejs /app/.next ./.next
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./package.json
COPY --from=builder /app/public ./public
CMD npm start
```

production stage일 때 실행되는 내용이다. 이 때 addgroup을 통해 nodejs라는 그룹을 생성하고, addUser를 통해 nextjs라는 유저를 생성하여 그룹 nodejs에 추가한다. USER nextjs를 통해 앞으로 사용되는 명령어는 루트가 아닌 nextjs 사용자로 수행된다. 이는 보안 강화를 위한 목적이다.

아래의 COPY 명령어들은 builder stage 에서 생성된 파일들을 설정된 위치로 복사하는 명령을 수행한다.

이제 루트에 docker-compose.yml을 새로 생성한다.

```yaml
version: '3.8'
services:
app:
image: openai-demo-app
build:
context: ./
target: dev
dockerfile: Dockerfile
volumes:
- .:/app
- /app/node_modules
- /app/.next
ports:
- "3000:3000"
```

이때 image는 이미지의 이름을 지정하는 것이며, target은 어떤 stage를 할 것인지 결정하는 명령어다. production으로 하고 싶다면 target:production으로 지정하면 된다. docker compose를 통해 간단하게 명령어를 실행시킬 수 있다.

```bash
COMPOSE_DOCKER_CLI_BUILD=1 DOCKER_BUILDKIT=1 docker-compose build
docker-compose up
```

첫 번째 명령어로 build가 끝난 뒤 docker-compose up 명령어를 통해 실행시킬 수 있다. localhost:3000으로 접속하면 docker에서 동작하는 trello 클론 코딩 웹사이트를 볼 수 있다.

![](https://blog.kakaocdn.net/dna/b0p33e/btsI0gsvZqC/AAAAAAAAAAAAAAAAAAAAAP969OwJyphjKmdQvFPbPBehsV2q107ImzeHRbkYzFVJ/img.png)

여기까지 standalone 기반 간단한 도커 적용에 대하여 정리해보았다. 기회가 된다면 jenkins와 연결하는 방법도 정리해봐야겠다.
