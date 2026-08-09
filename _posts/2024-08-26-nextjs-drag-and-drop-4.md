---
title: "Next.js 시작하기 - Drag and Drop 만들기 (4) Draggable"
date: 2024-08-26 12:00:00 +0900
categories: [Next.js]
tags: [Next.js, React, Drag and Drop, Trello, IndexedDB, MSW]
author: L.J
---

### **지난 시간...**

지난 시간에는 draggable을 활용해서 list 카드 간 이동이 가능하게 만들어보았다. 이번에는 동일하게 todo 카드들이 자유롭게 draggable 할 수 있도록 만드는 방법을 시도해볼 것이다. 방법은 list 카드 이동과 동일한 방식이지만, 서로 다른 list 카드간 이동 또한 고려해야 하는 것이 차이점이다.

#### **todo draggable**

이번에도 우리가 해야 할 일은 draggable에 해당하는 함수들을 만들어주는 것이다. 기존에 handle 함수를 만들었던 것 처럼, 이번에는 todo handle 함수를 만들어보자.

먼저 아래의 state 함수를 만든다. 해당 state들은 list때와 같이 이동 전 위치와 이동 후 위치를 저장하는 용도이다.

```tsx
const [draggingTodoId, setDraggingTodoId] = useState<number | null>(null);
const [dragOverTodoId, setDragOverTodoId] = useState<number | null>(null);
```

다음으로는 handle 함수를 만들어보자.

```tsx
const handleDragTodoStart = (
// 선택한 Todo 카드와 List id
e: React.DragEvent<HTMLDivElement>,
list: number,
id: number,
setDraggingListId: (props: number) => void,
setDraggingTodoId: (props: number) => void,
) => {
e.stopPropagation();
setDraggingListId(list);
setDraggingTodoId(id);
};
const handleDragTodoOver = (
// 이동하고하 하는 위치의 Todo 위치와 List id
e: React.DragEvent<HTMLDivElement>,
list: number,
id: number,
setDragOverListId: (props: number) => void,
setDragOverTodoId: (props: number) => void,
) => {
e.preventDefault();
setDragOverListId(list);
setDragOverTodoId(id);
};
```

```tsx
// 기존의 두 함수는 아래와 같이 변경하자
const handleDragEnd = (
e: React.DragEvent<HTMLDivElement>,
setDraggingTodoId: (props: null) => void,
setDragOverTodoId: (props: null) => void,
setDraggingListId: (props: null) => void,
setDragOverListId: (props: null) => void,
) => {
e.stopPropagation();
setDraggingTodoId(null);
setDragOverTodoId(null);
setDraggingListId(null);
setDragOverListId(null);
};
const handleDropOn = (
e: React.DragEvent<HTMLDivElement>,
prevList: number | null,
currentList: number | null,
prevTodo: number | null,
currentTodo: number | null,
) => {
if (draggingTodoId === null) {
e.preventDefault();
console.log("list", draggingTodoId);
console.log("drag on", prevList, currentList);
listDragMutate({ prev: prevList!, current: currentList! });
} else {
e.preventDefault();
console.log("todo", draggingTodoId);
console.log("drag on", prevList, currentList);
console.log("drag on", prevTodo, currentTodo);
}
};
```

아직 mutation 함수를 만들지 않았기에, handleDropTodoOn에서는 해당 state들이 제대로 우리가 의도한 값을 출력하는지 console을 찍어보도록 했다. 이때 draggingTodoId가 null이 아닐 경우에만 동작하게 하여 list 동작과 겹치지 않도록 조건을 부여하였다. 기존에 만들었던 함수보다 몇 가지가 더 늘어난 것을 확인할 수 있을 것이다.

Todo 카드가 이동할 수 있게 하는 원리는 다음과 같다.

위의 함수들은 아래와 같이 사용된다.

```tsx
{list.todo?.map((todo: string, todoIndex: number) => (
<div
draggable
onDragStart={(e) =>
handleDragTodoStart(
e,
listIndex,
todoIndex,
setDraggingListId,
setDraggingTodoId,
)
}
onDragOver={(e) =>
handleDragTodoOver(
e,
listIndex,
todoIndex,
setDragOverListId,
setDragOverTodoId,
)
}
onDrop={(e) =>
handleDropTodoOn(
e,
draggingListId,
dragOverListId,
draggingTodoId,
dragOverTodoId,
)
}
onDragEnd={(e) =>
handleDragEnd(
e,
setDraggingTodoId,
setDragOverTodoId,
setDraggingListId,
setDragOverListId,
)
}
style={{
userSelect: "none",
padding: "10px",
marginBottom: "10px",
borderRadius: "4px",
backgroundColor: "#eee",
}}
```

