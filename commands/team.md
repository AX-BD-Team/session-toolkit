---
name: team
description: |
  Agent Teams를 tmux in-window split에서 병렬 수행한다.
  리더 pane 옆에 worker pane이 직접 보이며, 실제 Claude Code TUI가 표시된다.
  완료 후 원래 레이아웃으로 자동 복원.
argument-hint: "<작업 설명>"
user-invocable: true
allowed-tools:
  - Bash
  - Read
  - Write
  - Glob
  - Grep
  - Edit
---

# Team — tmux In-Window Split Agent Teams

`$ARGUMENTS`로 전달된 작업을 분석하여, **리더와 같은 tmux window 내**에서 병렬 `claude` 인스턴스를 실행한다.
Worker pane은 리더 pane 옆에 직접 보이며, 완료 후 원래 레이아웃으로 자동 복원된다.

**리더 pane은 크기/위치/프로세스 모두 변경하지 않는다.**

## 환경 규칙

이 프로젝트는 **Windows + WSL** 또는 네이티브 Linux 환경에서 동작한다.

### 환경 자동 감지 (Step 0에서 수행)

Step 시작 전, 아래 명령으로 환경을 판별한다:
```bash
if grep -qi microsoft /proc/version 2>/dev/null; then
  echo "WSL_DIRECT"   # WSL 내부에서 직접 실행 — tmux 바로 호출
else
  echo "NATIVE"       # 네이티브 Linux — tmux 직접 호출
fi
```

## Steps

### 1. 작업 분석 및 팀 구성 결정

`$ARGUMENTS`의 작업 설명을 분석하여 다음을 결정한다:

- **팀 이름**: 작업 키워드 기반 kebab-case
- **worker 수**: 2~3명 (같은 column 내 분할이므로 **최대 3명** 권장)
- **역할 분배**: worker끼리 **같은 파일을 동시 수정하지 않도록** 파일/모듈/레이어 기준으로 분할
- **태스크 요약**: 각 worker의 작업을 **10자 내외 한줄**로 요약
- **allowedTools 결정**: 태스크에 필요한 도구 목록 결정
  - 읽기만: `Read,Glob,Grep`
  - 수정 포함: `Read,Edit,Write,Glob,Grep,Bash`

작업 분석 시 코드베이스를 탐색하여 실제 대상 파일과 범위를 파악한다.

### 2. Worker 프롬프트 작성

각 worker에게 전달할 프롬프트를 작성한다. 프롬프트에 반드시 포함:

- **구체적인 파일 경로와 수정 내용** (가장 중요)
- 작업 범위 제한
- 프로젝트 규칙 (CLAUDE.md의 관련 섹션, 간략히)
- 작업 완료 기준

먼저 임시 디렉토리를 생성하고, 프롬프트를 **임시 파일**에 저장한다:

```bash
TEAM_DIR="$PWD/.team-tmp"
mkdir -p "$TEAM_DIR"
```

```bash
cat > "$TEAM_DIR/team-{팀이름}-worker-{N}.txt" << 'PROMPT'
[worker 프롬프트 내용]
PROMPT
```

### 3. Worker 생성 (tmux in-window split)

> **CRITICAL**: worker는 리더와 **같은 window** 내에서 실행된다.
> 리더 pane 자체를 **가로 분할(`-h`)**하여 오른쪽에 worker 전용 컬럼을 생성한다.
> **리더 외 다른 pane의 크기/위치/프로세스는 절대 변경하지 않는다.**
> **break-pane, swap-pane, select-window 사용 금지**

**3a. pane 구조 감지 및 최소 넓이 확인**:

```bash
LEADER_PANE=$(tmux display-message -p '#{pane_id}')
LEADER_WIDTH=$(tmux display-message -p '#{pane_width}')

echo "=== Current Pane Layout ==="
tmux list-panes -F "pane=#{pane_id} left=#{pane_left} top=#{pane_top} size=#{pane_width}x#{pane_height}"
echo "Leader: $LEADER_PANE (width=$LEADER_WIDTH)"

if [ "$LEADER_WIDTH" -lt 80 ]; then
  echo "ERROR: Leader pane too narrow ($LEADER_WIDTH cols). Minimum 80 required."
  exit 1
fi
```

