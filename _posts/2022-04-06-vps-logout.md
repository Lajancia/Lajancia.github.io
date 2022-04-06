---
layout: post
title: "VPS - 5"
date: 2022-04-06
excerpt: "VPS 페이지 새로고침 및 logout 이슈 해결"
project: true
tag:
  - nodejs
  - vue
  - js
  - Hostinger
  - vps
  - vuex
---

# VPS-5

---

## VPS 시작할 때

```jsx
You need to run "nvm install N/A" to install it before using it.
```

이 뜨는 경우 실행 순서에 대하여 기록.

### 1. nvm list 확인

```jsx
nvm list
nvm use 14 // 기존에 있는 node가 14버전 기준으로 설치
```

이미 설치되어 있는 상태인지 다시 한 번 확인해보기

### 2. pm2 상태 확인

```jsx
pm2 status를 통해 현재 실행 상태를 확인한다.
```

pm2에서 특정 id를 중지하거나 삭제하는 과정은 이전 VPS-3을 참고

## VPS login VUEX 이슈 해결하기

> 로그인 시스템을 만들었는데 페이지를 새로고침하면 로그아웃을 하지 않았음에도 로그인 상태가 아닌 guest로 뜬다. 로그인은 그대로 되어있지만, vue 프레임워크가 현재의 상태를 제대로 기억하지 못하고 있는 것이다.

### 해결 방법

가장 일반적인 이슈이기에, 아주 간단한 해결법이 있다.

```jsx
import Vue from "vue";
import Vuex from "vuex";
import createPersistedState from "vuex-persistedstate";

Vue.use(Vuex);

export default new Vuex.Store({
  plugins: [
    createPersistedState({
      storage: window.sessionStorage,
    }),
  ],
  state: {
    user: null,
  },
  getters: {
    user: (state) => {
      return state.user;
    },
  },
  mutations: {
    setUser(state, user) {
      state.user = user;
    },
  },
});
```

- 기존의 store/index.js 에 vuex-persistedstate와 plugins를 추가하면 된다. 플러그인을 이용하면 간단하게, 페이지가 리프레시 되어도 자동으로 현재 상태를 복구하여 계속 로그인상태인 것을 보여주게 만들 수 있다.

### 문제

> 세션 상태를 복구하는 것 까지는 좋은데....api를 통해 로그아웃을 해도 계속 vuex가 로그인상태를 복구해버리는 문제가 생겼다. 이로 인해 로그아웃이 되지 않는다...

```jsx
methods: {
  handleLogout() {
  console.log("it works1")
  this.$store.commit("setUser", null);
  console.log("it works2")
    this.$http
      .get("/api/auth/logout")
      .then(() => {
        console.log('this is front')
        console.log(this.$store.state)
      //  console.log(this.$store.state.user)
        console.log('this os end');
      })

  },

}
```

> 위의 문제를 해결하기 위해서 해야 할 것은 vuex의 state를 null로 바꾸고, 세션을 끊어 로그아웃 상태로 만들어야 한다.

두 가지 기능을 해야 하기 때문에 따로 함수를 생성하였다. v-on:click 으로 버튼이 활성화되면 handleLogout()함수를 통해 두 가지 일을 수행하게 만들었다. 단순하게 this.$store.state=null로 만들어서 해결하려 했지만, 완전히 초기화 상태로 만들 수 없기 때문에 위와 같이 setUser을 이용하여 user값에 null을 넣어주면 잘 동작한다.

이제 페이지를 새로고침 해도 로그인상태를 유지하고, 로그아웃 시에도 세션을 삭제하고 state가 초기화되어 재로그인이 되는 문제가 해결되었다.
