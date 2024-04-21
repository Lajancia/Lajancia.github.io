---
layout: post
title: "Vanilla Javascript로 SPA 구현하기 -3"
date: 2024-02-10
excerpt: "Jest 테스트 케이스 작성"
project: true
tag:
  - Javascript
  - SPA
  - JEST
---


## 이전 시간 이슈

### /posts/0 새로고침 시 404 에러 발생

라우팅이 제대로 되었음에도 불구하고 해당 페이지를 새로고침할 경우에는 계속해서 404 에러가 발생하는 것을 확인할 수 있었다. 이는 바닐라 자바스크립트를 통해 구현했기 때문에 발생하는 문제가 아니라 react의 라우터에서도 충분히 발생할 수 있는 문제였다.

결론만 말하자면 webpack devserver에서 별도로 설정해주었어야 할 문제였다.
```
  devServer: {
    historyApiFallback: true,
    static: path.join(__dirname, 'dist'),
    port: 9000,
  },
};
```
다음과 같이 historyApiFallback 설정을 해줘야 한다.

[SPA에서 뒤로가기, 새로고침 시 404 Not Found 오류 해결](https://darrengwon.tistory.com/245)

[velog](https://velog.io/@wjdska245/react-router-dom-정상경로-404에러-발생-해결)

```jsx

output: {
        filename: 'index.js',
        path: path.resolve(__dirname, 'dist'),
        publicPath: '/',
    },
devServer: {
        historyApiFallback: true,
        static: path.join(__dirname, "dist"),
        port: 9000
	}
```

기본적으로 새로고침은 반드시 해당 경로에 대한 요청을 서버에 전달하고, 응답을 반환한다. 이때 서버에서 별도로 url에 대한 모든 경로가 index.html, index.js를 향하게 하지 않을 경우에는 path에 해당하는 파일 경로의 index.html을 찾는다. 여태 에러가 발생한 이유는 계속해서 새로고침을 할 때 /index.html이 아닌, /posts/0/index.html을 찾기 때문에 발생한다.

historyApiFallback은 개발 서버에서 요청된 모든 경로에 대해 항상 index.html 파일을 제공하도록 한다.

이는 react-dom-router에서도 발생할 수 있는 문제이기 때문에 webpack을 통해 설정해야 한다. 실 서버에서는 해당 서버가 기본으로 /index.html을 가리키도록 설정하면 된다.

## Jest 테스트케이스 작성

- 일반적인 경우, 하나의 테스트는 하나의 expect를 가지는 것이 바람직하다고 한다. (목적이 뚜렷하고, 테스트의 목적이 명확하다는 것)
- 테스트는 최대한 컴포넌트 단위와 모듈 단위로 하도록 한다. 상황에 따라 모듈/기능 위주의 테스트를 진행하는 것 또한 좋다.

```tsx
import Router from './Router';

describe('Router 모듈 동작 확인', () => {
  test('라우터에 따른 랜더 함수 실행 테스트', () => {
    const { location } = window; // window 저장
    console.log = jest.fn();
    const router = new Router();
    router.addRoute('', () => console.log('initial route'));
    router.addRoute('/', () => console.log('initial route'));
    router.loadInitialRoute();

    delete window.location;
    Object.defineProperty(window, 'location', {
      value: {
        href: '/',
        pathname: '/',
      },
      writable: true, // 재정의 가능하게 설정
    });

    expect(console.log).toHaveBeenCalledWith('initial route');
    window.location = location; // 재정의되었던 window 복구
  });

  test('에러 페이지 라우팅 테스트', () => {
    const { location } = window;
    Object.defineProperty(window, 'location', {
      value: {
        href: 'http://localhost:9000/',
        pathname: 'http://localhost:9000/',
      },
      writable: true, // 재정의 가능하게 설정
    });

    console.log = jest.fn();
    const router = new Router();
    router.addRoute('', () => console.log('initial route'));
    router.addRoute('/', () => console.log('initial route'));
    router.addRoute(null, () => console.log('Error Page Render'));
    router.loadInitialRoute();

    delete window.location;
    Object.defineProperty(window, 'location', {
      value: {
        href: 'http://localhost:9000/noPageExist',
        pathname: 'http://localhost:9000/noPageExist',
      },
      writable: true, // 재정의 가능하게 설정
    });

    expect(console.log).toHaveBeenCalledWith('Error Page Render');
    window.location = location;
  });

  test('쿼리 스트링 테스트', () => {
    const { location } = window;
    Object.defineProperty(window, 'location', {
      value: {
        href: 'http://localhost:9000/test?id=0&name=apple',
        pathname: 'http://localhost:9000/test?id=0&name=apple',
      },
      writable: true, // 재정의 가능하게 설정
    });

    const router = new Router();
    router.addRoute('', () => console.log('initial route'));
    router.addRoute('/', () => console.log('initial route'));
    router.addRoute('/test', () => console.log('test route'));
    router.addRoute(null, () => console.log('Error Page Render'));
    router.loadInitialRoute();

    expect(router.getData().name).toBe('apple');
    window.location = location;
  });
});
```

위와 같이 테스트 케이스를 작성하였다. window.location을 사용할 경우, 다음 테스트에도 영향을 주기 때문에 다음과 같이 테스트가 완료되면 원래의 location으로 되돌려 놓아야 다음 테스트에 문제가 생기지 않는다.