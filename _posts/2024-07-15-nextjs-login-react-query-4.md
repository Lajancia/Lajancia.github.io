---
title: "Next.js 시작하기 - 간단한 Login 페이지 만들기 (4) React-Query"
date: 2024-07-15 12:00:00 +0900
categories: [Next.js]
tags: [Next.js, React, React Query, Login, MSW, API]
author: L.J
---

### **React Query를 더 관리하기 좋게 만드는 법은 없을까?**

코드를 쪼갤 수 있을 만큼 쪼개고 관리하기 좋은 형태로 리팩토링하는 것은 어쩔 수 없는 프론트엔드 개발자의 고민이다. React-Query는 정말 좋은 기능들을 제공하지만, 많은 기능이 있는 만큼 코드가 굉장히 길어질 수도 있다. 어떻게 보면 React Query 또한 API 호출 함수와 같이 한 곳에 저장해뒀다가 필요할 때 마다 호출한다면 참 좋을 것 같은데, 더 나은 관리 방법은 없는 것일까?

#### **기존의 React-Query를 개선해보자**

동일한 react-query문을 하나의 파일에서 필요할 때 마다 생성해서 사용하는 것은 코드를 길게 만들고 불필요한 반복 작업을 요구한다. 또한 기능이 변경될 필요가 있을 때, 해당 기능이 적용된 페이지마다 작업을 해줘야 하는 불편함이 있다. 때문에 이번에는 지난 시간에 만들었던 React-query문을 utils 폴더 하위로 분리하는 작업을 해볼 예정이다.

```tsx
// utils 하위 login.ts
import axios from "axios";
const loginUser = async (data: { id: string; password: string }) => {
const response = await axios.post("/login", data);
return response.data;
};
export { loginUser };
```

```tsx
// login/page.tsx
const loginMutation = useMutation(loginUser, {
onSuccess: (data) => {
alert(data.message);
},
onError: (error) => {
alert(`Error: ${error}`);
},
}); // POST API일 경우 useMutation
```

현재 위와 같이 api 호출 함수와 react-query가 서로 다른 파일에서 사용되고 있다. 이 두 개의 코드를 하나의 파일에서 관리하고 선택적으로 호출하여 사용할 수 있게 변경을 해보았다.

```tsx
import { useMutation, useQueryClient } from "react-query";
// react-query를 위한 login API
import axios from "axios";
const loginUser = async (data: { id: string; password: string }) => {
const response = await axios.post("/login", data);
return response.data;
};
export const useLoginMutation = () => {
const loginMutation = useMutation(loginUser, {
onSuccess: (data) => {
alert(data.message);
},
onError: (error) => {
alert(`Error: ${error}`);
},
}); // POST API일 경우 useMutation
return loginMutation;
};
```

간단하게 기존의 loginMutation을 함수로 감싼 형태이다. 이러한 형태로 useMutation을 변경하게 되면, 기존의 login 페이지에서는 useLoginMutation을 호출하여 사용하기만 하면 된다.

```tsx
const { isLoading, mutate } = useLoginMutation();
```

코드가 굉장히 짧아졌다. 이제 기존에 loginMutation.mutate(data) 와 같이 호출하던 함수들을 단순히 mutate(data)로도 불러올 수 있게 되었다.

하지만 문제가 있다. 만약 useLoginMutation 이외에 다른 api 호출 함수가 있다면 isLoading과 mutate, isError등의 네이밍이 겹치게 될텐데, 각각의 useQuery 문에서 이러한 네이밍이 겹치지 않도록 할당하는 방법은 없을까?

#### **네이밍을 재지정 해보자**

동일한 이름을 가진 함수를 동시에 사용할 수 없다. 해당 기능이 어떤 것이었는지 헷갈릴 뿐더러, 제대로 동작하지 않는다. 때문에 여러개의 mutation 혹은 query가 있을 경우, 개별적인 네이밍 관리를 통해 사용할 수 있다.

방법은 간단하다.

```tsx
const { isLoading: loginLoading, mutate: loginMutation } = useLoginMutation();
```

다음과 같이, 원하는 이름을 기능명 뒤에 지정하는 것으로 충분하다. 이제 기존의 isLoading과 mutate를 loginLoading과 loginMutation으로 변경하면 이전과 같이 동일하게 동작하게 할 수 있다.

```tsx
// before
const onSubmit: SubmitHandler<Inputs> = (data) => {
loginMutation.mutate(data);
};
// after
const onSubmit: SubmitHandler<Inputs> = (data) => {
loginMutation(data);
};
```

![](https://blog.kakaocdn.net/dna/bq750l/btsIyZSvQkL/AAAAAAAAAAAAAAAAAAAAAIEs281MpgohH2aXeAMxBrkDBXH9qiIWlnQcnnQz_nvU/img.png)

이전과 동일하게 동작한다

~~단순한 변화지만 마음이 편해지는 형태다.~~

### **이제 다음은 Drag & Drop**

Next-Trello 클론 코딩의 가장 중요한 기능 중 하나인 TodoList 드래그 앤 드롭 기능. 일반적인 드래그 앤 드롭보다도 더 복잡하게, 컴포넌트 내부의 컴포넌트 또한 드래그 앤 드롭이 가능한 구조다. 하나까지는 쉽게 드래그 앤 드롭이 가능하게 할 수 있는데, 컴포넌트 내부의 컴포넌트까지 드래그 앤 드롭이 가능하고 순서를 변경할 수 있게 하는 구조는 대체 어떻게 작성할 수 있을까. 그리고 라이브러리 없이 해당 기능을 구현할 수 있을까? 당시에 클론 코딩 프로젝트를 하며 구현했던 기능을 다시 구현해보며 라이브러리 없이 작성한 Drag & Drop 기능을 다시 살펴볼 예정이다.