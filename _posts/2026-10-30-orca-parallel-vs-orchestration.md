---
title: "Orca 정리 2: 병렬 실행과 오케스트레이션은 무엇이 다른가"
date: 2026-10-30 10:00:00 +0900
categories: [Tools, AI Agents]
tags: [orca, orchestration, multi-agent, claude-code, codex]
description: 에이전트를 여러 개 띄우는 것과 에이전트에게 에이전트를 감독시키는 것의 차이를, Orca의 Run, Task, Dispatch 개념을 중심으로 정리했다.
---

> 공식 문서(onorca.dev/docs/cli/orchestration)와 이를 풀어 쓴 블로그 글(테크버킷, 2026-09-04)을 읽고 정리한 **공부 노트**다. 명령을 직접 실행해 보지는 않았다. Orchestration은 문서에서 **Experimental** 기능이고 명령이 자주 바뀐다고 하니, 따라 하기 전에 최신 가이드를 확인한다.
{: .prompt-warning }

[1편]({% post_url 2026-10-30-orca-agent-development-environment %})에서는 Orca가 에이전트마다 worktree를 따로 주어 **병렬 실행**을 가능하게 한다고 정리했다. 그런데 Orca 문서에는 *orchestration*이라는 별도 기능이 또 나온다. "여러 에이전트를 동시에 돌린다"는 점에서 둘은 같아 보인다. 이 글은 무엇이 다른지 정리한다.

## 1. 한 줄 요약

- **병렬 실행**은 에이전트를 동시에 **여는** 것이다. 일을 나누고 결과를 확인하는 사람이 **나**다.
- **오케스트레이션**은 그 **관리 역할을 에이전트(coordinator)에게 넘기는** 것이다. 일을 나누고, 맡기고, 끝났는지 추적하는 일을 coordinator가 한다.

핵심 차이는 "몇 개를 동시에 돌리나"가 아니라 **누가 작업을 나누고, 누가 완료를 확인하나**다.

## 2. 세 가지 방식 비교

참고한 글은 비슷해 보이는 방식을 세 가지로 구분한다.

| 방식 | 작업을 나누는 주체 | 기존 에이전트가 결과를 기다리나 | 적합한 상황 |
|---|---|---|---|
| 수동 병렬 실행 | 사용자 | 사용자가 직접 확인 | 같은 문제의 구현을 여러 개 비교 |
| handoff | 사용자 또는 기존 에이전트 | 아니요 | 다른 에이전트에게 작업 소유권을 완전히 넘김 |
| supervised orchestration | coordinator 에이전트 | 예 | 여러 Task, 완료 추적, 질문과 응답, 의존성 관리 |

예를 들어 보면 이렇다.

- "새 worktree에서 Codex가 이 작업을 이어서 맡아줘" → **handoff**. 넘기면 끝이고 결과를 기다리지 않는다.
- "두 worker에게 나눠 맡기고, 완료를 기다린 뒤 결과를 종합해줘" → **orchestration**. 맡긴 쪽이 끝까지 추적한다.

공식 문서도 같은 기준을 준다. 소유권과 완료 추적, 작업 간 의존 관계(DAG)가 필요하면 orchestration을 쓰고, 한 번짜리 프롬프트는 `orca terminal send`, 감독 없이 완전히 넘기는 경우는 worktree/terminal 명령(`orca-cli` 스킬)을 쓴다.

## 3. 오케스트레이션의 구성 요소

공식 문서의 핵심 모델은 다음과 같다.

```text
Run     전체 목표 + coordinator의 받은 편지함 (namespace)
├─ Task A   완료 조건이 있는 작업 단위
│   └─ Dispatch A   Task A를 특정 worker 터미널에 배정한 "한 번의 시도"
└─ Task B
    └─ Dispatch B
```

- **Run**: 지속되는 이름 공간이자 coordinator의 inbox다. **스스로 일정을 짜거나 worker를 배치하지 않는다.**
- **Task**: 설명(spec), 의존성, 상태를 가진 작업 항목이다. 상태는 `pending`, `ready`, `dispatched`, `completed`, `failed`, `blocked`.
- **Dispatch**: Task를 터미널에 배정한 한 번의 시도다. `worker_done`과 heartbeat의 **수명 주기 기준**이 된다. 재시도하면 새 Dispatch가 생긴다.
- **Message**: inbox 메일이다. 종류는 `status`, `dispatch`, `worker_done`, `escalation`, `question`, `heartbeat` 등이다.
- **Decision gate**: Task를 막아 두고 결정이 기록될 때까지 기다리는 coordinator 소유의 질문이다.

