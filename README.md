# worktree-context

Git worktree 사이에서 AI 작업 문맥을 가볍게 이어주는 Agent Skill입니다.

세션 전체를 공유하지 않습니다. 대신 **작업을 계속하는 데 필요한 최소한의 checkpoint**와 **다른 worktree에도 유용한 검증된 사실**만 Git의 common directory에 보존합니다.

## 목적

`worktree-context`는 다음 상황을 위한 Skill입니다.

- 하나의 AI CLI가 여러 worktree를 오가며 작업할 때
- Codex, Claude Code, Antigravity 등 여러 AI CLI가 서로 다른 worktree에서 병렬 작업할 때
- 세션을 `/clear`하거나 새로 시작한 뒤 이전 worktree 작업을 빠르게 이어갈 때

핵심 원칙:

> 계속할 만큼만 기억하고, 판단을 오염시킬 만큼 기억하지 않는다.

런타임 context는 repository에 commit하지 않고 Git common directory 아래에 저장합니다.

```text
.git/worktree-context/
├── shared.md
└── worktrees/
    ├── main.md
    ├── perf-analysis.md
    └── implementation.md
```

- `worktrees/*.md`: 각 worktree의 목적, 진행 상태, 다음 작업
- `shared.md`: 여러 worktree에서 재사용할 가치가 있는 **검증된** 사실/결정/실패만 저장

## 설치

전역 설치를 권장합니다. 새 worktree를 만들 때마다 다시 설치할 필요가 없습니다.

```bash
npx skills@latest add al-hub/worktree-context -g
```

설치 화면에서 사용할 AI agent(Codex, Claude Code, Antigravity 등)를 선택하면 됩니다.

여러 agent에 한 번에 설치하려면:

```bash
npx skills@latest add al-hub/worktree-context -g \
  --agent codex claude-code antigravity antigravity-cli --yes
```

> **현재 `skills` CLI 주의사항**
> `skills@1.5.21` 계열에는 Codex/Antigravity 같은 일부 universal agent의 global 설치가
> agent별 global skills 경로 대신 `~/.agents/skills`에만 설치되는 upstream 이슈가 보고되어 있습니다.
> 설치 후 해당 CLI에서 skill이 보이지 않으면 `npx skills@latest list -g`로 설치 상태를 확인하고,
> upstream 수정 전까지는 해당 agent의 global skills 경로에 연결이 필요한지 확인하세요.

## 삭제

간단히:

```bash
npx skills@latest remove worktree-context -g
```

모든 agent에서 바로 제거:

```bash
npx skills@latest remove worktree-context -g --agent '*' --yes
```

Skill 삭제는 각 repository의 저장된 checkpoint를 자동으로 지우지 않습니다.
checkpoint까지 지우려면 해당 Git repository 안에서:

```bash
rm -rf "$(git rev-parse --path-format=absolute --git-common-dir)/worktree-context"
```

## 간단한 사용 예시

### 1. worktree를 만들어 작업

AI CLI에서 평소처럼 요청합니다.

```text
성능 문제는 별도 worktree를 만들어서 분석해줘.
```

Skill은 생성된 worktree의 목적과 시작점을 checkpoint로 남깁니다.

### 2. 다른 worktree로 이동했다가 복귀

```text
일단 main으로 돌아가서 다른 작업하자.
```

현재 worktree의 핵심 진행 상태를 checkpoint합니다.

나중에:

```text
아까 성능 분석 worktree로 돌아가서 계속해줘.
```

저장된 checkpoint와 현재 Git 상태를 확인한 뒤 이어서 작업합니다.

### 3. 여러 AI CLI가 병렬 작업

```text
Codex       → perf-analysis worktree
Claude Code → implementation worktree
Antigravity → verification worktree
```

각 AI의 작업 상태는 worktree별로 분리합니다.

다른 worktree에도 가치가 있는 benchmark 결과, 검증된 제약, 실패한 접근 같은 정보만 `shared.md`에 공유합니다.

### 4. `/clear` 후 다시 시작

`/clear` 명령 자체를 가로채는 hook은 사용하지 않습니다.

대신 의미 있는 작업 단계마다 작은 checkpoint를 유지합니다. 따라서 새 세션에서 같은 worktree를 이어가면 최근 checkpoint를 읽고 필요한 문맥만 복원합니다.

```text
긴 session
   ↓
checkpoint
   ↓
/clear
   ↓
새 session
   ↓
작은 context로 resume
```

## 동작 원칙

- 일반 Git 작업에서는 불필요하게 개입하지 않습니다.
- worktree 생성, 전환, 복귀, 이어서 작업할 때 사용합니다.
- 전체 대화나 chain-of-thought를 저장하지 않습니다.
- 추측은 shared context에 올리지 않습니다.
- test, benchmark, 코드로 확인된 재사용 가능한 정보만 공유합니다.
- 현재 코드와 test 결과가 저장된 context보다 항상 우선합니다.
- checkpoint의 HEAD와 현재 HEAD가 다르면 오래된 정보는 참고 정보로 취급하고 필요한 부분을 재검증합니다.

## 업데이트

```bash
npx skills@latest update worktree-context
```

## License

MIT
