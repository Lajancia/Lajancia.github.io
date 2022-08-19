---
layout: post
title: "Bitburner"
date: 2022-08-18
excerpt: "Bitburner tutorial"
tags: [Steam, hacking]
comments: false
---

# Bitburner

---

~~프로젝트로 코딩하고 게임도 코딩하는 코딩러…~~

[Getting Started Guide for Beginner Programmers - Bitburner 1.6.4 documentation](https://bitburner.readthedocs.io/en/latest/guidesandtips/gettingstartedguideforbeginnerprogrammers.html)

## first steps - kill the noodles.script

$ kill noodles.script

Or

Alt s press kill script

## creating our first script

### 스크립트 작성법

$ home

$ nano early-hack-template.script

```jsx
// that we're going to hack. In this case, it's "n00dles"
var target = "n00dles";

// Defines how much money a server should have before we hack it
// In this case, it is set to 75% of the server's max money
var moneyThresh = getServerMaxMoney(target) * 0.75;

// Defines the maximum security level the target server can
// have. If the target's security level is higher than this,
// we'll weaken it before doing anything else
var securityThresh = getServerMinSecurityLevel(target) + 5;

// If we have the BruteSSH.exe program, use it to open the SSH Port
// on the target server
if (fileExists("BruteSSH.exe", "home")) {
  brutessh(target);
}

// Get root access to target server
nuke(target);

// Infinite loop that continously hacks/grows/weakens the target server
while (true) {
  if (getServerSecurityLevel(target) > securityThresh) {
    // If the server's security level is above our threshold, weaken it
    weaken(target);
  } else if (getServerMoneyAvailable(target) < moneyThresh) {
    // If the server's money is less than our threshold, grow it
    grow(target);
  } else {
    // Otherwise, hack it
    hack(target);
  }
}
```

### 실행

```
$ scan-analyze 2
```

위의 명령어를 실행하면 여러 서버들을 확인할 수 있다.

- `sigma-cosmetics`
- `joesguns`
- `nectar-net`
- `hong-fang-tea`
- `harakiri-sushi`

위의 서버들은 16기가 램을 가지고 있다. 또한 위의 서버들은 별도의 NUKE할 오픈 포트가 필요하지 않다. 쉽게 말해 위 서버들은 루트 엑세스를 하여 스크립트를 실행시킬 수 있는 서버들이다.

### 스크립트 성능 확인

```
$ mem early-hack-template.script
```

위의 명령어를 통해 우리가 작성한 스크립트가 얼마만큼의 스래드를 실행시킬 수 있는지 확인할 수 있다. 여기서는 하나의 스래드를 위해 2.6기가 램을 사용한다.

### 동작

1. 각각의 서버에 작성한 스크립트를 카피하기 위해 scp 터미널을 사용한다.
2. 서버와 컨넥트하기 위해 connect 터미널을 사용한다.
3. 루트 엑세스 권한을 얻기 위해 NUKE.exe를 실행한다.
4. 스크립트를 동작시키기 위해 run 커맨드를 다시 실행한다.
5. 위의 서버에 각각의 동작을 반복한다.

```
$ home
$ scp early-hack-template.script n00dles
$ scp early-hack-template.script sigma-cosmetics
$ scp early-hack-template.script joesguns
$ scp early-hack-template.script nectar-net
$ scp early-hack-template.script hong-fang-tea
$ scp early-hack-template.script harakiri-sushi
$ connect n00dles
$ run NUKE.exe
$ run early-hack-template.script -t 1
$ home
$ connect sigma-cosmetics
$ run NUKE.exe
$ run early-hack-template.script -t 6
$ home
$ connect joesguns
$ run NUKE.exe
$ run early-hack-template.script -t 6
$ home
$ connect hong-fang-tea
$ run NUKE.exe
$ run early-hack-template.script -t 6
$ home
$ connect harakiri-sushi
$ run NUKE.exe
$ run early-hack-template.script -t 6
$ home
$ connect joeusguns
$ connect nectar-net
$ run NUKE.exe
$ run early-hack-template.script -t 6
```

- 이때 nectar-net은 홈에서 바로 접근할 수 없다. 예제에서는 hong-fang-tea를 통해 접근했지만, 필자는 joeusguns를 통해 접근했다. nectar-net의 위치는 가장 처음에 서버를 스캔했을 때 어떤 서버의 하위 서버였는지 확인할 수 있다.

## 이후…

한창 돈을 뽑아먹고(?) 있는데 웬 메세지가 왔다. 대충 flight.exe 프로그램이 홈 컴퓨터에 추가되었다고 한다. jump3R이라는 사람이 보낸 것 같은데….일단 메세지도 j0.msg로 세이브 되어 있지만 혹시 모르니 기록해둬야겠다.
