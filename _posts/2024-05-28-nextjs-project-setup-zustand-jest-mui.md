---
title: "Next.js 시작하기 - 프로젝트 세팅 (3) Zustand/Jest/MUI"
date: 2024-05-28 12:00:00 +0900
categories: [개발]
tags: [Next.js, Zustand, Jest, MUI]
author: L.J
---

### **Zustand 설치**

```bash
npm install zustand
```

```tsx
import { create } from 'zustand'

const useStore = create((set) => ({
  count: 1,
  inc: () => set((state) => ({ count: state.count + 1 })),
}))
```

### **Jest 설치**

```bash
npm install -D jest @types/jest ts-jest
npm i -D babel-jest @babel/core @babel/preset-env
```

```javascript
// jest.config.js
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
};
```

```json
// package.json
"test": "jest"
```

#### **테스트 예시**

```ts
// src/components/sample.ts
const sum = (a: number, b: number) => a + b;
export default sum;
```

```ts
// sample.test.ts
import sum from './sample';

describe('sum', () => {
  test('adds 1 + 2 to equal 3', () => {
    expect(sum(1, 2)).toBe(3);
  });
});
```

### **MUI 설치**

```bash
npm i @mui/material-nextjs @mui/icons-material @emotion/styled @emotion/react @emotion/cache
```
