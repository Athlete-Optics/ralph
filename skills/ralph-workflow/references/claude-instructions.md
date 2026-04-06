---
name: claude-instructions
description: Claude Code CLI-specific guidance for running Ralph workflows. Use when running Ralph with the Claude Code CLI (claude -p).
---

# Ralph for Claude Code CLI

Claude Code CLI-specific guidance for the Ralph workflow.

---

## Setup (New Users)

Complete these steps once before running any Ralph workflow with Claude Code CLI.

### 1. Install Claude Code CLI

Install via npm (requires Node.js 18+):

```bash
npm install -g @anthropic-ai/claude-code
```

Verify:

```bash
claude --version
```

### 2. Authenticate

```bash
claude
```

On first run, Claude Code will prompt you to authenticate. Follow the browser-based auth flow. Required once per machine.

Alternatively, set the `ANTHROPIC_API_KEY` environment variable:

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
```

### 3. Verify

```bash
claude -p "hello" --model claude-sonnet-4-6
```

This should print a short response to stdout. If it fails, check your authentication.

---

## Authentication

Before starting any Ralph workflow:

```bash
claude --version  # verify installation
claude -p "hello" --model claude-sonnet-4-6  # verify authentication
```

Without valid auth, all spawned agent sessions will fail.

---

## CLI Flags

All `claude -p` invocations for non-interactive (headless) Ralph sessions use:

```bash
claude -p "<prompt>" \
  --model <model-id> \
  --dangerously-skip-permissions
```

| Flag | Purpose |
|------|---------|
| `-p "<prompt>"` | Non-interactive mode: provide prompt, print response, exit |
| `--model <id>` | Model to use (e.g. `claude-opus-4-6`, `claude-sonnet-4-6`) |
| `--dangerously-skip-permissions` | Skip all permission prompts; required for headless/unattended execution |

**Multi-turn (Phase A design):**

```bash
# Turn 1 — start new session
claude -p "<initial prompt>" --model <designer_model> --dangerously-skip-permissions

# Turn 2+ — continue most recent session
claude --continue -p "<follow-up>" --model <designer_model> --dangerously-skip-permissions

# Resume a specific session by ID
claude --resume "<session-id>" -p "<follow-up>" --model <designer_model> --dangerously-skip-permissions
```

`--continue` resumes the most recent Claude Code session in the current directory, preserving full conversation history.

**Important:** `--dangerously-skip-permissions` is required for non-interactive execution. Without it, agents will block waiting for tool permission confirmations.

---

## Agent Spawning

**Use the Bash tool to spawn agents via `claude -p`.**

The Claude Code Agent tool (spawning subagents) is suitable for short, bounded tasks. For Ralph's Phase B/C loop, spawn independent `claude -p` processes via Bash instead — each session has its own context, matching the Ralph loop's design.

```bash
claude -p "$(cat workdocs/NEXT_PROMPT<X>.md)" \
  --model <model-id> \
  --dangerously-skip-permissions
```

Run from the workspace root. Spawned sessions are independent processes with fresh context — not subagents of the parent session.

---

## Monitoring

**Real-time monitoring via transcripts:**

Claude Code session transcripts are written in real-time to:

```
~/.claude/projects/<project-hash>/<session-id>.jsonl
```

The `<project-hash>` is derived from the absolute path of the working directory. To find it:

```bash
ls ~/.claude/projects/
```

Each `.jsonl` file contains one JSON object per line representing tool calls, messages, and results. These are updated live as the agent works.

**Recommended monitoring cadence:**

1. Every 3–5 minutes: check `git log --oneline -5` and `git status` for file changes
2. If no changes for 5+ minutes: check the latest `.jsonl` file in `~/.claude/projects/<project-hash>/`
3. If no new tool calls in the transcript for 3+ minutes: agent may be stuck — investigate

```bash
# Find the latest transcript for the current project
ls -t ~/.claude/projects/$(echo -n "$(pwd)" | shasum -a 256 | cut -d' ' -f1)/*.jsonl | head -1
# Then: tail -n 20 <path-to-latest.jsonl>
```

---

## Output

`claude -p` writes the final response to stdout when the session ends. For monitoring purposes, the `.jsonl` transcript is the most reliable real-time signal — it is written continuously regardless of output flags.

---

## Session IDs

Each `claude -p` session generates a unique session ID visible in the transcript filename. To resume a specific session:

```bash
claude --resume "<session-id>" -p "<prompt>" --model <model-id> --dangerously-skip-permissions
```

To list recent sessions:

```bash
ls -lt ~/.claude/projects/<project-hash>/
```