**3b. worker runner 스크립트**를 생성한다:

```bash
PROJECT_DIR="$PWD"
TEAM_DIR="$PWD/.team-tmp"
CLAUDE_CFG="${CLAUDE_CONFIG_DIR:-$HOME/.claude}"

cat > "$TEAM_DIR/team-{팀이름}-run-{N}.sh" << RUNNER
#!/usr/bin/env bash
export PATH="\$HOME/.local/bin:\$PATH"
export CLAUDE_CONFIG_DIR="$CLAUDE_CFG"
cd $PROJECT_DIR

TASK_SUMMARY="{태스크요약}"
tmux select-pane -t "\$TMUX_PANE" -T "W{N}: \$TASK_SUMMARY ⏳" 2>/dev/null

echo ""
echo "╔══════════════════════════════════════════╗"
echo "║  Worker {N} — \$TASK_SUMMARY"
echo "║  시작: \$(date '+%H:%M:%S')"
echo "╚══════════════════════════════════════════╝"
echo ""

prompt=\$(cat "$TEAM_DIR/team-{팀이름}-worker-{N}.txt")
command claude "\$prompt" \\
  --allowedTools 'Read,Edit,Write,Glob,Grep,Bash' \\
  --max-turns 20

echo ""
echo "╔══════════════════════════════════════════╗"
echo "║  Worker {N} — 완료 ✅"
echo "║  종료: \$(date '+%H:%M:%S')"
echo "╚══════════════════════════════════════════╝"
echo '=== WORKER-{N} DONE ===' > "$TEAM_DIR/team-{팀이름}-worker-{N}.log"
tmux select-pane -t "\$TMUX_PANE" -T "W{N}: \$TASK_SUMMARY ✅" 2>/dev/null
RUNNER
chmod +x "$TEAM_DIR/team-{팀이름}-run-{N}.sh"
```

> **`command claude`**: `.bashrc`의 alias를 우회하여 실제 바이너리를 호출한다.
> **`CLAUDE_CONFIG_DIR`**: 리더 세션의 인증 컨텍스트를 worker에 전파한다.

**3c. launcher 스크립트를 생성**한다:

```bash
cat > "$TEAM_DIR/team-{팀이름}-launcher.sh" << LAUNCHER
#!/usr/bin/env bash
set -e
TEAM="{팀이름}"
TEAM_DIR="$TEAM_DIR"
LEADER_PANE="$LEADER_PANE"
PROJECT_DIR="$PROJECT_DIR"
WORKER_COUNT={N}

# 0) 기존 worker pane 정리 (재실행 시)
if [ -f "\$TEAM_DIR/team-\${TEAM}-worker-panes.txt" ]; then
  while read -r old_pane; do
    tmux kill-pane -t "\$old_pane" 2>/dev/null || true
  done < "\$TEAM_DIR/team-\${TEAM}-worker-panes.txt"
  rm -f "\$TEAM_DIR/team-\${TEAM}-worker-panes.txt"
fi

# 1) Worker 컬럼 생성 — 리더 pane을 가로 분할, 오른쪽 50%
WORKER_COL=\$(tmux split-window -h -d -t "\$LEADER_PANE" -l 50% -c "\$PROJECT_DIR" -P -F '#{pane_id}')
echo "\$WORKER_COL" > "\$TEAM_DIR/team-\${TEAM}-worker-panes.txt"
sleep 2

# 2) Worker 컬럼을 worker 수에 따라 세로 분할 + 실행
if [ "\$WORKER_COUNT" -eq 1 ]; then
  tmux send-keys -t "\$WORKER_COL" "bash \$TEAM_DIR/team-\${TEAM}-run-1.sh" Enter

elif [ "\$WORKER_COUNT" -eq 2 ]; then
  W2=\$(tmux split-window -v -d -t "\$WORKER_COL" -l 50% -c "\$PROJECT_DIR" -P -F '#{pane_id}')
  echo "\$W2" >> "\$TEAM_DIR/team-\${TEAM}-worker-panes.txt"
  sleep 2
  tmux send-keys -t "\$WORKER_COL" "bash \$TEAM_DIR/team-\${TEAM}-run-1.sh" Enter
  tmux send-keys -t "\$W2" "bash \$TEAM_DIR/team-\${TEAM}-run-2.sh" Enter

elif [ "\$WORKER_COUNT" -eq 3 ]; then
  W2=\$(tmux split-window -v -d -t "\$WORKER_COL" -l 67% -c "\$PROJECT_DIR" -P -F '#{pane_id}')
  echo "\$W2" >> "\$TEAM_DIR/team-\${TEAM}-worker-panes.txt"
  sleep 2
  W3=\$(tmux split-window -v -d -t "\$W2" -l 50% -c "\$PROJECT_DIR" -P -F '#{pane_id}')
  echo "\$W3" >> "\$TEAM_DIR/team-\${TEAM}-worker-panes.txt"
  sleep 2
  tmux send-keys -t "\$WORKER_COL" "bash \$TEAM_DIR/team-\${TEAM}-run-1.sh" Enter
  tmux send-keys -t "\$W2" "bash \$TEAM_DIR/team-\${TEAM}-run-2.sh" Enter
  tmux send-keys -t "\$W3" "bash \$TEAM_DIR/team-\${TEAM}-run-3.sh" Enter
fi

# 3) Pane 타이틀 표시 활성화
tmux set-option -w pane-border-status top 2>/dev/null
tmux select-pane -t "\$LEADER_PANE" -T "Leader: \$TEAM" 2>/dev/null

echo "Workers created: WORKER_COUNT=\$WORKER_COUNT"
tmux list-panes -F "pane=#{pane_id} left=#{pane_left} top=#{pane_top} size=#{pane_width}x#{pane_height}"
LAUNCHER
chmod +x "$TEAM_DIR/team-{팀이름}-launcher.sh"
```

