---
title: "Next.js 시작하기 - 간단한 Login 페이지 만들기 (1) Router"
date: 2024-05-30 12:00:00 +0900
categories: [개발]
tags: [Next.js, Router, App Router]
author: L.J
---

### **Next.js의 라우팅 방식**

Next.js에서는 **자동 라우팅**을 지원한다. 파일 혹은 폴더를 생성하는 것으로 라우팅이 가능하다.

**App Router**: 폴더명 기반. 폴더 하위에 page.tsx/layout.tsx가 존재하면 해당 폴더명으로 라우팅 경로 생성.
**Pages Router**: 폴더와 파일명 기반.

Next.js 공식 홈페이지는 **App Router 사용을 권장**한다.

#### **폴더 구조 예시**

```
src
├── app
│   ├── layout.tsx
│   ├── page.tsx
│   └── login
│       └── page.tsx
├── components
└── mocks
    └── handlers.ts
```

#### **이슈 해결: CommonJS vs ESModule**

next.config.mjs와 jest.config.js에서 호환성 문제 발생 시 **파일 확장자를 .cjs로 변경**하여 해결 가능. .cjs는 해당 파일을 CommonJS로 취급한다.

#### **라우팅 테스트**

```tsx
// src/app/login/page.tsx
const Login = () => {
  return 'hello world';
};
export default Login;
```

`/login` 경로로 접속하여 "hello world"가 표시되면 정상 동작.
