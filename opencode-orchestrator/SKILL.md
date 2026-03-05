---
name: opencode-orchestrator
description: Orchestrate multiple OpenCode instances in parallel to tackle large or complex coding tasks. Claude Code acts as the manager — breaking down the work, delegating to parallel OpenCode workers, and auditing the result. Use this skill whenever the user has a coding task that spans multiple files, features, or components that could benefit from parallel execution (e.g., "implement auth and payments", "add tests for all modules", "refactor the whole backend", "build frontend and backend simultaneously"). Also use it when the user explicitly asks to use OpenCode, run multiple OpenCode instances, or leverage OpenCode for any coding work. Even for single-file tasks, prefer dispatching to OpenCode over coding directly.
metadata:
  requires:
    bins:
      - opencode
---

# OpenCode Orchestrator

You are the **manager**. OpenCode workers do the actual coding — your job is to decompose, delegate, monitor, and audit.

## Worker Invocation

Each OpenCode worker is a Task subagent that runs:

```bash
opencode run "<instructions>" --agent build --dir /absolute/path/to/project
```

- Always use `--dir` with an absolute path — do NOT use `cd` (permissions fail outside project dir)
- Omit `--model` to use the user's default configured model
- Use `--agent plan` only for the initial planning phase, never for implementation
- Use `--format json` if you need to capture the session ID for follow-up work:
  ```bash
  opencode run "<instructions>" --agent build --dir /path --format json 2>&1 | grep -m1 sessionID | python3 -c "import sys,json; print(json.loads(sys.stdin.read())['sessionID'])"
  ```
- Continue a prior session with `--session <sessionID>` to preserve context

## Workflow

### Phase 1: Understand

Before dispatching any worker, explore the codebase yourself:
- Check the directory structure, key files, and entry points
- Understand existing conventions (naming, patterns, frameworks)
- Identify what already exists vs. what needs to be built

### Phase 2: Decompose

Break the task into **independent workstreams** — subtasks where workers won't conflict.

Good splits:
- By module boundary (auth, payments, notifications)
- By layer (API routes, service layer, DB models)
- By file ownership (each worker gets exclusive files)
- Frontend vs. backend (always safe to parallelize)
- Implementation vs. tests (different files)

Bad splits:
- Two workers touching the same file
- Worker B depends on Worker A's output (make it sequential instead)

For each workstream, define:
- **Scope**: exactly which files/directories the worker owns
- **Instructions**: a self-contained prompt — the worker can't ask you questions
- **Interface contracts**: if Worker A exports something Worker B imports, specify the signature explicitly in both prompts

### Phase 3: Dispatch

Spawn all independent workers **simultaneously** as Task subagents. Each subagent's task:

```
Run this opencode command and report the final output:

opencode run "<your specific instructions here>" --agent build --dir /absolute/path/to/project

Instructions should include:
- Exactly what to build
- Which files to create/modify
- Any interface contracts with other workers
- What "done" looks like (e.g., "all tests pass", "returns UserDTO")
```

For dependent workstreams (B depends on A), run A first, audit it, then dispatch B with A's actual output as context.

### Phase 4: Audit

Once all workers finish, you must personally verify before declaring done:

1. **Read the changed files** — understand what was built
2. **Check integration points** — imports, exports, type signatures match across worker boundaries
3. **Run tests/build** if available (e.g., `npm test`, `cargo check`, `pytest`, `tsc --noEmit`)
4. **Fix small issues yourself** — type mismatches, missing imports, wrong function signatures
5. **Re-dispatch a targeted worker** for anything complex — with precise instructions about what's wrong

Only report completion to the user after you've verified the work is coherent and correct.

## Worker Instruction Template

Write worker prompts that are:
- **Self-contained**: include all context the worker needs (no follow-up questions possible)
- **Bounded**: specify exact files to touch
- **Concrete**: define interfaces, types, and contracts explicitly

Example of a good worker prompt:
```
Implement the UserService in src/services/user.ts.

It should export:
- createUser(data: { email: string; name: string }): Promise<User>
- getUserById(id: string): Promise<User | null>
- deleteUser(id: string): Promise<void>

Use Prisma (already configured in prisma/schema.prisma) for DB access.
The User type is imported from @prisma/client.
Do not modify any other files.
```

## Parallelism Reference

| Scenario | Strategy |
|----------|----------|
| Multiple independent features | Dispatch all workers simultaneously |
| Feature + its tests | Simultaneously (different files) |
| Backend + frontend | Always parallel |
| Sequential dependency | Run first worker → audit → then second worker with first's output as context |
| Large refactor | Split by directory, each worker owns its subtree |

## One-Worker Tasks

Even for a single task, prefer dispatching to OpenCode rather than coding it yourself — OpenCode has full tool access and can iterate without token pressure. Use a single worker when:
- The task doesn't have obvious parallel splits
- The task is small and self-contained

## Available Agents

| Agent | Use for |
|-------|---------|
| `build` | All implementation work (default for workers) |
| `plan` | Initial analysis and planning only — cannot write code |
| `explore` | Read-only codebase exploration (subagent, falls back to `build` in `opencode run`) |
