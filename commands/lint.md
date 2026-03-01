---
name: lint
description: |
  변경된 파일의 ESLint/TypeScript 에러를 점검하고 수정한다.
  프로젝트의 lint/typecheck 명령을 자동 감지하여 실행.
user-invocable: true
allowed-tools:
  - Bash
  - Read
  - Edit
  - Glob
  - Grep
---

# Lint — 코드 품질 점검 + 자동 수정

## Steps

### 1. 프로젝트 패키지 매니저 감지

```bash
if [ -f "pnpm-lock.yaml" ]; then
  PM="pnpm"
elif [ -f "yarn.lock" ]; then
  PM="yarn"
elif [ -f "bun.lockb" ]; then
  PM="bun"
else
  PM="npm"
fi
echo "Package manager: $PM"
```

### 2. 변경된 파일 확인

```bash
git diff --name-only HEAD
git diff --name-only --staged
```

### 3. Lint 실행

```bash
$PM lint 2>&1
```

에러가 있으면:
1. 에러 메시지를 분석하여 자동 수정 가능한 것은 직접 수정
2. `$PM lint --fix` 사용 가능하면 실행
3. 수정 후 재실행하여 0 errors 확인

### 4. TypeScript 타입 체크

```bash
$PM typecheck 2>&1 || $PM tsc --noEmit 2>&1
```

에러가 있으면:
1. 에러 파일과 위치를 파악
2. 타입 에러를 직접 수정
3. 재실행하여 0 errors 확인

### 5. 결과 출력

```
## Lint 결과

- ESLint: ✅ 0 errors / ❌ N errors (수정 완료/수동 수정 필요)
- TypeScript: ✅ 0 errors / ❌ N errors (수정 완료/수동 수정 필요)
- 수정된 파일: [목록]
```
