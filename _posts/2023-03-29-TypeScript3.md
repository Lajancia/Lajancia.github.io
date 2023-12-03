# 4. Typescript

---

```tsx
class Word{
	constructor(
		public term:string,
		public def:string
	){}
}

kimchi.def = "xxx" ->가능
```

이때 term과 def를 public으로 설정했다.

그러나 이러한 설정은 수정도 가능하게 만든다.

→ 값을 보여주지만 수정은 불가능하게 만드려면 어떻게 해야 할까?

```tsx
class Word{
	constructor(
		public readonly term:string,
		public readonly def:string
	){}
}

	kimchi.def = "xxx" ->불가능
```

각각의 값에 readonly 값을 만들면 된다.

주로 누군가가 데이터 값을 덮어쓰지 못하게 하기 위해서 사용

## static

자바스크립트에서도 존재하는 타입이다.

## Interface

### 지금까지 사용했던 타입 사용법

```tsx
type Nickname=string
type Health = number
type Friends = Array<string>

type Player - {
	nickname:string,
	healthBar:number,
}
const nico:Player = {
	nickname:"Nico",
	healthBar:10
}

type Food = string;
const kimchi:Food="delicious"
```

### 이런것도 가능

```tsx
type Nickname=string
type Health = number
type Friends = Array<string>

Interface Player {
	nickname:string,
	healthBar:number,
}
const nico:Player = {
	nickname:"Nico",
	healthBar:10
}

type Food = string;
const kimchi:Food="delicious"
```

인터페이스는 오브젝트의 모양을 타입스크립트에 설명해 주기 위해서만 사용 가능하다. 별도의 단일 타입 지정은 하지 못한다.

type이 더 범용적인 특징을 가졌다.

```tsx
interface User{
	name:string
}
interface User{
	lastname:string
}
interface User{
	health:number
}

const nico:User{
 name:"nico"
	lastname:"lee"
health:10
}

//인터페이스는 축적의 기능이 있다.
```

## abstract class

```tsx
abstract class User{
	constructor(
		protected firstName:string,
		protected lastName:string	
	){}
	abstract sayHi(name:string):string
	abstract fullName():string
}

class Player extends User{
	fullName(){
		return `${this.firstName} ${this.lastname
	}
	sayHi(name:string){
		return `Hello ${name}.My name is ${this.fullName()}
	}
}
```

자바스크립트에서는 추상 클래스의 개념이 없다. 따라서 자바스크립트에서는 class로 바뀐다.

### abtract → interface

```tsx
Interface class User{
	
	firstName:string,
	lastName:string	
	sayHi(name:string):string
	fullName():string
}

class Player implements User{
	constructor(
		public firstName:string.
		public lastName:string
	){}
	fullName(){
		return `${this.firstName} ${this.lastname}
	}
	sayHi(name:string){
		return `Hello ${name}.My name is ${this.fullName()}
	}
}
```

abstract와 동일하게 동작하지만, 문제는 인터페이스는 private property들을 사용하지 못한다.

이때 type도 abstract를 대체할 수 있다.

## Polymorphism

```tsx
interface SStorage<T>{
	[key:string]:<T>
}

class LocalStorage<T>{
	private storage:SStorage<T>={}
	set(key:string,value:T){
		this.storage[key]=value;
	remove(key:string){
		delete this.storage[key]
{
	get(key:string):T{
return this.storage[key]
}	
}
}

const stringsStorage = new LocalStorage<string>()
stringStorage.get("ket")
stringsStorage.set("hello","how are you")
const booleansStorage = new LocalStorage<boolean>();
booleansStorage.get("xxx")

```