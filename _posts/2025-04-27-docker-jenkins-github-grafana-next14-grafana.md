---
title: "Docker Jenkins Github Grafana Next14 Grafana 붙이기"
date: 2025-04-27 08:00:00 +0900
categories: [개발]
tags: [Docker, Jenkins, Grafana, Next14, Prometheus, Node-Exporter, 모니터링]
author: L.J
---

## Docker + Jenkins + Github + Grafana + Next14

### **Grafana 붙이기**

전체적으로 세팅을 끝냈고, 이제 정성 가득한 포트폴리오에 접근하는 유저들의 트래픽과 사용 브라우저 등을 확인하고, 정상적으로 랜더링이 이뤄지고 있는지를 파악할 차례다. 또한 현재 필자가 사용중인 서버의 용량과 메모리 사용량에 대해서도 실시간으로 모니터링하여 지나치게 사용되는 것을 확인할 필요가 있다.

Grafana faro는 특히나 이러한 프론트엔드 애플리케이션 모니터링에 최적화된 기능들을 제공한다. 하지만 먼저 node-exporter를 통해 서버 모니터링을 먼저 세팅할 예정이다. 설치의 편의를 위해 도커 이미지를 통해 grafana를 설치할 예정이다.

#### **Grafana 도커 이미지**

<https://djlee118.tistory.com/844>

#### **Grafana 설치하기**

```bash
docker pull grafana/grafana
```

jenkins와 마찬가지로 docker 기반으로 설치할 예정이다. 도커 이미지로 grafana를 pull 했으니, 3000번 포트에서 grafana를 실행시켜주자.

```bash
docker run -d -p 3000:3000 --name grafana grafana/grafana
```

-d로 백그라운드에서 동작하게 옵션을 줄 수 있다. http://본인도메인:3000 을 통해 접속하면 다음과 같이 화면이 보일 것이다.

![](https://blog.kakaocdn.net/dna/bOpYHA/btsNCA6psff/AAAAAAAAAAAAAAAAAAAAABs8yl99nfZC1K-49Xrp-YJEIEkmaRYMRoXXmL1fxh3F/img.png)

최초 아이디/비밀번호는 admin/admin 이다. 로그인 한 뒤에 바로 비밀번호를 변경해주자.

### **Grafana Prometheus**

Grafana를 설치하고자 하는 이유는 첫 번째로 현재 서버의 상태를 모니터링 하기 위해서이고, 두 번째로는 애플리케이션 상태를 모니터링 하기 위해서이다. 후자의 경우에는 Grafana Faro를 사용할 예정이다. 우선 이번에는 Prometheus부터 세팅해보자.

#### **Prometheus Image**

```bash
# 이미지 받기
docker pull prom/prometheus
# 실행
docker run -d -p 9090:9090 --name prometheus prom/prometheus
```

현재 사용하는 port와 겹치지 않도록 9090으로 설정하였다. 이제 다시 Grafana로 돌아가서 Connections > Data Sources > Prometheus를 추가하자.

![](https://blog.kakaocdn.net/dna/ch9OvM/btsNBusAosG/AAAAAAAAAAAAAAAAAAAAAEV7W-a5CEeExJQypbpRua_HRZInOyW6-xBhT2ED1l2c/img.png)

Connection에 prometheus를 실행시키고 있는 container name을 통해 설정할 수 있는데, 아마 필자의 포스트를 따라 세팅을 진행했을 경우, 처음 시도할 때는 제대로 연결되지 않을 것이다. 이유는 현재 우리는 docker network를 통해 도커 컨테이너 간 네트워크 연결을 제어하고 있기 때문이다.

때문에 prometheus와 grafana 모두 같은 network 하위에서 동작해야 한다. 두 개의 컨테이너 모두 실행중에 있으니, 다음 명령어를 통해 연결해주자.

```bash
docker network connect my-network grafana
docker network connect my-network prometheus
```

위의 방법을 통해 두 개의 컨테이너를 하나의 네트워크 그룹으로 묶어주었다. 이 상태에서 save&test를 하면 정상적으로 실행될 것이다.

#### **Node-exporter 설치**

cpu와 memory를 모니터링 하기 위해, node exporter를 설치해줘야 한다.

```bash
docker run -d --name=node_exporter --network=my-network -p 9100:9100 prom/node-exporter
```

node_exporter를 설치한 뒤, prometheus.yml을 수정한다.

```yaml
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node_exporter:9100']
```

해당 설정이 적용되도록 재시작을 하자.

```bash
docker restart prometheus
```

이제 Dashboard에서 원하는 템플릿을 추가해보자. 필자는 아래의 대시보드를 선택했다.

<https://grafana.com/grafana/dashboards/15172-node-exporter-for-prometheus-dashboard-based-on-11074/>

![](https://blog.kakaocdn.net/dna/cnSKgv/btsNBwjlE5x/AAAAAAAAAAAAAAAAAAAAAKdzzGfgNJf5Kh9YQuLi66lJWzgFYwFklKoMT9TmeRc_/img.png)

node-exporter가 정상적으로 서버의 상태를 가져오는 것을 볼 수 있다!