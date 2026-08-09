---
title: "1월 프론트엔드 개발자 뉴스 2"
date: 2025-01-26 08:00:00 +0900
categories: [개발]
tags: [프론트엔드, 뉴스, htmx, React, Deno, Bun, pnpm, Node.js]
author: L.J
---

## 1월 프론트엔드 개발자 뉴스 #2

해당 1월 뉴스레터들은 대부분 2024를 마무리하는 취지로 작성된 글들이 많이 보인다. 그 중에서도 조금 생소하거나 흥미로운 주제들을 몇가지 정리하여 작성해둔다. 해당 뉴스레터는 지난번 Medium 게시글을 참고한 것과 달리 Javascript Weekly에서 보내주는 에디터픽 게시글들을 참조했다. 자세한 내용까지는 작성하지 않고, 전체적으로 어떤 내용이 주를 이루는지만 살필 것이다.

### **1월 뉴스레터**

<https://javascriptweekly.com/issues/718>
<https://javascriptweekly.com/issues/719>
<https://javascriptweekly.com/issues/720>

### **Javascript**

<https://risingstars.js.org/2024/en#section-framework>

2024년 한 해 동안 인기있었던 프레임워크와 라이브러리, 툴 등을 깃허브에서 받은 별 갯수를 기반으로 순위를 나열했다. 여기서 조금 눈에 띄는 것은 htmx인 것 같다. 무려 리액트를 뛰어넘어 1위를 차지하고 있으니 말이다. 2024년 가장 인기있는 프론트엔드 프레임워크 1순위는 htmx, 2순위가 react, 3순위가 svelte이다. svelte 또한 분명 몇 년 전까지만 하더라고 vue보다 순위가 낮았었는데, 이제는 당당히 3위를 차지하고 있다.

htmx는 HTML을 사용하여 동적으로 Page를 랜더링할 수 있도록 하는 자바스크립트 라이브러리이다. 리액트가 있는데 굳이 htmx를 사용해야 할까 생각이 들 수도 있지만, 사실 리액트를 굳이 사용할 필요까지는 없는 단순한 구성이거나 혹은 프론트엔드가 익숙하지 않은 백엔드 개발자들에게는 좋은 선택지가 될 수 있다. 목적은 단순하다. 복잡한 Javascript로 구현해야 하는 부분을 최소화 시키고 html에서 최대한 처리하게 두는 것이다.

htmx의 목적을 요약하면 다음과 같다.

1. 단순함 유지하기
2. HTML 태그에 사용자 정의 속성을 추가하고 Ajax와 유사한 요청 보내기
3. 리액트나 뷰와 같은 무거운 기능 대신 간단하고 기본적인 구현이 필요할 때 부담없이 선택 가능

<https://velog.io/@sehyunny/a-first-look-at-htmx-and-how-it-compares-to-react>

htmx의 사용법과 장점에 대해서는 해당 글에서 잘 번역하여 이해하는데 도움이 된다. 추후 백엔드 언어를 공부할 때 htmx로 간단한 테스트를 하는 ui를 구현해보는 것도 좋을 것 같다. 특히 Jquery를 대체할만한 라이브러리로 주목받는 것 같다.

<https://2ality.com/2025/01/nodejs-strip-type.html>

node 관련으로도 재미있는 뉴스가 있다. 바로 23.6.0 버전부터 typescript 파일을 빌트인으로 지원한다는 것이다. 주의할 점은, .tsx 파일은 지원되지 않는다. (.jsx도 마찬가지)

![](https://blog.kakaocdn.net/dna/ced7tB/btsL04P4T4Y/AAAAAAAAAAAAAAAAAAAAALr9_sSAfieSDaCG2vZyJaak4Zz6sK3pmG1VoPIAxvya/img.webp)

이외에도 Deno 관련 이슈들이 있었다. 아마 Deno와 함께 멍한 공룡이 그려진 이미지를 본 적 있을 것이다. 이 친구가 Deno이다. 원래 자바스크립트는 웹 브라우저에서 동작하는 스크립트 언어로 시작했지만, Node.js 등장 이후부터는 서버 사이드 프로그래밍 영역까지 확장되었고, 이후에 데노(Deno)가 등장하면서 Node.js와 함께 서버 사이드 자바스크립트를 이끌고 있다.

기존 node.js를 보완하기 위해 만들어진 만큼, 보안성 강화와 타입스크립트 기본 지원 등등의 개선이 이루어졌지만, 아직 Node.js가 압도적인 사용률과 안정적인 생태계를 구축하고 있다 보니, 아직은 Node.js 정도의 규모를 갖추지는 못한 상태이지만 계속해서 성장중에 있다. 개발자 입장에서는 서버 사이드 자바스크립트로 Deno와 Node.js라는 두 가지 선택지가 있는 셈이니 나쁘지 않은 변화이다. 이번 뉴스레터에서는 이런 Deno 2.0 출시에 대한 아티클이 언급되어 있다.

<https://deno.com/blog/v2.0>

bun 관련 업데이트도 눈에 띈다. 패키지 설치와 관리에서 엄청난 속도를 자랑한다는 것으로 본 것이 얼마 전이었던 것 같은데, 생각보다 빠른 속도로 npm과 yarn을 쫓아오는 것 같다. 이제 bun 으로 프론트엔드 빌드와 실행이 가능해진 모양이다. 호환성 문제도 완전히 해결되기까지 멀지 않은 것일까...

<https://bun.sh/blog/bun-v1.1.44>

추가로 pnpm 10이 릴리즈되었다. 해시 알고리즘이 SHA256으로 변경된 부분과 함께 자잘한 변경사항이 있다.

<https://github.com/pnpm/pnpm/releases/tag/v10.0.0>

이외에도, storybook과 NestJS, Inferno 업데이트가 진행되었다.

### **Articles**

<https://www.comfydeploy.com/blog/you-dont-need-nextjs>

아무래도 Next14.js 개발자이다보니, 눈에 띄는 제목이다. 블로그 저자의 경우 Next를 사용하다 다시 react로 돌아간 이야기에 대해 언급하고 있다. 아무래도 Next14의 빌드 타임이 너무 길고, 약간의 변화가 모든 SSR의 reload를 발생시키며 수많은 API들이 api routes와 얽혀 있어서 테스트하기에도 어려운 상황에 유저당 API 이용 횟수가 너무 커 2000$나 되는 요금 폭탄을 맞은 탓이었다.

React로 복귀한 후에는 이러한 문제들은 해결되었지만, Next만의 캐싱 기능과 prerender, initial load 및 서버 컴포넌트 등의 이점을 포기해야만 했다고 한다.

<https://tympanus.net/codrops/2024/12/19/crafting-a-dreamy-particle-effect-with-three-js-and-gpgpu/>

WebGL 아티클도 빠질 수 없다. Three.js로 Bloom을 활용해 몽환적인 분위기를 구현한 쉐이더 예제가 나왔다. 추후에 포트폴리오 웹 페이지 개발에 활용해보아도 좋을 것 같다.

<https://github.com/visgl/deck.gl/blob/9.1-release/docs/whats-new.md#deckgl-v91>

관련해서 데이터 시각화로 사용되는 deck.gl이 소개되었다. WebGPU를 기반으로 동작하며 React와 JS에서 사용 가능하다. 대규모의 데이터를 시각화하는데 유용할 것 같다.