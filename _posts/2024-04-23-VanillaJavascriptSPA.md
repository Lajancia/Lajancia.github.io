---
layout: post
title: "Vanilla Javascript로 SPA 구현하기 -4"
date: 2024-02-10
excerpt: "Vanilla Javascript로 구현된 부분 Typescript로 마이그레이션 하기"
project: true
tag:
  - Javascript
  - SPA
  - JEST
  - Typescript
---


## Typescript 마이레이션

**고려사항**

1. any 타입 없애기
2. as 타입 들어내기

### Jest에서 window.location 사용하기

[Jest에서 window 환경 커스텀 설정 (window, navigator, etc.)](https://somedaycode.tistory.com/18)

다양한 방법을 시도해 보았지만 기존의 설정을 그대로 유지한 채 window.location 부분만 변경하는 것은 여전히 제대로 동작하지 않았다.

때문에 다음과 같이 window.location을 조작하기 전에 먼저 해당 상태를 임시로 저장해둔 후, 다시 원상복귀 하는 방식으로 다른 테스트케이스에서 조작된 window.location 의 영향을 받지 않도록 했다.

```tsx
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
```

## ESLint+Prettier 설정

[VSCode에서 ESLint와 Prettier (+ TypeScript) 사용하기](https://velog.io/@das01063/VSCode에서-ESLint와-Prettier-TypeScript-사용하기)

요약하면 다음과 같다.

1. ESLint extension 설치
2. Prettier extension 설치
3. npm ESLint 설치
    
    ```json
    // .eslintrc.json
    {
      "parser": "@typescript-eslint/parser",
      "plugins": ["prettier"],
      "extends": [
        "plugin:@typescript-eslint/recommended",
        "prettier/@typescript-eslint",
        "plugin:prettier/recommended",
        "prettier"
      ],
      "parserOptions": {
        "ecmaVersion": 2018,
        "sourceType": "module"
      },
      "rules": {
        "prettier/prettier": "error",
        "@typescript-eslint/explicit-function-return-type": "off"
      }
    }
    
    ```
    
4. npm Prettier 설치
    
    ```json
    // .prettierrc
    {
      "semi": true,
      "trailingComma": "all",
      "singleQuote": true,
      "printWidth": 80,
      "tabWidth": 2
    }
    ```
    
5. setting.js에서 typescript 허용하기
6. text editor에서 default editor을 prettier로 설정하기

하지만 일반적인 과정으로는 도무지 저장했을 때 자동으로 포매팅이 되는 방식이 적용되지 않았다. 때문에 최후의 수단으로 visual studio Code의 버전을 업그레이드해보는 것을 시도한 결과, macOS에서는 Application 디렉토리에 vsCode가 존재하지 않을 경우, 업데이트 진행이 불가능하다는 것을 알게 되었다. ~~(VScode는 최초로 맥북을 산 이후로 한 번도 업데이트가 된 적이 없었던 것이다!)~~

이를 해결하기 위한 방법은 단순히 vsCode를 Application 디렉토리에 옮겨주면 된다. 이동 후에 VsCode를 restart 하고 업데이트를 실행하면 prettier가 정상적으로 적용된 것을 확인할 수 있다.

## 타입스크립트 세팅

기존의 js와 함께 사용중이었던 만큼, ts, js  모두 실행될 수 있는 환경이 필요했다.

이를 위해 jest, webpack,, tsconfig에 대하여 몇가지 수정이 이뤄졌다.

### jest.config.json

```tsx
{
  "verbose": true,
  "collectCoverage": true,
  "moduleNameMapper": {
    "\\.(css|less|scss|sass)$": "identity-obj-proxy",
    "\\.(jpg|jpeg|png|gif|eot|otf|webp|svg|ttf|woff|woff2|mp4|webm|wav|mp3|m4a|aac|oga)$": "<rootDir>/__mocks__/fileMock.js"
  },
  "testEnvironment": "jsdom",
  "moduleFileExtensions": ["js", "ts", "tsx", "json"],
  "transform": {
    "^.+\\.(ts|tsx)$": "ts-jest",
    "^.+\\.js$": "babel-jest"
  },
  "globals": {
    "ts-jest": {
      "tsconfig": "tsconfig.json"
    }
  }
}
```

### webpack.config.js

```tsx
import path from 'path';
import HtmlWebpackPlugin from 'html-webpack-plugin';
import { fileURLToPath } from 'url';

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

export default {
  mode: 'development',
  entry: './src/index.ts', // Update this if your entry point is now a .ts file
  output: {
    filename: 'index.js',
    path: path.resolve(__dirname, 'dist'),
    publicPath: '/',
  },
  resolve: {
    extensions: ['.tsx', '.ts', '.js'], // Add .ts and .tsx here
    alias: {
      '@': path.resolve(__dirname, 'src/'),
    },
  },
  module: {
    rules: [
      { test: /\.tsx?$/, loader: 'ts-loader' }, // Add this rule
      {
        test: /\.js$/,
        enforce: 'pre',
        loader: 'source-map-loader',
      },
      {
        test: /\.css$/,
        use: ['style-loader', 'css-loader'],
      },
      {
        test: /\.(woff(2)?|ttf|eot|svg|jpg|jpeg|png|glb|gltf)(\?v=\d+\.\d+\.\d+)?$/,
        use: [
          {
            loader: 'file-loader',
            options: {
              name: '[name].[ext]',
              outputPath: 'assets/',
            },
          },
        ],
      },
      {
        test: /\.js$/,
        exclude: /node_modules/,
        use: {
          loader: 'babel-loader',
        },
      },
    ],
  },
  plugins: [
    new HtmlWebpackPlugin({
      template: './src/index.html',
    }),
  ],
  devServer: {
    historyApiFallback: true,
    static: path.join(__dirname, 'dist'),
    port: 9000,
  },
};
```

tsx와 ts가 사용될 수 있도록 환경 세팅을 한 뒤, 기존의 javascript 코드를 하나씩 typescript로 변경 진행하였다. 

처음에는 모든 매개변수와 함수에 대하여 타입을 선언했는데, 타입 추론을 적극적으로 이용하자.

또한 interface로 타입을 선언하는 것도 좋지만, type에서 제공하는 기능이 더 실용적인 것 같다.(코드에 호버할 때 선언된 타입이 보이는 기능 등)