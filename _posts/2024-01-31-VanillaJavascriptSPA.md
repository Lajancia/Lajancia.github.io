---
layout: post
title: "Vanilla Javascript로 SPA 구현하기 -2"
date: 2024-01-31
excerpt: "router 모듈화하기"
project: true
tag:
  - Javascript
  - SPA
  - History API
---

## 코드 리뷰

- 모듈을 만드는 것을 목표로 하기(history API 활용)
- state를 관리할 수 있는 구조의 컴포넌트를 만들기
- devDependency 명확히 구분하기
- modules.css 사용하기
- id 를 class 로 변경해서 사용하기
- issue 관리 및 브랜치 관리
- sonnar cloud에서 코드 리뷰 확인하고 코드 스멜 개선해보기
- jestTest 지우기
- 테스트 케이스 다양하게 작성해보기

## history API를 활용한 라우터 구현

[How to build a router in vanilla JavaScript](https://accreditly.io/articles/how-to-build-a-router-in-vanilla-javascript)

[Vanilla Javascript로 구현하는 SPA - 라우팅](https://www.woong-jae.com/post/spa-from-scratch-2)

[How to build a Router with vanilla JavaScript](https://dev.to/kodnificent/how-to-build-a-router-with-vanilla-javascript-2a18)

[How to build a router in vanilla JavaScript](https://accreditly.io/articles/how-to-build-a-router-in-vanilla-javascript#content-creating-a-router)

## React-dom-router 동작 구조

[velog](https://velog.io/@nasikun/react-router-dom-v6-동작-원리의-모든-것)

1. **Router (라우터)**: 어플리케이션을 감싸고 라우팅 기능을 제공. 라우팅을 위한 컨텍스트를 생성하고, 다른 화면 간에 이동을 돕는다.

2. **Route (라우트)**: URL 경로와 React 컴포넌트 간의 매핑을 정의하는 데 사용. 지정된 경로와 현재 URL이 일치하는 경우 해당 컴포넌트를 렌더링

3. **Switch (스위치)**: 여러 개의 route 컴포넌트를 감싸는 데 사용. 현재 URL에 일치하는 첫 번째 Route 또는 Redirect 만 렌더링

4. **Link (링크)**: 애플리케이션 내에서 다른 화면으로 이동할 수 있는 링크를 생성하는 데 사용

## 라우터 모듈 만들기

```jsx
// router/Router.js
export default class Router {
    constructor() {
        this.routes = [];
        this.data = {};
        
        // history api에서 경로가 변경될 경우 새로고침 없이 재랜더링 수행
        window.addEventListener('popstate', () => this.loadInitialRoute());
    }

    addRoute(path, renderTemplate) {
        this.routes.push({ path, renderTemplate });
    }

    // 현재 url 경로 추출
    _getCurrentURL() {
        const path = window.location.pathname;
        return path;
    }

    // 동적으로 생성된 routes에서 해당 경로가 존재하는지 확인
    _matchUrlToRoute(urlSegs) {
        try {
            const matchedRoute = this.routes.find(route => {
            const routePathSegments = route.path.split('/').filter(Boolean);
            const currentPathSegments = urlSegs.filter(Boolean);

            if (routePathSegments.length !== currentPathSegments.length) {
                return false;
            }

            return routePathSegments.every((routePathSegment, index) => {
                return routePathSegment.startsWith(':') || routePathSegment === currentPathSegments[index];
                });
            });
            return matchedRoute;
        } catch {
            return false;
        }
        
    }

    // path 추출 및 해당 경로에 대한 컴포넌트 랜더링
    loadInitialRoute() {
        const pathnameSplit = this._getCurrentURL().split('/');
        const pathSegs = pathnameSplit.length > 1 ? pathnameSplit.slice(1) : '';
        this._loadRoute(...pathSegs);
    }

    _loadRoute(...urlSegs) {
        const matchedRoute = this._matchUrlToRoute(urlSegs);
        if (!matchedRoute) {
            const routeWithNullPath = this.routes.find(route => route.path === null);
            routeWithNullPath.renderTemplate();
            
        } else {
            matchedRoute.renderTemplate();
        }
    }

     // 해당 경로 history에 push
    navigateTo(path) {
        const pathnameSplit = this._getCurrentURL();
        if (path !== pathnameSplit) {
            window.history.pushState(null, '', path);
            const popStateEvent = new PopStateEvent('popstate', { state: null });
            dispatchEvent(popStateEvent);
        } else {
            console.log('no render')
        }
    }
}
```

마지막 navigateTo 함수를 사용하여 원하는 path를 spa 방식으로 이동할 수 있도록 한다.

```jsx
import HomePage from './pages/Home.js'
import Post from './pages/Post.js'
import Error from './components/Error.js'
import Router from './router/Router.js'

const routes = ($element) => {
    const router = new Router;
    router.addRoute('', () => new HomePage({ $element, router }));
    router.addRoute('/', () => new HomePage({ $element, router }));
    router.addRoute('/posts/:id', () => new Post({ $element, router }));
    router.addRoute(null, () => new Error({ $element,router}));
    router.loadInitialRoute();
    return router
};

export default routes;
```

동적으로  생성되는 /posts/:id가 정상적으로 처리될 수 있도록 라우터 모듈에서 현재 path의 : 앞 부분과 등록된 라우터 경로가 일치하는지 여부를 _matchUrlToRoute에서 확인한다.

routes에서 생성된 라우터들은 컴포넌트에서 재사용할 수 있도록 return 하였다.

## 컴포넌트 만들기

[Vanilla Javascript로 웹 컴포넌트 만들기 | 개발자 황준일](https://junilhwang.github.io/TIL/Javascript/Design/Vanilla-JS-Component/#_1-상태관리의-탄생)

### 기존 컴포넌트 방식

```jsx
import Header from './components/Header.js'
import Footer from './components/Footer.js'
import Router from './router/Router.js'
import routes from './routes.js'

const App = ($app) => {
    const header = document.createElement('header');
    const main = document.createElement('main');
    const footer = document.createElement('footer');

    $app.appendChild(header);
    $app.appendChild(main);
    $app.appendChild(footer);

    const render = () => {
        const router = new Router(routes(main));
        router;
        header.innerHTML = '';
        footer.innerHTML = '';

        new Header({ $app: header });
        new Footer({ $app: footer });
    }

    render();
}

export default App;
```

기존의 컴포넌트 방식으로는 router 모듈과 함께 사용하기가 어렵다. 

또한 모듈을 활용하여 생성된 상태가 아닌, 컴포넌트 자체로 생성된 상태이기 때문에 재사용을 위해 모듈의 형태로 다시 개발할 필요가 있다.

### 변경된 컴포넌트 모듈

```jsx
// core/Components.js

export default class Component {
    // 생성자 -> 할당된 $app에 컴포넌트를 삽입할 div 생성
    constructor ($element) { 
        this.$container = document.createElement('div');
        $element.appendChild(this.$container);
        this.$element = this.$container;
    }

    // 탬플릿에 대한 추가 컴포넌트 세팅 함수
    setTemplate() { }

    // 아벤트 셋업
    setup() { }
    
    // 랜더링할 template 지정
    template() {
        return '';
    }

    // 랜더링 수행
    render() {
        this.$element.innerHTML = this.template();
        this.setTemplate();
        this.setEvent();
    }

    // 랜더링 수행 이후 추가적으로 수행해야 할 작업
    setEvent() { }
    
    setState (newState) {
        this.state = { ...this.state, ...newState };
        this.render();
    }
}
```

### Home.js

```jsx
import Card from '../components/Card.js'
import Component from '../core/Components.js'
import styles from '../assets/css/Home.module.css'

const NUMBER_OF_CARD = 5;

export default class HomePage extends Component{
    constructor({ $element,router }) {
        $element.innerHTML = '';
        super($element);
        this.router = router;
        this.render()
        
    }

    template() {
        return `
            <h1 class = "${styles.develop}">
                개발
            </h1>
        `
    }

    setTemplate() {
        for (let i = 0; i < NUMBER_OF_CARD; i++) {
            this.card = new Card({ $element: this.$element, router:this.router, id: i });
        }
    }
}
```

## 이슈사항

- 여전히 새로고침 시에 404 에러가 발생하는 문제를 보인다.
- jest 테스트케이스 작성 시 클릭 이벤트에 따른 url 이동 테스트를 가능하게 해야 한다.
- 404 에러 페이지 추가가 필요하다.
- 쿼리스트링에 대한 처리가 되어있지 않다.