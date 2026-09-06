---
title: "AI 코딩 에이전트를 위한 워킹 메모리 볼트 — Agent Working Memory Template 설계와 활용"
date: 2026-09-06 12:00:00 +0900
categories: [스터디]
tags: [AI, Agent, LLM, Claude, Codex, Working Memory, Obsidian, Template]
author: L.J
---

### **들어가며**

AI 코딩 에이전트는 빠르게 발전하고 있다. Claude에는 Memory 기능이 추가되었고, 세션 검색도 가능해졌다. 에이전트마다 컨텍스트 윈도우가 커지고, 성능도 날로 좋아진다.

그런데 이런 발전이 오히려 새로운 문제를 만든다.

**정보가 에이전트에 종속된다.**

Claude에서 쌓은 지식은 Claude에 남는다. Codex에서 작업한 내용은 Codex 세션 히스토리에 묶인다. 내일 더 나은 에이전트가 나오면? 지금까지 쌓아온 컨텍스트, 결정 사항, 작업 패턴을 처음부터 다시 설명해야 한다.

이 템플릿의 목적은 하나다.

> **에이전트와 무관하게(agent-agnostic) 내 작업 지식을 중앙집중하고, 어떤 에이전트를 쓰든 볼트만 연결하면 즉시 활용할 수 있게 만드는 것.**

에이전트는 도구다. 도구는 바뀐다. 하지만 내가 쌓아온 작업 지식과 맥락은 도구를 넘어서 보존되어야 한다.

---

### **무엇을 해결하려는가**

#### **1. 정보의 에이전트 종속성**

Claude Memory는 Claude 안에서 유용하다. 세션 검색도 같은 에이전트 내에서만 의미가 있다. Codex로 갈아타면? Cursor로 작업하면? 다음 세대 에이전트가 나오면?

**지식이 특정 도구에 갇히면, 더 나은 도구로의 이 migration이 어렵다.**

| 상황 | 문제 |
|------|------|
| Claude Memory 사용 | Claude 안에서는 편리하지만, 다른 에이전트는 이 정보에 접근 불가 |
| 세션 검색 | 세션 히스토리는 증발하지 않지만, 구조화되지 않은 채팅 더미일 뿐 |
| 에이전트 migration | 새 에이전트가 나올 때마다 컨텍스트를 다시 설명해야 함 |

#### **2. 맥락의 파편화**

여러 에이전트를 병렬로 사용할수록 맥락은 더 분산된다. "이 결정은 Claude에서 했는데, 그 코드는 Codex에서 작성했고..." — 모든 정보가 각자의 세션에 흩어져 있다.

#### **3. 구조화되지 않은 정보**

세션 검색이 가능해졌다고 해도, 채팅 로그는 근본적으로 비정형 데이터다. 결정 사항, 작업 패턴, 반복 워크플로우가 메시지 사이에 섞여 있어 재사용하기 어렵다.

---

### **디자인 원칙**

| 원칙 | 핵심 아이디어 |
|------|-------------|
| **에이전트 불변 (Agent-Agnostic)** | 볼트는 특정 에이전트에 종속되지 않는다. Claude든 Codex든, 미래의 어떤 에이전트든 같은 볼트를 읽고 쓸 수 있어야 함 |
| **중앙집중 (Centralized)** | 모든 세션의 맥락이 한 곳에 모인다. 어떤 에이전트로 작업했든 정보는 볼트에 남는다 |
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

### **Claude Memory / Session Search와의 관계**

이 템플릿은 Claude의 Memory 기능이나 세션 검색을 **대체하지 않는다**. 오히려 보완한다.

| 기능 | 강점 | 한계 |
|------|------|------|
| **Claude Memory** | 에이전트 내에서 빠른 참조, 자동 학습 | Claude 전용. 내보내기/이식 불가. 구조화되지 않음 |
| **세션 검색** | 과거 대화 찾기 | 비정형 데이터. 결정/패턴 추출 어려움. 에이전트 종속 |
| **이 볼트** | 에이전트 불변. 구조화된 지식. 완전한 이식성 | 초기 설정 필요. 루틴 유지 필요 |

