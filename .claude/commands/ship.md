---
description: 구현부터 병합까지 7단계 파이프라인을 순서대로 수행합니다 (구현 → commit → PR → 리뷰 → 반영 → Ready → 병합).
argument-hint: "[작업 설명] [--loop=N] [--from=N] [--stop-at=N] [--base=<branch>] [--no-cleanup] [--post] [-y|--yes] [-h|--help]"
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, AskUserQuestion, Task, mcp__github__create_pull_request, mcp__github__get_pull_request, mcp__github__list_pull_requests, mcp__github__update_pull_request, mcp__github__merge_pull_request, mcp__github__add_issue_comment, mcp__github__list_workflow_runs, mcp__github__list_workflow_jobs, mcp__github__get_pull_request_files
---

# /ship

작업을 **구현부터 병합까지** 한 번에 흘려보낸다.

```
1. 구현        2. /commit      3. /create-pr(Draft)   4. /pr-review
5. /apply-review   6. Ready 전환(CI)   7. 병합
```

한국어로 응답한다. 각 단계 결과를 한 줄로 보고하며 진행한다.

## 입력 인자

사용자가 넘긴 인자: `$ARGUMENTS`

- `--loop=N`: 1단계(구현)를 지속 반복 모드로, 최대 N회. **미지정이 기본**(직접 구현)
- `--from=N`: N단계부터 재개. 실패 후 이어받을 때 쓴다
- `--stop-at=N`: N단계까지만 수행 (예: `--stop-at=6` = 병합하지 않음)
- `--base=<branch>`: base 브랜치 강제 지정. 미지정 시 자동 추론
- `--no-cleanup`: 병합 후 브랜치 삭제 생략
- `--post`: 병합 후 진행 문서 갱신·실기기 체크리스트 출력까지 수행
- `-y`, `--yes`: 병합 직전 확인을 생략
- `-h`, `--help`: 도움말 출력 후 종료

인자 없이 호출하면 현재 브랜치 상태를 판정해 **다음에 해야 할 단계부터** 이어서 진행한다.

### Help Mode

`-h`/`--help` 가 있으면 아래만 출력하고 **아무것도 실행하지 않는다**.

```
=== /ship 도움말 ===

Usage: /ship [작업 설명] [options]
Example: /ship "W6 App 조립"
         /ship --from=4
         /ship --stop-at=6 --loop=10

-- 단계 --
  1 구현            2 /commit        3 /create-pr (항상 Draft)
  4 /pr-review      5 /apply-review  6 Ready 전환 + CI 대기
  7 병합 (+ 브랜치 정리)

-- Options --
  --loop=N       구현을 지속 반복 모드로, 최대 N회 (기본: 직접 구현)
  --from=N       N단계부터 재개
  --stop-at=N    N단계까지만
  --base=<br>    base 브랜치 지정 (기본: 자동 추론)
  --no-cleanup   병합 후 브랜치 삭제 생략
  --post         병합 후 진행 문서 갱신·실기기 체크리스트
  -y, --yes      병합 직전 확인 생략
  -h, --help     이 도움말

-- 정지 조건 --
  게이트(빌드·테스트·린트) 실패 · CI red · 리뷰 CRITICAL · merge conflict
  → 그 단계에서 멈추고 원인을 보고한다. 다음 단계로 넘어가지 않는다.
========================
```

---

## 설계 원칙 (구현 전 반드시 읽을 것)

### A. PR 은 항상 Draft 로 만들고, 반영이 끝난 뒤에 Ready 로 올린다

CI 가 Draft 를 건너뛰는 설정이면, Ready 전환 후의 모든 push 가 CI 를 재실행시킨다.
**3→4→5 를 Draft 상태에서 끝내고 6 에서 한 번만 Ready** 로 올리면 CI 가 1회로 끝난다.
순서를 바꾸면(생성→Ready→리뷰→반영) CI 가 2회 이상 돈다.

### B. 진행 상태는 파일이 아니라 실물에서 판정한다

상태 파일을 만들지 마라. 반드시 실제 진행과 어긋난다.
매 단계 시작 전 아래를 **직접 조회**해 어디까지 됐는지 판정한다.

| 판정 | 근거 |
|------|------|
| 구현/커밋 완료 | 워킹트리 clean **그리고** `<base>..HEAD` 에 커밋 존재 |
| PR 생성됨 | `list_pull_requests(head: 현재 브랜치, state: open)` |
| 리뷰 완료 | 그 PR 에 자체 리뷰 결과 코멘트 존재 |
| Ready 상태 | PR 의 `draft` 필드 |
| CI 결과 | `list_workflow_jobs(run_id)` 의 `conclusion` |

