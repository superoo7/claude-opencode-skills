# claude-opencode-skills

Claude Code skills for [OpenCode](https://opencode.ai). Use Claude Pro as the orchestrator while your local self-hosted model does the actual code generation.

## The story

I have a Claude Pro subscription. I also built a home server: DGX Spark with 128GB unified memory, running Qwen3.5-122B-A10B via llama.cpp. The machine sits there most of the day doing nothing useful.

The problem is that Claude Pro burns through context fast when doing repetitive code generation. Write a file, write another file, write tests, write more files. Claude is smart enough to do all of this, but it feels like a waste, and you hit limits. The obvious fix is Claude Max, but I already paid for the GPU. I don't want to pay again.

So I started thinking about whether I could use Claude Pro for the reasoning part (planning, decomposing, reviewing) and push the actual coding work to my local model. The local model is free to run, fast enough, and handles code generation well. Claude stays focused on the parts it is actually better at.

OpenCode is what makes this work. It is an open-source AI coding agent that can run headless, point at any model, and take instructions from the command line. Claude Code can spin up multiple OpenCode workers in parallel, each talking to the local model, each working on a different part of the codebase. When they finish, Claude reviews the output.

That is the setup this repo is built around.

## What is this?

[Claude Code](https://claude.ai/claude-code) supports skills, which are instruction files that tell Claude how to behave in specific situations. This repo contains skills that connect Claude Code with [OpenCode](https://opencode.ai).

The split is simple: Claude Code (your Pro subscription) is the manager. OpenCode (pointing at your local model) is the worker. Claude thinks, OpenCode codes.

## Skills

### `opencode-orchestrator`

Orchestrates multiple OpenCode instances in parallel for large or complex coding tasks.

**What it does:**
- Breaks your task into independent workstreams
- Spawns parallel OpenCode workers simultaneously, one per workstream
- Audits the combined output for integration issues before reporting done

**When it triggers:**
- Tasks spanning multiple files, features, or components
- Anything where you want to do two things at once
- Any prompt that mentions OpenCode explicitly

**Example prompts:**
- *"Implement auth and payments in parallel"*
- *"Add tests for all API endpoints"*
- *"Refactor the backend and update the frontend to match"*
- *"Use opencode to build this feature"*

### `opencode-agent`

Delegates a single coding task to OpenCode and iterates until tests pass.

**What it does:**
- Plans a self-contained prompt for OpenCode based on your codebase
- Dispatches a single OpenCode worker and captures the session ID
- Runs your test suite to verify the result
- Continues the session with targeted follow-ups until all tests pass

**When it triggers:**
- Building a single feature or module
- Fixing a bug or failing test
- Implementing one function with existing tests to satisfy

**Example prompts:**
- *"Add a /login endpoint — the tests in test/auth.test.js should pass"*
- *"Fix the failing test in sum.test.js"*
- *"Implement formatCurrency() in src/utils/currency.ts, tests already exist"*

## Installation

### Prerequisites

- [Claude Code](https://claude.ai/claude-code) installed
- [OpenCode](https://opencode.ai) installed (`opencode --version` should work)
- At least one LLM provider configured in OpenCode (`opencode auth list`)

> **Self-hosting tip:** OpenCode works with any OpenAI-compatible API. Point it at your local inference server in `~/.config/opencode/opencode.json` and all workers will use your local model with no API costs for code generation.

### Recommended: clone directly into Claude Code skills directory

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/superoo7/claude-opencode-skills.git ~/.claude/skills/claude-opencode-skills
```

Then symlink the skill(s) you want:

```bash
ln -s ~/.claude/skills/claude-opencode-skills/opencode-orchestrator ~/.claude/skills/opencode-orchestrator
ln -s ~/.claude/skills/claude-opencode-skills/opencode-agent ~/.claude/skills/opencode-agent
```

### Manual install (single skill file only)

If you just want the `SKILL.md` files without cloning the whole repo:

```bash
mkdir -p ~/.claude/skills/opencode-orchestrator
curl -o ~/.claude/skills/opencode-orchestrator/SKILL.md \
  https://raw.githubusercontent.com/superoo7/claude-opencode-skills/main/opencode-orchestrator/SKILL.md

mkdir -p ~/.claude/skills/opencode-agent
curl -o ~/.claude/skills/opencode-agent/SKILL.md \
  https://raw.githubusercontent.com/superoo7/claude-opencode-skills/main/opencode-agent/SKILL.md
```

The skill is available automatically in your next Claude Code session.

## Usage

Just describe your task naturally in Claude Code. The skill triggers on its own:

```
Implement the auth module and the payments module at the same time
```

```
Use opencode to add tests for all my API endpoints
```

```
Refactor the backend and update the frontend to match, run both in parallel
```

Claude Code reads the skill, breaks down the work, dispatches OpenCode workers, and audits the results.

## How it works

Claude Code loads skills from `~/.claude/skills/`. Each skill is a folder with a `SKILL.md` file that Claude reads when the skill is relevant.

The orchestrator dispatches workers using `opencode run --agent build --dir /path/to/project`. The `--dir` flag is required in non-interactive mode since changing directories causes permission issues.

## Contributing

PRs welcome. If you have built a useful OpenCode skill for Claude Code, open a PR to add it here.

## License

MIT
