---
layout: post
title: "3. vue.js + express calendar 만들기"
date: 2021-09-28
excerpt: "웹사이트에서 사용할 달력을 v-calendar를 이용하여 제작한다"
project: true
tag:
  - nodejs
  - vue
  - express
  - backend
---

# 3. vue.js 웹페이지 구현

---

> 달력을 구글 캘린더와 연동해야 한다.

> 구글 자체에서 제공해주는 코드가 있지만, 예쁘지 않기 때문에 다른 방법을 찾아보아야 한다.

> vuetify에서 제공하는 달력이 있지만, 미리 설치하지 않으면 기존의 프로젝트 파일을 변경하는 문제가 있다. 이제서야 사용하려니 너무 늦음...

## Fullcalendar+google calendar 연동

참고 동영상

[](https://11q.kr/g5s/bbs/board.php?bo_table=co4&wr_id=555)

> fullcalendar를 이용하여 구글api와 연동해서 사용하려고 했지만 계속되는 충돌로 인해 다른 켈린더를 사용하기로 했다.

> vue에서 vuetify가 아닌 v-calendar 또한 사용 가능하다. 이걸 사용하면 다른 파일의 변경 없이 사용 가능하다.

## V-Calendar 사용하기

[Layouts | V-Calendar](https://vcalendar.io/layouts.html#full-width)

사용법이 아주 친절하게 나와있다.

현재 내가 사용하고 있는 vue는 3이기 때문에

```python
npm install v-calendar@next
```

위를 이용하여 설치해주도록 한다.

main.js

```python
import { createApp } from 'vue'
import ElementPlus from 'element-plus'
import 'element-plus/dist/index.css'
import App from './App.vue'
import './assets/tailwind.css'
import axios from 'axios'
import VueAxios from 'vue-axios'
import VCalendar from 'v-calendar';

// createApp(App).mount('#app')

const app = createApp(App)
app.use(VueAxios, axios)
app.use(ElementPlus)
app.use(VCalendar, {})
app.mount('#app')
```

v-calendar를 사용 가능하도록 위와같이 기존의 코드에 VCalendar를 추가한다.

필요한 기능

- 일정이 기록된 부분에 dot 표시 되기
- 오늘 날짜 하이라이트
- 큰 달력

Calendar.vue

```python
<template>
  <div>
    <v-calendar is-expanded :attributes="attrs" />
    <!-- <v-date-picker v-model="date" /> -->
  </div>
</template>

<script>
export default {
  data() {
    return {
      attrs: [
        { key: "today", highlight: true, dates: new Date() },
        {
          dot: "red",
          key: "today",
          dates: new Date(2021, 8, 1),
          highlight: {
            backgroundColor: "#ff8080",
          },
          // Just use a normal style
          contentStyle: {
            color: "#fafafa",
          },
          // Our new popover here
          popover: {
            label: "You just hovered over today's date!",
          },
        },
      ],
    };
  },
};
</script>
```

- 위의 파일은 App.vue에서 Calendar 에 해당하는 바에서 나타날 예정이다.

## 완성된 화면

![Screenshot 2021-09-24 at 17.53.15.jpg](3%20vue%20js%20%E1%84%8B%E1%85%B0%E1%86%B8%E1%84%91%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%8C%E1%85%B5%20%E1%84%80%E1%85%AE%E1%84%92%E1%85%A7%E1%86%AB%203dd0d75db1134a11b50ceaa452a6d1c7/Screenshot_2021-09-24_at_17.53.15.jpg)

- 할 일이 표시된 1 위에 마우스를 hover 하면 텍스트가 나타난다.
- 이제 google 아니면 naver 달력에서 일정 데이터를 추출해내는 방법을 알아내어, 일정이 있는 날짜에 자동으로 텍스트가 등록되고 날짜가 표시될 수 있도록 연결해주어야 한다.
- 데이터를 연결하기 위해서 google calendar에서 스케쥴 데이터를 json으로 가져오는 방법과 json 사용 방법을 알아봐야 할 것 같다.
