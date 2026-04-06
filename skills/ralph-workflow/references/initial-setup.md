---
name: initial-setup
description: General setup for new Ralph users. Covers GitHub, Git, repo cloning, and workspace setup. Read once when setting up a new machine or starting with Ralph for the first time.
---

# Ralph Initial Setup

General setup instructions for the Claude Code CLI harness. For CLI-specific details, see `claude-instructions.md` in the same directory.

---

## Prerequisites

### GitHub Account and `gh` CLI

1. Create a GitHub account at https://github.com if you don't have one.
2. Install the `gh` CLI: https://cli.github.com/manual/installation
3. Authenticate:

```bash
gh auth login
# Follow the prompts: choose GitHub.com, HTTPS, and browser-based auth
```

Verify:

```bash
gh auth status
# Should show: ✓ Logged in to github.com
```

### Git Configuration

Set your identity globally (required for commits):

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

### Claude Code CLI

Install and authenticate — see `claude-instructions.md`.

---

## Verify Setup

Run the following to confirm all prerequisites are met:

```bash
# Git identity
git config user.name   && echo "✓ git user.name"
git config user.email  && echo "✓ git user.email"

# GitHub CLI
gh auth status 2>&1 | grep -q "Logged in" && echo "✓ gh auth OK" || echo "✗ gh auth: run gh auth login"

# Claude Code CLI
claude --version && echo "✓ claude CLI OK" || echo "✗ claude CLI missing: npm install -g @anthropic-ai/claude-code"

# ripgrep (used in success criteria)
rg --version | head -1 && echo "✓ rg OK" || echo "✗ rg missing: brew install ripgrep"
```

---

## Workspace Context Suggestions

When a user's initial prompt references external context, the orchestrator agent should check for each of the following and suggest adding it to the workspace before proceeding:

| Context type | Detection | Suggested action |
|---|---|---|
| **Repo mentioned** | Prompt includes a GitHub URL or repo name | "Want me to help clone `<repo>` and add it to the workspace?" |
| **Documentation URL** | Prompt includes a URL to docs, wiki, or design spec | "Want me to fetch and save that documentation to `workdocs/` so it survives session boundaries?" |
| **Confluence page** | Prompt includes a Confluence URL | Check if a Confluence skill is available. If not: "I can retrieve that page if you add a Confluence skill." |
| **Jira / issue tracker** | Prompt includes a Jira URL or ticket number | "Want me to fetch the Jira ticket details and save them to `workdocs/`?" |
| **PDF or local file** | User says "see attached" or references a file path | "Please drop that file in `workdocs/` directly and I'll include it as context." |
| **Design mockup / Figma** | Prompt includes a Figma URL | "Want me to save a summary of that design spec to `workdocs/`?" |

---

## External Skills

Optional skills that extend Ralph's capabilities when available in the workspace.

| Skill | When useful | Relevant steps |
|-------|-------------|----------------|
| Security review | Assessing security implications | A11, B7, C3 |
| Confluence / internal docs | Searching internal documentation | A12 |
| Browser automation (Chrome MCP) | UI testing, simulated UAT | B7, C3 |
| Code search / exploration | Navigating unfamiliar codebases | A12, B5 |

If a skill is available and applicable, agents should use it proactively.
