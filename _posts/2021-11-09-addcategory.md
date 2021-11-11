---
layout: post
title: "jekyll moon 테마에 카테고리 추가법"
date: 2021-11-09
excerpt: "jekyll moon 테마에서 원하는 카테고리를 새로 생성하고 마크다운을 등록하는 법 정리."
tags: [jekyll, blog]
comments: false
---

# jekyll moon 테마에 카테고리 추가법-check

2021-11-09

---

> 여러 사이트를 찾아보아도 제대로 카테고리 추가와 포스트 작성에 대한 설명이 없어서 직접 찾아낸 방법 작성하였다. 제대로된 방법인지는 잘 모르겠지만 작동은 잘 되니 그냥 사용한다.

## 1. \_data>navigation.yml에 사이트 네비게이션 추가

```
- title: Blender
  url: /Blender/
```

- 새로 추가하려는 카테고리의 사이트 네비게이션을 추가한다. 기존에 project가 생성되어 있을 것이다. 동일한 형식으로 추가해주자.

## 2. \_layouts 파일 내부에 카테코리 html 생성

```html
---
layout: compress
---

<!DOCTYPE html>
<!--[if lt IE 7]><html class="no-js lt-ie9 lt-ie8 lt-ie7"> <![endif]-->
<!--[if (IE 7)&!(IEMobile)]><html class="no-js lt-ie9 lt-ie8"><![endif]-->
<!--[if (IE 8)&!(IEMobile)]><html class="no-js lt-ie9"><![endif]-->
<!--[if gt IE 8]><!-->
<html class="no-js">
  <!--<![endif]-->

  <head>
    {% include head.html %}
  </head>

  <body>
    {% include nav.html %}
    <!-- Header -->
    <header class="header" role="banner">
      <div class="wrapper animated fadeIn">
        <div class="content">
          <div class="post-title">
            <h1>{{ page.title }}</h1>
            <a class="btn zoombtn" href="{{site.url}}">
              <i class="fa fa-home"></i>
            </a>
          </div>
          <div class="post-list">
            {% for post in site.posts %} {% if post.Blender %}
            <ul>
              <li class="wow fadeInLeft" data-wow-duration="1.5s">
                <a class="zoombtn" href="{{ site.url }}{{ post.url }}"
                  >{{ post.title }}</a
                >
                <p>{{ post.excerpt }}</p>
                <a href="{{ site.url }}{{ post.url }}" class="btn zoombtn"
                  >Read More</a
                >
              </li>
            </ul>
            {% endif %} {% endfor %}
          </div>
        </div>
      </div>
    </header>
    {% include scripts.html %}
    <script src="{{ site.url }}/assets/js/wow.min.js"></script>
    <script type="text/javascript">
      new WOW().init();
    </script>
  </body>
</html>
```

- 생성하려는 카테고리 이름이 Blender이기 때문에 Blender.html을 생성하고 위의 코드를 넣어준다.
- 이때 다른 카테고리 명을 할 것이라면 post-list 클래스 아래의 {% if post.Blender %}에서 Blender 부분을 다른 것으로 대체한다.

## 3. 상위 폴더 안에 Blender 파일 생성.

상위폴더>Blender폴더>index.md 생성

```
---
layout: Blender
title: Blender studies
excerpt: "Studying blender"
comments: false
---
```

- layout은 꼭 설정한 카테고리 명과 똑같이 해주어야 한다.

## 4. post 작성

```
---
layout: post
title: "2. blender 단축키 정리 - 업데이트"
date: 2021-10-19
excerpt: "blender 단축키 설명"
Blender: true
tags: [blender, key]
comments: false
---
```

- 기존의 post 카테고리에도 마크다운을 포함시킬 것이라면 layout: post를 추가하고, 그것이 아니라면 쓰지 않아도 된다.
- Blender 카테고리에 포함시키기 위해서는 위와 같이 `Blender:true` 라고 작성해야 한다.