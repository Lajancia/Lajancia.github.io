---
title: "12월 프론트엔드 개발자 뉴스 1"
date: 2024-12-22 08:00:00 +0900
categories: [개발]
tags: [프론트엔드, 뉴스, Astro, CSS, React, JavaScript, WebGL]
author: L.J
---

## 12월 프론트엔드 개발자 뉴스 #1

### **웹 개발 뉴스**

#### **Astro 5.0**

<https://astro.build/blog/astro-5/>

**Astro란?**

블로그, 이커머스, 마케팅 등의 컨텐츠 중심 웹사이트를 구축하기 위한 프레임워크이다. Island 아키텍처 도입을 통해 자바스크립트 오버헤드와 복잡성을 줄이고 빠른 로딩 속도와 뛰어난 검색 엔진 최적화를 제공한다.

**Astro 5.0 업데이트**

- Content Layer: 다양한 소스에서 컨텐츠를 원활하게 로드
- Server Islands: 정적 컨텐츠와 동적 개인화 컨텐츠를 쉽게 결합

#### **Viewport Control with interactive-widget**

<https://htmhell.dev/adventcalendar/2024/22/>

기본적으로 모바일 환경에서 가상 키보드가 올라올 때, Visual Viewport는 변하지만 Layout viewport는 변하지 않는 문제로 인해 가상 키보드에 화면이 가려지는 현상이 종종 발생합니다. 이러한 문제를 해결하기 위해 interactive-widget 메타 태그를 사용하여 해결할 수 있다.

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0, interactive-widget=resizes-content">
```

*해당 interactive-widget 키워드는 Chrome 108+, Firefox 132+ 이상 버전에서 사용 가능.*

#### **Debugging with console.log()**

<https://debugagent.com/front-end-debugging-part-2-console-log-to-the-max>

기본적으로 디버깅을 위해 console.log를 사용하는 것이 일반적이지만, 이러한 로그에 대하여 레벨을 부여하여 가독성있는 디버깅이 가능하게 할 수 있다.

**console.log 레벨링**

```javascript
console.log()    // 일반 로그
console.info()   // 정보
console.warn()   // 경고
console.error()  // 에러
```

**console.log 스타일링**

```javascript
console.customLog = function(msg) {
  console.log("%c" + msg, "color:black;background:pink;font-family:system-ui;font-size:4rem;-webkit-text-stroke: 1px black;font-weight:bold")
}
console.customLog("Dazzle")
```

**console.table()**

```javascript
console.table(["Simple Array", "With a few elements", "in line"])
```

테이블 형태의 로그 출력

**console.count()**

```javascript
function myFunction() {
  console.count('myFunction called');
}
```

해당 함수가 몇 번이나 호출되었는지 카운트

**console.group()**

```javascript
console.group('My Group');
console.log('Message 1');
console.log('Message 2');
console.groupEnd();
```

로그의 특정 그룹으로 묶는 방법

**monitor()**

함수와 함수에 넘긴 argument를 호출 시에 출력

#### **npm vs npx**

<https://blog.logrocket.com/npm-vs-npx/>

**npm**: Node Package Manager의 역할을 수행한다. package.json을 통해 의존성을 관리할 수 있으며, 버전 관리도 가능하다. 필요한 패키지들을 설치하여 사용할 수 있도록 돕는다.

**npx**: npm이 패키지의 설치와 관리를 담당한다면 npx는 실행을 담당한다. npx는 node 패키지를 설치하지 않고 바로 실행할 수 있으며, 특정 패키지가 한 번만 실행될 필요가 있을 때 npx를 사용할 수 있다.

#### **5 Technical Trends To Help Web Developers Stand Out in 2025**

<https://thenewstack.io/5-technical-trends-to-help-web-developers-stand-out-in-2025/>

1. Vanilla JS의 힘을 무시하지 말 것 - React나 Angular, Vue와 같은 프레임워크들이 많이 사용되지만, 간단한 프로젝트에 대해서 오히려 불필요한 복잡성을 가질 수 있기에 Vanilla JS를 통해 가볍고 온전히 제어 가능한 웹을 만드는 것도 좋은 방법이 될 수 있다.

2. 전통적인 형식의 포트폴리오 웹사이트 대신 Three.js를 사용한 특색 있는 웹 애플리케이션을 제작하기 - AI의 사용으로 인해 더 이상 일반적인 형태의 웹 애플리케이션은 시각적으로 이목을 끌기가 어렵다. Three.js와 같은 라이브러리를 통해 인터랙티브한 웹사이트를 만들어보자.

3. 보안 지식이 점점 중요해지고 있음 - 보안은 더 이상 옵션이 아니며, 매 해 회사에서 보안을 위해 투자하는 금액이 증가하고 있다. XSS, SQL Injection과 같은 보안적인 부분에 대해 신경쓰며 개발을 할 필요가 있다.

4. 로컬에서 AI 코딩 어시스턴트 사용하기 - AI 툴을 활용하여 코딩을 하는 것은 더 이상 특이한 일이 아니다. 하지만 민감한 데이터 등과 외부 API 의존성을 낮추기 위해 로컬에서 AI 모델을 돌려 사용하는 방법 또한 고려해볼 필요가 있다. 가장 접근성이 좋은 로컬 LLM 모델은 Ollama가 있다.

5. No-code와 low-code 툴의 힘을 무시하지 말 것 - 개발자의 입장에서 이러한 툴을 사용하는 것에 회의적인 사람들이 많지만, 다른 시각에서 이러한 툴은 개발 시간을 효율적으로 단축시키고 복잡한 로직에 더 집중할 수 있도록 돕는다.

#### **CSS 뉴스**

#### **CSS Wrapped 2024**

<https://chrome.dev/css-wrapped-2024/>

### **Javascript 뉴스**

#### **5 Javascript Libraries you should say goodbye to in 2025**

<https://thenewstack.io/5-javascript-libraries-you-should-say-goodbye-to-in-2025/>

1. JQuery - 자바스크립트의 조부모님 격이었던 JQuery가 2025년부로 지원 종료됩니다.
2. Moment.js - 날짜를 핸들링 할 때 자주 사용된 이 라이브러리는 비교적 무겁고 유연하지 못한 특징으로 인해 다른 대체 라이브러리에 밀려 2025년부로 지원이 종료됩니다.
3. Lodash - 다양한 편의 기능 지원 라이브러리인 Lodash의 대부분의 기능이 ES6에서 지원됩니다.
4. Underscore.js - Lodash와 비슷한 라이브러리로 ES6에서 이제 대부분의 기능이 지원됩니다.
5. Require.js - webpack, vite와 같은 대체 라이브러리로 인해 더 이상 지원되지 않게 되었습니다.

#### **React.js 19**

<https://react.dev/blog/2024/12/05/react-19>

#### **SOLID principles for JavaScript**

<https://blog.logrocket.com/solid-principles-javascript/>

JavaScript에서의 SOLID 원칙 준수 방법 설명

#### **Mastering Progressive Hydration for enhanced web performance**

<https://thenewstack.io/mastering-progressive-hydration-for-enhanced-web-performance/>

lazy-loading vs Progressive Hydration에 대한 차이점과 각각의 상세 동작 방식 설명

### **참고**

<https://frontender-ua.medium.com/frontend-weekly-digest-392-9-15-december-2024-5e3f9125f232>
<https://frontender-ua.medium.com/frontend-weekly-digest-391-2-8-december-2024-35fe18049f8d>