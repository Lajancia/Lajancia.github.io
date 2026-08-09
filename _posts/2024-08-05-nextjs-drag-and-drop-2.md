---
title: "Next.js 시작하기 - Drag and Drop 만들기 (2) MSW + indexedDB"
date: 2024-08-05 12:00:00 +0900
categories: [Next.js]
tags: [Next.js, React, Drag and Drop, MSW, IndexedDB, React Query]
author: L.J
---

### **시작하기...**

지난 시간에 indexedDB에 샘플 데이터가 저장되는 것 까지 세팅을 했다. 오늘은 indexedDB에서 데이터를 가져오는 get API와 데이터를 추가하는 post API 두 개를 만들어 실제로 데이터가 추가되는 것을 해볼 예정이다.

### **indexedDB 데이터를 가져와보자**

먼저, useQuery를 통해 todoList 데이터를 가져오는 get API 호출 함수를 만들어야 한다.

```tsx
import { useQuery } from "react-query";
import axios from "axios";
const todoList = async () => {
const response = await axios.get("/todoList");
return response.data;
};
export const useTodoListQuery = () =>
useQuery({
queryKey: ["todoList"],
queryFn: todoList,
staleTime: Infinity,
onSuccess: (data) => {
console.log("data", data);
},
});
```

staleTime은 해당 get 데이터의 지속 시간을 설정하는 것이다. 우리는 get 호출이 다시 일어나기 전 까지 바뀌지 않도록 Infinity로 설정할 것이다. queryKey는 해당 쿼리의 키 역할로, 키에 변화가 있을 경우 해당 useQuery가 자동으로 호출된다. useQuery는 처음 페이지가 랜더링될 때 자동으로 get을 호출하는 만큼 자동으로 todoList 데이터가 가져와진다.

이제 해당 api를 호출하면 msw에서 mocking하여 indexedDB의 데이터를 가져올 수 있게 세팅해보자.

```tsx
// src/mocks/handlers.ts
import { rest } from "msw";
import { db } from "./db";
export const handlers = [
rest.post("http://localhost:3000/login", async (req, res, ctx) => {
const { id, password } = await req.json();
if (id === "admin" && password === "admin") {
return res(
ctx.delay(5000),
ctx.status(200),
ctx.json({ message: "Login successful" }),
);
} else {
return res(
ctx.delay(5000),
ctx.status(401), // 401 Unauthorized
ctx.json({ message: "Invalid credentials" }),
);
}
}),
rest.get("http://localhost:3000/todoList", async (req, res, ctx) => {
const lists = await db.todoList.toArray();
return res(
ctx.status(200),
ctx.json({
todoList: lists,
}),
);
}),
];
```

기존의 handlers에 추가로 /todoList 의 get에 대한 handler를 작성한다. 이 때 db.todoList.toArray()를 통해서 indexedDB의 todoList 테이블 데이터를 배열로 가져오게 된다.

여기까지 끝났으면, 이제 /dashboard 페이지에서 해당 데이터에 따라 List와 Todo가 나타나도록 한다.

*추가로 card에 대한 title 이름을 지난 시간에 까먹고 넣지 않아, dexie에서 임시 데이터를 세팅할 때 title:string;을 하나 더 추가하여야 한다.*