여기까지 하고 카드를 이동하려고 하면 500 에러가 뜨면서도 list 카드 이동은 정상적으로 될 것이다. 이유는 기존에 만든 list의 draggable과 todo의 draggable이 동시에 동작되면서 발생하는 문제다. 이번에도 조건문을 통해 두 동작이 겹치지 않도록 해야 한다.

```tsx
const handleDropOnList = (
e: React.DragEvent<HTMLDivElement>,
prev: number | null,
current: number | null,
) => {
if (draggingTodoId === null) {
e.preventDefault();
console.log("list", draggingTodoId);
console.log("drag on", prev, current);
listDragMutate({ prev: prev!, current: current! });
}
};
```

List에 적용해뒀던 handleDropOnList에 다음과 같이 draggingTodoId가 null일 경우에만 동작할 수 있도록 했다. 이제 한 번 todo 카드를 다른 곳으로 옮겨보자.

![](https://blog.kakaocdn.net/dna/Br4LM/btsJfpIPnWl/AAAAAAAAAAAAAAAAAAAAAHzHgosg-7VxPpKImKap92fFTZ2zUWLLOzi5_mSZG5do/img.png)

todo card를 옮기면 다음과 같이 원래 있던 위치와 옮기고자 하는 위치의 index가 console로 찍히는 것을 확인할 수 있다. 이제 이 index와 id를 api로 넘겨서 실제 indexedDB가 변경되도록 해보자.

### **mutation 만들기**

방법은 List와 동일하지만 이번에는 넘겨야 할 것들이 두 개 더 늘었다.

```tsx
const todoDrag = async (data: {
prevList: number;
currentList: number;
prevTodo: number;
currentTodo: number;
}) => {
const response = await axios.post("/todoDrag", data);
return response.data;
};
export const useTodoDragMutation = () => {
const queryClient = useQueryClient();
const listMutation = useMutation(todoDrag, {
onSuccess: () => {
queryClient.invalidateQueries("todoList");
},
onError: (error) => {
alert(`Error: ${error}`);
},
});
return listMutation;
};
```

이번에는 List id와 todo index를 모두 보내야 한다. 이제 해당 api를 mocking 해줄 handler를 만들어보자.

```tsx
rest.post("http://localhost:3000/todoDrag", async (req, res, ctx) => {
const { prevList, currentList, prevTodo, currentTodo } = await req.json();
console.log(
"drag prev to current",
prevList,
currentList,
prevTodo,
currentTodo,
);
try {
const todoList = await db.todoList.toArray();
const [prevTodoList] = todoList[prevList].todo.splice(prevTodo, 1);
todoList[currentList].todo.splice(currentTodo, 0, prevTodoList);
// 데이터베이스 업데이트
db.todoList.clear();
await db.transaction("rw", db.todoList, async () => {
for (const item of todoList) {
await db.todoList.put({ title: item.title, todo: item.todo });
}
});
return res(ctx.json({ message: "Item updated successfully" }));
} catch (error) {
console.error("Failed to update List:", error);
return res(
ctx.status(500),
ctx.json({ message: "Failed to update List" }),
);
}
}),
```

여기까지 하면 이제 자유롭게 List와 todo를 옮길 수 있는 TodoList 기능이 완성되었다!

![](https://blog.kakaocdn.net/dna/b4sxRP/btsJeUW7tgp/AAAAAAAAAAAAAAAAAAAAADITVe3bfW7JK_6Fkt0maZr-ZhUzJPd38d0yvmO-FX6S/img.png)
![](https://blog.kakaocdn.net/dna/vDJzt/btsJe97r6I5/AAAAAAAAAAAAAAAAAAAAAHs86Fgrd_QOBWSNGr0IR0PQ2hnL4D9clD46NA7kWQmX/img.png)

### **마무리...**

Next14를 처음 접하게 되면서 허겁지겁 만들었었던 trello clone coding 프로젝트를 정리해보았다. 정말 간단한 프로젝트지만 안에 들어가는 여러 라이브러리와 사용법은 next14에서 센스있는 개발을 하기 위한 많은 내용들이 담겨있었다. 어디까지나 Next14를 사용하는 방법에 대한 가이드인 만큼 디자인적인 부분이나 추가적인 기능까지는 고려하지 않고 있지만 조금 더 개선을 해보자면 delete 기능을 넣거나 가로 무한 스크롤 기능을 넣는 방법들도 생각해볼 수 있을 것 같다. react-query에서는 페이지멸 데이터 로딩에 대한 좋은 지원들도 있다!

해당 프로젝트는 깃허브 public에 올려두고 추후 next14 관련 여러 테스트가 필요할 때 학습용으로 꺼내 쓸 예정이다.

[깃허브 프로젝트 링크](https://github.com/Lajancia/nextjs-trello)

혹시 필요한 사람들이 있다면 자유롭게 해당 프로젝트를 사용하고 next14 스터디에 도움이 되었으면 한다.
