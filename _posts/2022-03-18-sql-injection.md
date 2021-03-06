---
layout: post
title: "Hacking-2"
date: 2022-03-18
excerpt: "SQL Injection"
tags: [study, security, hack, SQL]
comments: false
---

# SQL Injection

---

[SQL Injection](https://www.hacksplaining.com/exercises/sql-injection#/sql-injection-vulnerability)

## SQL Injection이란?

로그인을 수행할 때 비밀번호를 sql 데이터베이스에서 합불을 판단하는 동안, 패스워드를 sql에 바로 주입하여 확인하는 경우, 치명적인 보안 취약을 야기할 수 있다. 말 그대로 SQL에 inject(주입)할 때 발생하는 문제다.

아주 단순하게 `' or 1=1--` 를 입력하면 비밀번호를 유추하지 않아도 간단하게 뚫을 수 있게 된다. 이와 같은 — 문자는 데이터베이스로 하여금 나머지 sql 스테이트를 무시할 수 있게 한다. 현재 개발중인 웹페이지에도 적용되는지 한번 테스트해봐야겠다.

## SQL Injection 막기

[Protecting Against SQL Injection](https://www.hacksplaining.com/prevention/sql-injection)

이러한 형태의 해킹은 Injection attack이라고 말한다. 잘못된 입력을 통해 의도하지 않은 실행을 야기하는 것이다. 가장 흔한 형태의 인터넷 공격이라고 볼 수 있다.

은행 등의 서비스에서 이러한 공격이 일어날 경우, 피해가 매우 클 수 있다.

### 이러한 해킹...

- 민감한 중요 정보 빼내기에 사용된다.
- 웹 사이트에 등록된 사용자의 인증 세부 정보를 열람한다.
- 데이터베이스 테이블을 드롭하거나 삭제할 수 있다.
- 치명적인 바이러스성 코드를 주입할 수 있다.

~~의외로 이런 해킹은 매우 흔하다고 한다...~~

## 방어하기

**결과적으로 SQL 주입은 심각한 문제를 야기할 수 있다. 이러한 문제를 사전에 방지하려면 어떻게 해야 할까?**

### Parameterized Statement

프로그래밍 언어는 `데이터베이스 드라이버` 를 통해 SQL 데이터베이스와 소통한다. 이러한 드라이버는 어플리케이션이 데이터베이스에서 SQL을 실행하고 개발하는 것을 허용해준다. 이때 변수화된 스테이트먼트를 이용함으로서 변수가 SQL에서 안전하게 사용될 수 있도록 한다.

~~파이썬과 노드를 자주 사용하니까 두 가지 위주로 정리...~~

### 파이썬의 경우

```python
# 안전한 코드
# SQL and parameter is sent off separately to the database driver.
cursor.execute("select user_id, user_name from users where email = ?", email)

for row in cursor.fetchall():
  print row.user_id, row.user_name

# 위험한 코드# String concatenation is vulnerable.
cursor.execute("select user_id, user_name from users where email = '%s'" % email)

for row in cursor.fetchall():
  print row.user_id, row.user_name
```

## Node.js의 경우

```python
## 안전한 코드 node-sql
var sql = require('sql');

// Queries are constructed as parameterized by default.
var query = user.select(user.star()))
                .from(user)
                .where(
                   user.email.equals(email)
                 ).toQuery();

## 안전한 코드 my-sql
var mysql = require('mysql');

var connection = mysql.createConnection({
  host     : HOST,
  user     : USERNAME,
  password : PASSWORD
});

connection.connect();

// Query and parameters passed separately.
connection.query(
  'select * from users where email = ?',
  [email],
  function(err, rows, fields) {
    // Do something with the retrieved data.
  });

connection.end();
```

- 인프런 노드 버드에서 학습했던 mysql 연결 api의 형태가 안전한 형태의 mysql로 작성되어 있었기에 sql injection을 해보았을 때 별달리 문제가 없었던 모양이다.
- 추후에 코드와 한번 비교해두도록 하자.

## 다른 고려 사항

### 권한을 최소화하기

어플리케이션이 각각의 프로세스에 접근할 때 필수적인 요소들을 필요한 것들만 영향을 미치도록 하는 것이 중요하다. 제한된 권한 설정을 하는 것도 인잭션 공격을 막는 좋은 방법 중 하나이다.

[Principle Of Least Privilege](https://www.hacksplaining.com/glossary/principle-of-least-privilege)

### password hashing

비밀번호를 해시화하고 유저로 하여금 강한 비밀번호 설정과 원 웨이 해시 등을 통해 유저의 정보를 훔치는 것을 방지하도록 한다.

### Third party Authentication

구글, 페이스북, 카카오톡과 같은 제 3의 플렛폼에서 제공하는 인증 수단을 사용하는 것이다. 혼자 개발하는 과정에서 필요한 모든 인증 과정과 보안을 신경쓰는 것 보다, 이러한 인증 수단을 사용하는 쪽이 훨씬 간편하고 안전하다.
