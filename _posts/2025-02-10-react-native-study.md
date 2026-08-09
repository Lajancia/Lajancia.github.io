---
title: "React Native 스터디"
date: 2025-02-10 08:00:00 +0900
categories: [개발]
tags: [ReactNative, Expo, NomadCoder, 모바일개발, React]
author: L.J
---

## React Native 스터디 -1

### **시작하기**

대세가 모바일 개발이라는 것은 스마트폰의 등장 시절부터 이야기된 부분이기도 하지만, 최근 여러 공고들을 보면서 확실히 프론트엔드 개발자 자격 요건 중에 React-native를 다룰 수 있는 요건이 많이 보이고 있었다. 아무래도 React를 사용하던 개발자에게 있어서 React native의 접근성이 좋다 보니 둘 다 할 수 있는 개발자를 많이 찾는 듯 했다. Next14도 다루고 있는데 React Native까지 해야 할 판이었지만, 그래도 같은 React 파생 상품(?)이라서 다행일 따름이었다. React로 웹개발도 하고 VR도 만들고 AR도 만들더니 모바일 앱도 만들 수 있다. React는 만능이었다.

### **뉴비를 위한 무료 React Native 찍먹 강의**

오래 전부터 이런 새로운 기술들은 Nomad Coder에서 한번씩 찍먹을 해왔다. 이번에도 역시 니꼬쌤이 우리를 위한 무료 강의를 열어두셨다.

<https://nomadcoders.co/react-native-for-beginners/lobby>

처음 React Native를 하는 만큼 해당 강의를 통해 배워볼 생각이다.

#### **환경 세팅**

모바일 앱 개발을 하기 위해서는 필수적으로 에뮬레이터와 java 설치가 요구되지만, 지금은 앱을 직접 출시하는 것 보다는 react-native를 어떻게 사용할 수 있는지가 더 큰 목표이기 때문에 이러한 귀찮은 절차는 생략되고 expo를 통해 곧바로 스마트폰에서 테스트할 수 있는 환경을 마련할 것이다.

가장 먼저 expo 계정을 만들자

<https://expo.dev/>

다음으로 이미 node.js가 설치된 상태라면 다음의 명령어를 통해 쉽게 설치할 수 있다.

```bash
npx create-expo-app@latest
```

설치 후에는 모바일 어플리케이션에서 Expo go 어플을 깔아주어야 한다.

### **React 와 React native**

익숙한 코드 구조와 파일들이 보이지만, React Native에서는 react와 달리 몇가지 룰이 존재한다.

1. 기존에 사용하던 div, span 과 같은 태그를 사용할 수 없다. 대신 react-native에서 View와 Text를 import해서 사용한다.
2. 텍스트는 기본적으로 Text 태그에 넣어서 사용해야 한다.
3. Style은 거의 대부분 React에서 사용하는 css를 그대로 사용할 수 있지만 특정 태그, border와 같은 태그들은 사용이 불가하다.

<https://reactnative.dev/>

그런데 이상한 점이 있다. 버전이 점점 올라갈수록 React Native docs에 있는 components와 API들이 점점 줄어들고 있다는 것이다. tag를 import 해오는 것도 모두 react-native에서 가져오는 것이 아니라 외부 라이브러리에서 가져오는 것들이 많은데, 대표적으로 StatusBar이 있다. 왜 이렇게 된 것일까?

#### **초심**

React Native팀도 처음에는 필요한 모든 컴포넌트들을 React-Native 기반으로 제공하려 했던 시도들이 이전 버전들에 유산처럼 남겨있는 것을 확인할 수 있다. 하지만, 이는 곧 React native 자체의 유지 관리와 업데이트가 느려진다는 것을 뜻하기도 한다. 때문에 React-native팀은 이러한 APIs, components의 서비스 규모를 줄이고 핵심 기능들만 제공하는 쪽으로 변경되었고 사라진 기능들은 커뮤니티에서 개발된 것들로 채워지게 되었다.

이러한 형태로 운영되는 것은 기능의 확장에 최적화되어 개발자들의 입장에서도 선택지가 늘어난다는 좋은 점도 있지만, 문제는 공식적으로 지원되지 않는 기능이라는 치명적인 단점이 있다. 보안이 필수적일 경우에는 보안적 이슈가 언제든지 발생할 잠재적 위험성을 가지고 있다는 것이기도 하기 때문이다. 때문에 이러한 문제로 인해 Flutter를 더 선호하는 모바일 앱 개발자도 많은 모양이다.

하지만 React 개발자의 입장에서는 React Native야말로 최고의 선택지가 아닐 수 없었다.

#### **Expo Go와의 콜라보**

React Native의 공식 문서에서 제공되는 Components들은 샘플 예제를 확인할 수 있는 QA코드가 존재한다. 이것을 휴대전화 카메라로 스캔하면 모바일에 설치되어있는 Expo Go를 통해 해당 예제가 어떻게 동작하는지를 확인할 수 있다.

<https://reactnative.dev/docs/vibration>

하지만 실제로 앱을 만들기 위해서 필요한 기능들은 현저히 부족한 상황이다. 이를 해결하려면 외부 라이브러리를 가져다 사용해야 한다. 우리가 사용할 수 있는 여러 라이브러리들은 여기서 확인할 수가 있다.

<https://reactnative.directory/?search=storage>

커뮤니티를 통해 무수한 기능들을 제공받을 수 있지만, 동시에 기능들을 커뮤니티에 의존해야 한다는 단점 또한 존재한다. 하지만 이러한 핵심 기능이 React-native에서 자체적으로 제공되지 않는 부분은 Expo팀에게도 과연 부담이 된 모양이었는지 Expo SDK를 통해 자체적으로 기능을 만들어 제공하고 있다.

<https://docs.expo.dev/versions/latest/sdk/expo/>

React-native에서 제공되지 않는 기능들은 대부분 Expo에서 제공하고 있다.

당분간 테스트 개발을 Expo Go를 최대한 활용해서 진행하고 추후 프로젝트가 완성되면 실제 애뮬레이터로 한번 동작시켜보도록 하자.