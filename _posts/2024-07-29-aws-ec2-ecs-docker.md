---
title: "AWS EC2와 ECS 그리고 Docker"
date: 2024-07-29 12:00:00 +0900
categories: [스터디]
tags: [AWS, EC2, ECS, Docker, Cloud, DevOps]
author: L.J
---

### **AWS, 아는 만큼 보인다**

전 직장에서는 클라우드 서비스를 사용하지 않았다. 언제 샀는지 알 수 없고, 더 이상 부품조차 구할 수 없는 오래된 물리 서버로 모든 것을 해결했다. 하지만 이제 AWS를 사용하는 곳으로 왔으니, 다시 클라우드 서비스의 대표 AWS의 개념을 오랜 기억속에서 꺼낼 때가 되었다.

### **ECS, EC2 뭐가 다른가?**

최근 일을 하면서 자주 접하는 단어들이다. 둘 다 클라우드 서비스라는 것은 알겠는데, 정확히 무슨 차이가 있는지 잘 알지 못해서 이번 기회에 정리를 해보려고 한다.

#### **EC2**

![](https://blog.kakaocdn.net/dna/cgVtDR/btsIO4eVlRj/AAAAAAAAAAAAAAAAAAAAAPdx7_et3MwMqLhKwLtyXEhWt7__4YJ1s68LPfkM8Y8b/img.png)

AWS EC2 참고 이미지

아마 AWS를 입문하는 사람들, 특히 학생일 경우 가장 많이 사용하는 서비스일 것이다. 첫 가입일 경우 1년간 무료 크레딧을 제공해주기 때문에 학습을 하기에도 매우 용이하기 때문이다. 간단하게 물리 서버가 가상으로 제공되어 유연하게 확장하고 축소할 수 있는 서비스이다. 아주 기본적인 클라우드 서비스라고 볼 수 있으며, 이외의 권한들은 수동으로 설정해줘야 한다.

#### **ECS (Elastic Container Service)**

![](https://blog.kakaocdn.net/dna/6SRMU/btsIQvbflaN/AAAAAAAAAAAAAAAAAAAAAB0mmas4yojQ9vlzVNcArPIfmaJWHz1peTqvTnsF0v6a/img.png)

AWS ECS 참고 이미지

EC2는 알겠지만, ECS가 뭔지는 잘 몰랐다. 대충 EC2보다 더 좋겠거니라는 생각을 했지만, 막상 두 가지는 서로 다른 개념으로 보아도 무방했다. 뭐가 더 좋다 나쁘다의 개념이 아니라, ECS는 하나의 종합패키지의 개념에 더 가깝다고도 볼 수 있다. 배포와 관리를 더 용이하게 한 서비스인 것이다.

ECS는 크게 세 가지 계층으로 나눠진다.

1. **Cluster** - 컨테이너를 실행하는 인프라의 논리적 그룹
2. **Task Definition** - 컨테이너를 실행하기 위한 설정 (Docker 이미지, 포트, 환경 변수 등)
3. **Service** - Task Definition을 기반으로 컨테이너를 실행하고 유지 관리

Capacity Option 중 선택지로 EC2를 옵션으로도 선택할 수 있지만 이외에도 Fargate와 외부 가상 머신 또는 서버를 클러스터에 등록하여 사용할 수도 있다.

### **ECS와 Docker**

기존의 EC2에서 Docker로 배포하는 것 또한 가능하지만, 직접 컨테이너를 관리하고 실행해야 하는 문제가 있다. 하지만 ECS는 다음과 같은 과정으로 실행된다.

ECS에서 도커를 사용하면 해당 배포와 관리를 AWS에서 담당하기 때문에 더 편리하게 컨테이너를 사용할 수 있다. 어떻게 보면 ECS는 kubernetes와 유사한 서비스를 제공한다고도 볼 수 있다.

### **정리하자면...**

| 구분 | EC2 | ECS |
|------|-----|-----|
| 개념 | 가상 서버 | 컨테이너 오케스트레이션 |
| 관리 수준 | 수동 (OS, 미들웨어 등) | AWS 관리형 |
| 확장 | 수동 또는 Auto Scaling | 자동 |
| 사용 사례 | 기존 애플리케이션, 커스텀 환경 | 컨테이너 기반 애플리케이션 |
| Docker | 직접 설치 및 관리 | 기본 지원 |

### **추후 참고하면 좋을 것 같은 자료들**

[Hands On] CI/CD – Jenkins pipeline을 이용한 ECS 배포 - NDS Cloud Tech Blog

자동화 배포 및 관리를 하기 위해 사용해야 할 것들은 다음과 같다.

- **GitHub** - 소스 코드 저장 및 버전 관리
- **Jenkins** - CI/CD 파이프라인 구축
- **Docker** - 애플리케이션 컨테이너화
- **Docker Hub / ECR** - Docker 이미지 저장소
- **AWS ECS / EC2** - 배포 대상

~~많기도 하다...~~ 간단하게 서로 어떤 순서와 관계로 동작하는지 알아둔다면 추후 docker 세팅을 할 때 도움이 되지 않을까 하며 여기 기록해둔다.
