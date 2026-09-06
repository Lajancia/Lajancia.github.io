---
title: "AI 코딩 에이전트를 위한 워킹 메모리 볼트 — Agent Working Memory Template 설계와 활용"
date: 2026-09-06 12:00:00 +0900
categories: [스터디]
tags: [AI, Agent, LLM, Claude, Codex, Working Memory, Obsidian, Template]
author: L.J
---

### **들어가며**

Claude, Codex, Cursor 등 AI 코딩 에이전트를 실무에 사용하다 보면 공통된 불편함이 있다. 세션이 끝나면 모든 컨텍스트가 사라지고, 다음 세션에서 같은 내용을 다시 설명해야 한다.

- "이 프로젝트는 pnpm을 사용해요"
- "응답은 한국어로, 간결하게 부탁해요"
- "지난주에 결정한 아키텍처는 이렇습니다"

이런 대화가 매일 반복된다. AI 에이전트의 세션은 기본적으로 **휘발성(ephemeral)**이다. 대화 기록은 저장되지만, 각 도구마다 다른 히스토리를 가지고 있어서 에이전트 간의 이동도 쉽지 않다.

이 문제를 해결하기 위해 만든 것이 **Agent Working Memory Template**이다. 파일 기반의 개인 워킹 메모리 볼트(vault)로, AI 에이전트가 세션 간 연속성을 유지하고 작업 지식을 자동으로 축적할 수 있게 설계되었다.

---

### **어떤 문제를 해결하는가**

이 템플릿은 세 가지 구체적인 문제를 해결한다:

| 문제 | 증상 |
|------|------|
| **세션 휘발성** | 도구가 바뀌거나 세션이 리셋되면 이전 컨텍스트가 사라짐. 현재 상태, 작업 스타일, 녹화 경계를 매번 다시 설명해야 함 |
| **에이전트 간 단절** | Codex(코드)와 Claude(문서/장문)는 강점이 다른데, 각자 다른 채팅 히스토리를 가지고 있어 수동 핸드오프가 필요함 |
| **지식 비축적** | 회의록, 결정 사항, 수정 사항, 반복 솔루션 패턴이 채팅에만 남고 재사용 가능한 자산이 되지 않음 |

---

### **디자인 원칙**

| 원칙 | 핵심 아이디어 |
|------|-------------|
| **개인 볼트 우선** | 한 사람의 작업 스타일, 프로젝트 상태, 선호도에 최적화. 여러 사람이 공유하는 용도는 아님 |
| **최소 루틴으로 시작** | Morning Briefing과 Wrap-up이 기본 트리거. 실제 사용을 통해 타이밍과 범위를 조정 |
| **명시적 출처와 불확실성** | 에이전트는 실제로 확인한 파일과 접근할 수 없었던 외부 출처를 구분. 확인하지 않은 시스템은 "not queried"로 표시 |
| **계층형 녹화 레벨** | Must / Should / Maybe / Do not write — 4단계로 신호 대비 잡음 비율 유지 |
| **다음 세션 부트 가능성** | `hot.md`가 다음 세션이 시작될 때 "지금 어디에 있고, 다음에 무엇을 해야 하는지"를 담은 부트 캐시 역할 |

---

### **전체 구조**

템플릿은 세 개의 레이어로 구성된다.

```
Agent_Working_Memory_Template/
├─ AGENTS.md                         # 에이전트 운영 규칙
├─ README.md                         # 사용자 퀵스타트
├─ SETUP.md                          # 초기 설정 질문 가이드
├─ hot.md                            # 다음 세션의 부트 캐시
├─ log.md                            # 볼트 작업 로그 (append-only)
├─ 00_Index.md                       # 볼트 전체 인덱스
├─ 01_Daily/                         # 데일리 노트 및 작업 로그
├─ 10_Projects/                      # 프로젝트 위키
├─ 20_Meetings/                      # 회의록 및 액션 아이템
├─ 30_Agent/
│  ├─ Working_Style.md               # 사용자 선호도 및 실행 규칙
│  ├─ Observations.md                # 반복 관찰 패턴
│  └─ Journal.md                     # 최근 1-2개 세션 요약
├─ 70_Entities/                      # 사람, 도구, 개념
│  ├─ People/
│  ├─ Tools/
│  └─ Concepts/
├─ 90_Retrospectives/                # 일일/세션 회고
└─ 99_Templates/                     # 노트 템플릿
```

**세 레이어의 역할:**

| 레이어 | 역할 | 대표 파일 |
|--------|------|----------|
| **Agent Continuity** | 세션 간 동일한 컨텍스트와 스타일 복원 | `hot.md`, `Working_Style.md`, `Observations.md` |
| **Autonomous Wiki Growth** | 작업 지식을 개인 위키로 축적 | `10_Projects/`, `20_Meetings/`, `70_Entities/` |
| **Routine-Based Refresh** | 볼트가 낡은 정보로 머물지 않도록 갱신 | `01_Daily/`, `90_Retrospectives/`, `Journal.md` |

---

### **핵심 루틴**

**1. Setup (최초 설정)**

사용자가 "이 폴더를 워킹 메모리 볼트로 사용해줘"라고 말하면, 에이전트는 `AGENTS.md`와 `SETUP.md`를 읽고 3-6개의 질문을 통해 사용자 정보, 프로젝트, 브리핑 소스, 녹화 경계를 설정한다.

**2. Morning Briefing (데일리 브리핑)**

```text
Please give me a morning briefing from this working memory vault.
```

