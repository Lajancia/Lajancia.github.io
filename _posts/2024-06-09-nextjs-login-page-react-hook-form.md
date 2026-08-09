---
title: "Next.js 시작하기 - 간단한 Login 페이지 만들기 (2) React Hook Form"
date: 2024-06-09 12:00:00 +0900
categories: [개발]
tags: [Next.js, React Hook Form, MUI, Login]
author: L.J
---

### **MUI App Bar로 레이아웃 구성**

모든 페이지가 공유하는 App Bar를 layout.tsx에 배치한다.

```tsx
// src/components/Header
import { AppBar, Box, Toolbar, Typography, Button, IconButton } from '@mui/material';
import MenuIcon from '@mui/icons-material/Menu';

const Header = () => {
  return (
    <Box sx={{ flexGrow: 1 }}>
      <AppBar position="static">
        <Toolbar>
          <IconButton size="large" edge="start" color="inherit" aria-label="menu" sx={{ mr: 2 }}>
            <MenuIcon />
          </IconButton>
          <Typography variant="h6" component="div" sx={{ flexGrow: 1 }}>
            News
          </Typography>
          <Button color="inherit">Login</Button>
        </Toolbar>
      </AppBar>
    </Box>
  );
};
export default Header;
```

### **React Hook Form을 사용한 로그인 폼**

```tsx
"use client";
import { useForm, SubmitHandler } from "react-hook-form";
import { TextField, Button } from "@mui/material";

interface Inputs {
  id: string;
  password: string;
}

const Login = () => {
  const { register, handleSubmit, watch, formState: { errors } } = useForm<Inputs>();

  const onSubmit: SubmitHandler<Inputs> = (data) => {
    console.log("login proceed", data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <TextField defaultValue="" {...register("id")} variant="outlined" label="ID" fullWidth margin="normal" />
      <TextField {...register("password", { required: true })} variant="outlined" label="Password" type="password" fullWidth margin="normal" error={Boolean(errors.password)} helperText={errors.password && "This field is required"} />
      <Button type="submit" variant="contained" color="primary">SUBMIT</Button>
    </form>
  );
};
export default Login;
```

#### **React Hook Form 주요 기능**

1. **register**: input을 react-hook-form에 등록
2. **watch**: input 변화 감지
3. **handleSubmit**: form 성공/실패 시 실행될 함수 등록
4. **formState**: form의 상태 (주로 error 핸들링에 사용)
