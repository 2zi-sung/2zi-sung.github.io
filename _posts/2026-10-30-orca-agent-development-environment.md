---
title: "Orca 정리: 여러 AI 코딩 에이전트를 병렬로 돌리는 개발 환경"
date: 2026-10-30 09:00:00 +0900
categories: [Tools, AI Agents]
tags: [orca, ade, claude-code, codex, git-worktree]
description: Orca가 무엇인지, 어떤 문제를 푸는지, 처음 설치해서 3개 에이전트를 돌려보는 순서까지 공식 문서 기준으로 정리했다.
---

> 이 글은 Orca 공식 문서(onorca.dev/docs)와 GitHub 저장소를 읽고 정리한 **공부 노트**다. 아직 직접 써본 후기가 아니라서, 직접 해보며 알게 된 점은 아래 "해보고 나서 채울 것"에 추가할 예정이다.
{: .prompt-info }

## 1. Orca가 뭔가

**Orca는 여러 AI 코딩 에이전트를 나란히 돌리기 위한 데스크톱 앱**이다. 공식 소개에서는 이것을 **ADE(Agent Development Environment)**, 즉 에이전트 개발 환경이라고 부른다.

- 만든 곳은 Stably AI이고, **MIT 라이선스 오픈소스**다 (저장소: `stablyai/orca`).
- macOS, Windows, Linux 데스크톱 앱이 있고, iOS와 Android 모바일 컴패니언 앱도 있다.
- **모델이 아니다.** Claude Code, Codex, Cursor CLI 같은 에이전트를 직접 제공하지 않는다. 이미 쓰고 있는 구독이나 CLI를 가져와서 한곳에서 돌려주는 도구다.
- Git을 대체하는 것도 아니다. 모든 작업 공간은 진짜 `git worktree`라서 그 폴더에서 평소처럼 git을 써도 된다.

공식 문서는 이 도구가 **이미 코드를 쓰고, diff를 읽고, 커밋을 신경 쓰는 사람**을 대상으로 한다고 밝힌다. 노코드 도구가 아니다.

## 2. 왜 필요한가: IDE와 무엇이 다른가

에이전트 여러 개를 한 폴더에서 동시에 돌리면 서로의 파일을 덮어쓴다. 그래서 보통은 브랜치를 바꾸고 stash를 하면서 하나씩 돌리게 된다. Orca의 접근은 이렇다.

| 문제 | Orca의 방식 |
|---|---|
| 에이전트끼리 파일이 충돌 | 작업마다 **별도 git worktree**(폴더 + 브랜치)를 만든다 |
| 어느 에이전트가 뭘 하는지 안 보임 | 한 화면에 분할 패널로 터미널을 나란히 띄운다 |
| AI가 만든 diff 검토가 번거로움 | diff 화면에서 줄별로 코멘트를 달아 에이전트에게 되돌려 보낸다 |
| 자리를 비우면 진행 상황을 모름 | 모바일 앱으로 상태를 보고 후속 지시를 보낸다 |

공식 문서가 드는 대표 사용 사례는 "같은 버그를 에이전트 3개가 동시에 시도하게 하고 가장 좋은 결과를 고르는 것"이다.

## 3. 핵심 개념: Worktree

Orca를 이해하는 데 가장 중요한 개념이다.

- 저장소마다 **base ref**(보통 `origin/main`)가 있다.
- 작업(worktree)마다 **start-from ref**(어디서 갈라질지), 자기만의 **브랜치**, 자기만의 **디스크 폴더**, 자기만의 **에이전트 터미널**이 있다.
- worktree를 삭제하면 폴더와 브랜치가 함께 지워진다(확인 절차가 있다).

새 worktree는 깨끗한 체크아웃이라서, `.gitignore`에 있는 파일(`node_modules`, `.env` 등)은 처음에 없다. 이를 위해 세 가지 방법이 문서에 나온다.

```yaml
# orca.yaml (저장소 루트): 큰 폴더는 복사하지 않고 공유(symlink)
worktree:
  sharedDirectories:
    - node_modules
    - .cache
```

```text
# .worktreeinclude (저장소 루트): worktree마다 복사할 파일
.env
.env.local
.vscode/settings.json
```

- `sharedDirectories`: 다시 만들 수 있는 큰 폴더를 공유한다.
- `.worktreeinclude`: 시크릿 같은 파일을 worktree마다 **복사**한다. 문서에는 글롭(`*`)은 아직 지원하지 않는다고 되어 있다.

## 4. 설치

