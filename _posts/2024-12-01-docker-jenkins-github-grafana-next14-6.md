---
title: "6. Docker + Jenkins + Github + Grafana + Next14"
date: 2024-12-01 09:00:00 +0900
categories: [개발, Next.js]
tags: [Next.js, Docker, Jenkins, Github, Grafana, Panda CSS, Dark Mode, CI/CD]
author: L.J
---

6. Docker + Jenkins + Github + Grafana + Next14

### **Panda CSS 다크모드 적용해보기**

전체적인 웹사이트의 형태를 갖추고 난 뒤, panda css에 동적으로 스타일을 적용해보는 몇가지 시도를 진행해보았다. 가장 먼저, 동적으로 내가 원하는 색상을 보이도록 styled css를 작성해보았다.

```
const circleButton = (props) =>
css({
width: '3rem',
height: '3rem',
backgroundColor: props.color === 'white' ? 'white' : props.color === 'Orange' ? 'Orange' : 'Black',
border: 'none',
borderRadius: '50%',
textAlign: 'center',
lineHeight: '4.4rem',
fontSize: '1.5rem',
cursor: 'pointer',
})
// html
<div className={circleButton({ color: 'white' })} />
```

이때 내가 어떤 색상을 원하는지 props로 넘기게 되면, 조건문에 따라 특정 색상을 세팅하는 구조를 가진다. 문제는 위의 조건문이 아니라 직접 색상을 props로 넘기는 방법으로 진행할 경우, 제대로 색상이 적용되지 않는 이슈를 발견하였다. 기본적으로 정적인 스타일링 라이브러리이다 보니, 조건문의 형태가 아닌 바로 넘겨주는 형태의 경우, 색상이 사전에 지정되지 않는 상태이다 보니 이러한 이슈가 발생하는 것 같다.

동그라미에 색상이 잘 적용된 모습

조건문의 형태기는 하지만 그대로 동적으로 잘 css가 적용된 모습이다. 이제 전체 웹사이트에 대하여 다크모드와 화이트 모드를 적용하여 내가 원하는 색상이 나타나도록 해보자.

#### **다크모드 적용하기**

panda css에서는 다른 테마들과 같이 다크모드와 화이트모드 지원이 가능하다. 필자와 같은 경우, 설정한 theme의 상태를 쿠키에다 저장하여 기본은 다크모드로 세팅하고 바뀐 설정에 대하여 쿠키에 저장해둘 예정이다.

특이한 점은, 기존에 사용하던 styled-css에서와 같이 Themeprovider와 같은 태그가 필요한 것이 아닌, html에 직접 data-color-mode를 통해 테마를 제어할 수 있다는 것이다.

다크 모드와 화이트 모드에서 서로 다른 색상이 적용될 수 있도록 다음과 같이 panda.config.js를 설정하자.

```
theme: {
extend: {
semanticTokens: {
colors: {
background: { value: { base: '#E9E9E9', _dark: '#1e1e1e' } },
MainText: { value: { base: '#000', _dark: '#fff' } },
primary: { value: { base: '#07c', _dark: '#0cf' } },
},
},
},
},
```

기본 테마는 base이지만, dark가 적용될 때는 _dark가 적용되도록 설정되어 있다.

해당 theme는 layout.jsx에서 제어하도록 설정한다.

<https://panda-css.com/docs/guides/multiple-themes#ssr-example-with-nextjs>

Panda CSS - Build modern websites using build time and type-safe CSS-in-JS

panda-css.com

```
import { Layout } from '../components/dom/Layout'
import { cookies } from 'next/headers'
import '../../index.css'
import Underlay from '../parts/keyboard/Underlay'
import { Jersey_25 } from 'next/font/google'

const jersey25 = Jersey_25({
subsets: ['latin'],
weight: ['400'],
})

export const metadata = {
title: 'SoominLab Portfolio',
description: 'A minimal starter for Nextjs + React-three-fiber and Threejs.',
}

export default function RootLayout({ children }) {
const store = cookies()
const themeName = store.get('theme')?.value || 'dark'
const theme = themeName
return (
<html lang='en' className='antialiased' data-color-mode={theme}>
<head />
<body className={jersey25.className}>
<Underlay />
<Layout>{children}</Layout>
</body>
</html>
)
}
```

next/headers에서 제공하는 cookies를 사용하여 현재 쿠키에 저장되어있는 theme value를 가져온다. 만약 value가 없다면 dark로 설정하게 한다.

해당 테마를 변경하기 위해서는 버튼에 동작을 추가해야 한다.

```
const toggleTheme = () => {
const currentTheme = document.cookie
.split('; ')
.find((row) => row.startsWith('theme='))
?.split('=')[1]
const newTheme = currentTheme === 'dark' ? 'light' : 'dark'
document.cookie = `theme=${newTheme}; path=/`
window.document.documentElement.setAttribute('data-color-mode', newTheme)
}
// button에 적용
<button className={StyledThemeButton} onClick={toggleTheme} />
```

해당 함수는 클릭 시에 현재 쿠키에 저장되어 있는 데이터를 가져오고 data-color-mode에 바뀐 테마를 교체하는 기능을 수행한다.

이때 자연스러운 색상 변경을 위해 background-color의 transition을 0.5s로 설정해두었다. 여기까지 panda css를 활용하여 동적으로 스타일링 하는 방법을 정리해 보았다.
