---
title: "React-Three-Fiber + Next14"
date: 2024-12-08 09:00:00 +0900
categories: [개발, Next.js]
tags: [React, Three.js, R3F, React Three Fiber, Next.js, 3D, GLB, drei]
author: L.J
---

React-Three-Fiber + Next14

### **시작하기...**

일전에도 Next14에서 React three Fiber, 줄여서 R3F를 사용하는 방법에 대해 언급했었다. 이번에는 간단하게 웹사이트를 만들면서 참고한 예제들과 사용 방법에 대해 언급해보려 한다.

<https://github.com/pmndrs/react-three-next>

GitHub - pmndrs/react-three-next: React Three Fiber, Threejs, Nextjs starter

React Three Fiber, Threejs, Nextjs starter. Contribute to pmndrs/react-three-next development by creating an account on GitHub.

github.com

위의 템플릿을 사용했다고 했을 때, 기본적으로 제공하는 방식은 R3F에서 자주 사용되는 drei의 View 기능이다. 해당 View를 통해 우리가 원하는 3D 모델 glb 파일을 화면에 보일 수 있도록 할 수 있다.

간단한 사용 방법은 다음과 같다.

#### **GLB 3D 모델링 불러오기**

```
"@react-spring/three": "^9.7.5",
"@react-three/drei": "9.88.3",
"@react-three/fiber": "^8.15.12",
"@react-three/postprocessing": "2.15.0",
"three": "^0.160.0",
"three-stdlib": "^2.28.9",
```

필자가 사용중인 라이브러리들이다. 중요한 것은, 해당 라이브러리들이 버전마다 사용방법과 지원 기능들이 조금씩 다르다 보니, 최신 라이브러리를 사용할 경우 동일한 결과가 나오지 않거나 에러가 발생할 수 있다는 것이다. 업데이트를 꾸준히 해주는 것은 좋지만 가끔 좋은 기능이 최신 버전에서 사라져 있는 것을 보면 아쉽다는 생각이 많이 든다.

```
export function Keyboard(props) {
const Keyboard = useRef()
const { scene } = useGLTF('/keyboard_website.glb')
return (
<>
<primitive ref={Keyboard} object={scene} {...props} />
</>
)
}
```

primitive 태그는 자동적으로 복잡한 Three.js 오브젝트를 jsx에서 랜더링될 수 있도록 도와주는 태그이다. 여기서 public 하위에 넣어둔 glb 파일을 불러와 사용한다.

```
const Keyboard = dynamic(() => import('../../components/canvas/Examples').then((mod) => mod.Keyboard), { ssr: false })
const View = dynamic(() => import('../../components/canvas/View').then((mod) => mod.View), {
ssr: false,
loading: () => (
<div>
<svg className={spinnerStyles} fill='none' viewBox='0 0 24 24'>
<circle className='opacity-25' cx='12' cy='12' r='10' stroke='currentColor' strokeWidth='4' />
<path
className='opacity-75'
fill='currentColor'
d='M4 12a8 8 0 0 1 8-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 0 1 4 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z'
/>
</svg>
</div>
),
})
const spinnerStyles = css({
marginLeft: '-0.25rem',
marginRight: '0.75rem',
height: '1.25rem',
width: '1.25rem',
animation: 'spin 1s linear infinite',
color: 'black',
'@keyframes spin': {
from: { transform: 'rotate(0deg)' },
to: { transform: 'rotate(360deg)' },
},
})
```

이 keyboard 컴포넌트와 해당 템플릿에서 제공하는 View를 활용하여 dynamic으로 불러올 수 있다.

```
import { Lightformer, Environment, Float, ContactShadows, Text, OrbitControls } from '@react-three/drei'
import { Bloom, EffectComposer, N8AO, TiltShift2 } from '@react-three/postprocessing'
export default function KeyboardParts() {
<View orbit className={css({ height: '100%' })}>
<spotLight position={[10, 10, 10]} penumbra={3} castShadow angle={1} />
<ambientLight position={[10, 10, 10]} />
<pointLight position={[10, 10, 10]} />
<Suspense fallback={null}>
<Float floatIntensity={2}>
<Keyboard scale={'0.3'} position={[0, 0, 0]} rotation={[Math.PI / 3.7, 5.5, 0]} />
</Float>
<ContactShadows position={[0, -5, 0]} opacity={0.8} scale={30} blur={1.75} far={4.5} />
</Suspense>
<Environment preset='city'>
<Lightformer
intensity={1}
position={[10, 5, 10]}
scale={[10, 10, 1]}
onUpdate={(self) => self.lookAt(10, 10, 0)}
/>
</Environment>
<EffectComposer disableNormalPass>
<Bloom />
</EffectComposer>
<OrbitControls enableZoom={false} />
</View>
}
```

