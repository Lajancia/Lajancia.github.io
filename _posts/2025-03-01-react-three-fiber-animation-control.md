---
title: "React Three Fiber 애니메이션 컨트롤"
date: 2025-03-01 08:00:00 +0900
categories: [개발]
tags: [R3F, ReactThreeFiber, Three.js, Blender, Mixamo, 3D, 애니메이션]
author: L.J
---

## React-Three-Fiber 애니메이션 컨트롤

### **시작하기**

React Three Fiber 라이브러리로 오브젝트에 애니메이션을 부여하여 AboutMe 페이지를 실감나게 바꿔볼 계획을 세우고 있었다. 페이지가 로딩되면 우주비행사가 걸어들어오고 인사를 건내며 특정 질문에 대하여 특정 제스쳐를 표시하는 것이다.

하지만 우선 문제가 있다. 먼저 Blender에 어떻게 여러가지 동작을 한꺼번에 넣을 수 있을지와 해당 동작들을 어떻게 가져와서 사용할지, 그리고 추가적으로 자연스러운 그림자 옵션을 어떻게 넣을 수 있는지에 대해서였다. 생각보다 애니메이션 컨트롤을 하기 위해서 준비해야 할 것들이 너무 많았다.

#### 차근차근

### Blender 사용하기

<https://codesandbox.io/p/sandbox/pecl6>

3D skeleton 기반 애니메이션 제어를 위해 참고한 코드는 위와 같다. 위의 Stacy에 해당하는 모델을 우리가 준비한 모델로 교체하고 원하는 타이밍에 원하는 애니메이션이 동작하는지 확인되면 거의 다 완성이다.

그럼 모델을 준비할 때 문제가 있다. 첫 번째는 어디서 모션 데이터를 가져오느냐이다. 필자는 예전에 무료로 제공받은 우주 비행사 모델을 가지고 있기 때문에 해당 모델을 Mixamo에 올려서 자동으로 리깅 작업부터 애니메이션까지 작업된 결과물을 받아낼 수 있었다. 만약 이러한 캐릭터 애니메이션 모션 데이터가 필요한 사람들은 Mixamo를 적극 이용해보자. 좋은 모션 캡쳐본이 많이 있다.

<https://www.mixamo.com/>

자, Mixamo에서 원하는 애니메이션과 Skeleton 작업이 끝난 결과물을 fbx파일로 받았다면, 현재 해당 모델에는 하나의 애니메이션이 담겨있게 된다. 하지만 위의 Stacy와 같은 경우 다양한 애니메이션을 클릭할 때 마다 다르게 보여준다. 해당 애니메이션은 어디에 넣어져 있는 것일까?

### **다이나믹 애니메이션**

Stacy 모델을 Blender에서 뜯어보면, NLA 폴더 하위에 여러개의 애니메이션들이 들어있는 것을 볼 수 있다. 해당 애니메이션은 Blender의 옵션에서 각각 등록시킬 수 있다. 따라서 우리는 원하는 Mixamo 애니메이션을 다운받아 추가해야 한다.

### **모델 올리기**

기존에 MacroKeyboard 모델을 올렸을 때와 같이 primitive 태그를 사용하려 했지만, 이 경우에는 그림자가 제대로 랜더링되지 않는 문제로 인해 Stacy의 예제와 같이 skinnedMesh 태그를 사용하기로 했다.

```jsx
// @ts-expect-error type ignore
function Model(props) {
  const halo = useRef()
  const { nodes, animations, materials } = useGLTF('/astronaut_animation.glb')
  const { ref, actions, names } = useAnimations(animations)
  // @ts-expect-error type ignore
  useEffect(() => {
    console.log('name', names, props.animationSet)
    console.log(materials, materials.CLAY)
    // @ts-expect-error type ignore
    actions[props.animationSet].reset().fadeIn(0.5).play()
    // @ts-expect-error type ignore
    return () => actions[props.animationSet].fadeOut(0.5)
  }, [actions, names, props.animationSet])
  return (
    <group ref={ref} {...props} dispose={null}>
      <group rotation={[Math.PI / 2, 0, 0]} scale={0.01}>
        <primitive object={nodes.mixamorigHips} />
        <skinnedMesh
          castShadow
          receiveShadow
          geometry={nodes.astronaut_1.geometry}
          material={materials.CLAY}
          skeleton={nodes.astronaut_1.skeleton}
        />
        <skinnedMesh
          castShadow
          receiveShadow
          geometry={nodes.astronaut_2.geometry}
          material={materials.CLAY}
          skeleton={nodes.astronaut_2.skeleton}
        />
        <skinnedMesh
          castShadow
          receiveShadow
          geometry={nodes.astronaut_3.geometry}
          material={materials.CLAY}
          skeleton={nodes.astronaut_3.skeleton}
        />
      </group>
      <mesh
        // @ts-expect-error type ignore
        ref={halo}
        receiveShadow
        position={[0, 1, -1]}
      >
        <circleGeometry args={[1, 64]} />
        <meshStandardMaterial />
      </mesh>
    </group>
  )
}
```

해당 gltf 파일에서 nodes와 materials를 통해 3D를 올릴 수 있다. 여기서 필자의 경우에는 astronaut로 등록을 할 때 에러가 발생하는 문제가 있었는데, 자신의 geometry 이름이 확실하지 않을 경우에는 콘솔에 nodes를 찍어 해당 이름이 실제로 존재하는지 확인해볼 수 있다. 확인해보니 astronaut가 아니라 astronaut_1,2,3으로 나눠져 있는 것을 확인할 수 있었다.

materials도 별도로 추가했다면 blender에서 해당 material 이름을 동일하게 설정해주면 된다.