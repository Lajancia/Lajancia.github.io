---
title: "React Native Layout 시스템"
date: 2025-02-12 08:00:00 +0900
categories: [개발]
tags: [ReactNative, Layout, Flexbox, Expo, 모바일개발]
author: L.J
---

## React Native -2

### Layout 시스템 알아보기

React 웹과 React Native의 레이아웃은 거의 유사하게 동작하지만 분명하게 차이점이 존재한다. 가장 대표적인 예시로는 'display:flex'가 있다.

웹에서는 Flex Direction을 설정하기 위해서는 기본적으로 display를 flex로 설정한 뒤에 해당 row, 혹은 column을 세팅할 수 있지만, React native에서는 별도의 display 설정 없이 flex direction을 곧바로 선언할 수 있다는 것이 큰 특징이다. **기본적으로 React native의 View는 Flex Container라고 생각해두면 좋다.**

**/app 폴더 하위 index.tsx를 수정해보자**

```tsx
export default function HomeScreen() {
  return (
    <View>
      <View style={{ width: 200, height: 200, backgroundColor: 'red' }}></View>
      <View
        style={{ width: 200, height: 200, backgroundColor: 'orange' }}
      ></View>
      <View style={{ width: 200, height: 200, backgroundColor: 'blue' }}></View>
    </View>
  );
}
```

위의 코드의 경우 아래와 같이 나타난다.

![](https://blog.kakaocdn.net/dna/bceyRk/btsMgvMtXYT/AAAAAAAAAAAAAAAAAAAAAIfTeaqgk0jMzW_OBH-HzlAudDnZxCUnHRINDZ0BhBDb/img.png)

이제 해당 코드를 row direction으로 설정해보자.

```tsx
export default function HomeScreen() {
  return (
    <View style={{ flexDirection: 'row' }}>
      <View style={{ width: 200, height: 200, backgroundColor: 'red' }}></View>
      <View
        style={{ width: 200, height: 200, backgroundColor: 'orange' }}
      ></View>
      <View style={{ width: 200, height: 200, backgroundColor: 'blue' }}></View>
    </View>
  );
}
```

![](https://blog.kakaocdn.net/dna/G0Yum/btsMgkYLNCy/AAAAAAAAAAAAAAAAAAAAAP1Gq2dg6oEwrZ9fBxaAN0-edjZ6MGBvjzRP_vbQMwc3/img.png)

앞서 설명한 것과 같이 동일하게 display: flex를 하지 않아도 row로 나타나는 것을 볼 수 있다. 또한 react native에서는 **default로 flex-direction이 column으로 설정되어 있다는 것을 잊지 말자!**

#### 모바일 Layout의 주의사항

대부분의 경우 모바일 애플리케이션의 레이아웃 개발에 있어서는 width와 height를 직접적으로 설정하는 일은 거의 없다. 사이즈가 기종마다 다르다보니 기본적으로 반응형을 생각해야 한다.

react-native에서는 직접적인 사이즈 설정 대신 flex를 통해 비율 설정을 할 수 있다.

```tsx
export default function HomeScreen() {
  return (
    <View style={{ flex: 1 }}>
      <View style={{ flex: 1, backgroundColor: 'red' }}></View>
      <View style={{ flex: 1, backgroundColor: 'orange' }}></View>
      <View style={{ flex: 1, backgroundColor: 'blue' }}></View>
    </View>
  );
}
```

![](https://blog.kakaocdn.net/dna/pQMbu/btsMgSAEeX0/AAAAAAAAAAAAAAAAAAAAACDqj1_o7R1P3_j32zjfYYWECHKqnxwZFIpnBctnaf1r/img.png)

부모 태그에서 flex를 지정하고 하위 자식에 대한 비율을 설정해줄 수 있다.

### **프로젝트 세팅해보기**

react-native를 최초로 세팅하고 난 뒤에 파일이 조금 많이 만들어져있는 것을 확인할 수 있다. 모두 예제를 위한 파일들이기 때문에 실제 개발을 할 때는 불필요한 파일들이다. 때문에 프로젝트 클론 코딩을 시작하기에 앞서, 개발을 위한 파일 세팅을 진행해야 할 필요가 있다.

```bash
npm run reset-project
```

해당 명령어를 실행하면 개발에 불필요한 파일들을 정리해준다.

#### 상단 Navigation 제거

![](https://blog.kakaocdn.net/dna/bAOszu/btsMg5GJXk6/AAAAAAAAAAAAAAAAAAAAAGKkqLjIlh9OJ-58QZDyDDQZ4LLCtpxsW_wk0TA4xVJj/img.png)

react-native에서 기본적으로 상단 navigation 이름이 계속해서 나타나는 것을 볼 수 있다. index로 파일명이 적혀있는 것을 확인할 수 있는데, 해당 부분이 default로 나타나는 것을 원치 않기 때문에 지워줄 것이다. 방법은 다음과 같이 _layout.tsx를 수정하는 것이다.

```tsx
import { Stack } from 'expo-router';

export default function RootLayout() {
  return (
    <Stack
      screenOptions={{
        headerShown: false,
      }}
    >
      <Stack.Screen name="index" />
    </Stack>
  );
}
```

위와 같이 headerShown을 false로 하면 전역적으로 상단 navigation을 제거할 수 있다.

![](https://blog.kakaocdn.net/dna/sO1ie/btsMftoLYGX/AAAAAAAAAAAAAAAAAAAAADT1Y87rOuFZ_l-MHXltAui1A_g4auFXZt9alGVL_5I0/img.png)

깔끔해졌다!

현재 Nomad coder에서 진행된 react-native와 현재 버전이 변화가 생겨 이러한 UI 부분과 관련한 차이점이 발생하는 듯 하다. 동영상에서는 위와 같은 default 헤더가 있지 않았기 때문이다. 심지어 코드를 App.js에 작성하는데 필자의 경우에는 index.tsx에서 개발을 진행하였다. 아무래도 그새 꽤 많은 업데이트가 진행된 모양이니 강의를 진행하면서 동일한 화면이 나오지 않을 경우에는 변경된 업데이트 사항에 대해서 확인하고 직접 방법을 찾아봐야 할 듯 하다...