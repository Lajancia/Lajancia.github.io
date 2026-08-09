---
title: "React-Three-Fiber + WebXR (1)"
date: 2024-08-31 12:00:00 +0900
categories: [R3F]
tags: [React, Three.js, R3F, WebXR, VR, Oculus]
author: L.J
---

### **R3F?**

Three.js라는 3D 모델을 웹상에서 제어할 수 있게 해주는 라이브러리가 있었다. 이 라이브러리를 React 프레임워크에서 쉽게 사용될 수 있도록 만들어진 Three.js 파생 라이브러리를 React-three-fiber, 줄여서 R3F라고 한다. 내 개인 포트폴리오 웹사이트 또한 이 R3F를 사용해서 만들어졌다. ~~현재는 젠킨스 테스트를 위해 내려간 상태다.~~

blender로 만든 3D 모형을 직접 웹상에 띄워 인터랙팅을 할 수 있다는 점은 굉장히 매력적이다. 놀랍게도 이 라이브러리에서 더 나아가 WebXR을 다룬다면 React로 직접 VR 기기에서 상호작용 할 수 있는 서비스도 만들 수 있다는 것이다. 오큘러스 프로를 오랫동안 사용중인 필자로서는 사용해보지 않을 수가 없다.

#### **뭐가 필요할까?**

안타깝게도 WebXR은 VR 기기가 필수적으로 필요하다. 물론 해당 장비가 없다면 R3F까지만 구현하는 것도 가능하다. 다만 이번 프로젝트의 최종 목표는 VR까지 구현되는 것임을 알았으면 한다.

이외 준비물은 웹개발을 위한 컴퓨터와, React에 대한 기본 지식이다. 추가적으로 Blender 혹은 Unity와 같은 3D 모델을 다룰 줄 안다면 훨씬 더 좋을 것 같다. 직접 만든 3D 모델을 올려볼 수 있는 절호의 기회다!

#### **아이디어 수집**

웹 디자인은 잘 할 줄 모르기에, 레퍼런스가 될 만한 이미지들을 찾아보았다. 핀터레스트에서 XR을 검색했을 때 뜨는 여러 이미지들 중, 가장 간단해보이면서도 멋진 디자인을 찾아보았다. 책상 위에 여러 그래프들이 떠 있는 이미지들이 눈에 띄었다. 시작할 때는 최대한 간단한 오브젝트를 위주로 만들어서 사용해볼 예정이다.

### **시작하기**

가장 먼저, react 프로젝트를 하나 생성해준다. 타입스크립트를 사용할 것이며, 추가로 yarn도 사용할 것이다.

```bash
yarn create react-app webxr-study --template typescript
```

우리가 참고할 튜토리얼은 아래의 깃허브이다.

<https://github.com/pmndrs/xr>

깃허브의 안내에 따라 필요한 라이브러리를 설치해주자.

```bash
yarn add three @react-three/fiber @react-three/xr@latest
```

설치가 완료되면 가장 기본 튜토리얼 코드를 App.tsx에 복사해서 정상적으로 동작하는지 확인해보자.

```jsx
import { Canvas } from "@react-three/fiber";
import {
useHover,
createXRStore,
XR,
XROrigin,
TeleportTarget,
} from "@react-three/xr";
import { useRef, useState } from "react";
import { Mesh, Vector3 } from "three";
const store = createXRStore({
hand: { teleportPointer: true },
controller: { teleportPointer: true },
});
export default function App() {
const [position, setPosition] = useState(new Vector3());
return (
<>
<button onClick={() => store.enterVR()}>Enter VR</button>
<button onClick={() => store.enterAR()}>Enter AR</button>
<Canvas style={{ width: "100%", flexGrow: 1 }}>
<XR store={store}>
<ambientLight />
<XROrigin position={position} />
<Cube />
<TeleportTarget onTeleport={setPosition}>
<mesh scale={[10, 1, 10]} position={[0, -0.5, 0]}>
<boxGeometry />
<meshBasicMaterial color="green" />
</mesh>
</TeleportTarget>
</XR>
</Canvas>
</>
);
}
function Cube() {
const ref = useRef<Mesh>(null);
const hover = useHover(ref);
return (
<mesh
onClick={() =>
store.setHand(
{ rayPointer: { cursorModel: { color: "green" } } },
"right"
)
}
position={[0, 2, 0]}
pointerEventsType={{ deny: "grab" }}
ref={ref}
>
<boxGeometry />
<meshBasicMaterial color={hover ? "red" : "blue"} />
</mesh>
);
}
```

세팅을 완료한 뒤 yarn start를 통해 로컬호스트를 실행시켜 보면, 아래와 같은 창이 나타난다.

![](https://blog.kakaocdn.net/dna/bToRuB/btsJniJqGx5/AAAAAAAAAAAAAAAAAAAAAD7-zfQCvwHwpgXUFiNwCJeKiJMuE17lw_MLZygoaXBh/img.png)

정말 조그만 상자와 함께 AR로 접근할 수 있는 버튼이 생겨났다. 실제 VR기기가 존재한다면 직접 테스트 해볼수도 있다. 포럼을 이리저리 돌아다녀본 결과, 가장 간단한 방법은 다음과 같다.

기본적으로 WebXR을 실행시키기 위해서는 localhost이거나 https 환경이어야 한다. 하지만 현재 우리의 react app을 와이파이 ip를 통해 연결하면 http 환경이다. 때문에 개발 환경에서만큼은 https로 연결해줘야 할 필요가 있다.

```bash
HTTPS=true yarn start
```

해당 방법으로 실행해주면 안전하지 않은 페이지가 뜰 것이다. 고급 을 눌러 해당 웹사이트로 접근하면 된다. oculus quest pro에서도 내 PC와 동일한 와이파이를 잡은 뒤 와이파이의 IP:port를 https일 때로 연결하면 정상적으로 WebXR이 실행되는 것을 확인할 수 있다!

![](https://blog.kakaocdn.net/dna/cX1gYU/btsJnJ0SezG/AAAAAAAAAAAAAAAAAAAAAK77B854OTeJrbya-ORHqv-Q31hrwATgN7zJczMALYLH/img.png)

네모가 나타났다!
