---
title: "React-Three-Fiber + WebXR (2)"
date: 2024-09-02 12:00:00 +0900
categories: [R3F]
tags: [React, Three.js, R3F, WebXR, VR]
author: L.J
---

### **우리가 해볼 것...**

Mixed Reality(혼합 현실) 위주로 WebXR을 만들 예정이기는 하지만, 대다수는 VR 기기가 없는 만큼 브라우저 상 등장하는 3D 모델의 배치도 고려 대상이다. 현재 구상중인 WebXR 스터디는 크게 두 가지 상호작용을 위주로 고려하고 있다.

필요한 3D 모델을 제작하기 전에, 먼저 간단한 예제 코드들을 테스트 해보며, 랜더링 성능을 파악해보도록 하자.

#### **컨트롤러**

오큘러스 프로에서는 두 가지 컨트롤러를 제공한다. 하나는 흔히 리모컨과 같이 동작하는 물리적 컨트롤러고, 다른 하나는 실제 손을 사용하는 컨트롤러다. 다행스럽게도 WebXR을 구동할 경우 해당 VR 기기에서 핸드 컨트롤러를 지원하면 자동으로 전환되어 사용할 수 있다. 지난 시간에 테스트한 큐브는 단순히 hover을 하기만 하면 색이 바뀌는 구조지만, 우리가 원하는 것은 터치시 동작하는 것이다. 추가적으로 개발을 진행하면서 최종적으로는 네모 박스가 아닌 우리가 직접 만든 모델을 올려서 사용할 것이기에, 이번에는 옛날에 만들었던 모델을 테스트 용으로 사용해 보기로 했다.

혹시 테스트할 모델이 필요하다면 아래의 모델을 자유롭게 사용하면 된다.