참고 글이 강조하는 점이 중요하다. **Orca는 작업을 스스로 쪼개지 않는다.** Orca는 Run, Task, Dispatch, 메시지의 상태만 관리하고, 목표를 Task로 나누고 어떤 에이전트와 worktree에 맡길지는 **coordinator 에이전트가 정한다.** 수정 파일 충돌 가능성도 Orca가 판단해 주지 않는다.

## 4. 감독 루프(supervised loop)

공식 문서가 권장하는 흐름은 `run-create → task-create → worker-start → check → worker-release`다.

```bash
# 1) 목표 하나에 Run 생성
orca orchestration run-create --objective "Split checkout QA and summarize blockers" --json

# 2) 완료 조건을 담은 Task 생성
orca orchestration task-create --task-title "Billing audit" \
  --spec "Audit billing settings for mobile layout" --json

# 3) Task를 worker에 배정 (새 worktree에서 Codex 실행)
orca orchestration worker-start --task <taskId> --worktree new-child \
  --name billing-audit --agent codex --setup run --json

# 4) 질문/완료/에스컬레이션 메시지를 기다림
orca orchestration check --wait --types worker_done,escalation,question \
  --timeout-ms 900000 --json

# 5) 결과를 검토한 뒤 끝난 worker 터미널 정리
orca orchestration worker-release --dispatch <dispatchId> --json
```

수동 병렬 실행과 비교하면 사람이 하던 일이 이렇게 옮겨 간다.

| 사람이 직접 하던 일 | orchestration에서는 |
|---|---|
| worktree 만들고 에이전트 실행 | `worker-start`가 worktree와 터미널을 만든다 |
| 프롬프트를 각 터미널에 복사 | Task의 `--spec`이 worker에게 주입된다 |
| 끝났는지 터미널을 번갈아 확인 | `check --wait`로 `worker_done` 메시지를 기다린다 |
| 에이전트 질문에 직접 답변 | worker가 `ask`, coordinator가 `reply` |
| 선후 관계 관리 | Task `--deps`로 의존성을 기록한다 |

## 5. 병렬 실행에 없던 기능들

### 5-1. 완료 계약(worker contract)

worker는 시작할 때 coordinator와 소통하는 규칙을 받는다.

- 실패하더라도 `worker_done`을 **정확히 한 번**, `--outcome succeeded|failed`와 함께 보낸다.
- Task ID와 Dispatch ID를 함께 보내서, 늦게 도착한 재시도가 엉뚱한 Dispatch를 완료 처리하지 않게 한다.
- 긴 작업에서는 heartbeat를 보낸다(기본 5분 간격, 설정에서 끄거나 1~60분으로 조정 가능).
- 막히면 터미널 입력창에 묻지 않고 `orca orchestration ask`로 coordinator에게 묻는다.

### 5-2. 질문과 응답

```bash
# worker가 선택지를 포함해 질문
orca orchestration ask --question "기존 API 응답 형식을 유지할까요?" \
  --options "유지,변경" --timeout-ms 600000 --json

# coordinator가 답변
orca orchestration reply --id <message_id> --body "기존 응답 형식을 유지하세요." --json
```

대기 시간이 지나도 질문은 사라지지 않고, 같은 질문은 `--resume <message_id>`로 이어서 기다린다. 일반 질문에는 `ask/reply`, Task 자체를 막는 승인 단계에는 **decision gate**(`gate-create`, `gate-resolve`)를 쓴다.

### 5-3. 의존성

테스트나 리뷰가 구현 결과에 의존하면 `--deps`로 선행 Task를 지정한다.

```bash
orca orchestration task-create --task-title "구현 결과 리뷰" \
  --spec "구현 diff를 검토하고 회귀 위험을 보고한다. 코드는 수정하지 않는다." \
  --deps '["<implementation_task_id>"]' --json

orca orchestration task-list --ready --brief --json   # 실행 가능한 Task만 보기
```

