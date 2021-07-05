---
layout: post
title: "깃허브 블로그 커스터마이징"
date: 2021-07-06
excerpt: "jekyll chirpy로 테마 변경."
tags: [github, jekyll, blog]
comments: false
---

# 깃허브 블로그 테마 적용

---

## 깃허브 블로그 테마 적용

기존의 minimal-mistake 테마가 마음에 들지 않아 테마를 새로 적용하게 되었다. 내가 사용한 테마는 jekyll Chirpy 테마이다. 문제는 이전에 다른 테마를 적용하면서 rbenv를 설치했는데, 이 테마는 2.5 이상 3 미만의 ruby 버전이 호환 가능하다. 따라서 ruby의 버전을 다운그레이드 해야 했다. (2021년 7월 기준)

```jsx
Install ruby-build
rbenv install 2.7.3
rbenv install --list
rbenv global 2.7.3
```

위의 코드를 실행하고 난 뒤 테마를 새로 적용했다. 깃허브를 클론하거나 파일을 다운받아 직접 넣어주거나 둘 중 하나를 하면 된다.

```jsx
git clone https://github.com/cotes2020/jekyll-theme-chirpy.git #클론할 경우
```

이후 gem dependencies 설치와 초기화를 진행한다.

```jsx
$ bundle
$ bash tools/init.sh
```

위의 방법들은 이미 이전에 다른 테마를 적용했다가 변경하는 방법이기에, 새로 테마를 적용하는 경우라면

[Github blog 만들기 (chirpy theme)](http://blog.ju-ing.com/posts/Github-blog-%EB%A7%8C%EB%93%A4%EA%B8%B0-chirpy-theme/)

위의 사이트를 참고하면 좋다고 생각한다.
