---
name: worktree-context
description: >-
  Preserve and restore minimal AI working context across Git worktrees and AI CLI
  sessions. Use when creating a git worktree, switching between worktrees,
  continuing work in an existing worktree, resuming after a cleared or restarted
  Codex/Claude/Antigravity session, or coordinating multiple AI agents working in
  separate worktrees. Keep per-worktree checkpoints separate and share only
  verified reusable facts, decisions, constraints, and failed approaches. Do not
  activate for ordinary single-worktree Git work that does not involve worktree
  creation, switching, resumption, or cross-worktree coordination.
---

# Worktree Context

Preserve just enough durable context to resume work across Git worktrees and disposable AI sessions.

Do not preserve full conversations or chain-of-thought.

## Core model

Use two kinds of memory:

1. **Local checkpoint** — private to one worktree. Stores where that worktree's task currently stands.
2. **Shared context** — visible to all worktrees. Stores only verified information that is reusable across worktrees.

Current code and test results always outrank stored context.

## Context location

Keep runtime state inside Git's common directory so every worktree of the same repository sees the same store without symlinks or committed files.

Resolve paths with:

```bash
COMMON_DIR="$(git rev-parse --path-format=absolute --git-common-dir)"
GIT_DIR="$(git rev-parse --path-format=absolute --git-dir)"

if [ "$COMMON_DIR" = "$GIT_DIR" ]; then
  WORKTREE_ID="main"
else
  WORKTREE_ID="$(basename "$GIT_DIR")"
fi

CONTEXT_DIR="$COMMON_DIR/worktree-context"
LOCAL_CONTEXT="$CONTEXT_DIR/worktrees/$WORKTREE_ID.md"
SHARED_CONTEXT="$CONTEXT_DIR/shared.md"

mkdir -p "$CONTEXT_DIR/worktrees"
touch "$SHARED_CONTEXT"
```

Do not create context files outside the current repository's Git common directory.

## Activate only for worktree workflows

Use this skill when any of these is true:

- The user asks the agent to create a Git worktree for a task.
- The agent is about to create or use a Git worktree as part of the requested work.
- The user asks to switch to another worktree.
- The user asks to return to or continue work in an existing worktree.
- A new/cleared AI session is continuing work inside an existing worktree.
- Multiple AI CLIs or agents are working in separate worktrees of the same repository.

Do not inject this workflow into ordinary Git work that has no worktree lifecycle or resumption need.

## On worktree creation

Create the new worktree normally.

Then create its local checkpoint with only:

```markdown
# Worktree Checkpoint

PURPOSE
<one concise task goal>

HEAD
<current commit>

STATUS
Not started

DONE
- None

CURRENT
- <first active step>

NEXT
- <next concrete action>
```

Include durable constraints only when they materially affect the task.

Never copy the full parent conversation into the checkpoint.

## On enter, resume, or session restart

Before substantial work:

1. Resolve the current local and shared context paths.
2. Read the local checkpoint if it exists.
3. Read `shared.md` only for information relevant to the current task.
4. Compare the checkpoint's recorded HEAD with the current HEAD.
5. If code changed since the checkpoint, treat affected stored claims as leads rather than truth and revalidate them against current code/tests.
6. Continue from `NEXT` when it is still valid.

Keep the restored prompt small. Do not replay history that is unnecessary to continue.

When useful, tell the user in one short line what was restored, for example:

```text
[perf-analysis restored] benchmark completed; next: validate persistent worker.
```

## Checkpoint the local worktree

Update the local checkpoint:

- after a meaningful milestone,
- before switching away from the worktree,
- after a material change to the plan,
- before finishing a substantial task when future continuation is plausible.

Keep it concise:

```markdown
# Worktree Checkpoint

PURPOSE
<stable task goal>

HEAD
<current commit>

DONE
- <completed durable result>

CURRENT
- <unfinished state>

NEXT
- <single best next action>

BLOCKERS
- <only real blockers, or None>
```

Rewrite only this worktree's local checkpoint. Never rewrite another worktree's checkpoint.

A `/clear` command may be handled directly by the AI CLI and cannot be assumed to trigger this skill. Therefore, checkpoint after meaningful milestones rather than relying on a clear hook.

## Share only verified reusable knowledge

Promote an item from local work to `shared.md` only when all are true:

1. It matters to another worktree or future parallel task.
2. It has been verified by code inspection, test, benchmark, command output, or another concrete artifact.
3. It is concise enough to reuse without the originating conversation.
4. It is not merely a hypothesis or temporary implementation detail.

Good shared items:

```markdown
## VERIFIED — <short title>
- Fact: <verified reusable fact>
- Evidence: <test/benchmark/code reference>
- Source: <worktree id> @ <HEAD>
```

Also share a failed approach when repeating it elsewhere would waste meaningful work:

```markdown
## FAILED — <approach>
- Result: <what happened>
- Evidence: <test/benchmark>
- Source: <worktree id> @ <HEAD>
```

Prefer appending one compact block to `shared.md`. Avoid rewriting the entire shared file, especially when multiple AI CLIs may be running concurrently.

## Never share these

Do not store in shared context:

- chain-of-thought,
- full transcripts,
- speculative diagnoses,
- ideas that have not been tested,
- temporary scratch notes,
- large code excerpts that can be read from Git,
- information already obvious from the current repository,
- secrets, credentials, tokens, or private user data.

## Conflict and staleness rules

If stored context conflicts with current code, tests, or command output:

```text
current evidence > stored context
```

If two worktrees have conflicting claims, keep them unresolved until current evidence distinguishes them. Do not let the first entry in `shared.md` become truth by default.

If a shared entry was produced against an old HEAD and the relevant code has changed, revalidate it before using it as a decision premise.

## Switching worktrees

Before leaving the current worktree:

1. checkpoint its local state;
2. switch worktrees normally;
3. resolve the new worktree's local checkpoint;
4. restore only what is needed to continue there.

Do not merge local contexts between worktrees.

## Completion

When a worktree task is fully complete:

- mark its local checkpoint `STATUS: Complete`;
- record the final HEAD;
- keep only reusable verified results in shared context;
- do not automatically delete the checkpoint, because a later session may need to understand why the worktree exists.

The user may delete runtime context at any time with:

```bash
rm -rf "$(git rev-parse --path-format=absolute --git-common-dir)/worktree-context"
```
