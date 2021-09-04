---
layout: post
title: "이진탐색"
date: 2021-09-04
excerpt: "파이썬 이진트리"
tags: [백준, 파이썬, 알고리즘, 이진탐색]
comments: false
---

# 이진 탐색

---

> 백준 1654번 랜선 자르기

## 기본 풀이법

**이진탐색**

이진 탐색 부문인 만큼, 일반적인 방법으로는 시간 초과가 발생할 수밖에 없도록 설계된 문제이다.

그러나 이 문제에서는 이진 탐색에서 더 나아가 upper bound 아이디어 또한 필요하다.

관련 개념 출처

[이진 탐색](https://terms.naver.com/entry.naver?docId=2270440&cid=51173&categoryId=51173)

이진 탐색은 다음과 같이 동작한다.

먼저, 데이터 15를 탐색하는 과정을 알아본다. 오름차순으로 정렬된 리스트의 중간값 11과 찾고자 하는 15를 비교한다.

![https://dbscthumb-phinf.pstatic.net/3523_000_1/20141020113555714_IEKSY8IFE.jpg/ka7_130_i2.jpg?type=w340_fst_n&wm=Y](https://dbscthumb-phinf.pstatic.net/3523_000_1/20141020113555714_IEKSY8IFE.jpg/ka7_130_i2.jpg?type=w340_fst_n&wm=Y)

11보다 찾고자 하는 값인 15가 더 크기 때문에 11의 오른쪽에 위치한 데이터들에서 다시 탐색을 수행한다.

![https://dbscthumb-phinf.pstatic.net/3523_000_1/20141020113555774_IY6YS5DGY.jpg/ka7_130_i3.jpg?type=w340_fst_n&wm=Y](https://dbscthumb-phinf.pstatic.net/3523_000_1/20141020113555774_IY6YS5DGY.jpg/ka7_130_i3.jpg?type=w340_fst_n&wm=Y)

17과 15를 비교했을 때, 찾고자 하는 값이 더 작기 때문에 17의 왼쪽 데이터를 다시 탐색한다.

![https://dbscthumb-phinf.pstatic.net/3523_000_1/20141020113556189_EYSIGTDPY.jpg/ka7_130_i4.jpg?type=w340_fst_n&wm=Y](https://dbscthumb-phinf.pstatic.net/3523_000_1/20141020113556189_EYSIGTDPY.jpg/ka7_130_i4.jpg?type=w340_fst_n&wm=Y)

탐색 영역 값인 15와 찾고자 하는 값이 일치한다.

![https://dbscthumb-phinf.pstatic.net/3523_000_1/20141020113556899_XIOWBPWXJ.jpg/ka7_130_i5.jpg?type=w340_fst_n&wm=Y](https://dbscthumb-phinf.pstatic.net/3523_000_1/20141020113556899_XIOWBPWXJ.jpg/ka7_130_i5.jpg?type=w340_fst_n&wm=Y)

```python
a, b =map(int,input().split())

li=[]
for i in range(a):
  temp=int(input())
  li.append(temp)

li=sorted(li)
l=len(li)

count=li[l-1]

def find(li, result_start,result_end,answer):
  if result_start>result_end: #최종 결과
    return answer
  sum=0 #만들어지는 랜선의 최대 개수를 저장
  result=(result_start+result_end)//2 #탐색 범위의 중간값

  for i in li:
    sum=sum+i//result #만들어진 랜선 개수 저장

  if sum>=b: #랜선의 개수가 목표로 하는 랜선 개수보다 크거나 같을 경우
    if(answer<result): #랜선의 길이 최대값을 구하기 위한 조건문
      answer=result
    return find(li,result+1,result_end,answer)

  elif sum<b: #만들어진 랜선 개수가 목표보다 작을 경우
    return find(li,result_start,result-1,answer)

  else: return result

result=find(li, 1,count,0)
print(result)
```