셋을 함께 사용하는 것이 가장 효과적이다. Claude Memory로 자주 쓰는 정보를 빠르게 참조하고, 이 볼트로 구조화된 장기 지식을 보존하며, 필요할 때 어떤 에이전트든 연결해서 사용한다.

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

- **어떤 에이전트를 쓰든** 같은 볼트로 즉시 작업 가능
- Claude에서 Codex로, Codex에서 미래의 더 나은 에이전트로 **자유롭게 migration**
- 작업 중 결정과 수정 사항이 **에이전트가 바뀌어도 소실되지 않음**
- 여러 에이전트를 병렬로 사용해도 **맥락이 한 곳에 집중**
- 채팅에 흩어진 지식이 **개인 작업 위키로 축적**
- Claude Memory와 볼트를 병행하면 **단기 참조 + 장기 보존** 최적화
- Morning Briefing과 Wrap-up이 볼트를 낡은 정보로 머물지 않게 유지
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

### **실제 사용 경험 — 완벽하지 않다, 그래서 개선한다**

이 템플릿을 실제로 사용하면서 가장 크게 느낀 점은 이것이다. **완벽한 자동화는 없다.**

Morning Briefing과 Wrap-up 루틴을 정의해놨지만, 에이전트가 이를 간간히 빼먹는 경우가 생긴다. 볼트는 에이전트가 참조하는 마지막 레이어다 보니, 에이전트가 바쁘게 작업을 하다 보면 "아, 오늘 브리핑을 안 했네" 하는 상황이 발생한다. 결국 내가 "모닝 브리핑 해줘" 또는 "wrap up 해줘"라는 트리거를 직접 넣어줘야 하는 날도 적지 않다.

이건 템플릿의 한계가 아니라, **어떤 지식 관리 도구든 피할 수 없는 현실**이다.

| 현실 | 대응 |
|------|------|
| 에이전트가 가끔 Morning Briefing을 건너뜀 | 직접 "morning briefing" 트리거 — 1초면 끝 |
| Wrap-up 없이 세션이 끝나는 경우 | `hot.md`에 마지막 상태라도 남아 있어 다음 세션에서 복원 가능 |
| 기록 깊이가 일관적이지 않음 | Working Style을 수정해서 기준을 조정 |
| 볼트에 정보가 너무 쌓임 | 주기적으로 `Observations.md`로 승격하고 오래된 Journal은 정리 |

중요한 건 **처음부터 완벽한 규칙을 만드는 것이 아니라, 사용하면서 불편한 지점을 지속적으로 `Working_Style.md`와 `AGENTS.md`에 반영해 개선해나가는 것**이다.

---

### **마치며 — 볼트는 에이전트보다 오래간다**

Claude가 발전하고, 더 좋은 에이전트가 나와도 이 템플릿의 가치는 줄어들지 않는다. 오히려 반대다.

**에이전트가 발전할수록, 개인의 작업 지식을 에이전트로부터 독립시켜야 할 필요성은 더 커진다.**

내일 나온 더 뛰어난 에이전트가, 오늘 내가 Claude에서 쌓은 컨텍스트를 전혀 모른다면? 그 에이전트도 처음부터 나를 이해해야 한다면?

이 볼트는 그 간극을 메운다. 에이전트가 바뀌어도, 볼트만 연결하면 언제든 내 작업 맥락을 복원할 수 있다. 완전하지는 않지만, 이런 아이디어로 나는 에이전트를 업무에 활용하고 있다. 만능이 아니라는 걸 인정하는 것이 오히려 현실적으로 오래 사용할 수 있는 비결이다.

> **GitHub**: [github.com/Lajancia/agent-working-memory-template](https://github.com/Lajancia/agent-working-memory-template)