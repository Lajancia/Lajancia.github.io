---
title: "Next.js 시작하기 - Drag and Drop 만들기 (1) IndexedDB"
date: 2024-08-04 12:00:00 +0900
categories: [Next.js]
tags: [Next.js, React, Drag and Drop, IndexedDB, Dexie, MSW, Trello]
author: L.J
---

### **시작하기...**

Trello 클론 코딩의 가장 메인 기능이라고 할 수 있는 드래그앤드롭 기능을 구현할 차례다. 라이브러리를 사용하는 방법도 있지만, 사용하지 않고 개발을 해볼 예정이다.

그냥 Drag and Drop은 간단하지만, Trello에서 제공하는 Drag and Drop 기능은 생각보다 복잡하다. 각각의 카드별로 드래그앤드롭이 가능한 것 뿐만 아니라 내부에 있는 카드 또한 자유롭게 이동이 가능한 구조로 되어있다.

우선, 간단하게 UI를 만들어보자.

### **Draggable**

```tsx
"use client";
import React, { useState, useEffect } from "react";
import { Card, Button } from "@mui/material";
import { useMutation } from "react-query";
const Dashboard = () => {
return (
<div
style={{
display: "flex",
height: "90vh",
alignItems: "flex-start",
overflowX: "auto",
}}
>
<Card
draggable
style={{
minHeight: "50px",
minWidth: "20vw",
margin: "8px",
padding: "10px",
backgroundColor: "#333",
display: "inline-block",
flexShrink: 0,
}}
>
<h2 style={{ color: "#eee", marginBottom: "8px" }}>todo Title</h2>
<div>
<div
draggable
style={{
userSelect: "none",
padding: "10px",
marginBottom: "10px",
borderRadius: "4px",
backgroundColor: "#eee",
}}
>
Some Text
</div>
<Button
sx={{ mt: 1, flexShrink: 0, minWidth: "20vw" }}
variant="contained"
>
+ Add List
</Button>
</div>
</Card>
</div>
);
};
export default Dashboard;
```

src/app/dashboard 폴더 하위 page.tsx에 위와 같이 간단한 html 코드를 작성한다. 여기서 draggable은 해당 태그와 그 하위 요소들을 드래그하여 이동할 수 있게 하는 기능이다. ~~물론 다시 원래의 자리로 돌아간다.~~

