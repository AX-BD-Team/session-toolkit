---
name: s-end
description: |
  세션 종료 시 코드 커밋 + 문서 갱신 + git push + CI/CD 배포.
  프로젝트에 맞게 동작: SPEC.md/CHANGELOG.md가 있으면 갱신, 없으면 건너뜀.
argument-hint: "[추가 메모]"
user-invocable: true
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# Session End — 커밋 + 문서 갱신 + Push + 배포

## 아키텍처

```
1. Git 커밋 (코드 변경)
2. 프로젝트 문서 갱신 (있으면)
3. Auto Memory 갱신 (MEMORY.md)
4. 문서 커밋
5. Git push (CI/CD 자동 트리거)
```

## Steps

### Phase 1: Git 커밋

1. `git status` + `git diff`로 변경사항 확인
2. 코드 변경사항 커밋 (문서 파일 제외):
   - 논리적 단위로 분리 가능하면 여러 커밋으로 나누기
   - 컨벤션: `feat:`, `fix:`, `refactor:`, `chore:`, `docs:`
   - `.env`, 자격 증명 파일 커밋 금지
3. 프로젝트의 검증 명령 실행 (lint, typecheck 등)

### Phase 2: 프로젝트 문서 갱신 (선택)

프로젝트에 다음 파일이 있으면 갱신한다. 없으면 건너뜀.

**SPEC.md** (또는 유사 상태 파일):
- 숫자/지표만 업데이트 (테스트 수, 빌드 상태 등)
- 세션 히스토리는 추가하지 않음

**CHANGELOG.md** (또는 docs/CHANGELOG.md):
- 파일 상단에 이번 세션 기록 추가:
```markdown
### 세션 NNN (YYYY-MM-DD)
**[작업 요약 1줄]**:
- ✅ [변경 1]
- ✅ [변경 2]

**검증 결과**:
- ✅ typecheck / lint / tests / build
```

### Phase 3: Auto Memory 갱신

Auto Memory 디렉토리의 MEMORY.md를 업데이트:

1. **현재 버전 & 상태**: 최신화
2. **최근 세션 요약** (sliding window, 최대 3개):
   - 이번 세션을 맨 위에 1줄 요약으로 추가
   - 3개 초과 시 가장 오래된 것 제거
3. **주요 지표**: 변경된 숫자만 업데이트
4. **다음 작업**: `$ARGUMENTS`의 추가 메모 반영

### Phase 4: 문서 커밋

```bash
git add SPEC.md docs/CHANGELOG.md  # 존재하는 파일만
git commit -m "docs: update SPEC.md + CHANGELOG — 세션 NNN [요약]"
```

### Phase 5: Git Push + 배포

```bash
git push origin $(git branch --show-current)
```

Push 후 CI/CD가 설정되어 있으면 배포 상태를 확인:
```bash
gh run list --limit 1 2>/dev/null || echo "GitHub Actions 미설정"
```

- CI/CD 성공 시 문서에 배포 상태 갱신 + 추가 커밋+push
- 실패 시 `gh run view --log-failed`로 원인 확인 후 보고

### 최종 요약 출력

```
## 세션 종료 완료

### Git 커밋
- `abc1234` feat: [메시지]
- `def5678` docs: update SPEC.md + CHANGELOG — 세션 NNN

### 배포
- CI/CD: ✅ 성공 / ❌ 실패 / ⏭️ 미설정

### 업데이트
- SPEC.md: [변경된 지표]
- MEMORY.md: 컨텍스트 갱신 완료
- CHANGELOG.md: 세션 NNN 추가
```

## 주의사항

- 프로젝트에 SPEC.md/CHANGELOG.md가 없으면 해당 Phase를 건너뜀
- MEMORY.md는 Git 추적 대상이 아님 (auto memory 디렉토리)
- CHANGELOG.md는 최신이 파일 상단에 오도록 prepend
