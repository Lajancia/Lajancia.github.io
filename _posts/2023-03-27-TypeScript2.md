---
layout: post
title: "TypeScript Study - 2"
date: 2023-03-27
excerpt: "타입스크립트 사용 방법 공부하기"
tags: [TypeScript, JavaScript]
comments: false
---

# 2. TypeScript

---

## 타입스크립트만의 특별한 타입

> 타입스크립트의 핵심은 타입 체커와 소통하는 것이다.
> 

### Unknown

```
let a:unknown; 
let b=a+1 //에러 발생

if (typeof a==='number'){
	let b=a+1
}//동작

if(typeof a==='string'){
	let b = a.toUpperCase();
}

```

### void

```
function hello(){
	console.log('x')
}
```

- void는 아무것도 주지 않는 값
- 따로 쓸 필요는 없다.

### never

```
function hello():never{
	throw new Error('xxx')
}
```

- 무조건 에러를 발생시킨다
- 거의 안씀

### or 연산

```
function hello(name:string|number){
	if(typeof name==='string'){
		name //type string
	}else if (typeof name==='number'){
	name //type number
	}else{
	name //type never ->실행되면 안되는 코드
	}
}
```

## call signatures

```
function add(a:number,b:number){
	return a+b
}

//화삺표
const add = (a:number,b:number)=>a+b
//이때 호출될 함수의 인자의 타입과 반환 타입을 알려주는 call signature 선언하기

type Add = (a:number, b:number) => number;
const add:Add = (a,b) =>a+b
```

## overloading

다른 사람들이 만든 외부 모듈에 오버로딩을 하는 일이 많음

```tsx
type Config = {
	path:string,
	state:object
}
type push = {
(path:string):void,
	(config:Config):void
}

const push:Push = (config)=>{
	if(typeof config === 'string'){
 console.log(config)
}else{
	console.log(config.path,config.state)
}
}
```

```tsx

//타입의 개수가 다를 경우 추가되는 변수는 옵션이다.
type Add = {
	(a:number, b:number):number,
	(a:number, b:number,c:number):number
}

const add:Add = (a,b,c?:number)=>{
	if(c) return a+b+c
	return a+b
}

add(1+2)
add(1+2+3)

//둘 다 가능
```

- 여러개의 call signatures를 가지고 있을 때 사용

## Polymorphism(다형성)

```tsx
//기존
type SuperPrint = {
	(arr:number[]):void
	(arr:boolean[]):void
	(arr:string[]):void
}

const superPrint : SuperPrint = (arr) =>{
	arr.forEach(i=>console.log(i))
}

superPrint([1,2,3,4])
superPrint([true,false,true])
superPrint(['a','b','c'])

//generic

type SuperPrint = {
	<TypePlaceholder>(arr:TypePlaceholder[]):void //알아서 타입 추측
	
}

const superPrint : SuperPrint = (arr) =>{
	arr.forEach(i=>console.log(i))
}

superPrint([1,2,3,4])
superPrint([true,false,true])
superPrint(['a','b','c'])
```

## generic Recap

```tsx
//좋지 않은 예시
type SuperPrint = (a:any[])=>any
	
const superPrint : SuperPrint = (arr) => a[0]
superPrint([1,2,3,4])
superPrint([true,false,true])
superPrint(['a','b','c'])

//좋은 예시, generic recap 예시
type SuperPrint = <T>(a:T[])=>T
	
const superPrint : SuperPrint = (arr) => a[0]
superPrint([1,2,3,4])
superPrint([true,false,true])
superPrint(['a','b','c'])

//2개 이상 generic
type SuperPrint = <T,M>(a:T[],b:M)=>T
	
const superPrint : SuperPrint = (arr) => a[0]
superPrint([1,2,3,4],"x") //true
superPrint([true,false,true])//error
superPrint(['a','b','c'])//error

//function 사용
function superPrint<T>(a:T[]){
	return a[0]
}

superPrint([1,2,3,4])
superPrint([true,false,true])
superPrint(['a','b','c'])

//extra object
type Player<E> = {
	name:string
	extraInfo:E
}
type NicoExtra={
	favFood:string
}

type NicoPlayer = Player<NicoExtra>

const nico:NicoPlayer<>{
	name:"nico"
	extraInfo:{
		favFood:"kimshi"
}
}
```