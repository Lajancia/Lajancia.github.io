---
layout: post
title: "Vanilla Javascript로 SPA 구현하기 -1"
date: 2024-01-23
excerpt: "Vanilla Javascript와 History API를 활용하여 SPA 구현 세팅을 해보기"
project: true
tag:
  - Javascript
  - SPA
  - History API
---


## 참고 사이트

[토스 기술 블로그, 토스 테크](https://toss.tech/)

[VanillaJS를 React 같이 코딩하기 - 시작하기](https://velog.io/@bepyan/Vanilla-JS를-React-같이-코딩하기)

## 라우터 구현 참고 사이트

[VanilaJS로 SPA 개발해보기 (No Framework)](https://sihus.tistory.com/49)

[Vanilla JS에서 SPA 라우팅 시스템 구현하기](https://kdydesign.github.io/2020/10/06/spa-route-tutorial/)

## 개발 과정

### Javascript에서 HTML 랜더링

```jsx
// Card.js
import '../assets/style/components/Card.css'
import Image0 from '../assets/images/Planet.jpeg';
import Image1 from '../assets/images/Architect.jpeg';
import Image2 from '../assets/images/Space.jpeg';
import Image3 from '../assets/images/TENET.jpeg';
import Image4 from '../assets/images/Winter.jpeg';

const Card = ({ app, route }) => {
    const images = [Image0,Image1,Image2,Image3,Image4]
    const content = document.createElement('div');
    content.className = "container"
    app.appendChild(content);

    const img = document.createElement('img');
    img.className = "thumbnail"
    img.src = images[route];

    const render = () => {
        content.innerHTML = `
            <a class = "contents" href="#/post/${route}">
                ${img.outerHTML}
                <div class = "posts">
                    <h1 id="title">그 많은 개발 문서는 누가 다 만들었을까</h1>
                    <h3 id="contents">토스트 테크니컬 라이터가 하는 일에 이어, 개발자 경험 전반으로 역할을 확장해온 이야기를 공유해요</h3>
                    <h4 id="date">2024. 01. 25</h4>
                </div>
                
            </a>
    `
        content.querySelector('a').addEventListener('click', (e) => {
            e.preventDefault();
            window.location.hash = `/post/${route}`;
        });
    }
    
    render();
}

export default Card;
```

```jsx
// Home.js
import Card from '../components/Card.js'
import '../assets/style/pages/Home.css'

const HomePage = ({ app }) => {
    const content = document.createElement('div')
    content.className = 'content';
    app.appendChild(content);

    const render = () => {
        content.innerHTML = `
            <h1 id = "develop">
                개발
            </h1>
        `
    }
    render();
    
    for(let i = 0; i < 5; i++) {
        const card = new Card({ app:content, route:i });
    }
}

export default HomePage;
```

위와 같이 javascript에서 HTML을 랜더링할 수 있도록 구현하였다. Card 컴포넌트가 Home.js에서 여러번 재사용되며, app을 통해 content를 넘겨 상위의 HTML 내부에 하위 컴포넌트의 HTML이 생성될 수 있도록 하였다.

### History API 를 활용한 라우팅 구현

History API를 활용하여 상태와 title, url을 지정하는 것이 가능하다.

url의 형태를 지정하는 것이 가능하지만 서버 측 지원이 필요하다는 단점이 존재한다. → 해당 경로를 새로고침 할 경우 자원을 찾지 못해 404 에러가 발생한다.

### Hash History Mode를 활용한 라우팅 구현

해시는 history가 관리되지 않는다. 하지만 history와 같이 리프레시 없이 경로를 이동할 수 있으며, hash가 변경될 때 마다 이벤트가 발생하여 hashChange를 통해 라우팅을 변경할 수 있다. 하지만 URL에 필연적으로 #을 사용해야만 하며 앞서 언급했듯 history가 관리되지 않고 상태를 전달할 수 없다.

우선 정적인 웹사이트를 개발하였기 때문에 Hash를 활용하여 라우팅을 구현해보았다.

```jsx
// Card.js
content.querySelector('a').addEventListener('click', (e) => {
            e.preventDefault();
            window.location.hash = `/post/${route}`;
        });
```

위의 Card  컴포넌트에서는 위와같이 a 태그가 클릭될 경우 리프레시를 막고 해시를 통해 원하는 포스트 경로로 갈 수 있도록 지정하였다.

```jsx
import Header from './components/Header.js'
import Footer from './components/Footer.js'
import HomePage from './pages/Home.js'
import Post from './pages/Post.js'

const App = (app) => {
    const render = () => {
        app.innerHTML = '';
        const page = window.location.hash;
        new Header({ app });
        if (page.startsWith('#/post/')) {
            const route = parseInt(page.replace('#/post/', ''));
            new Post({ app,route });
        } else {
            new HomePage({ app });
        }
        new Footer({ app });
    }

    window.addEventListener('hashchange', render);
    window.onpopstate = render;
    render();
}

export default App;
```

이를 App.js에서 감지하여 일치하는 url일 경우 해당 페이지의 요소들로 컴포넌트가 교체될 수 있도록 한다.

그러나 이러한 방식은 페이지에 상태를 전달할 수 없으며, 페이지가 많아질 경우 조건문의 길이도 길어진다. 또한 라우팅을 모듈화 할 수 없다.

## 이슈사항

`target undefined 에러 발생` 

화살표 함수 변경 시 this.target = document.createElement('div'); 에서 this가 지정되지 않아 undefined 발생

`History API로 라우팅 구현 시 페이지 새로고침 때 404 에러 발생`

이러한 문제가 발생하는 이유는 /post/0 url에서 새로고침이 이루어질 경우 서버에서는 해당 자원의 위치를 찾지 못하기 때문에 404 에러가 발생한다. 

→ 이 문제를 해결하기 위해 해시 라우팅을 구현하였다.

`Hash 라우팅 구현 시 / 경로로 돌아갈 때 #/로 경로가 지정됨`

해시 라우팅은 필연적으로 url에 #을 붙여 주소를 이동한다. 루트 경로에 가기 위해서는 기존의 History API를 활용하여 구현할 수 있다.

`Jest 사용 시 import 사용 불가`

이는 현재 babel 및 package.json의 세팅으로 인해 ECMA script의 버전 충돌이 발생하였기 때문이다. 이를 해결하기 위해 취한 방법은 아래와 같다.

```jsx
//package.json
{
  "name": "vanillajs-to-typescript",
  "version": "1.0.0",
  "description": "Vanilla javascript SPA",
  "main": "./src/index.js",
  "type": "module",
  "scripts": {
    "start": "webpack-dev-server --mode development --config ./webpack.config.js",
    "build": "webpack --mode production --config ./webpack.config.js",
    "test": "jest"
  },
  "keywords": [],
  "author": "Soomin Hwang",
  "license": "ISC",
  "devDependencies": {
    "@babel/core": "^7.23.9",
    "@babel/preset-env": "^7.23.9",
    "@testing-library/dom": "^9.3.4",
    "babel-loader": "^9.1.3",
    "css-loader": "^6.9.1",
    "file-loader": "^6.2.0",
    "html-webpack-plugin": "^5.6.0",
    "identity-obj-proxy": "^3.0.0",
    "jest": "^29.7.0",
    "style-loader": "^3.3.4",
    "webpack": "^5.90.0",
    "webpack-cli": "^5.1.4",
    "webpack-dev-server": "^4.15.1"
		"@babel/plugin-transform-modules-commonjs": "^7.23.3",
    "@babel/plugin-transform-runtime": "^7.23.9",
    "jest-environment-jsdom": "^29.7.0"
  },
	}
```

type을 module로 변경하고 필요한 babel 패키지를 설치한다.

```jsx
//babel.config.json

{
  "presets": ["@babel/preset-env"]
}
```

위와 같이 세팅을 끝내면 jest를 실행시켰을 때 import가 가능해지지만, css와 이미지 등을 import 할 시에는 또 다른 에러가 발생할 것이다. 이를 위해 설정해줘야 하는 것은 다음과 같은 jest.config.json 파일이다.

```jsx
//jest.config.json
{
  "verbose": true,
  "collectCoverage": true,
  "moduleNameMapper": {
    "\\.(css|less|scss|sass)$": "identity-obj-proxy",
    "\\.(jpg|jpeg|png|gif|eot|otf|webp|svg|ttf|woff|woff2|mp4|webm|wav|mp3|m4a|aac|oga)$": "<rootDir>/__mocks__/fileMock.js"
  },
  "testEnvironment": "jsdom"
}
```

identity-obj-proxy는 별도로 npm을 통해 설치해주어야 한다.

```jsx
// /__mocks__/fileMock.js

module.exports = {};
```

위의 세팅은 jest에서 해당 이미지나 css 등의 파일들이 js와 같이 실행시키려는 것을 방지할 수 있다.

`package.json의 type을 module로 변경 후 발생하는 “ReferenceError: module is not defined in ES module scope"에러`

이러한 문제는 코드 내에서 사용되는 require로 인해 발생하는 문제일 가능성이 크다. package.json에서 type을 module로 설정하였을 경우, require를 사용할 수 없다. 때문에 require로 작성한 코드를 import로 바꿔 에러를 해결하였다.