View 안에 세팅하고자 하는 환경을 작성해주면 된다. 필자는 오브젝트가 잘 보일 수 있도록 spotLight와 ambientLight, pointLight를 추가해준 뒤, 그림자가 나타날 수 있도록 ContactShadows도 세팅하였다. 해당 기능들은 모두 drei를 통해 설정할 수 있다. 이외에도 Evironment를 사용하여 오브젝트가 전체적으로 뚜렷하게 보일 수 있도록 주변 환경 세팅을 City로 설정하였다.

Float 기능은 말 그대로 오브젝트가 허공에 둥둥 뜨는 느낌이 들도록 해주는 기능이다. 자연스럽게 동작할 수 있도록 2 정도로 세팅하였다.

#### **가로 스크롤 이미지 Display**

<https://codesandbox.io/p/sandbox/l4klb>

React Three Fiber에서는 다양한 Example들을 소개하는데, 그 중 위의 Example을 사용하여 Gallery 페이지를 꾸며보려 했다.

```
'use client'
import * as THREE from 'three'
import { useRef, useState, Suspense } from 'react'
import { Canvas, useFrame, useThree } from '@react-three/fiber'
import { Image, ScrollControls, Scroll, useScroll, Text } from '@react-three/drei'
import { proxy, useSnapshot } from 'valtio'
import { easing } from 'maath'
import { css } from '../../../styled-system/css'
const material = new THREE.LineBasicMaterial({ color: 'white' })
const geometry = new THREE.BufferGeometry().setFromPoints([new THREE.Vector3(0, -0.5, 0), new THREE.Vector3(0, 0.5, 0)])
const imagePaths = []
for (let i = 1; i <= 17; i++) {
imagePaths.push(`/img/gallery/${i}.jpeg`)
}
const state = proxy({
clicked: null,
urls: imagePaths,
})
function Minimap() {
const ref = useRef()
const scroll = useScroll()
const { urls } = useSnapshot(state)
const { height } = useThree((state) => state.viewport)
useFrame((state, delta) => {
ref.current.children.forEach((child, index) => {
const y = scroll.curve(index / urls.length - 1.5 / urls.length, 4 / urls.length)
easing.damp(child.scale, 'y', 0.15 + y / 6, 0.15, delta)
})
})
return (
<group ref={ref}>
{urls.map((_, i) => (
<line
color={'#000'}
key={i}
geometry={geometry}
material={material}
position={[i * 0.06 - urls.length * 0.03, -height / 2 + 0.6, 0]}
/>
))}
</group>
)
}
function Item({ index, position, scale, c = new THREE.Color(), ...props }) {
const ref = useRef()
const scroll = useScroll()
const { clicked, urls } = useSnapshot(state)
const [hovered, hover] = useState(false)
const click = () => (state.clicked = index === clicked ? null : index)
const over = () => hover(true)
const out = () => hover(false)
useFrame((state, delta) => {
const y = scroll.curve(index / urls.length - 1.5 / urls.length, 4 / urls.length)
easing.damp3(ref.current.scale, [clicked === index ? 4.7 : scale[0], clicked === index ? 5 : 4 + y, 1], 0.15, delta)
ref.current.material.scale[0] = ref.current.scale.x
ref.current.material.scale[1] = ref.current.scale.y
if (clicked !== null && index < clicked) easing.damp(ref.current.position, 'x', position[0] - 2, 0.15, delta)
if (clicked !== null && index > clicked) easing.damp(ref.current.position, 'x', position[0] + 2, 0.15, delta)
if (clicked === null || clicked === index) easing.damp(ref.current.position, 'x', position[0], 0.15, delta)
easing.damp(ref.current.material, 'grayscale', hovered || clicked === index ? 0 : Math.max(0, 1 - y), 0.15, delta)
easing.dampC(
ref.current.material.color,
hovered || clicked === index ? 'white' : '#aaa',
hovered ? 0.3 : 0.15,
delta,
)
})
return (
<Image ref={ref} {...props} position={position} scale={scale} onClick={click} onPointerOver={over} onPointerOut={out} alt='gallery' layout='fill' />
)
}
function Items({ w = 0.7, gap = 0.15 }) {
const { urls } = useSnapshot(state)
const { width } = useThree((state) => state.viewport)
const xW = w + gap
return (
<ScrollControls horizontal damping={0.1} pages={(width - xW + urls.length * xW) / width}>
<Minimap />
<Text font={Jersey} fontSize={0.3} position={[0, -2.3, 0]} color={'#373737'}>
3D Modeling
</Text>
<Scroll>
{urls.map((url, i) => <Item key={i} index={i} position={[i * xW, 1, 0]} scale={[w, 4, 1]} url={url} />)}
</Scroll>
</ScrollControls>
)
}
const App = () => (
<>
<Canvas gl={{ antialias: false }} dpr={[1, 1.5]} onPointerMissed={() => (state.clicked = null)} className={css({ height: '90%' })}>
<Suspense fallback={null}>
<Items />
</Suspense>
</Canvas>
</>
)
export default App
```

