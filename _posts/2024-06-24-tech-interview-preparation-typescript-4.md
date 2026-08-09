---
title: "기술 면접 준비 - Typescript 기본 (4)"
date: 2024-06-24 12:00:00 +0900
categories: [개발]
tags: [TypeScript, 기술 면접, 프론트엔드]
author: L.J
---

### **다형성**

다형성이란 제네릭과 같이 특정 객체가 하나의 타입이 아닌 여러 가지 타입으로 표현될 수 있어서 재사용 가능한 형태로 구현될 수 있는 것을 말한다.

#### **타입 시그니쳐**

```typescript
function functionName (parameter1: type, parameter2: type): returnType {
  return value;
}
```

함수, 서브루틴, 메소드 등의 입력과 출력을 정의하는 것.

#### **호출 시그니처**

```typescript
type Operation = {
  (a: number, b: number): number;
};
```

#### **인터페이스와 타입의 차이**

객체 타입을 정의할 때는 interface를 사용하는게 좋고, 단순 원시값이나 튜플, 유니언 타입을 선언할 때는 type을 사용하는 것이 좋다.

#### **infer 키워드**

infer는 조건부 타입 중 하나로 조건에 따라 타입을 정의할 수 있는 기능이다. infer를 통해 제네릭 파라미터의 타입을 추론할 수 있다.

#### **as 키워드**

타입 단언을 수행하는데 사용되는 키워드다. 개발자가 컴파일러에 특정 변수에 대한 타입 힌트를 주는 것으로 컴파일 단계에서 타입스크립트가 추론하지 못하는 타입을 as 키워드를 통해 명시해주는 것이다.

#### **타입 가드 (Type Guard)**

변수의 타입을 좁히는데 사용하는 메커니즘이다.

- **typeof A**: A 타입을 문자열로 반환
- **A instanceof B**: B의 프로토타입 체인에 A가 포함되었는지 boolean 반환
- **a in A**: A의 속성중에 a가 있는지 boolean 반환

#### **unknown 타입**

모든 타입을 허용한다. any 타입과는 다르게 프로퍼티 또는 연산을 하는 경우 컴파일러가 체크한다.

#### **never 타입**

타입스크립트에서 never 타입은 값의 공집합이다. bottom type이라고도 불린다.

### **SOLID 원칙**

특히 OCP (Open-Closed Principle): 소프트웨어 엔티티는 확장에 대해서는 열려있어야 하지만 변경에 대해서는 닫혀 있어야 한다.