```tsx
// db.ts
import Dexie, { type EntityTable } from "dexie";
interface Data {
id: number;
title: string;
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

```tsx
// layout.tsx
const checkDatabase = async () => {
try {
await Dexie.exists("TodoListDatabase").then((exists) => {
if (!exists) {
db.todoList.add({ title: "todo Title", todo: ["hello", "world"] });
console.log("Database created");
} else {
console.log("Database already exists");
}
});
} catch (error) {
console.error("Error checking database existence: " + error);
}
};
```

```tsx
"use client";
import React from "react";
import { Card } from "@mui/material";
import TodoListButton from "@/components/TodoListButton";
import { useTodoListQuery } from "@/utils/todoList";
const Dashboard = () => {
const { data, isLoading } = useTodoListQuery();
if (isLoading) {
return <div>Loading...</div>;
}
return (
<div
style={{
display: "flex",
height: "90vh",
alignItems: "flex-start",
overflowX: "auto",
}}
>
{data.todoList.map((list: any) => (
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
<h2 style={{ color: "#eee", marginBottom: "8px" }}>{list.title}</h2>
<div>
{list.todo.map((todo: string) => (
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
{todo}
</div>
))}
<TodoListButton>+ Add Todo</TodoListButton>
</div>
</Card>
))}
<TodoListButton>+ Add List</TodoListButton>
</div>
);
};
export default Dashboard;
```

위와 같이 data 내부에서 map을 통해 각 객체를 꺼내와 나열하게 된다. 여기까지 완료된다면, 화면에는 indexedDB에 저장해뒀던 임시 데이터에 대한 내용이 나타나게 된다.

![](https://blog.kakaocdn.net/dna/ba3iJ8/btsITOW6TTm/AAAAAAAAAAAAAAAAAAAAABTPw_pj6VLHVtsZ6Z1zS5t4pwhg2fSJIzC4ymwtS3Uy/img.png)

이제 ADD TODO를 할 때와 ADD LIST를 할 때 각각 리스트 혹은 todo가 늘어갈 수 있게 post api를 만들고 연결해보자.

#### **버튼 post 기능 연결해보기**

가장 먼저 카드 내부에 데이터를 추가하는 기능을 만들 예정이다. 먼저 useMutation으로 todo를 추가하는 api 호출 react-query 함수를 만들어보자

```tsx
const todo = async (data: { id: number; title: string }) => {
const response = await axios.post("/todo", data);
return response.data;
};
export const useTodoMutation = () => {
const queryClient = useQueryClient();
const todoMutation = useMutation(todo, {
onSuccess: () => {
queryClient.invalidateQueries("todoList");
},
onError: (error) => {
alert(`Error: ${error}`);
},
});
return todoMutation;
};
```

해당 api는 현재 추가되고 있는 카드의 id와 title을 받아 해당 id 카드에 데이터를 추가해 줄 것이다. 이 api를 mocking하기 위한 handler는 다음과 같이 준비한다.

```tsx
rest.post("http://localhost:3000/todo", async (req, res, ctx) => {
const { id, title } = await req.json();
console.log("id", id, "title", title);
try {
const item = await db.todoList.get(id);
if (item) {
item.todo.push(title);
await db.todoList.put(item);
console.log("Item updated successfully");
return res(ctx.json({ message: "Item updated successfully" }));
} else {
throw new Error("Item not found");
}
} catch (error) {
console.error("Failed to update item:", error);
return res(
ctx.status(500),
ctx.json({ message: "Failed to update item" }),
);
}
}),
```

해당 handlers를 통해 api post를 mocking 하여 indexedDB의 todoList 테이블에서 해당 id 가 포함된 객체를 찾아내고, 해당 객체의 todo 배열에 새로운 title 데이터를 추가하는 로직이다.

이제 api 호출이 상호작용 될 수 있게 하였으니, 해당 react-query가 confirm 클릭 시 동작하도록 해야 한다. api는 페이지 단위에서 사용할 것이다.

```tsx
// dashboard/index.tsx
const handleTodoAdd = (_id: number, _title: string) => {
mutate({ id: _id, title: _title });
};
<TodoListButton id={list.id} onClickConfirm={handleTodoAdd}>
+ Add Todo
</TodoListButton>
```

이때, ADD LIST에는 아직 적용하지 않았기에 잠시 주석처리를 해두도록 하자.

TodoListButton에서 해당 데이터를 받을 수 있도록 props를 수정해야 한다.

```tsx
// TodoListButtonProps/index.tsx
interface TodoListButtonProps {
children: React.ReactNode;
id: number;
onClickConfirm: (id: number, title: string) => void;
}
const TodoListButton = ({
children,
id,
onClickConfirm,
}: TodoListButtonProps)
const handleConfirmClick = () => {
onClickConfirm(id, todoName);
setIsAdding(false);
setTodoName("");
};
```

기존의 interface, handleConfirmClick, props를 다음과 같이 수정하여 함수를 사용할 수 있게 해야 한다.

이제 + ADD TODO에 텍스트를 타이핑하고 confirm을 누르면 추가되는 것을 확인할 수 있다. + ADD LIST도 동일한 방법으로 함수를 만들어 추가될 수 있게 하면 된다.

![](https://blog.kakaocdn.net/dna/7S250/btsIWc3cs0q/AAAAAAAAAAAAAAAAAAAAAF__UimNAt97Hm5uQW6SGUNli_Bkt72fruoxPsB2B40z/img.png)

데이터가 추가되고 있다!

다음 시간에는 나머지 List 추가 로직 작성과 카드간 드래그앤드롭으로 위치이동이 될 수 있게 해보자.
