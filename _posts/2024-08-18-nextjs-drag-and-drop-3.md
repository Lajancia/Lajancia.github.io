---
title: "Next.js 시작하기 - Drag and Drop 만들기 (3) Draggable"
date: 2024-08-18 12:00:00 +0900
categories: [Next.js]
tags: [Next.js, React, Drag and Drop, Trello, MSW, IndexedDB]
author: L.J
---

### **시작하기...**

지난 시간에 indexedDB와 MSW를 활용하여 프론트엔드의 Todo를 추가할 수 있게 해보았다. 같은 방법으로 빠르게 List를 추가할 수 있는 기능을 만든 뒤, 대망의 Drag and Drop 기능을 구현해보자. 해당 DnD 기능은 라이브러리를 사용하지 않고 만들 예정이다. ~~어지간하면 라이브러리를 쓰는게 맞다.~~

### **빠르게 List 추가 기능 만들어보기**

Todo를 만들었을 때 사용법을 설명하였으니, List는 자세한 설명을 생략하겠다.

```tsx
//list를 추가하는 mutation 함수
const list = async (data: { title: string }) => {
const response = await axios.post("/list", data);
return response.data;
};
export const useListMutation = () => {
const queryClient = useQueryClient();
const todoMutation = useMutation(list, {
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

```tsx
// MSW mocking
rest.post("http://localhost:3000/list", async (req, res, ctx) => {
const { title } = await req.json();
console.log("title", title);
try {
await db.todoList.add({ title, todo: [] });
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

List Add를 사용하게 되면서 추가적으로 타입에 대하여 수정해야 할 필요도 있다. List에서는 id가 자동 생성되기 때문에 필요없기 때문이다.

```tsx
// TodoListButton
interface TodoListButtonProps {
children: React.ReactNode;
id?: number;
onClickConfirm: (title: string, id?: number) => void;
}

//DashBoard
const { mutate, isLoading: todoLoading } = useTodoMutation();
const { mutate: listMutate, isLoading: listLoading } = useListMutation();

const handleTodoAdd = (_title: string, _id?: number) => {
if (_id !== undefined) {
mutate({ id: _id, title: _title });
}
};

// List Add
const handleListAdd = (_title: string) => {
listMutate({ title: _title });
};

<TodoListButton onClickConfirm={handleListAdd}>+ Add List</TodoListButton>
```

원리는 Todo Add 버튼 생성 때와 같다. 정상적으로 적용되었다면 List Add 버튼을 통해 추가하였을 때 카드가 새로 생성되는 것을 확인할 수 있다.

![](https://blog.kakaocdn.net/dna/J9on8/btsI8ia5zFs/AAAAAAAAAAAAAAAAAAAAAOAHtolJr_IARkuXwl_BG9FU7X4bTl3HjybitDP4l2ti/img.png)

생성된 리스트 하위에 todo도 추가할 수 있게 되었다!

### **List Drag and Drop**

먼저 가장 바깥의 List간 드래그앤드롭이 가능하도록 한번 만들어보자. draggable이 적용된 태그는 다음 네가지를 설정할 수 있다.

먼저, 드래그 대상 list id와 드래그 할 위치의 list id를 저장할 두 개의 useState를 만들어보자.

```tsx
const [draggingListId, setDraggingListId] = useState<number | null>(null);
const [dragOverListId, setDragOverListId] = useState<number | null>(null);
```

다음으로 각각의 동작에 대해서 어떻게 동작할지를 결정하도록 함수를 생성할 것이다.

```tsx
const handleDragStart = (
e: React.DragEvent<HTMLDivElement>,
id: number,
setDraggingId: (props: number) => void,
) => {
e.stopPropagation();
setDraggingId(id);
};
const handleDragOver = (
e: React.DragEvent<HTMLDivElement>,
id: number,
setDragOverId: (props: number) => void,
) => {
e.preventDefault();
setDragOverId(id);
};
const handleDragEnd = (
e: React.DragEvent<HTMLDivElement>,
setDraggingId: (props: null) => void,
setDragOverId: (props: null) => void,
) => {
e.stopPropagation();
setDraggingId(null);
setDragOverId(null);
};
const handleDropOnList = (
e: React.DragEvent<HTMLDivElement>,
prev: number | null,
current: number | null,
) => {
e.preventDefault();
console.log();
console.log("drag on", prev, current);
};
```

가장 먼저, handleDragStart를 통해 Drag 하는 컴포넌트의 id를 draggingListId에 저장해둔다. 다음으로 handleDragOver에서 드래그 하여 위치를 이동할 때 해당 변경 위치에 대한 id를 setDragOverId를 통해 저장해둔다. 이후 원하는 위치에 놓을 경우, onDrop에서 이전 id와 현재 변경된 위치의 id를 console로 출력하고 onDragEnd에서 저장해뒀던 id state를 null로 비운다.

```tsx
draggable
onDragStart={(e) => handleDragStart(e, list.id, setDraggingListId)}
onDragOver={(e) => handleDragOver(e, index, setDragOverListId)}
onDrop={(e) => handleDropOnList(e, draggingListId, dragOverListId)}
onDragEnd={(e) =>
handleDragEnd(e, setDraggingListId, setDragOverListId)
}
```

이때 주의해야 할 것은, prev에 해당하는 id는 Card의 id이고, current는 todoList 배열의 index인 점이다.

정상적으로 동작한다면 다음과 같이 카드 위치를 옮길 때 마다 console.log가 찍힐 것이다. 물론 아직 위치가 바뀌지는 않는다.

![](https://blog.kakaocdn.net/dna/bjDor3/btsI7lT4fQF/AAAAAAAAAAAAAAAAAAAAAPI8FTcIpaN3TW2TyKfd9XPftS3WHpuga_Y1dS79OQOL/img.png)

프론트엔드 단에서는 정상적으로 해당 카드 간의 이동에 대하여 이전과 이후 위치가 찍히고 있다. 이제 해당 id를 api로 백단에 넘겨서 이동된 위치로 데이터를 갱신할 수 있도록 해야 한다. 이 부분은 역시 MSW Mocking에서 처리해줘야 한다.

먼저 해당 두 개의 id를 post로 보낼 react-query mutation 함수를 만들어 보자.

```tsx
const listDrag = async (data: { prev: number; current: number }) => {
const response = await axios.post("/listDrag", data);
return response.data;
};
export const useListDragMutation = () => {
const queryClient = useQueryClient();
const listMutation = useMutation(listDrag, {
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

해당 listDrag는 위치를 이동하고자 하는 List의 id와 이동할 위치의 index값을 post로 날리고, 성공적으로 완료될 경우 todoList 키를 가진 react-query get을 트리거하여 새로운 데이터로 페이지를 리렌더링 할 수 있게 만들어졌다. 해당 api를 가로채는 MSW는 아래와 같이 작성한다.

```tsx
rest.post("http://localhost:3000/listDrag", async (req, res, ctx) => {
const { prev, current } = await req.json();
console.log("drag prev to current", prev, current);
try {
// 데이터베이스에서 todoList 가져오기
const todoList = await db.todoList.toArray();
// prev와 동일한 id를 가진 항목 찾기
const itemIndex = todoList.findIndex((item) => item.id === prev);
if (itemIndex === -1) {
throw new Error("Item with the given id not found");
}
// 해당 항목을 배열에서 제거
const [item] = todoList.splice(itemIndex, 1);
// current 위치에 항목 삽입
todoList.splice(current, 0, item);
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

handlers 하위에 추가하면 된다. 다만 해당 방식의 처리는 매우 비효율적이다. 데이터가 별로 없을 때는 문제가 없지만, 데이터가 늘어갈 수록 for문으로 매번 초기화된 db에 다시 데이터를 집어넣어 id를 갱신해야 하기 때문이다. 하지만 이 부분은 frontend의 동작을 확인하기 위해 만들어진 임시 동작인 만큼 어디까지나 임시적인 목적으로 사용할 것이다.

이러한 과정으로 draggable 동작 함수들을 생성하게 되면, card를 옮길 때 마다 제대로 해당 위치에 이동하는 것을 확인할 수 있다.

![](https://blog.kakaocdn.net/dna/bfiPW8/btsI7syKJV4/AAAAAAAAAAAAAAAAAAAAAGL9hY35d3nZ05up4r5WFpAXxyIzF6WfwOxR-qj6FOrZ/img.png)

### **마무리**

list 데이터 간 이동을 확인하였으니 다음 시간에는 더 복잡한 todo 이동을 진행하여 마무리할 예정이다. todo가 더 복잡한 이유는, 각각의 card 내에서 위치가 이동되어야 함과 동시에, 카드 간 위치 이동도 진행되어야 하기 때문이다. 때문에 list의 id와 list 내 todo 배열의 index 또한 고려해야 한다.

물론 프론트엔드에서 해야 할 건, 단순히 옮기고자 하는 컴포넌트의 id와 todo의 index, 그리고 이동하고자 하는 위치 list index와 todo의 index만 넘기면 되는 부분이다. 따라서 MSW에서 처리되는 indexedDB 간 처리 방식은 효율성 보다는 동작에 더 초점을 맞춰 진행할 것이다. ~~실제로 production에서 사용될 API 처리는 백엔드 개발자가 멋지게 처리해 줄 것이라 믿기 때문이다.~~
