---
title: "Next.js 시작하기 - 프로젝트 세팅 (2) React Hook Form / MSW"
date: 2024-05-19 12:00:00 +0900
categories: [개발]
tags: [Next.js, React Hook Form, MSW]
author: L.J
---

### **React Hook Form 설치**

```bash
npm install react-hook-form
```

React Hook Form은 form 태그를 React에서 편리하게 사용할 수 있도록 기능을 제공한다. submit 후 페이지 refresh가 발생하지 않는다.

### **MSW (Mock Service Worker) 설치**

백엔드 API가 개발되지 않은 상태에서 프론트엔드 개발을 가능하게 해준다. Service Worker API를 활용하여 실제 요청을 가로챈다.

**MSW 1버전 사용 (Next.js 14 호환성 이슈)**

```bash
npm install -D msw@1.3.2
```

```typescript
// src/mocks/handlers.ts
import { rest } from "msw";

export const handlers = [
  rest.post("http://localhost:3000/login", (req, res, ctx) => {
    return res(ctx.status(200), ctx.json({ message: "Login successful" }));
  })
];
```

```tsx
// layout.tsx
"use client";
import { useEffect } from "react";
import { setupWorker } from "msw";
import { handlers } from "@/mocks/handlers";

useEffect(() => {
  if (typeof window !== "undefined") {
    const worker = setupWorker(...handlers);
    worker.start();
  }
}, []);
```

```bash
npx msw init public/
```

MSW의 worker는 클라이언트 컴포넌트이므로 use client와 useEffect를 사용하여 동작시킨다.
