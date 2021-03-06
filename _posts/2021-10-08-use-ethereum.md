---
layout: post
title: "2. 이더리움 시작하기"
date: 2021-10-08
excerpt: "이더리움을 이용하기 위한 트러플 사용"
Blockchain: true
project: true
tag:
  - 이더리움
  - 블록체인
  - 트러플
---

# 2. 이더리움 시작하기-2

---

## truffle setting

```python
truffle init
```

트러플 초기 세팅을 하는 방법이다.

## 생성된 파일

- contracts → lottery 생성할 파일
- migrations
- test - solidity에서 자체 테스트와 js를 활용한 테스트 이용 가능
- truffle-config.js

## truffle 컴파일

```python
truffle compile
```

- 트러플을 컴파일할 수 있다.
- 컴파일 시, 폴더에 build 폴더가 추가적으로 생성된다.

## migrations 파일 추가

```python
const Lottery = artifacts.require("Lottry");

module.exports = function (deployer) {
  deployer.deploy(Lottery);
};
```

- 기존의 initial_migration.js 파일 코드를 변형
- 서버를 돌리기 위해서 먼저 깔아둔 ganache-cli를 사용해야 함

## Ganache-cli

```python
ganache-cli -d -m tutorial
```

- 위의 명령어를 사용하여 계정과 키 등을 알 수 있음

## 트러플 마이그레이션 등록

truffle-config.js

```python
development: {
     host: "127.0.0.1",     // Localhost (default: none)
     port: 8545,            // Standard Ethereum port (default: none)
     network_id: "*",       // Any network (default: none)
    },
```

위의 development 부분의 주석을 해제한다.

## 트러플 마이그레이트

```python
truffle migrate
```

- 위의 명령어를 실행시키면 migrations 파일에 등록되어있는 파일들에서 사용되는 계정과 가스 프라이스 등을 확인할 수 있다.
- 다시 migrate 할 때는 truffle migrate —reset을 사용하도록 한다.

> 학습 중에는 어지간하면 기존에 생성된 파일들은 건드리지 않고 새로 만들어서 사용하도록 한다.
