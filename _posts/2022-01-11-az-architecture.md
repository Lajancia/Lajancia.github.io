---
layout: post
title: "Azure 900 - 4"
date: 2022-01-11
excerpt: "azure 아키텍쳐 구성 요소, compute 서비스"
tags: [azure, ms, az-900]
comments: false
---

# az 아키텍쳐 구성요소

---

## 지역

- 지역region은 데이터센터 모음을 나타냄
- 사용자와 가장 가까운 지역에 리소스를 배포 가능
- 전 세계 55개 지역 140개국
- 리소스 배치 시 지역 가용성에 유의
- 지역 독립적인 글로벌 서비스

> 지역 가용 서비스 확인 필수

## BCDR Business continuity and disaster recovery

- azure 지역은 항상 쌍을 이루는 다른 지역이 있음
- 데이터센터 간 300마일 이상의 분리를 선호
- 일부 서비스의 경우 지역 쌍에 자동 복제를 제공(storage account 등)

> 300마일은 자연 재해 영향권과 관련이 있다.

## Geography

- 데이터 상주 및 규정 준수 경계를 유지하는 개별 시장
- 일반적으로 둘 이상의 영역이 포함됨
- 특정 지역 및 규정 준수 요구 사항이 있는 고객은 데이터와 응용 프로그램을 가까이에 두어야 함
- 아메리카, 유럽, 아시아 태평양, 중동 및 아프리카로 분류

## 가용성 옵션

## 가용성 세트 availability sets

유지관리 또는 하드웨어 오류 중에 응용프로그램을 온라인 상태로 유지

- update domains(UD) - 예약된 유지 관리, 성능, 보안 업데이트는 업데이트 도메인을 통해 순서를 결정
- Fault domains(FD) - 데이터 센터의 여러 하드웨어에서 워크로드를 물리적으로 분리
- 고가용성을 마이크로소프트에서 관리

## 가용 영역 Availability zones

- azure 지역 내에서 물리적으로 위치를 구분
- 가용성 세트를 한 단계 끌어 올림
- 독립적인 전원, 냉각 및 네트워킹 기능을 갖춘 하나 이상의 데이터 센터를 포함
- 격리 경계 역할을 수행
- 한 가용 영역이 다운되면 다른 가용 영역이 계속 작동
- 고가용성을 사용자가 관리

## 고가용성 구현

- 존 1과 존 2 등 비용이 발생. 서로 통신 비용이 발생할 수 있음

## 재해 복구 DR 구현

## 리소스 그룹

- 동일한 수명 주기를 가지는 리소스들의 묶음
- 리소스는 리소스 그룹을 이동할 수 있음
- 리소스는 하나의 리소스 그룹을 가짐
- 역할 기반 접근 제어 (RBAC)를 사용하여 리소스 그룹( 또는 리소스) 레벨의 보안 유지

## azure resource manager

- azure를 관리하는 계층
- 리소스 또는 리소스 그룹을 생성, 구성, 관리, 삭제
- 리소스 조직화 및 자동화
- azure ad를 통한 접근 제어
- azure portal, cli, powershell, api

# azure compute 서비스

---

## Azure Compute

- azure에서 compute 작업을 하는 서비스
- Web server를 운영하거나 Application Server를 운영 시 사용
- 기본적으로 virtual machine이 있음
- 사용자가 직접 프로세스에 관여

## Azure Compute 서비스

- Azure virtual machine - IaaS의 대표적인 서비스로 워크로드를 수행가기에 가장 유연한 서비스이다. 운영체제를 관리해야 하며 사용자가 직접 운영한다.
- VM Scale Sets - Azure VM Image를 이용하여 자동으로 확장 또는 축소할 수 있다.
- App Services - 사용자는 소스 파일만 업로드하면 알아서 동작하는 PasS이다. Web App, API App, Mobile App, Logic App, Functions와 같은 서비스들이 있다.
- Functions - App Services의 하나로 이벤트를 기반으로 컴퓨트 작업을 수행한다.

## 실습

영상 참조

## Azure container 서비스

- azure container instances -azure가 관리하는 container cluster에 container image를 업로드할 수 있는 PaaS 서비스
- Azure Kubernetes Service - 많은 수의 컨테이너를 관리하기 위한 container orchestrator. ㅁ켝ㄷ 프dp Kubernetes Cluster를 구성해 주며, Master Node는 Azure에서 무료로 제공한다.

## 실습

영상 참조
