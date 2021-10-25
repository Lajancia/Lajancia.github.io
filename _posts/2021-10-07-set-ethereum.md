---
layout: post
title: "1. 이더리움 시작하기"
date: 2021-10-07
excerpt: "이더리움을 이용하기 위한 초기 세팅 진행. 트러플 이용"
project: true
tag:
  - 이더리움
  - 트러플
  - 블록체인
---

# 1. 이더리움 시작하기-초급

## 기초 정리 참고

[[무료] 이더리움 입문 바이블: 모든 이더리움 입문자를 위하여 - 인프런 | 강의](https://www.inflearn.com/course/ethereum-bible/dashboard)

- 전체적으로 이더리움의 기본적인 지식에 대하여 글로 정리되어 있다.
- 따로 앱을 만드는 방법은 나와있지 않다는 것 주의.
- 정의와 사용에 필요한 개념만 들어있다.

## 이더리움 앱 만들어보기

---

### 이더리움 실전 초보자들을 위한 로터리 Dapp 개발

[[무료] Ethereum 실전! 초보자를 위한 Lottery Dapp 개발 - 인프런 | 강의](https://www.inflearn.com/course/ethereum-dapp/dashboard)

이더리움에 대하여 아무것도 모르는 상태기 때문에 우선 초급반을 먼저 수강하고 중급을 수강할 예정이다. 무료 강의이고, 동영상으로 설명되어있으며, 참고할 깃허브가 존재한다. 처음 앱은 위의 동영상을 토대로 앱 개발을 따라갈 예정이다.

## 기본 환경 설정

---

> mac os 기준

1. node.js→설치됨
2. vscode→설치됨
3. truffle
4. ganashe-cli
5. vscode - solidity extension
6. metamask

### 1. truffle 설치

강의에서 5.0.3 버전을 설치해서 동일하게 해준다.

```python
sudo npm install truffle@5.0.3 -g
```

- 언제부터인지는 잘 모르겠지만 global로 npm을 설치하려면 이제 sudo로 설치해야 한다.

### 그럼 그렇지

당연하지만 설치부터 문제가 생겼다.

```
npm WARN deprecated mkdirp@0.5.1: Legacy versions of mkdirp are no longer supported. Please update to mkdirp 1.x. (Note that the API surface has changed to use Promises in 1.x.)
```

- 이거 저거 개발해본다고 깔아둔게 많은데 버전이 충돌한 모양이다.

### 해결

```python
sudo npm install -g truffle
truffle version

#Truffle v5.4.13 (core: 5.4.13)
#Solidity v0.5.16 (solc-js)
#Node v14.16.1
#Web3.js v1.5.3
```

버전 문제였다. 딱히 해결방법이 나와있지 않아서 최신을 까는게 답이었다.

> 깔아둔게 많아서 그런지 WARN이 아주 많이 떴지만 버전은 제대로 체크되었다.

### 2. ganache-cli 설치하기

~~살바토레 가나치인줄~~

간단하게 블록체인을 테스트할 수 있도록 도와주는 것이라고 한다.

```python
sudo npm install ganache-cli -g
ganache-cli version
```

다행이 문제없이 설치되었다.

### 3. vscode-solidity extension

- vscode market에서 solidity 검색→ 가장 상단에 있는 것 다운로드

### 4. metamask 설치

- 지갑 역할을 해주는 툴이라고 한다.
- 크롬 확장 기능이기 때문에 웨일에서도 잘 설치가 되었다. (웨일은 크롬 기반 브라우저)