`--from` 이 주어지면 그 단계부터, 없으면 위 판정 결과의 **다음 단계**부터 시작한다.
이미 끝난 단계는 건너뛰고 그 사실을 한 줄로 알린다.

### C. 게이트를 통과해야 다음으로 간다

커밋 전·반영 후에 **CI 와 동일한 검증을 로컬에서** 돌린다. 실패하면 그 자리에서 멈춘다.
검증하지 않고 커밋하면 컴파일도 안 되는 커밋이 남는다.

---

## 0단계: 사전 점검

```bash
git rev-parse --abbrev-ref HEAD
git status --porcelain
git remote get-url origin
```

- 현재 브랜치가 `main`/`master`/`develop` 등 보호 브랜치면 **중단**하고 알린다.
- base 브랜치 결정: `--base` → `CLAUDE.md`/`AGENTS.md` 의 명시 → `origin/develop` → `origin/main` → `origin/master`.
- 워킹트리가 dirty 한데 `--from` 이 2 이상이면, 이전 작업 잔여물일 수 있으므로 목록을 보여주고 진행 여부를 확인한다(AskUserQuestion).

### 검증 명령 자동 감지

`.github/workflows/*.yml` 을 읽어 **CI 가 실제로 실행하는 스텝**을 추출해 게이트로 쓴다.
`run:` 으로 시작하는 명령 중 빌드·테스트·린트에 해당하는 것을 순서대로 모은다.

감지에 실패하면 프로젝트 관례를 찾는다(`package.json` scripts, `Makefile`, `CLAUDE.md`).
그래도 없으면 사용자에게 검증 명령을 묻고, 답이 없으면 **게이트 없이 진행하되 그 사실을 명시**한다.

> CI 와 같은 명령을 로컬에서 먼저 돌리는 것이 이 커맨드의 핵심이다. 6단계 CI 실패를 앞당겨 잡는다.

### 문서 전용 변경 판정

변경 파일이 전부 `docs/**` · `**/*.md` 류이면 **docs-only** 로 표시한다.
CI 워크플로에 `paths-ignore` 로 그 경로가 있으면 6단계에서 CI 를 기다리지 않는다.

---

## 1단계: 구현

`--loop=N` 이 있으면 지속 반복 모드로, 없으면 직접 구현한다.

- 작업 설명이 인자로 주어졌으면 그것을, 아니면 브랜치명·기획 문서에서 범위를 파악한다.
- 이 레포에 주차별 기획 문서 관례가 있으면 **해당 문서를 먼저 읽고** 그 결정을 따른다.
  기획 문서가 있어야 할 자리에 없으면 경고한다(구현 전 기획 원칙).

**게이트**: 0단계에서 감지한 검증 명령을 전부 실행한다.

- 하나라도 실패하면 **중단**. "게이트 실패 = 미완성"이다. 테스트를 지우거나 약화시켜 통과시키지 마라.
- 테스트를 새로 추가했다면 **실행 로그에 그 이름이 찍히는지 확인**한다.
  통과 개수만 보면 "수집되지 않아 조용히 건너뛴 테스트"를 놓친다.

### 위임에 대해

병렬화 이득이 확실한 큰 작업이 아니면 직접 구현하는 편이 빠르다.
위임하는 경우에도 **산출물을 직접 읽어 검증**한다. 에이전트의 완료 보고를 근거로 삼지 마라.
정체된 에이전트는 기다리지 말고 중단시키고 직접 처리한다.

---

## 2단계: /commit

`~/.claude/commands/commit.md` 의 절차를 따른다. 재구현하지 마라.

- **관심사·계층별로 커밋을 나눈다.** 도메인/데이터/화면/테스트/문서를 한 덩어리로 묶지 않는다.
- 여러 에이전트가 동시에 파일을 만드는 중이면 **경로를 명시해 선택적으로 스테이징**한다(`git add -A` 금지).
- 레포의 기존 커밋 컨벤션을 감지해 따른다(gitmoji 사용 여부, Co-Authored-By 사용 여부).
- push 는 3단계에서 한다.

---

## 3단계: /create-pr

`~/.claude/commands/create-pr.md` 의 절차를 따르되, **한 가지를 덮어쓴다.**

> **PR 은 항상 Draft 로 생성한다** (`draft: true`). `--open` 을 넘기지 마라.
> Ready 전환은 6단계의 일이다. 원칙 A 참조.

본문에 다음을 반드시 포함한다.

- 기획 문서 링크(있으면)
- **구현 중 기획과 달라진 점**과 그 근거
- **구현 중 발견한 결함**과 조치
- 검증 결과(테스트 건수·린트)
- 자동으로 채울 수 없는 것(스크린샷·실기기 검증)은 **미수행이라고 명시**한다. 비워두고 넘어가지 마라.

---

## 4단계: /pr-review

