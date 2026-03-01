---
name: s-start
description: |
  세션 시작 시 프로젝트 컨텍스트를 복원한다.
  Auto Memory(MEMORY.md)에서 즉시 맥락 파악 후, SPEC.md 등 프로젝트 사양 파일을 보충 읽기.
argument-hint: "[오늘 작업]"
user-invocable: true
allowed-tools:
  - Read
  - Glob
  - Grep
  - Bash
---

# Session Start — 프로젝트 컨텍스트 복원

## Steps

### 1. Auto Memory 확인

MEMORY.md는 자동 로딩되므로, 내용을 기반으로 현재 상태를 파악한다:
- 최근 세션 요약
- 주요 지표
- 다음 작업

### 2. 프로젝트 사양 파일 읽기

프로젝트에 다음 파일이 있으면 읽어 컨텍스트를 보충한다:

```bash
# 프로젝트 사양 파일 탐색
for f in SPEC.md docs/SPEC.md spec.md PROJECT.md; do
  [ -f "$f" ] && echo "Found: $f"
done
```

SPEC.md (또는 유사 파일)가 있으면 읽어 현재 상태 섹션을 파악한다.

### 3. Git 상태 확인

```bash
git status --short
git log --oneline -3
```

### 4. 작업 계획 수립

`$ARGUMENTS`에 오늘 작업이 명시되어 있으면:
- 관련 파일/모듈 탐색
- 작업 범위 파악

명시되지 않았으면:
- MEMORY.md의 "다음 작업" 항목을 기반으로 제안

### 5. 세션 시작 안내

```
## 세션 시작

### 프로젝트 상태
- 버전: [버전]
- 마지막 세션: [요약]
- Git: [브랜치, clean/dirty]

### 오늘 작업
- [작업 1]
- [작업 2]

### 관련 파일
- [파일 목록]
```
