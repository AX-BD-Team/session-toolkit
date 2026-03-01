---
name: git-sync
description: |
  Git을 통한 멀티 환경(Windows/WSL/Mac) 간 프로젝트 동기화.
  로컬과 GitHub 리모트 상태를 비교하고 push/pull로 동기화한다.
  Use when user mentions: git sync, 환경 동기화, push, pull, 코드 동기화, 환경 전환
argument-hint: "[push|pull|status|stash]"
user-invocable: true
allowed-tools:
  - Bash
  - Read
---

# Git Sync — 멀티 환경 프로젝트 동기화

여러 환경에서 GitHub을 허브로 사용하여 프로젝트를 동기화한다.

## 서브커맨드 분기

- `$ARGUMENTS` 비어 있거나 `status` → **Status 플로우**
- `push` → **Push 플로우** (환경 떠나기 전)
- `pull` → **Pull 플로우** (환경 진입 후)
- `stash` → **Stash 상태 확인**

---

## 공통 로직 (모든 플로우에서 먼저 실행)

### Step 1: 로컬 상태 수집

```bash
git status --short
git stash list
git log --oneline -5
```

### Step 2: 리모트 최신 정보 가져오기

```bash
git fetch origin
```

### Step 3: 로컬 ↔ 리모트 비교

```bash
git branch --show-current
git rev-list --left-right --count HEAD...origin/$(git branch --show-current)
```

### Step 4: 상태 분류

| 조건 | 상태 |
|------|------|
| LOCAL_AHEAD=0, REMOTE_AHEAD=0 | `synced` |
| LOCAL_AHEAD>0, REMOTE_AHEAD=0 | `ahead` (push 필요) |
| LOCAL_AHEAD=0, REMOTE_AHEAD>0 | `behind` (pull 필요) |
| LOCAL_AHEAD>0, REMOTE_AHEAD>0 | `diverged` (주의) |

---

## Status 플로우

공통 로직 실행 후 결과를 출력한다.
`diverged` 상태인 경우 경고를 표시하고 rebase/merge 중 택일을 확인.

## Push 플로우

1. 로컬 변경사항 확인
2. 변경사항이 있으면 WIP 커밋 또는 정식 커밋 선택
3. 정식 커밋 시 검증 실행 (lint, typecheck 등)
4. `git push origin <현재브랜치>`
5. 결과 출력

## Pull 플로우

1. behind=0이면 "이미 최신" 출력 후 종료
2. 미커밋 변경사항이 있으면 stash 또는 WIP 커밋 후 진행
3. `git pull --rebase origin <현재브랜치>`
4. stash 복원 (해당 시)
5. WIP 커밋 정리 제안
6. 결과 출력

## Stash 플로우

```bash
git stash list
```

stash가 있으면 목록 표시 + pop/drop/show 선택지 제공.

## 엣지 케이스

- **diverged**: push 전에 반드시 pull --rebase 선행. force push 금지.
- **WIP 커밋 누적**: 3개 이상이면 squash 제안.
- **민감 파일**: `.env`, `.dev.vars` 등 절대 커밋하지 않음.