`~/.claude/commands/pr-review.md` 의 절차를 따른다. 대상은 3단계에서 만든 PR 번호.

**반드시 지킬 것:**

- 이 파이프라인에서는 **작성자와 리뷰어가 동일**하다. 리뷰 결과와 PR 코멘트에 그 사실을 명시한다.
  구조적 사각이 남는다는 점을 숨기지 마라.
- 변경 500줄 초과면 외부 리뷰어를 권고하는 문장을 넣는다.
- 위임한 리뷰 에이전트가 **빈 응답으로 끝나면 그 결과를 리뷰로 취급하지 마라.**
  직접 검토하고, 위임이 실패했다는 사실을 보고한다.
- Draft PR 이므로 submit 은 `COMMENT` 로 제한된다.

**CRITICAL 이 1건 이상이면** 5단계에서 반드시 반영한다. 반영 없이 6단계로 넘어가지 마라.

---

## 5단계: /apply-review

`~/.claude/commands/apply-review.md` 의 절차를 따른다.

- CRITICAL·WARNING 은 반영한다. SUGGESTION 은 판단하되 **미반영 사유를 남긴다**.
- 반영 후 **게이트를 다시 통과**해야 한다(1단계와 동일한 검증 명령).
- 반영분을 커밋하고 push 한다. **이 시점까지 PR 은 Draft 다.**
- 반영 내역을 PR 코멘트로 남긴다. 무엇을 왜 안 고쳤는지도 함께.

리뷰에서 지적한 내용이 **잘못된 것으로 드러나면** 고치지 말고 정정을 공개한다.
자기 리뷰의 오진을 그대로 코드에 반영하면 더 나빠진다.

---

## 6단계: Ready 전환 + CI

```
mcp__github__update_pull_request(draft: false)
```

**docs-only 이면** CI 가 트리거되지 않는다. `list_workflow_runs(branch)` 로 실행 0건을 확인하고 7단계로 간다.

**코드 변경이면** CI 를 기다린다.

- `list_workflow_runs(branch, perPage: 1)` 로 run id 를 확보한다(응답이 크므로 1회만).
- 이후에는 `list_workflow_jobs(run_id)` 로 확인한다(응답이 작다).
- **백그라운드 대기 후 조회**한다. 짧은 간격 폴링은 응답 크기 때문에 비싸다.
  첫 확인은 그 레포의 과거 실행 시간에 맞춰(모르면 10분), 이후 2분 간격.
- `conclusion: success` 가 아니면 **중단**하고 실패한 스텝과 로그를 보고한다.

---

## 7단계: 병합

**`--yes` 가 없으면 병합 직전에 확인한다**(AskUserQuestion). 되돌리기 비용이 가장 큰 단계다.

확인 시 다음을 함께 제시한다: PR 번호·제목, CI 결과, 커밋 수, 변경 규모, 미반영 리뷰 지적, 미수행 검증(실기기 등).

- 병합 방식은 레포 관례를 따른다(`git log --merges` 로 기존 병합 커밋 형태를 확인).
- 병합 후: `main` 동기화 → 병합된 브랜치 원격·로컬 삭제(`--no-cleanup` 이면 생략).
- **merge conflict 가 있으면 병합하지 말고 중단**한다.

---

## 8단계 (선택, `--post`)

- 진행 문서(PEP·로드맵 등)의 체크박스·상태를 실제와 맞춘다.
- 다른 병합 완료 브랜치가 남아 있으면 정리한다.
- **자동화할 수 없는 검증을 사람에게 넘긴다** — 실기기 확인 항목을 체크리스트로 출력한다.
  특히 영속·재시작·오프라인처럼 단위 테스트가 덮지 못하는 것.

---

## 최종 보고

```
=== /ship 완료 ===
- 브랜치   : <head> → <base>
- 커밋     : N개
- PR       : #NN <URL>  (병합됨 | Ready | Draft)
- CI       : success (M분 SS초) | 스킵(docs-only) | 미실행
- 리뷰     : CRITICAL n / WARNING n / SUGGESTION n  → 반영 n건
- 병합     : <merge sha> | 미수행(사유)
- 정리     : 브랜치 삭제 완료 | 생략

미완 사항
- <실기기 검증 등 사람이 해야 할 일>
- <미반영 리뷰 지적과 사유>
```

## 안전 규칙

- **force push · base 브랜치 직접 커밋 · 다른 PR 수정/클로즈를 하지 않는다.**
- 게이트 실패를 우회하려고 테스트를 지우거나 약화시키지 않는다.
- 중단할 때는 **어느 단계에서 왜 멈췄는지**와 `--from=N` 재개 방법을 함께 알린다.
- 검증하지 않은 것을 검증했다고 보고하지 않는다. 스크린샷·실기기처럼 못 한 것은 못 했다고 적는다.
