---
title: "기술 면접 준비 - Javascript 기본 (1)"
date: 2024-06-16 12:00:00 +0900
categories: [개발]
tags: [JavaScript, 기술 면접, 프론트엔드]
author: L.J
---

### **자바스크립트란?**

자바스크립트는 클라이언트 사이드에서 동작하는 **싱글스레드 프로그래밍 언어**이며 **인터프리터 방식**으로 동작한다.

싱글 스레드이지만 오래 걸리는 작업들은 브라우저 내부의 Web APIs에서 비동기+논블로킹으로 처리할 수 있다.

### **이벤트 루프**

이벤트 루프는 call stack, callback queue, Web APIs 등을 모니터링하며 비동기 작업들을 순서대로 처리한다.

Callback Queue 처리 순서: microtask queue → macrotask queue

### **비동기 처리**

**Promise**: 비동기 작업이 미래에 완료되거나 실패할 것이라는 약속

```javascript
const promise = new Promise((resolve, reject) => {
  if (condition) resolve('resolved');
  else reject('rejected');
});
```

**Promise.all()**: 모든 프로미스가 이행될 때까지 대기. 하나라도 거부되면 즉시 거부.
**Promise.race()**: 가장 먼저 완료된 프로미스를 기준으로 이행 또는 거부.

**async/await**: async를 붙인 함수는 항상 Promise를 반환. await로 Promise 완료 대기.

### **타입**

| 기본형 | 참조형 |
|--------|--------|
| 숫자 | object |
| 문자 | Date |
| undefined, null | map, set |
| symbol | WeakMap, WeakSet |

#### **Symbol**
ES6에서 추가된 기본형. 생성될 때마다 고유한 값을 가진다.

#### **BigInt**
2^53-1보다 큰 정수를 표현할 때 사용.

#### **WeakMap, WeakSet**
참조하는 식별자가 없을 경우 메모리에서 자동 삭제. 효율적인 메모리 활용 가능.

### **실행 컨텍스트와 this**

**실행 컨텍스트**: 코드를 실행하는데 필요한 환경정보를 모아둔 객체.
**this**: 실행 컨텍스트의 소유자를 가리킨다. call, apply, bind로 명시적 바인딩 가능.

### **클로저 (Closure)**

함수가 선언된 환경의 Lexical Scope를 기억하여 함수가 스코프 밖에서 실행될 때에도 이 스코프에 접근할 수 있게 한다.

**디바운스 (Debounce)**: 짧은 시간 동안 동일 이벤트가 많이 발생하면 마지막 한 번만 처리
**스로틀 (Throttle)**: 이벤트를 일정 주기마다 발생시키는 기술

### **호이스팅 (Hoisting)**

변수 식별자와 함수 선언을 코드의 가장 위로 끌어올리는 현상. 실행 컨텍스트의 환경 레코드로 인해 발생한다.

### **var와 let**

var는 함수 범위, 호이스팅 발생. let은 블록 범위, TDZ 영향으로 선언 전 사용 시 에러.

### **프로토타입 (Prototype)**

자바스크립트는 프로토타입 기반 객체 지향 언어. 프로토타입 체인을 따라 상위 객체의 프로퍼티를 순차적으로 검색한다.

### **이터레이터와 제너레이터**

**iterator()**: 반복을 위한 인터페이스. value와 done을 반환하는 next() 메서드를 가진다.
**generator()**: 실행을 일시 중지하고 재개할 수 있는 함수. function* 키워드와 yield 사용.

### **이벤트 델리게이션 (Event Delegation)**

상위 부모 요소에서 하위 자식 요소의 이벤트를 위임하여 처리하는 방식.

```javascript
const ul = document.querySelector("ul");
ul.addEventListener("click", (event) => {
  if (event.target.tagName == "LI") {
    event.target.classList.add("selected");
  }
});
```
