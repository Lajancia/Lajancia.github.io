---
layout: post
title: "Hostinger+git 연결하기"
date: 2022-02-27
excerpt: "shared hosting과 깃허브 연결"
tags: [study, shared_hosting, hostinger]
comments: false
---

# Hostinger+git 연결

---

[How to Add SSH Keys to Your GitHub Account](https://www.inmotionhosting.com/support/server/ssh/how-to-add-ssh-keys-to-your-github-account/)

[](http://ajitgupta.in/deployment/automatic-deployment-of-a-website-on-hostinger-using-github/)

- 이때 directory is not empty 에러가 발생한다면, hostinger 파일 관리에서 public_html 내부의 모든 파일을 삭제하여 비워주도록 한다.

## Issue

정상적으로 깃허브에 연결이 되었지만 public_html에 모든 파일이 올라가버려 정상적으로 웹페이지가 배포되지 않았다.

이 문제는 내가 현재 사용하는 서비스가 shared hosting 이기 때문에 기본적으로 root directory를 수정하는 것을 허용하지 않는다. 이 기능을 이용하기 위해서는 vps 를 이용해야 한다는데, 지금 내가 부담하기에는 금액대가 비싸다...

## Solution

그러나 방법은 있다. .htaccess 파일을 이용하여 웹페이지에서 사용할 디렉토리를 바꿔주는 것이다.

[How to change the document root directory](https://www.a2hosting.com/kb/developer-corner/apache-web-server/changing-the-document-root-directory)

```jsx
# .htaccess main domain to subfolder redirect
RewriteEngine on

RewriteCond %{HTTP_HOST} ^(www.)?example.com$
RewriteCond %{REQUEST_URI} !^/subfolder/

RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d

RewriteRule ^(.*)$ /subfolder/$1

RewriteCond %{HTTP_HOST} ^(www.)?example.com$

RewriteRule ^(/)?$ subfolder/index.html [L]
```

이 코드의 subfolder에 빌드된 웹페이지가 들어있는 폴더 명을 입력하고 examples.com에 웹사이트명을 작성한다.

## Issue-2

무슨 이유에서인지 deploy가 될때마다 파일이 정상적으로 불러와지지 않는다.

여러가지 시도를 해보았지만 깃허브에 push가 되면 동작하지 않는다...

수동으로 deploy를 할 때 이런 문제가 일어나는 건지 확인해볼 필요가 있다.

## Solution

deploy 문제가 아니라 .htaccess 파일의 문제였다. 위의 코드만으로는 js 파일을 가져올 때 발생하는 404 에러를 막지 못한다.

```jsx
<IfModule mod_rewrite.c>
RewriteEngine on

RewriteCond %{HTTP_HOST} ^(www.)?website$
RewriteCond %{REQUEST_URI} !^/subfile/

RewriteCond %{REQUEST_FILENAME} !-f
        RewriteCond %{REQUEST_FILENAME} !-d

RewriteRule ^(.*)$ /subfile/$1

RewriteCond %{HTTP_HOST} ^(www.)?website$

RewriteRule ^(/)?$ subfile/index.html [L]
</IfModule>
```

위의 subfile에 파일명을 적고 website에 자신의 웹사이트를 작성하면 정상적으로 모든 파일들이 동작한다.

이제 깃허브에 push를 하면 웹사이트가 자동적으로 깃허브의 리파지토리를 통해 코드를 읽어내어 웹사이트를 업데이트한다.