전체 코드는 다음과 같다. 이미지를 나열하는 쪽은 Items 컴포넌트 쪽이며, drei 쪽에서 제공하는 가로 스크롤 기능도 추가되었다. public 내에 있는 이미지를 자동으로 카운트하여 나열할 수 있게 하였는데, 추가로 Text를 넣고 싶어 drei의 Text 테그도 사용하였다.

주의해야 할 점은, drei 라이브러리를 사용한 컴포넌트 내에서는 일반적인 div 태그를 사용할 수 없다. 때문에 텍스트를 작성하기 위해서는 Text 태그를 사용해야 한다.

스크롤을 할 때 부드럽게 이동되는 것을 볼 수 있다.

### **이슈 사항**

문제는 한 가지 이슈 사항이 발생한다. 기존에 메인 페이지에서 View를 통해 불러온 이미지는 drei의 View이고, Canvas는 Fiber의 기능이다. 또한 react가 아닌 Next14에서 라이브러리를 사용한 탓인지, 기존에 View를 통해 불러온 오브젝트가 route로 페이지를 이동할 때 캐시가 제대로 지워지지 않아 계속해서 보이는 문제가 있는 것이다. 하지만 매번 이동할 때 마다 캐시를 지웠다가 돌아올 때 다시 랜더링을 하자니, 3D오브젝트 특성 상 성능적으로 비효율적이었다.

때문에 필자와 같은 경우, 약간의 꼼수를 사용하기로 했다. 페이지 이동 전에 컴포넌트가 정리되듯 사라지게 한 다음 다른 페이지로 이동될 수 있도록 하는 것이었다.

```
import { useSpring, animated } from '@react-spring/three'
```

해당 라이브러리는 오브젝트의 scale 혹은 position 등에 애니메이션을 주도록 하는 기능을 제공한다.

```
const Keyboard = dynamic(() => import('../../components/canvas/Examples').then((mod) => mod.Keyboard), { ssr: false })
const AnimatedKeyboard = animated(Keyboard)
```

기존에 keyboard를 불러온 것을, AnimatedKeyboard에 넣어 animated를 적용한다.

```
export default function KeyboardParts({ showKeyboard }) {
const { scale } = useSpring({ scale: showKeyboard ? 0.3 : 0, config: { duration: 200 } })
return (
<>
<View orbit className={css({ height: '100%' })}>
<spotLight position={[10, 10, 10]} penumbra={3} castShadow angle={1} />
<ambientLight position={[10, 10, 10]} />
<pointLight position={[10, 10, 10]} />
<Suspense fallback={null}>
<Float floatIntensity={2}>
<AnimatedKeyboard scale={scale} position={[0, 0, 0]} rotation={[Math.PI / 3.7, 5.5, 0]} />
</Float>
<ContactShadows position={[0, -5, 0]} opacity={0.8} scale={30} blur={1.75} far={4.5} />
</Suspense>
<Environment preset='city'>
<Lightformer
intensity={1}
position={[10, 5, 10]}
scale={[10, 10, 1]}
onUpdate={(self) => self.lookAt(10, 10, 0)}
/>
</Environment>
<EffectComposer disableNormalPass>
<Bloom />
</EffectComposer>
<OrbitControls enableZoom={false} />
</View>
</>
)
}
```

scale을 showKeyboard state에 따라 0.3 크기 혹은 0 크기가 되도록 하고 duration은 0.2초로 설정한다. 이제 해당 애니메이션은 0.2초동안 동작하게 된다. 그렇다면 페이지 이동은 0.2초 뒤에 일어나면 자연스럽게 보일 것이니, header를 다음처럼 0.3초 뒤에 route.push를 하도록 해야 한다.

```
const handleGalleryMove = () => {
if (router.pathname === '/gallery') return
handleKeyboard()
setTimeout(() => {
router.push('/gallery')
}, 300)
}
```

클릭 시에 0.3초 뒤에 /gallery로 이동하게 함으로서 메인 페이지의 오브젝트가 사라지고 난 다음 /gallery로 이동되도록 하였다.
