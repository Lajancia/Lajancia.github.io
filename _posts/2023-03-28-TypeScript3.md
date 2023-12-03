---
layout: post
title: "TypeScript Study - 3"
date: 2023-03-28
excerpt: "타입스크립트 사용 방법 공부하기"
tags: [TypeScript, JavaScript]
comments: false
---
# 3. Typescript

---

## Class 생성

```tsx

class Player{
	constructor(
		private firstName:string,
		private lastNmae:string,
		public nickname:string
	){}
}

const nico = new Player("nico","las","니꼬")
```

### 추상클래스

추상클래스는 다른 클래스가 상속받을 수 있는 클래스이다.

추상 클래스의 인스턴스는 만들 수 없다. 오직 다른 곳에서 상속받고 난 다음 사용할 수 있다.

```tsx
abstract class User{
	constructor(
		private firstName:string,
		private lastNmae:string,
		public nickname:string
	){}
	getFullName(){
		return `${this.firstName} ${this.lastNmae}`
	}
}

class Player extends User{

}
const nico = new Player("nico","las","니꼬")

nico.getFullName()
```

이러한 상속, public, private는 자바스크립트에서는 문제없이 모두 동작되지만 타입스크립트에서는 에러를 발생시켜 잘못된 사용을 막는다.

### private vs protected

```tsx
private : 다른 자식 클래스에 extend 되어도 사용 불가
protected : private 와 동일하게 외부에서 사용 불가하지만 자식 클래스로 사용될 경우 사용 가능
```

## 추상 메소드

```tsx
abstract class User{
	constructor(
		private firstName:string,
		private lastNmae:string,
		public nickname:string
	){}
	getFullName(){
		return `${this.firstName} ${this.lastNmae}`
	}
	abstract getNickName():void //추상 메소드
}

class Player extends User{

}
const nico = new Player("nico","las","니꼬")

nico.getFullName()
```

추상 메소드와 같이 call signature만 있을 경우, 추상 클래스를 상속받는 클래스에서 추상 메소드를 구현해주어야 한다.

- public
- private
- protected
- abstract

→ 자바스크립트에서 제대로 동작하지 않는 기능들이다.

```tsx
type Words = {
	[key:string]:string
}

let dict:Words = {
		"potato":"food"
}

class Dict{
	private words:Words
	constructor()(
		this.words={}
	)
	add(word:Word){
		if(this.words[word.term] === undefined){
			this.words[word.term]=word.def;
		}
	}
	def(term:string){
		return this.words[term]
	}
}

class Word{
	constructor(
		public term:string,
		public def:string
	){}
}

const kimchi = new Word("kimchi","한국의 음식")
const dict = new Dict()

dict.add(kimchi);
dict def("kimchi")
```

→ property의 이름을 모르지만 타입은 알 경우 위와 같이 사용