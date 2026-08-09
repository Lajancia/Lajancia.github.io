---
title: "Next.js 시작하기 - 프로젝트 세팅 (1)"
date: 2024-05-19 12:00:00 +0900
categories: [개발]
tags: [Next.js, 프로젝트 세팅, SSR]
author: L.J
---

### **왜 Next.js인가?**

많은 회사에서 요구하는 기술 스택 중 하나는 Next.js 프레임워크였다.

React는 CSR, Next.js는 SSR로 동작한다. CSR은 빈 HTML 파일과 JS를 클라이언트로 보내 클라이언트 측에서 렌더링한다. SSR은 서버 측에서 렌더링을 끝낸 페이지를 클라이언트로 보내준다.

많은 회사에서 SSR로 전환하려는 이유는 **SEO (검색 엔진 최적화)** 때문이다. 검색 엔진 입장에서 CSR은 자바스크립트 실행 후 완성된 HTML을 필요로 하므로 SSR을 더 선호한다.

### **프로젝트 생성**

```bash
npx create-next-app@latest
```

설정이 끝난 후:

```bash
npm run build
npm run start
```

`http://localhost:3000/`에서 확인할 수 있다.

**사용할 라이브러리**: React Hook Form, Zustand, MSW, Jest, MUI, React Query
