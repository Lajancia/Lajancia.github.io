---
title: "5. Docker + Jenkins + Github + Grafana + Next14"
date: 2024-11-23 09:00:00 +0900
categories: [개발, Next.js]
tags: [Next.js, Docker, Jenkins, Github, Grafana, Panda CSS, React Three Fiber, CI/CD]
author: L.J
---

5. Docker + Jenkins + Github + Grafana + Next14

### **지난 시간...**

지난 시간동안 Jenkins에서 도커 이미지를 빌드할 수 있도록 설정하고, 간단한 Next14 프로젝트를 통해 해당 동작이 정상적으로 동작하는지 테스트를 수행해 보았다. 전체적인 자동 빌드 배포 프로세스가 설정되었으니, 이제 우리는 본격적으로 Next14 애플리케이션을 개발하여 우리가 구현하고자 하는 멋진 웹을 개발해야 한다.

### **Next14 포트폴리오 애플리케이션**

세팅 환경은 다음과 같다.

사실 이번 Next14에서 가장 핵심이 되는 부분은 React Three Fiber라고도 할 수 있다. 줄여서 R3F라고도 불리는 이 라이브러리는 기존 React에서 3D 모델을 쉽게 컨트롤하여 웹사이트에 띄워줄 수 있는 Three.js를 더 쉽게 사용할 수 있도록 하는 라이브러리이다. 하지만 문제는 해당 라이브러리가 기본적으로 클라이언트 사이드에서 동작하는 라이브러리라는 사실이다. 사실 이러한 면에서 차라리 React 프레임워크를 처름부터 쓰는 편이 훨씬 좋을 것이지만, 어디까지나 해당 프로젝트는 Next14와 app router를 배우기 위한 점이 크고, 약간의 불편함을 감수하여 Next14에서 R3F가 동작되도록 할 예정이다.

다행스럽게도 필자와 같은 사람들을 위해 친절한 누군가가 미리 Next14 app router와 R3F가 호환되도록 한 프로젝트 세팅을 미리 만들어두었다.

<https://github.com/pmndrs/react-three-next>

GitHub - pmndrs/react-three-next: React Three Fiber, Threejs, Nextjs starter

github.com

### **간단한 디자인**

간단하게 figma로 Next14에서 만들고 싶은 디자인을 만들어보았다. 최대한 다음과 같은 형태로 구현될 수 있도록 해볼 예정이다.

다크모드

화이트모드

#### **Panda CSS**

아마 대부분의 경우 Next14 app router를 사용하는 과정에서 CSS 라이브러리로는 대체로 Tailwind css를 사용하고 있을 것이다. 하지만 문제가 있다. 필자의 경우 page router를 사용하면서 emotion css in js를 사용해 스타일링을 해왔는데 app router로 전환할 경우 해당 emotion css in js가 app router와 완벽하게 호환되지 않아 스타일이 제대로 적용되지 않는다는 이슈가 발생하는 것이다. 마치 깜빡이는 현상처럼 스타일이 적용되지 않다가 나타나버리는 이슈는 아무도 원하지 않을 것이다. 하지만 그렇다고 Tailwind로 모두 교체하기에는 공수도 많이 들 뿐더러.... 우리 개발팀은 태그에 너무 많은 스타일이 들어가 길어지는 것을 원치 않았다. 스타일은 스타일대로 관리하고 싶은데, app router에서도 기존의 css in js를 사용할 수는 없을까. 이러한 고민 끝에 panda css를 사용해보자는 결론이 나왔다.

물론 panda css 로의 전환을 위해서도 이런 저런 변경이 필요하다. 우선은 어떻게 사용할 수 있는지 알아보자.

```
// panda css 설치
npm install @pandacss/dev
// panda css 프로젝트에 세팅
npx panda init
```

다음으로 루트 디렉토리에 index.css를 생성하고 아래의 코드를 추가한다.

```
@layer reset, base, tokens, recipes, utilities;
```

만약 global.css에서 tailwind를 사용하고 있다면, 해당 global.css 의 css를 index.css 하위로 옮기고 @tailwind 관련 문구를 삭제한 뒤 global.css를 import 하는 파일에서 index.css를 import 하도록 변경한다. 필자와 같은 경우는 아래와 같이 index.css를 변경하였다.

```
@layer reset, base, tokens, recipes, utilities;
@layer base {
* {
box-sizing: border-box;
}
html,
body,
#root {
width: 100%;
height: 100%;
margin: 0;
padding: 0;
overflow: hidden;
}
}
```

추가로 panda css는 빌드 타임에 해당 css 들이 적용되는 관계로 develop 단계에서는 이러한 panda가 변화할 때 마다 감지할 수 있게 해야 한다. package.json에 아래의 명령어를 추가하자.

```
"panda": "npx panda",
"panda:dev": "npx panda --watch"
```

또한 만약 npm run panda:dev를 실행중에 tailwind css 관련 이슈가 발생하였을 경우, 본인의 postcss.config.js 안에 tailwind plugin 설정이 있는지 확인해보자. 해당 tailwind css를 삭제하면 정상적으로 동작할 것이다.

#### **emotion styled -> panda css 마이그레이션**

이제 한번 panda css를 적용해보자. 기존에 emotion styled 혹은 styled-component를 사용하고 있을 경우, 페이지를 새로고침할 때 스타일이 하나도 적용되지 않은 상태였다가 깜빡이며 적용되는 현상을 자주 발견할 수 있을 것이다. 이 현상을 해결하기 위해 해당 스타일들을 panda로 변경해보자.

```
// example 변경 이전
import styled from 'styled-components'
export default function Page() {
return (
<>
<Header />
<FlexContainer>
<HalfWidthContainer>
<KeyboardParts />
</HalfWidthContainer>
<HalfWidthContainer>
<Info />
</HalfWidthContainer>
</FlexContainer>
</>
)
}
const FlexContainer = styled.div`
display: flex;
align-items: center;
height: 100%;
`
const HalfWidthContainer = styled.div`
width: 50%;
height: 100%;
`
```

```
// example panda css 변경 후
import { css } from 'styled-system/css'
export default function Page() {
return (
<>
<Header />
<div className={FlexContainer}>
<div className={HalfWidthContainer}>
<KeyboardParts />
</div>
<div className={HalfWidthContainer}>
<Info />
</div>
</div>
</>
)
}
const FlexContainer = css({ display: 'flex', alignItems: 'center', height: '100%' })
const HalfWidthContainer = css({ width: '50%', height: '100%' })
```

태그에 바로 이름을 넣어 사용하는 것과 다르게, className에 해당 스타일을 적용해주는 방식으로 변경된 것과 더불어 스타일명도 카멜케이스 방식으로 변경된 것을 확인할 수 있다. 이러한 방식으로 스타일을 적용하게 되면 app router에서 화면이 깜빡이면서 스타일이 적용되는 이슈가 해결된다. 또한 기존 css in js 처럼 props를 넘겨 스타일을 제어할 수도 있다!