![](https://blog.kakaocdn.net/dna/d4uWdW/btsISWOiQCu/AAAAAAAAAAAAAAAAAAAAAH3EW7_iq1xbTALDhVVLO-RP6fFywxd49fyTIV2cmMbw/img.png)

간단한 UI 세팅

이제 우리가 만들어야 하는 기능이 무엇인지를 한 번 살펴보자.

#### **만들어야 하는 기능들**

1. Add List, Add Todo 버튼 - 검은색 카드들을 추가하는 버튼과 하얀색 카드를 추가하는 버튼을 만들어야 한다. 버튼을 클릭할 때 텍스트가 입력될 수 있어야 하고, 입력된 텍스트들은 어딘가에 저장되어야 한다. react-query로 post를 날리고, 응답에 대해서 다시 state가 업데이트 되는 구조가 되어야 할 것 같다.

2. Card간 드래그앤드롭 후 해당 위치로 변경될 수 있도록 해야 한다.

3. Card 내부에 있는 컨텐츠들이 Card 내부 혹은 외부 위치로 이동될 수 있게 해야 한다.

우선 첫번째 버튼부터 만들어보자.

#### **추가 버튼 만들기**

만들어야 하는 버튼은 두 가지지만, 결국 기능은 같다. 무언가를 '추가' 하거나 취소하는 기능이다. 버튼을 클릭하면 텍스트 창과 save/cancel 버튼이 있으면 좋을 것 같다.

```tsx
import React, { useState, FC } from "react";
import Button from "@mui/material/Button";
import TextField from "@mui/material/TextField";

interface TodoListButtonProps {
children: React.ReactNode;
}

const TodoListButton = ({ children }: TodoListButtonProps) => {
const [isAdding, setIsAdding] = useState(false);
const [todoName, setTodoName] = useState("");

const handleAddClick = () => {
setIsAdding(true);
};

const handleConfirmClick = () => {
setIsAdding(false);
setTodoName("");
};

const handleCancelClick = () => {
setIsAdding(false);
};

if (isAdding) {
return (
<div
style={{
display: "inline-block",
flexShrink: 0,
minWidth: "20vw",
marginTop: "8px",
}}
>
<TextField
sx={{
flexShrink: 0,
minWidth: "20vw",
backgroundColor: "#eee",
borderRadius: "8px",
marginBottom: "8px",
}}
value={todoName}
onChange={(e) => setTodoName(e.target.value)}
/>
<div>
<Button
sx={{ mr: 1, flexShrink: 0 }}
variant="contained"
onClick={handleConfirmClick}
>
Confirm
</Button>
<Button variant="contained" onClick={handleCancelClick}>
Cancel
</Button>
</div>
</div>
);
}

return (
<Button
sx={{ mt: 1, flexShrink: 0, minWidth: "20vw" }}
variant="contained"
onClick={handleAddClick}
>
{children}
</Button>
);
};
export default TodoListButton;
```

components 폴더 하위에 TodoListButton 폴더를 생성하여 위와 같은 index.tsx를 작성하였다. 하나는 카드 내부에, 하나는 카드 외부에 배치하여 각각 다른 용도로 재사용 가능하다.

![](https://blog.kakaocdn.net/dna/p9SzH/btsITWGSbLj/AAAAAAAAAAAAAAAAAAAAAOeDeHwyH2dBzUFYqW82Rms60j44oFaQ5KaJApAytxRf/img.png)

TodoListButton 적용 후

이제 간단한 UI는 갖춰졌다. 이제 카드가 추가되거나 카드 내부에 텍스트가 추가되는 기능을 만들 차례다.

하지만 먼저 고민해봐야 할 문제가 있다. 텍스트를 저장하려면 데이터를 저장할 데이터베이스가 필요하지만, 애석하게도 우리는 디비와 통신할만한 백엔드가 없다. 프론트엔드 개발을 할 때만 테스트 할 용도로 사용할 수 있는 임시 데이터베이스 같은 것이 필요한 순간이다.

#### **indexedDB**

브라우저에서 제공하는 기능 중 하나로 일반적인 storage보다 더 많은 용량을 사용할 수 있다. 테이블 형태가 아닌 객체 형태로 저장되는 특징을 가지고 있다. 이러한 기능과 MSW를 합쳐 confirm 버튼을 누를 때 해당 텍스트가 api post를 호출하고, msw에서 해당 api를 가로채 text를 indexedDB로 저장하게 하는 것이다.

하지만 또 다른 문제가 있다. 카드와 카드 내부의 텍스트를 어떻게 관리할지 고민해봐야 한다. 모든 카드와 내부 텍스트들이 자유롭게 이동 가능하게 하기 위해서는 각각의 위치를 기억하게 할 필요가 있기 때문이다.

간단하게 Card는 List라고 하고, 내부 텍스트는 Todo라고 부르겠다.

```ts
let data = [
{ id: 0, todo: ['a', 'b', 'c'] },
{ id: 1, todo: ['e', 'f', 'g'] },
{ id: 2, todo: ['h', 'i', 'z'] }
];
```

id는 List를 나타내고, 내부 todo는 Todo에 대한 데이터이다. 위치 데이터는 각 배열의 index로 표현할 수 있을 것이다. List의 위치가 변경될 경우, 해당 객체의 index 위치를 변경하여 이동하는 구조로 활용할 예정이다.

#### **세팅 방법**

Next.js에서 indexedDB를 사용하기 위해 몇가지 세팅을 진행해줘야 한다. 이를 위해 사용할 것은 Dexie이다.

```bash
npm install dexie
npm install dexie-react-hooks
```

모듈을 설치한 후, mocks 하위에 db.ts를 생성한다.

```ts
import Dexie, { type EntityTable } from "dexie";

interface Data {
id: number;
todo: string[];
}

const db = new Dexie("TodoListDatabase") as Dexie & {
todoList: EntityTable<
Data,
"id" // primary key "id" (for the typings only)
>;
};

// Schema declaration:
db.version(1).stores({
todoList: "++id, todo", // primary key "id" (for the runtime!)
});

export type { Data };
export { db };
```

여기서 TodoListDatabase는 우리가 사용할 indexedDB의 이름이 될 것이고, ++는 autoincrement를 적용하게 한다. 이제 페이지가 처음 랜더링될 때 TodoListDatabase가 있으면 세팅을 스킵하고, 없으면 임시 데이터를 추가하는 로직을 layout.tsx에 추가해준다.

```tsx
"use client";
import { useEffect } from "react";
import { Metadata } from "next";
import { Inter } from "next/font/google";
import "./globals.css";
import { setupWorker } from "msw";
import { handlers } from "@/mocks/handlers";
import Header from "@/components/Header";
import { QueryClient, QueryClientProvider } from "react-query";
import Dexie from "dexie";
import { db } from "@/mocks/db";

const inter = Inter({ subsets: ["latin"] });

const queryClient = new QueryClient();

const RootLayout = ({ children }: { children: React.ReactNode }) => {
useEffect(() => {
const checkDatabase = async () => {
try {
await Dexie.exists("TodoListDatabase").then((exists) => {
if (!exists) {
db.todoList.add({ todo: ["hello", "world"] });
console.log("Database created");
} else {
console.log("Database already exists");
}
});
} catch (error) {
console.error("Error checking database existence: " + error);
}
};
checkDatabase();
}, []);

// msw mocking
useEffect(() => {
if (typeof window !== "undefined") {
const worker = setupWorker(...handlers);
worker.start();
}
}, []);

return (
<QueryClientProvider client={queryClient}>
<html lang="en">
<body className={inter.className}>
<Header />
<div>{children}</div>
</body>
</html>
</QueryClientProvider>
);
};
export default RootLayout;
```

이제 개발자탭에서 application > storage를 확인해보면 아래와 같이 나타난다.

![](https://blog.kakaocdn.net/dna/dgVmbe/btsIUw2gSZG/AAAAAAAAAAAAAAAAAAAAAFkRg-wlP3adMS3sPIlCI8UY6aLeCvMaDCWfpCf0EpOO/img.png)

하지만 뭔가 이상하다. 두 번이나 저장되는 것이 확인되었다. 다시 console창을 확인해보자.

![](https://blog.kakaocdn.net/dna/b9Z9JX/btsIVFqHCmB/AAAAAAAAAAAAAAAAAAAAANgz3YQ9G1i5CrQLpgEON0xZ_Ky4h7l_eMavv3ZBrSTw/img.png)

위와 같이 두 번이나 랜더링 되는 것이 보였다. 당황할 필요 없다. 이는 Next14에서 자주 발생하는 현상이다. react18을 사용하면서 발생하는 문제 중 하나인데, strict mode로 인해 생명주기가 두 번 발생하여 여러가지 사이드 이팩트를 검사하는 과정을 수행한다. 개발 모드에서만 일어나는 현상이지만 이 경우, 계속해서 두 번 랜더링이 될 테니, strict mode를 꺼주도록 하자. 방법은 간단하다.

```js
/** @type {import('next').NextConfig} */
const withJest = require("next/jest");
const isTest = process.env.NODE_ENV === "test";
const nextConfig = {
reactStrictMode: false,
// other Next.js configurations
};
const jestConfig = {
transform: {
"^.+\\.(ts|tsx)$": ["babel-jest", { presets: ["next/babel"] }],
},
};
module.exports = isTest ? withJest(jestConfig) : nextConfig;
```

기존의 next.config.cjs를 다음과 같이 작성한다. 만약 위의 세팅으로도 두 번 랜더링이 된다면 파일명을 .js 확장자로 변경해야 한다. 해당 코드를 통해 test일 경우와 dev일 경우 별도의 세팅으로 동작할 수 있게 할 수 있다. 세팅을 완료하면 이제 한 번만 랜더링을 하는 것을 확인 가능하다.

![](https://blog.kakaocdn.net/dna/rv3g2/btsIUSRA61Y/AAAAAAAAAAAAAAAAAAAAAE6oTkz-PoK-ovlD0WMQqGgCpjh5vxXQOypTu4Di6Trj/img.png)

어플리케이션의 indexedDB를 확인할 때도 정상적으로 하나의 객체가 저장된 것을 확인 가능하다. 이제 다음 포스트에서 저장된 indexedDB 데이터와 MSW를 연결하여 마치 프론트와 백엔드가 상호작용하는 것과 같이 만들어볼 예정이다.