**3d. launcher를 실행**한다:
```bash
bash "$TEAM_DIR/team-{팀이름}-launcher.sh"
```

**3e. worker pane 생성을 검증**한다.

### 4. 모니터링

DONE 마커를 확인하여 완료를 감지한다:

```bash
TEAM_DIR="$PWD/.team-tmp"
ALL_DONE=true
for i in 1 2; do
  if grep -q "WORKER-${i} DONE" "$TEAM_DIR/team-{팀이름}-worker-${i}.log" 2>/dev/null; then
    echo "Worker ${i}: DONE"
  else
    echo "Worker ${i}: RUNNING"
    ALL_DONE=false
  fi
done
echo "ALL_DONE=$ALL_DONE"
```

### 5. 검증

모든 worker 완료 후, 리더가 직접 검증한다. 프로젝트의 검증 명령을 실행:
- lint, typecheck, test 등 프로젝트에 맞는 명령을 사용

### 6. 정리 및 결과 출력

1. 로그 파일에서 결과 수집
2. **worker pane 회수**:
```bash
TEAM_DIR="$PWD/.team-tmp"
TEAM="{팀이름}"
while read -r pane_id; do
  tmux kill-pane -t "$pane_id" 2>/dev/null || true
done < "$TEAM_DIR/team-${TEAM}-worker-panes.txt"

REMAINING_TEAMS=$(ls "$TEAM_DIR"/team-*-worker-panes.txt 2>/dev/null | grep -v "team-${TEAM}-" | wc -l)
if [ "$REMAINING_TEAMS" -eq 0 ]; then
  tmux set-option -w pane-border-status off 2>/dev/null
fi
```
3. 임시 파일 정리
4. 결과 요약 출력

## 주의사항

- **자기 영역만 분할**: 리더 pane을 `-h`로 분할, worker pane kill 시 자동 복원
- **기존 pane에 send-keys/kill 금지**: 새로 생성한 worker pane에만 사용
- **`command claude`**: alias 충돌 방지 필수
- **`sleep 2`**: pane 생성 후 필수 (shell init delay)
- **`--allowedTools`**: 미지정 시 승인 프롬프트로 pane이 멈춤
- **git 작업은 리더만** 수행 (worker에게 commit/push 시키지 않음)
- `$ARGUMENTS`가 비어있으면 사용자에게 작업 설명을 요청한다
