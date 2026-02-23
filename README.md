# /dispatch

**Stop juggling terminals.** One command dispatches work to background AI agents — Claude, GPT, Gemini — while you keep coding.

<p align="center">
  <img src="skills/dispatch/assets/architecture.svg" alt="dispatch architecture: you run /dispatch, it plans the task, fans out to parallel workers (Claude, GPT, Gemini), and reports results back" width="700" />
</p>

```
/dispatch "do a security review of this project"
```

The dispatcher plans the task, spawns background workers, and reports back. You never leave your session.

---

## Install

**User-level** (available in all your projects):

```bash
npx skills add bassimeledath/dispatch -g
```

**Project-level** (shared with your team via version control):

```bash
npx skills add bassimeledath/dispatch
```

## How it works

1. You run `/dispatch "task description"`
2. A checklist plan is created at `.dispatch/tasks/<id>/plan.md`
3. A background worker picks it up and checks off items as it goes
4. You get results when it's done — or ask for status anytime

Workers can use **any model** you have access to. Mix Claude for deep reasoning, GPT for broad tasks, Gemini for speed — all from one interface.

## Setup

On first run, `/dispatch` auto-detects your CLIs (`claude`, `agent`, `codex`), discovers available models, and generates `~/.dispatch/config.yaml`. No manual config needed.

## Configuration

Three sections in `~/.dispatch/config.yaml`:

**Backends** — CLI commands for each provider:
```yaml
backends:
  claude:
    command: >
      env -u CLAUDE_CODE_ENTRYPOINT -u CLAUDECODE
      claude -p --dangerously-skip-permissions
  cursor:
    command: >
      agent -p --force --workspace "$(pwd)"
  codex:
    command: >
      codex exec --full-auto -C "$(pwd)"
```

**Models** — one line each, mapped to a backend:
```yaml
models:
  opus:            { backend: claude }
  sonnet:          { backend: claude }
  gpt-5.3-codex:  { backend: codex }
  gemini-3.1-pro:  { backend: cursor }
```

**Aliases** — named shortcuts with optional role prompts:
```yaml
aliases:
  security-reviewer:
    model: opus
    prompt: >
      You are a security-focused reviewer. Prioritize OWASP Top 10.
```

## Worker IPC

Workers can ask clarification questions **without exiting**. When a worker hits a blocker, it surfaces the question to you. After you answer, the worker picks up where it left off with full context preserved.

## Prerequisites

At least one AI CLI backend:
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) (`claude`)
- [Cursor CLI](https://docs.cursor.com/) (`agent`)
- [Codex CLI](https://github.com/openai/codex) (`codex`)

## License

MIT