macOS는 Homebrew 캐스크로 설치할 수 있다.

```bash
brew install --cask stablyai/orca/orca
```

- 업데이트: `brew upgrade --cask orca`
- 첫 실행 때 홈 디렉터리 접근 권한을 묻고, `~/.claude`, `~/.codex`, Ghostty 터미널 설정이 있으면 가져올지 물어본다.
- Linux에서는 CLI 이름이 GNOME의 Orca 스크린 리더와 겹치지 않도록 `orca-ide`다.

## 5. 처음 써보기: 에이전트 3개 병렬 세션

공식 문서가 "가장 중요한 페이지"라고 하는 흐름이다. 전체는 **저장소 추가 → worktree → 에이전트 → 분할 → diff → 배포**로 요약된다.

1. **저장소 추가**: 사이드바의 *Add Repo*에서 로컬 체크아웃을 지정한다. 기본 브랜치가 base ref가 된다.
2. **worktree 만들기**: 저장소 이름 옆 `+`를 누르고 작업 이름(예: `fix-login-race`)을 입력한다. 이름을 비우면 바다 생물 이름이 붙는다.
3. **에이전트 고르기**: 새 worktree 터미널에서 Claude Code, Codex, Cursor CLI 등을 고른다. 에이전트는 올바른 작업 폴더에서 실행된다.
4. **같은 작업을 3개에 시키기**: 2~3단계를 두 번 더 반복하고 같은 프롬프트를 붙여 넣는다.
   - `fix-login-race` → Claude Code
   - `fix-login-race-2` → Codex
   - `fix-login-race-3` → Cursor CLI
5. **분할 패널**: worktree 탭을 오른쪽이나 아래쪽 가장자리로 끌어서 세 에이전트를 한 화면에서 본다.
6. **승자 고르기**: 각 worktree의 diff를 열어 비교하고, *Annotate AI Diff*로 줄별 코멘트를 달아 가장 가까운 에이전트에게 돌려보낸다. 커밋과 푸시는 Orca 안에서 하고, 나머지 두 worktree는 한 번에 지운다.

## 6. 그 밖의 기능 (문서 목차 기준)

- **터미널**: WebGL 렌더링, 무제한 분할, 재시작해도 남는 스크롤백
- **Design Mode**: 내장 Chromium에서 UI 요소를 클릭하면 HTML, CSS, 잘라낸 스크린샷이 에이전트 프롬프트로 들어간다
- **GitHub / Linear / Jira 연동**: PR이나 이슈에서 바로 worktree를 만든다
- **원격 실행**: SSH, 자체 호스팅 Orca 서버, 클라우드 VM에서 에이전트를 돌린다
- **Orca CLI**: 에이전트가 `orca worktree create` 같은 명령으로 환경을 직접 조작한다
- **사용량 추적**: Claude와 Codex 사용량, 레이트 리밋 초기화 시점을 보여준다

## 7. 내 생각과 의문 (공부 메모)

- **병렬 실행의 비용**: 같은 작업을 3개 에이전트에 시키면 구독 사용량도 그만큼 쓴다. 어떤 작업에서 병렬이 값어치가 있는지 판단 기준이 필요하다.
- **검토 병목**: 에이전트를 늘려도 결국 diff를 읽는 사람의 속도가 한계가 된다. Annotate 기능이 이 병목을 얼마나 줄이는지 확인하고 싶다.
- **보안**: `.worktreeinclude`로 `.env`를 worktree마다 복사하면 시크릿이 여러 폴더에 퍼진다. 원격 실행까지 쓰면 어디에 시크릿이 놓이는지 따로 정리해야 한다.
- **회사 코드**: 회사 저장소를 외부 에이전트와 함께 쓸 때는 회사 정책을 먼저 확인해야 한다.

## 8. 해보고 나서 채울 것

- [ ] 작은 개인 저장소 하나로 3-agent 세션을 실제로 돌려보기
- [ ] 같은 프롬프트에 대한 에이전트별 결과 차이 비교
- [ ] `orca.yaml`과 `.worktreeinclude` 설정이 의도대로 동작하는지 확인
- [ ] 모바일 앱 알림과 후속 지시 흐름 확인

## 참고 자료

- 공식 사이트: <https://www.onorca.dev/>
- 문서: <https://www.onorca.dev/docs> (설치: `/docs/install`, 첫 세션: `/docs/first-session`, Worktrees: `/docs/model/worktrees`)
- GitHub: <https://github.com/stablyai/orca>
