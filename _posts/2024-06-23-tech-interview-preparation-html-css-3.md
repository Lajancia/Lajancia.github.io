---
title: "기술 면접 준비 - HTML, CSS 기본 (3)"
date: 2024-06-23 12:00:00 +0900
categories: [개발]
tags: [HTML, CSS, 기술 면접, 프론트엔드]
author: L.J
---

### **HTML, CSS**

#### **!Doctype의 역할**

#### **메타 태그**

```html
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<meta name="description" content="페이지에 대한 요약 설명">
<meta name="keywords" content="키워드를 콤마로 구분하여 나열">
```

#### **Box Model**

모든 HTML 요소는 box 형태의 영역을 가지고 있다. box는 content, padding, border, margin으로 구성되어 있다.

#### **Semantic Web**

기계가 이해할 수 있는 형태로 웹을 구성한 것. non-semantic 요소는 div, span이고 semantic 요소는 form, table, img, header, nav, aside, section, article, footer 등이 있다.

### **Browser Rendering 최적화**

**DOM 트리와 Render 트리**: DOM 트리는 HTML의 모든 요소를 담고 있지만 Render 트리는 실제로 화면에 구현되는 요소들만으로 구성된다. display: none은 Render 트리에서 제외된다.

**리플로우와 리페인트**: 리플로우는 기하학적 변화 시 발생하며 리페인트를 수반한다. 리페인트는 색상 등 픽셀 변화 시 발생한다.

**Transform의 translate**: position과 달리 GPU를 사용하여 reflow, repaint를 발생시키지 않는다.

#### **이벤트 버블링과 캡쳐링**

**버블링**: 하위 요소에서 상위 요소로의 이벤트 전파 방식
**캡쳐링**: 상위 요소에서 하위 요소로의 이벤트 전파 방식
**이벤트 위임 (Event Delegation)**: 상위 요소에서 하위 요소의 이벤트를 제어하는 방식

#### **스크립트 불러오기**

**async**: DOM에 접근하는 스크립트에는 권장되지 않음, 실행 순서가 보장되지 않음
**defer**: 모든 DOM이 로드된 후에 실행, 선언한대로 실행 순서 보장

#### **CLS (Cumulative Layout Shift)**

페이지 콘텐츠가 예기치 않게 이동하는 현상. 리소스가 비동기식으로 로딩되거나 DOM 요소가 동적으로 추가되어 발생한다.