다만 참고 글은 한계도 짚는다. Orca는 의존성을 **기록**할 뿐이고, 어느 worktree의 변경을 어떻게 리뷰 worker에게 넘길지는 coordinator가 설계해야 한다. 커밋, 브랜치, diff 경로를 Task 설명에 적어 두라는 조언이다.

## 6. 언제 무엇을 쓰나

```text
같은 문제를 여러 에이전트가 풀어서 가장 좋은 결과를 고르고 싶다
  → 수동 병렬 실행 (사람이 비교하는 것이 목적)

한 에이전트에서 다른 에이전트로 작업을 완전히 넘기고 싶다
  → handoff (기다리지 않는다)

나눈 일의 완료를 추적하고, 질문에 답하고, 순서를 관리해야 한다
  → supervised orchestration
```

처음에는 수동 병렬 실행부터 익히는 것이 낫다는 조언이 있다. worktree, 터미널, diff, 브랜치 관계를 알아야 coordinator가 worker를 엉뚱한 곳에 배치했을 때 알아챌 수 있기 때문이다. 시작은 worker 하나를 감독하는 최소 흐름이 좋다.

## 7. 주의할 점

- **Experimental**: Settings → Experimental에서 켜야 하고, 켜지 않으면 CLI가 있어도 Run과 Task를 만들 수 없다.
- **명령이 바뀐다**: 예전 `orchestration run`, `run-stop`, `coordinator-start/stop`은 지금은 아무 일도 하지 않는다(은퇴한 명령). 에이전트가 기억에 의존하지 않도록 `orca skills get orchestration --full`로 현재 가이드를 먼저 읽게 하라고 문서가 권한다.
- **스킬은 지침일 뿐**: 스킬을 설치해도 실행 중인 Orca runtime과 각 에이전트 CLI(Codex, Claude Code 등)의 설치와 로그인이 필요하다.
- **검토는 여전히 사람 몫**: `worker_done` 보고만 믿지 않고 diff와 테스트 결과를 확인해야 한다.
- **격리 한계**: worktree는 코드 변경을 나눌 뿐 운영체제 권한을 격리하는 보안 샌드박스가 아니다. 비밀값과 배포 권한은 필요한 worker에게만 최소 범위로 준다.
- **파일이 겹치는 작업**: 같은 파일을 수정하는 worker를 같은 worktree에서 동시에 돌리면 변경이 섞인다. 무리하게 병렬화하지 말고 파일 소유권을 나누거나 리뷰 Task를 뒤에 둔다.
- **중첩 깊이**: 기본적으로 worker가 다시 worker를 만들 수 없다(depth 1). 늘릴 수는 있지만 상태 추적과 충돌 관리가 어려워진다.

## 8. 내 정리와 남은 의문

- 병렬 실행과 오케스트레이션의 차이는 **동시성**이 아니라 **관리 책임의 위치**다. 병렬은 사람이, 오케스트레이션은 coordinator 에이전트가 관리한다.
- 관리를 에이전트에게 넘기면 사람의 수작업은 줄지만, coordinator가 Task를 잘못 나누면 그 오류가 전체로 퍼진다. **Task 설명(spec)의 품질**이 결과를 좌우한다.
- 의문: coordinator와 worker가 같은 모델일 때 coordinator의 검토가 실제로 독립적인 검증이 되는가?
- 의문: 작업이 작을 때도 orchestration의 오버헤드(Run, Task, 메시지 관리)가 이득인가? 어느 규모부터 이득인지 직접 재 봐야 한다.

### 직접 해보고 채울 것

- [ ] Experimental 켜고 `orca skills get orchestration --full` 읽어 보기
- [ ] worker 하나만 감독하는 최소 흐름 실행
- [ ] 독립 Task 두 개를 병렬로 실행하고, 의존 Task로 바꿔 순서 확인
- [ ] 수동 병렬 실행과 소요 시간, 사용량, 결과 품질 비교

## 참고 자료

- Orca 공식 문서, Orchestration: <https://www.onorca.dev/docs/cli/orchestration>
- 테크버킷, "Orca orchestration 사용법: coordinator와 worker 연결하기" (2026-09-04): <https://techbukket.com/blog/orca-orchestration-guide>

## 다른 주제

- [Mistral Timestral 정리: 시계열 파운데이션 모델은 무엇이고 어떻게 쓰나]({% post_url 2026-10-30-mistral-timestral-time-series-foundation-model %})