에이전트는 `hot.md`, 최신 데일리 노트, 프로젝트 인덱스, 설정된 브리핑 소스를 읽고 다음을 요약한다:
- 오늘의 포커스
- 대기 중인 항목
- 블로커
- 외부 입력
- 독립 작업

오늘의 데일리 노트가 없으면 `99_Templates/daily.md`에서 자동 생성한다.

**3. During Work (작업 중)**

에이전트는 자동으로 중요한 정보만 선택적으로 기록한다:
- **Must write** — 사용자 수정, 명시적 결정, 핸드오프, 외부 시스템 상태 변경, 미팅/문서 수집 결과
- **Should write** — 반복 작업 패턴, 재사용 가능한 워크플로우, 버그 원인/수정 패턴
- **Maybe write** — 브레인스토밍, 일회성 의견, 미검증 해석
- **Do not write** — 단순 대화 반응, 이미 `hot.md`에 있는 중복 정보, 미검증 추측, 민감 정보

**4. Wrap-Up (마무리)**

```text
Please wrap up today.
```

에이전트는 데일리 노트를 업데이트하고, 회고 노트를 생성/갱신하며, `hot.md`를 다음 세션을 위한 부트 캐시로 재작성하고, `Journal.md`와 `log.md`에 요약을 추가한다. 반복 패턴은 `Observations.md`나 프로젝트 노트로 승격한다.

---

### **정보 승격 흐름**

```
Conversation
    → Daily note / Journal
    → hot.md
    → Project / Meeting / Entity notes
    → Observations
    → Working_Style
```

| 단계 | 역할 |
|------|------|
| **Conversation** | 즉시 작업에 필요한 raw 컨텍스트 |
| **Daily note / Journal** | 하루 또는 세션의 압축된 기록 |
| **hot.md** | 다음 세션을 위한 짧은 부트 캐시 |
| **Project / Meeting / Entity notes** | 장기 작업 지식 |
| **Observations** | 반복 관찰 패턴 |
| **Working_Style** | 충분히 검증된 실행 규칙 |

이 흐름은 일회성 정보가 영구적인 사용자 규칙이 되는 것을 방지한다.

---

### **참조한 패턴**

이 템플릿은 기존에 잘 알려진 몇 가지 패턴을 참고했다:

| 패턴 | 핵심 아이디어 | 템플릿에서의 반영 |
|------|-------------|-------------------|
| **Karpathy LLM Wiki** | LLM이 읽고, 정리하고, 연결할 수 있는 파일 기반 위키 | 프로젝트, 회의, 사람, 도구, 개념을 마크다운 위키로 관리 |
| **OpenClaw Auto-Dream** | 세션 히스토리를 주기적으로 요약하고 중요한 정보를 장기 메모리로 승격 | Journal → Observations → Working_Style 흐름으로 단순화 |
| **Daily Review / Shutdown** | 하루의 시작과 끝을 정의하는 리뷰와 종료 루틴 | Morning Briefing과 Wrap-up이 볼트의 최소 갱신 사이클 |

---

### **기대 효과**

- 세션 리셋 후 컨텍스트 복원에 드는 시간 감소
- Claude와 Codex 사이를 같은 컨텍스트로 이동 가능
- 작업 중 결정과 수정 사항이 소실되지 않음
- 에이전트가 사용자의 작업 스타일을 점진적으로 학습
- 채팅에 흩어진 지식이 개인 작업 위키로 축적
- Morning Briefing과 Wrap-up이 볼트를 낡은 정보로 머물지 않게 유지
- 확인한 출처와 확인할 수 없는 출처가 명확히 구분됨
- 사람이 직접 검사하고 수정할 수 있는 파일 구조

---

### **제한 사항 및 권장 사용법**

이 템플릿은 **한 사람의 작업 스타일**에 맞춰 설계되었다. 팀 공유 지식베이스로는 권장하지 않는다.

- 각 사용자가 템플릿을 복사하여 개인 볼트로 사용
- 개인 볼트는 작업 스타일, 현재 상태, 핸드오프, 관찰 정보를 저장
- 팀 레벨의 결정이나 프로젝트 지식은 별도의 팀 위키나 이슈 트래커로 승격
- 민감 정보는 명시적 승인 없이 자동 기록하지 않음
- 업데이트 타이밍과 녹화 범위는 실제 사용을 통해 조정

---

### **유용한 프롬프트 예시**

| 의도 | 프롬프트 |
|------|----------|
| 초기 설정 | "Use this folder as my working memory vault. Please run setup." |
| 하루 시작 | "Please give me a morning briefing." |
| 하루 마무리 | "Please wrap up today." |
| 회의 수집 | "Please turn this meeting transcript into meeting notes and update the vault." |
| 수정 반영 | "The corrected schedule is important for the next session, so please reflect it in hot.md and the daily note." |
| 기록 제외 | "This is personal, so please do not record it in the vault." |

---

### **마치며**

Agent Working Memory Template은 AI 코딩 에이전트의 가장 큰 약점인 **세션 휘발성**을 해결하기 위한 실용적인 도구다. 복잡한 설정이나 외부 인프라 없이, 단순한 마크다운 파일과 Obsidian 볼트 구조만으로 에이전트가 맥락을 유지하고 지식을 축적할 수 있게 한다.

핵심은 "완벽한 규칙을 처음부터 만들기보다, 실제 사용하면서 어색한 부분을 `Working_Style.md`와 `AGENTS.md`를 통해 개선해나가자"는 접근이다.

> **GitHub**: [github.com/Lajancia/agent-working-memory-template](https://github.com/Lajancia/agent-working-memory-template)