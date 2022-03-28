---
layout: post
title: "VPS-4"
date: 2022-03-15
excerpt: ".htaccess를 이용하여 외부에서 다이랙트로 서브 페이지 접속하기"
project: true
tag:
  - nodejs
  - vue
  - js
  - Hostinger
  - vps
---

# VPS-4

---

## vue+history mode 이슈

아무리 찾아도 도통 이유를 알 수 없었던 이슈... 그것은 바로 멀쩡하게 잘 돌아가는 페이지를 다시 리프레시 하면 404에러가 뜨는 문제였다. hostinger에서만 일어나는 일은 아닌 듯 하지만 분명 무언가가 도메인 네임 뒤에 붙은 폴더를 곧바로 접속하려 할때 막아버려서 들어가지지 않는 것 같았다.

### 해결방법

```jsx
<IfModule mod_rewrite.c>
  RewriteEngine On
  RewriteBase /
  RewriteRule ^index\.html$ - [L]
  RewriteCond %{REQUEST_FILENAME} !-f
  RewriteCond %{REQUEST_FILENAME} !-d
  RewriteRule . /index.html [L]
</IfModule>
//루트 디랙토리에 있는 파일들이 페이지를 새로고침해도 404 에러가 뜨지 않게 리다이랙트 해주는 코드
```

```jsx
<IfModule mod_rewrite.c>
RewriteEngine on

RewriteCond %{HTTP_HOST} ^(www.)?lajanciadev.com$
RewriteCond %{REQUEST_URI} !^/dist/

RewriteCond %{REQUEST_FILENAME} !-f
        RewriteCond %{REQUEST_FILENAME} !-d

RewriteRule ^(.*)$ /dist/$1

RewriteCond %{HTTP_HOST} ^(www.)?lajanciadev.com$

RewriteRule ^(/)?$ dist/index.html [L]
</IfModule>
//위의 코드는 public에서 dist로 루트 파일을 변경하는 .htaccess 코드이다.
```

```jsx
//두 코드를 적절히 잘 섞어서 dist 파일로 root 경로를 설정하는 동시에 리다이랙트로 외부 url을 방문하고 되돌아와도 404에러가 뜨지 않게 만든 코드
<IfModule mod_rewrite.c>
RewriteEngine on
RewriteBase /dist/
RewriteRule ^(.*)$ /dist/$1
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule ^(/)?$ dist/index.html [L]
</IfModule>
```

한참을 찾아다닌 해결방법. 어쩌면 당연하게도 .htaccess를 사용하여 도메인 네임 뒤에 붙은 라우터 페이지들에도 잘 접속될 수 있도록 설정해주어야 했다. 본인의 경우, hostinger에서 apache를 사용하여 위와 같이 작성해주었지만, 다를 경우에는 코드 또한 달라지는 것 같으니 아래 참고 사이트를 확인해보는 것을 추천한다.

## 참고

[How to serve VueJS history mode on shared hosting server in subdirectory](https://www.nuomiphp.com/eplan/en/26609.html)

[Different History modes | Vue Router](https://router.vuejs.org/guide/essentials/history-mode.html#apache)
