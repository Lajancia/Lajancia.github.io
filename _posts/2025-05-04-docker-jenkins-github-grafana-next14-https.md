---
title: "Docker Jenkins Github Grafana Next14 HTTPS 설정"
date: 2025-05-04 08:00:00 +0900
categories: [개발]
tags: [Docker, Jenkins, Grafana, Next14, HTTPS, SSL, Nginx, Certbot]
author: L.J
---

## Docker + Jenkins + Github + Grafana + Next14

### **HTTPS 설정하기**

여태 http 프로토콜로 통신하는 방식을 통해 포트폴리오를 운영해왔다. 문제는 이제 대부분의 경우 최신 브라우저에서는 더 이상 http 프로토콜을 안전하지 않은 것으로 판단하여 사이트 접근 전에 경고를 주거나 접근을 막는 경우가 허다하다는 것이었다. 사용중인 hostinger에서 이러한 https 프로토콜 지원에 대해서도 구독이 가능한 것으로 알고 있지만, 더 이상 도메인 비용과 서버 비용 이외에 추가적인 납부하고 싶지 않았기 때문에 무료로 HTTPS 인증을 받을 수 없을까 고민하던 중, Let's encrypt에서 지원하는 무료 SSL 인증서를 찾게 되었다. 드디어 내 포트폴리오도 무료로 HTTPS 통신이 가능해질 단서를 발견한 것이다!

### **Docker Nginx + SSL**

기존에 필자의 포트폴리오는 Nginx를 통해서 접근이 가능하게 되어 있었다. 최초로 통신이 요청될 경우에 직접적으로 TCP/IP 통신이 아닌 SSL(TLS) 통신을 통해서 인증서를 확인하게 되는데, 이 과정에서 cerbot을 사용하면 이 인증서를 발급하는 과정에서 인증 문자를 nginx에 설정하여 정상적으로 인증되도록 할 수 있다.

#### **Let's encrypt 인증서 받기**

먼저, 현재 nginx가 켜져 있다면 내려주어야 한다.

```bash
systemctl status nginx.service
systemctl stop nginx.service
```

nginx가 내려가 있다면 이제 다음을 설치해야 한다. 참고로 아래의 update 중에 docker container가 모두 내려가는 일이 있었다. 업데이트 후에 다시 올려주도록 하자.

```bash
sudo apt update
sudo apt upgrade -y
sudo apt-get install letsencrypt -y
sudo apt install certbot python3-certbot-nginx
```

인증서는 다음 명령어로 받을 수 있다.

```bash
sudo letsencrypt certonly -a standalone -d <도메인네임>
```

필자의 경우, 도메인 네임에 soominlab.com으로 작성했다. 여기까지 수행하면 **/etc/letsencrypt/live/도메인**경로에 파일들이 생성된다.

#### **nginx-prod.conf**

기존에 etc/nginx/nginx-prod.conf에 작성했던 nginx 세팅을 수정해야 한다.

```nginx
events {
  worker_connections 1024;
}
http {
  upstream next-app {
    server next-r3f:3000;
  }
  server {
    listen 80;
    server_name soominlab.com;
    location /.well-known/acme-challenge/ {
      allow all;
      root /var/www/certbot;
    }
    location / {
      return 301 https://$host$request_uri;
    }
  }
  server {
    listen 443 ssl;
    server_name soominlab.com;
    ssl_certificate /etc/letsencrypt/live/soominlab.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/soominlab.com/privkey.pem;
    location / {
      proxy_pass http://next-app;
      proxy_set_header Host $http_host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
  }
}
```

여기서 /.well-known/acme-challenge/ 이 부분을 추가해야 하며, ssl_certificate에서 기존에 인증을 받아온 경로와 연결해준다.

#### **docker-compose.yml**

```yaml
networks:
  my-network:
    external: true
services:
  nginx:
    image: nginx:latest
    restart: unless-stopped
    networks:
      - my-network
    volumes:
      - ./nginx-prod.conf:/etc/nginx/nginx.conf
      - /etc/letsencrypt:/etc/letsencrypt
      - /var/www/certbot:/var/www/certbot
    ports:
      - 80:80
      - 443:443
    command: "/bin/sh -c 'while :; do sleep 6h & wait $${!}; nginx -s reload; done & nginx -g "daemon off;"'"
  certbot:
    image: certbot/certbot
    restart: unless-stopped
    networks:
      - my-network
    volumes:
      - /etc/letsencrypt:/etc/letsencrypt
      - /var/www/certbot:/var/www/certbot
    entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"
```

volumes는 copy 형태가 아닌, 직접적으로 해당 폴더와 연결하는 방식이기 때문에 실시간으로 동일한 파일을 공유하여 사용한다. docker에서 제공하는 cerbot으로 인증서가 만료될 경우 자동으로 갱신할 수 있도록 설정하였다. 추가적으로 다른 앱들과 네트워크를 공유할 수 있도록 my-network 설정도 추가해주었다.

이제 해당 nginx를 다시 올려주자.

```bash
docker compose up -d
```

정상적으로 잘 동작한다면 이제 https로 도메인에 접근할 수 있다.

<https://soominlab.com/en>