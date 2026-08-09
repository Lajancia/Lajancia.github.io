---
title: "Next.js 시작하기 - 간단한 Login 페이지 만들기 (3) React-Query"
date: 2024-07-06 12:00:00 +0900
categories: [Next.js]
tags: [Next.js, React, React Query, Login, MSW, API, MUI]
author: L.J
---

### **React-Query 사용하기**

지금부터 간단한 Login api를 React-query를 사용하여 구현해볼 예정이다. 앞서 언급했듯이, React-Query는 프론트엔드 api 사용에 있어 다양한 편의기능을 제공해 준다. 대표적으로 data가 갱신되면 useState를 사용할 필요 없이, 해당 data의 상태 변화를 확인하고 랜더링이 되는 기능부터, 페이지별 데이터 호출, loading 감지 등의 기능을 제공한다.

더 자세한 내용은 아래의 카카오 기술 블로그를 참고하면 좋을 것 같다.

[카카오페이 프론트엔드 개발자들이 React Query를 선택한 이유](https://tech.kakaopay.com/post/react-query-1/)

#### **어떻게 만들면 될까?**

react-query를 작성할 때 주의해야 할 점은, Get api는 useQuery를 사용해야 하며, 이외의 api는 useMutation을 사용해야 한다. 또한 이번에 만들 것은 React-query를 활용한 API 사용이기 때문에, 실질적으로 비밀번호를 암호화 하는 작업이나 토큰 발행 등의 동작이 포함되어있지는 않다 ~~(이 부분은 백엔드 API 개발자들이 처리해줄 것이기 때문에...)~~ 프론트엔드 개발자가 해야 할 일은, 추후 백엔드가 전달해준 API가 화면 상에서 어떻게 동작할 지 확인하는 것인 만큼, ***이번 구현은 화면의 동작에 초점을 맞추고 있다.***

**첫 번째로, src 폴더 하위에 utils 폴더를 생성한다.**

![](https://blog.kakaocdn.net/dna/c4pWZk/btsIphskfbS/AAAAAAAAAAAAAAAAAAAAAFuXsPYpmDEm8PoZTBf_PBu06bXXqeYLNH2vf_L0pLiI/img.png)

해당 폴더는 지금부터 api나 함수들을 넣어둘 폴더로 사용할 것이다. 이번에 만들 api는 login 에서만 사용되는 api인 만큼, utils 폴더 하위에 추가로 login.ts 파일을 생성한다. Login은 form을 통해 백엔드로 해당 아이디와 비밀번호가 올바른지 진위여부를 확인하는 용도지만, 동시에 보안적으로 중요한 기능이다. 따라서 Get 방식 보다는 Post 방식으로 만들어진 API를 전달받을 것이다.

**api 호출을 위해 사용할 라이브러리는 axios이다. 추가적으로 axios 모듈을 설치해야 한다.**

```bash
npm install axios
```

**POST /login API에 id와 password를 전달하는 loginUser 함수를 만든다.**

```tsx
// react-query를 위한 login API
import axios from "axios";
const loginUser = async (data: { id: string; password: string }) => {
  const response = await axios.post("/login", data);
  return response.data;
};
export { loginUser };
```

하지만 해당 API는 실질적으로 백엔드가 존재하지 않기 때문에 당연히 동작하지 않는다. 풀스텍 개발자라면 직접 만들어서 사용할 수 있겠지만, React-Query 사용법을 배우기 위해 백엔드 서버를 만들고 API까지 개발하기에는 너무 비효율적이다. 슬프게도 프로젝트 마감 시간은 계속해서 다가오고, 그냥 화면만 동작하게 해달라는 요청이 들어온다. API 호출 결과에 따라 유기적으로 동작하는 프론트일 경우에, 이러한 요청은 정말 난감하다.

*MSW는 이러한 상황에 처해있는 프론트엔드 개발자에게 도움의 손길을 내민다.*

**api가 요청되었을 때 msw가 가로챌 수 있게 해보자**

지난 포스트를 따라 정상적으로 개발자 도구에서 msw enable이 뜨고 있다면, 해당 기능을 사용할 준비는 거의 끝난 것이다. 이제 우리가 할 일은 handlers를 추가하는 일 뿐이다.

해당 방법은 next14버전과 msw 2버전의 호환 문제로 msw 1버전임을 다시 한 번 언급하고 지나간다.

```tsx
// src/mocks/handlers.ts
import { rest } from "msw";
export const handlers = [
  rest.post("http://localhost:3000/login", async (req, res, ctx) => {
    const { id, password } = await req.json();
    if (id === "admin" && password === "admin") {
      return res( ctx.delay(5000), ctx.status(200), ctx.json({ message: "Login successful" }));
    } else {
      return res(
       ctx.delay(5000),
        ctx.status(401), // 401 Unauthorized
        ctx.json({ message: "Invalid credentials" }),
      );
    }
  }),
];
```

로그인을 진행할 때 5초 정도 응답을 지연시켜 loading 애니메이션이 동작할 수 있게 handlers를 만들어 보았다. 추가적인 다른 handlers를 추가하고 싶다면 handlers 배열 하위에 계속해서 추가하면 된다.

id와 password가 모두 admin일 경우 login이 성공하고 아닐 경우 실패하는 결과를 돌려주도록 만들었으니, 이제 login 페이지에서 react-query를 사용해 직접 호출해볼 차례이다.

**QueryClient 세팅**

react-query가 정상적으로 동작하게 하기 위해서는 최상위 컴포넌트에서 QueryClient를 세팅해야 한다. 이를 위해서 layout.tsx에 조금 추가를 해줘야 한다.

```tsx
"use client";
import { useEffect } from "react";
import { Metadata } from "next";
import { Inter } from "next/font/google";
import "./globals.css";
import { setupWorker } from "msw";
import { handlers } from "@/mocks/handlers";
import Header from "@/components/Header";
import { QueryClient, QueryClientProvider } from "react-query";
const inter = Inter({ subsets: ["latin"] });

const queryClient = new QueryClient();

const RootLayout = ({ children }: { children: React.ReactNode }) => {
  // msw mocking
  useEffect(() => {
    if (typeof window !== "undefined") {
      const worker = setupWorker(...handlers);
      worker.start();
    }
  }, []);
  return (
    <QueryClientProvider client={queryClient}>
      <html lang="en">
        <body className={inter.className}>
          <Header />
          <div>{children}</div>
        </body>
      </html>
    </QueryClientProvider>
  );
};
export default RootLayout;
```

추가된 부분은 QueryClient 부분이다.

**react-query useMutation 사용**

post API를 사용하기 때문에 useMutation을 활용하여야 한다. ./src/app/login/page.tsx 의 Login 컴포넌트에 간단한 useMutation 함수를 만들어 보자

```tsx
import { useMutation } from "react-query";
import { loginUser } from "@/utils/login";
const loginMutation = useMutation(loginUser, {
    onSuccess: (data) => {
      alert(data.message);
    },
    onError: (error) => {
      alert(`Error: ${error}`);
    },
  }); // POST API일 경우 useMutation
```

api가 성공할 경우 onSuccess의 동작을 수행하고, 실패할 경우 onError를 수행한다.

여기까지 진행할 경우, admin을 id와 password에 입력할 경우 5초 뒤에 alert가 뜨지만, 시각적으로 loading이 되고 있는지 여부를 파악할 수 없다. 조금 더 수정을 해보자.

**react-query 에서 loading 기능 사용하기**

간단하게 MUI를 활용하여 loading 애니메이션을 추가해준다. 로딩되는 동안에는 사용자가 다른 행동을 하는 것을 막아야 하기 때문에, MUI Backdrop을 사용할 예정이다.

```tsx
"use client";
import { useEffect } from "react";
import { useForm, SubmitHandler } from "react-hook-form"; // 폼 생성을 위한 import
import { TextField, Button, Backdrop, CircularProgress } from "@mui/material"; // MUI 라이브러리
import styles from "./page.module.css";
import { useMutation } from "react-query";
import { loginUser } from "@/utils/login";

// Form에 입력된 데이터를 받을 Inputs 인터페이스 지정
interface Inputs {
  id: string;
  password: string;
}

const Login = () => {
  const {
    register,
    handleSubmit,
    watch,
    formState: { errors },
  } = useForm<Inputs>();

  const loginMutation = useMutation(loginUser, {
    onSuccess: (data) => {
      alert(data.message);
    },
    onError: (error) => {
      alert(`Error: ${error}`);
    },
  }); // POST API일 경우 useMutation

  const onSubmit: SubmitHandler<Inputs> = (data) => {
    loginMutation.mutate(data);
  };

  const watchID = watch("id");
  useEffect(() => {
    console.log(watchID); // watch를 통해 "example"에 데이터가 전달되는지 체크
  }, [watchID]);

  return (
    <>
      <Backdrop
        sx={{ color: "#fff", zIndex: (theme) => theme.zIndex.drawer + 1 }}
        open={loginMutation.isLoading}
      >
        <CircularProgress color="inherit" />
      </Backdrop>
      <form className={styles.loginForm} onSubmit={handleSubmit(onSubmit)}>
        <h1>Login</h1>
        <TextField
          defaultValue=""
          {...register("id")}
          variant="outlined"
          label="ID"
          fullWidth
          margin="normal"
        />
        <TextField
          {...register("password", { required: true })}
          variant="outlined"
          label="Password"
          type="password"
          fullWidth
          margin="normal"
          error={Boolean(errors.password)}
          helperText={errors.password && "This field is required"}
        />
        <Button type="submit" variant="contained" color="primary">
          SUBMIT
        </Button>
      </form>
    </>
  );
};
export default Login;
```

Backdrop과 CircularProgress를 사용하여 loading이 되는 동안 애니메이션이 동작하게 했다. 이 때, react-query에서 제공하는 isLoading은 api가 호출되는 동안 true, 호출 전과 완료는 false로 결과를 반환하기 때문에 간단하게 loading 여부를 체크하여 제어할 수 있다.

### 결과

![](https://blog.kakaocdn.net/dna/bQloMn/btsIpFM3BVH/AAAAAAAAAAAAAAAAAAAAAGafgxIadZ2r99pDaMoz8LJZl__Zf3JtjkRdarHvjz_y/img.png)

Console에서 정상적으로 msw가 /login api를 가로채고 동작하는 것을 확인할 수 있다.

지금이야 간단하지만, 매번 .isLoading .isSuccess를 덧붙여서 사용하거나, 추가적으로 useMutation에 key를 부여하여 자동으로 data가 변경될 때 마다 감지하게 하기 위해 key를 사용하는 등, 하나의 파일에서 관리하기에는 react-query 함수는 매우 너무 복잡해질 수 있다.

**때문에 이러한 관리의 복잡성을 해결하고 생성한 react-query를 분리해서 관리하는 방법에 대해 다음 포스트에서 이야기해 볼 것이다.**