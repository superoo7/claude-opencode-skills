---
name: opencode-agent
description: Delegate a single, self-contained coding task to OpenCode and iterate until tests pass. Use when the user asks to build a feature, fix a bug, or implement something that is one coherent unit of work. OpenCode does the coding; Claude verifies by running tests and drives follow-up sessions until the quality bar is met. Do NOT use for tasks that span multiple independent workstreams — use opencode-orchestrator instead.
metadata:
  requires:
    bins:
      - opencode
---

# OpenCode Agent

You are the **verifier**. OpenCode does the coding — your job is to craft a precise prompt, inspect the result, run the tests, and drive follow-up sessions until the work is correct.

## Worker Invocation

```bash
# Initial run — capture session ID for follow-up
opencode run "<task>" --agent build --dir /absolute/path --format json 2>&1 | grep -m1 sessionID | python3 -c "import sys,json; print(json.loads(sys.stdin.read())['sessionID'])"

# Continue the same session with follow-up instructions
opencode run -s <sessionID> "<follow-up instructions>" --agent build --dir /absolute/path

# Continue the last session (shorthand)
opencode run -c "<follow-up>" --agent build --dir /absolute/path
```

- Always use `--dir` with an absolute path — do NOT use `cd`
- Always use `--format json` on the initial run to capture the session ID
- Use `--agent build` for all implementation work
- Use `--agent plan` only if you need a read-only breakdown before writing the prompt

## Workflow

### Phase 1: Plan task

Before invoking OpenCode, explore the codebase yourself:
- Check the directory structure, key files, and test setup
- Understand existing conventions and what "done" looks like
- Identify which files will be touched

Write a self-contained prompt — include all context the worker needs:
- Exactly what to build and in which files
- Relevant type signatures, interfaces, or contracts
- Existing patterns to follow (e.g., "follow the pattern in `src/auth/login.ts`")
- Acceptance criterion (e.g., "all tests in `src/payments/payments.test.ts` should pass")

Run the initial session and capture the session ID:

```bash
SESSION=$(opencode run "<your task prompt>" --agent build --dir /abs/path --format json 2>&1 | grep -m1 sessionID | python3 -c "import sys,json; print(json.loads(sys.stdin.read())['sessionID'])")
echo "Session: $SESSION"
```

### Phase 2: Analyse result

Once OpenCode finishes, inspect the output yourself:

1. **Read the changed files** — understand what was built
2. **Run the test suite** — `npm test`, `pytest`, `cargo test`, `go test ./...`
3. **Run the type checker / linter** if available — `tsc --noEmit`, `cargo check`, `mypy`
4. **Identify specific failures** — note exact error messages, failing test names, line numbers

If everything passes → declare done. Otherwise proceed to Phase 3.

### Phase 3: Improve

Continue the session with targeted instructions about exactly what is wrong:

```bash
opencode run -s $SESSION "<precise description of the failure and what to fix>" --agent build --dir /abs/path
```

Then return to Phase 2. Repeat until:
- All existing tests pass
- No new test failures introduced (no regressions)
- Type checks and lints are clean (if applicable)

**Stop after 2 failed follow-up sessions** — escalate to the user with a summary of what is stuck and why.

## Quality Gate

| Check | Command |
|-------|---------|
| Tests pass | `npm test` / `pytest` / `cargo test` / `go test ./...` |
| Types clean | `tsc --noEmit` / `cargo check` / `mypy` |
| No regressions | Run the full suite, not just new tests |

Only report completion after you have personally run the tests and confirmed they pass.

## When to use this skill vs. opencode-orchestrator

| Situation | Use |
|-----------|-----|
| Single feature, single workstream | `opencode-agent` |
| Fix a bug or failing test | `opencode-agent` |
| Implement one function or module | `opencode-agent` |
| Multiple independent features at once | `opencode-orchestrator` |
| Task explicitly spans frontend + backend | `opencode-orchestrator` |
| "Build everything" / whole-project tasks | `opencode-orchestrator` |

## Available Agents

| Agent | Use for |
|-------|---------|
| `build` | All implementation work (default) |
| `plan` | Initial analysis and planning only — cannot write code |
| `explore` | Read-only codebase exploration |