[work.glb.zip](https://blog.kakaocdn.net/dna/b7hB68/btsJnastMMr/AAAAAAAAAAAAAAAAAAAAADpbJVtEoHZH7q6Bm4P-bRgZXwRbRtNzm5Wbtm6Y41OY/work.glb.zip?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1788188399&allow_ip=&allow_referer=&signature=Tm0eJa%2FRMYsmcKJLb9wI59h8D8w%3D&attach=1&knm=tfile.zip)

#### **터치 시 색 전환**

간단하게 터치 동작을 수행할 경우 모델의 색깔이 흰색에서 검은색이 되는 동작을 구현해 볼 것이다. 이 때, 사용할 모델의 구조가 복잡하여 프레임 드랍이 심각하기에, 버튼의 역할을 수행할 정육면체를 대신 클릭하여 색상이 바뀔 수 있게 하였다.

```jsx
function Cube() {
const ref = useRef<Mesh>(null);
const Cube = useRef();
// @ts-ignore
const work = useGLTF("/work.glb") as any; // Adjust the path and type as necessary
const [white, setWhite] = useState(false);
const glassMaterial = new THREE.MeshPhysicalMaterial({
transparent: true,
opacity: 0.5,
color: white ? "white" : "black",
roughness: 0,
side: THREE.FrontSide,
blending: THREE.AdditiveBlending,
polygonOffset: true,
polygonOffsetFactor: 1,
envMapIntensity: 21,
});
useFrame(() => {
(Cube.current as any).rotation.y += 0.01;
});
return (
<>
<mesh
onClick={() => setWhite(!white)}
position={[0.3, 1.2, -0.3]}
pointerEventsType={{ deny: "grab" }}
ref={ref}
scale={[0.05, 0.05, 0.05]}
>
<boxGeometry />
<meshBasicMaterial color={white ? "white" : "black"} />
</mesh>
<primitive
ref={Cube}
position={[1.5, 1.2, -0.6]}
scale={[0.08, 0.08, 0.08]}
object={work.nodes.Cube}
// @ts-ignore
material={glassMaterial}
/>
</>
);
}
```

이번에는 mesh에 hover가 아닌 onClick 발생 시 state가 바뀌면서 glassMaterial 내부의 색상이 변경되도록 하였다. three.js는 타입 관련 이슈가 꽤 많이 터지다 보니, 어쩔 수 없이 @ts-ignore을 사용하여 잠시 타입 에러를 막아두었다. onClick이 적용된 것은 boxGeometry이기 때문에 work 내부의 Cube 노드는 터치가 적용되지 않는다. 성능 향상을 위해서는 폴리곤 3D 모델을 사용할 필요가 있지만, 지금은 임시적으로만 사용할 예정이니 어쩔 수 없다.

위의 코드에서 primitive는 외부에서 Import 한 모델을 그대로 랜더링하기 위해 사용된다. 때문에 이미 Mesh로 구성되어 별도의 mesh 테그가 필요하지 않다. 필자의 경우, 유리 재질의 불투명한 느낌을 주고 싶어 별도로 material 을 통해 추가 설정을 진행하였다.

전체 코드는 아래와 같다.

```jsx
import { Canvas, useFrame } from "@react-three/fiber";
import { useGLTF } from "@react-three/drei";
import { createXRStore, XR, XROrigin, TeleportTarget } from "@react-three/xr";
import { useRef, useState } from "react";
import { Mesh, Vector3 } from "three";
import * as THREE from "three";
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
<mesh scale={[10, 1, 10]} position={[0, -0.5, 0]}></mesh>
</TeleportTarget>
</XR>
</Canvas>
</>
);
}
function Cube() {
const ref = useRef<Mesh>(null);
const Cube = useRef();
// @ts-ignore
const work = useGLTF("/work.glb") as any; // Adjust the path and type as necessary
const [white, setWhite] = useState(false);
const glassMaterial = new THREE.MeshPhysicalMaterial({
transparent: true,
opacity: 0.5,
color: white ? "white" : "black",
roughness: 0,
side: THREE.FrontSide,
blending: THREE.AdditiveBlending,
polygonOffset: true,
polygonOffsetFactor: 1,
envMapIntensity: 21,
});
useFrame(() => {
(Cube.current as any).rotation.y += 0.01;
});
return (
<>
<mesh
onClick={() => setWhite(!white)}
position={[0.3, 1.2, -0.3]}
pointerEventsType={{ deny: "grab" }}
ref={ref}
scale={[0.05, 0.05, 0.05]}
>
<boxGeometry />
<meshBasicMaterial color={white ? "white" : "black"} />
</mesh>
<primitive
ref={Cube}
position={[1.5, 1.2, -0.6]}
scale={[0.08, 0.08, 0.08]}
object={work.nodes.Cube}
// @ts-ignore
material={glassMaterial}
/>
</>
);
}
```

해당 코드를 동일하게 https 환경에서 실행한 뒤 오큘러스 퀘스트의 Browser에서 와이파이 IP와 port로 접속해보자.

![](https://blog.kakaocdn.net/dna/czk4TS/btsJnNwNExg/AAAAAAAAAAAAAAAAAAAAABvmXqdLuSsmUfnWIZd_5Pckr4zQv1YpsNs8TS8DPARC/img.png)

허공에 뜬 큐브

검은 큐브를 클릭할 경우 우리가 제작한 모델의 색상이 변경되는 것을 확인할 수 있다. 이 때, 인식 문제로 인해 빠르게 정육면체를 톡 두드리는 제스쳐를 취해야 제대로 인식되는 문제가 있다. 잘 되지 않는다면, 엄지와 검지를 꼬집는 제스쳐로 터치를 대신할 수 있다.

![](https://blog.kakaocdn.net/dna/bbQwVP/btsJppOMhVP/AAAAAAAAAAAAAAAAAAAAAFRTh11dopQHpwgawN2xkWLWLrqybtyFgWcepjdV4Yk5/img.gif)

PC에서 위치를 변경하고 다시 VR 기기를 쓰며 확인하는 것이 불편하다면, 아직 베타 버전이지만 remote display 기능을 통해 VR 기기 내부에서 원격으로 PC 화면을 띄워 동시에 작업이 가능하다.

다음 시간에는 큐브를 자유롭게 이동하는 동작을 적용해보자!
