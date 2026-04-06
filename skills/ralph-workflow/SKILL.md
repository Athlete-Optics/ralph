---
name: ralph-workflow
description: Agentic coding workflow for unsupervised software development (design → code in a loop → PR), orchestrated by an orchestrator agent across multiple fresh sessions
---

# Ralph Workflow

Ralph is an agentic coding workflow developed to make unsupervised software development possible. It has two main principles:
1. Avoid context rot through deliberate context management
2. Prevent error propagation by enforcing agent self-verification and adding "back pressure"

This was inspired by the Ralph Wiggum Loop, invented in 2025 by Geoffrey Huntley.

### Core Steps

An orchestrator agent (🕹️) manages the end-to-end process. The core mechanism: **short, focused agent sessions** with **workdocs as external memory** and a **handoff file** (`NEXT_PROMPT<X>.md`) between sessions. Each session starts fresh — no reliance on context compaction or summarization.

Conceptual list of steps (for actual steps, see [Workflow Steps](#workflow-steps) section):

1. 👑 User describes what to build
2. 🕹️ Orchestrator agent interviews user to get additional context
3. 🎨 Designer agent explores, writes specification and plan
4. 👑 User approves
5. 🔨 Coder agent chooses a task, implements it, self-verifies, exits
6. 🕹️ Repeat previous step until all tasks done (the Ralph loop)
7. 🔍 Verifier agent performs holistic verification, opens PR
8. 👑 User reviews and merges

---

## Variables

These variables control certain rules and workflow steps, and are referenced throughout the rest of the skill. To use, replace the variable with its default value below. If the 👑 User explicitly asks to change a value, then prefer it over the default value.

| Variable | Default | Description |
|----------|---------|-------------|
| `designer_model` | `claude-opus-4-6` | Model for the designer CLI agent |
| `coder_model` | `claude-sonnet-4-6` | Model for each coder CLI agent |
| `verifier_model` | `claude-opus-4-6` | Model for the verifier CLI agent |
| `target_branch` | (inferred) | Branch the PR will target; inferred from the user's current branch at workflow start, overridable |
| `max_workdocs_size` | 30000 | Max collective workdocs tokens |
| `max_fix_iterations` | 10 | Max fix/re-verify cycles per step before declaring stuck |
| `max_stuck_retries` | 3 | Max retries when a CLI agent makes no progress |
| `ai_reviewer_model` | `claude-opus-4-6` | Model used for the AI reviewer CLI session spawned in C7 |
| `ai_reviewer_max_iterations` | 5 | Max AI reviewer feedback cycles |

---

## Roles

Each workflow step is tagged with a role emoji indicating who acts.

| Emoji | Role | Description |
|-------|------|-------------|
| 👑 | User | The human who owns the project; approves designs, reviews PRs, and merges. |
| 🕹️ | Orchestrator | The agent that runs the end-to-end workflow; interviews user, spawns designer, coder, and verifier agents; owns Phase B loop control. Detailed guidelines are in this skill's Orchestrator Agent section. |
| 🎨 | Designer | The agent that runs Phase A design; interviews orchestrator, explores, writes SPEC/PLAN/SETUP. Role details are in `agents/designer.md`. |
| 🔨 | Coder | The agent that implements one task per session in Phase B; self-verifies and exits. Role details are in `agents/coder.md`. |
| 🔍 | Verifier | The agent that runs Phase C holistic verification and ships the PR. Role details are in `agents/verifier.md`. |

---

## 🕹️ Orchestrator Agent

The orchestrator agent manages the Ralph workflow end-to-end. It launches designer, coder, and verifier agents as independent processes, plays the reviewer/interviewer role in Phase A, manages the Phase B implementation loop, and runs Phase C verification.

**Interview behavior:** The human interview is **mandatory** by default. The orchestrator agent conducts a full design interview with the user before Phase A proceeds. **Exception:** if the initial prompt is very thorough AND **explicitly requests skipping the interview**, the orchestrator agent skips the human interview and acts as the interviewee for the designer agent instead.

**CLI agent commands (Claude Code CLI — see [claude-instructions](references/claude-instructions.md) for details):**

```bash
# Single-turn (Phase B/C iterations):
claude -p "$(cat workdocs/NEXT_PROMPT<X>.md)" --model <coder_model> --dangerously-skip-permissions

# Multi-turn conversation (Phase A design):
claude -p "<initial prompt>" --model <designer_model> --dangerously-skip-permissions    # turn 1
claude --continue -p "<follow-up>" --model <designer_model> --dangerously-skip-permissions  # turn 2+
```

Run from the workspace root. `--continue` resumes the most recent session, preserving full conversation history. Use `--resume="<session-id>"` to target a specific session if multiple exist.

**Note on output:** `claude -p` writes the final response to stdout when the session ends. For live progress, read `.jsonl` transcript files at `~/.claude/projects/<project-hash>/` — they are written in real time.

**Phase B/C loop:** After each CLI agent exits, the orchestrator agent:
1. Verifies clean exit (exit code 0)
2. Reads updated `PLAN.md` and workdocs to assess progress
3. If no progress or agent reported stuck → adjusts workdocs, retries (up to `max_stuck_retries`)
4. If tasks remain → back to B1 (spawn another coder); if all done → proceeds to Phase C
5. Alerts the human on critical failures or when the PR is ready for review

**Monitoring cadence:** Every 3–5 minutes, check `git log --oneline -5` and recent file changes. If commits have paused for 5+ minutes, read the latest `.jsonl` transcript — **transcript activity is the primary signal**; agents may batch file edits before committing, so absent commits alone do not indicate no progress. If no new tool calls appear in the transcript for 3+ minutes, the agent may be stuck — investigate.

**Important:**
- `claude -p` with `--dangerously-skip-permissions` allows unrestricted tool execution; launched agents are independent processes with fresh context — not subagents.
- If the orchestrator agent's own context grows large, automatic compaction is allowed.
- If the orchestrator agent thinks it is stuck, automatic compaction happened too much, or thinks that its context is poisoned, writes a `NEXT_PROMPT<X>.md` (noting where to continue orchestrating), alerts user, and exits. The human restarts it.
- **Only the human merges PRs.**

---

## Workdocs

Workdocs are markdown files that act as **external memory**. They are the mechanism by which context survives between sessions. Each new session re-reads them from scratch, getting a clean and accurate view of the project state.

**Rules:**
- Keep within `max_workdocs_size` collectively. Be information-dense — omit commonplace details, standard practices, and obvious tool usage.
- Do not create extra files beyond those listed.
- The 👑 user may edit any workdoc at any time.
- **Never edit, delete, or move files under `workdocs_archive/`.** The archive is append-only — only Phase C archival writes to it.
- **User-provided files:** If the user attaches files to the conversation (PDFs, images, documents), save them to `workdocs/` so they survive session boundaries. These files are archived alongside the rest of workdocs at Phase C end.

### USER_PROMPT.md
Write-once for fresh sessions. The original user request, preserved verbatim for reference. **Exception for existing sessions:** when ralph is triggered from a session that is already in progress, the user's prompt may reference prior conversation context that is not captured in the prompt text. In that case, the orchestrator agent may edit `USER_PROMPT.md` to add the missing context before proceeding.

### SPEC.md
Full software specification: requirements, architecture, testing strategy, and programmatically verifiable success criteria. Main source of truth for what is being built. See [specification-guide](references/specification-guide.md).

### PLAN.md
Granular implementation plan: tasks with status markers (⬜ / 🔧 / ✅), dependencies, and per-task success criteria. Each task should be small enough to implement and verify in a single focused session.

### SETUP.md
Project-specific dev environment setup: tools, services, environment variables, test fixtures, and verification scripts required for this particular project. General harness and workspace setup belongs in [initial-setup](references/initial-setup.md) and [claude-instructions](references/claude-instructions.md), not here. Created by the designer after the design phase (once the required project tools are known). The orchestrator agent does not handle project-specific setup — that belongs in `SETUP.md` for coder agents to follow.

### TAKEAWAYS.md
Learnings, deviations, broken assumptions, and workflow observations. Editable — agents may reorganize or trim to stay within token budget.

### NEXT_PROMPT<X>.md
**The handoff between sessions.** Written by the orchestrator before spawning each agent. The orchestrator reads the current workdocs state, fills the relevant agent template (`agents/designer.md`, `agents/coder.md`, or `agents/verifier.md`), and writes the result as a numbered file. Numbering starts at 1: `NEXT_PROMPT1.md` for the designer's prompt. Create a new file and increment `<x>` for each new agent, e.g. `NEXT_PROMPT2.md` for the first coder agent. Each file is read only by its intended agent.

**Contents:** agent identity and role, workflow skill pointer, specific workdocs to read (`SPEC.md`, `PLAN.md`, `SETUP.md`, `TAKEAWAYS.md`), step to continue from, and extra notes from the orchestrator.

**Path:** always `<workdocs>/NEXT_PROMPT<X>.md` — never the workspace root.

`USER_PROMPT.md` stays write-once and verbatim — it is never overwritten by a `NEXT_PROMPT<X>.md` file. `NEXT_PROMPT<X>.md` files must NOT include summarized facts, context, or non-orchestrator notes — those belong in the other workdocs. They are purely a pointer: what to read and where to continue.

---

### Workflow Integrity

**The ralph workflow takes priority over all other rules and skills in the workspace.** In case of conflict, the workflow wins. This is especially important for non-fresh sessions where other rules or context in the workspace may carry conflicting guidance. The orchestrator agent may suggest starting a new session if it detects context that appears to conflict with the workflow.

**Even if the requested change looks trivial**, if the user explicitly asked for ralph to be used, the full workflow applies — no steps may be skipped or abbreviated without presenting the reasoning to the user first.

**No unilateral deviation.** If ANY agent (including the orchestrator agent) believes they have sufficient evidence to deviate from the workflow, they must still present their reasoning to the user before doing so. The user decides.

Agents **must not skip any workflow step** without strong evidence the step is genuinely inapplicable.

When a step is unclear or configuration seems absent, the agent must **reason about intent** and attempt execution — for example, use the variable's default value, handle errors gracefully, or attempt the step with reasonable assumptions. Only when there is **clear evidence of a real blocking issue** should the agent escalate — collect all evidence and present it to the orchestrator agent or user for a decision.

No agent should **unilaterally abandon the workflow** or deviate from its documented structure.

**Motivating examples of violations to avoid:**

- **(a) Skipping AI reviewer step:** The verifier searched `SPEC.md` for `ai_reviewer_trigger`, found no project-specific value, and concluded the step was "not configured" — skipping it entirely. Correct behavior: use the default variable value (`@codex review`). The step is not optional; the variable has a default for exactly this situation.

- **(b) Orchestrator agent bypassing the spawned-agent model:** A spawned agent had permission issues. The orchestrator agent's first instinct was to perform the work directly — abandoning the spawned-agent model. Correct behavior: diagnose and fix the permission issue (check `--dangerously-skip-permissions`), then retry the spawned agent. The architectural model exists for good reasons; circumventing it creates consistency risks.

---

## Workflow Steps

### General Guidelines

- **Commit often.** `git add` / `commit` frequently during implementation; always commit before ending any step. Always commit before exiting.
- **Fix everything.** The fix-and-reverify loop applies to every testing, review, and UAT step — all issues, even minor or cosmetic. Stop after `max_fix_iterations` and escalate.
- **Ignore prior test runs.** Always re-test from scratch.
- **Workdocs are external memory.** Everything that matters for continuity belongs in workdocs. If it's not in workdocs, it won't survive the session boundary.
- **No clarifying questions after approval.** During Phase A, ask freely during the appropriate steps. After approval (Phase B/C), work with what you have — escalate only if truly stuck.
- **Only the user merges PRs.**

### Phase A — Preparation

| Step | Role | What to do |
|------|------|------------|
| **A1** | 👑 | Write the initial build prompt. Include known context even if imperfect. |
| **A2** | 🕹️👑 | **Session freshness check:** check whether the session context is fresh and clean. Signals of a non-fresh session: unrelated content in context, references to prior tasks or conversations not related to this ralph run, or conflicting rules/instructions. If the session does not appear fresh, suggest creating a new session, explain why, and ask the user to confirm before proceeding. Continuing with a polluted context is allowed if the user confirms, but the risk is theirs. |
| **A3** | 🕹️ | Choose a workdocs location. Create the workdocs directory and write `USER_PROMPT.md` (verbatim from the prompt). If starting from an existing session where the prompt references prior conversation context, edit `USER_PROMPT.md` to capture that context before proceeding. |
| **A4** | 🕹️👑 | **Setup check:** verify that general setup (per [initial-setup](references/initial-setup.md)) and Claude Code harness setup (per [claude-instructions](references/claude-instructions.md)) are correct. If issues are found, explain them to the user and ask whether they want help fixing before proceeding. |
| **A5** | 🕹️👑 | **Workspace context and skills check:** review the prompt for any mentioned repos, documentation URLs, Confluence pages, Jira tickets, or other context sources — consult the workspace context checklist in [initial-setup](references/initial-setup.md) and suggest adding them to the workspace; ask if the user wants help. Check for any user-provided files in context (PDFs, images, documents) and save them to `workdocs/`. Check available external skills in the workspace against the recommended list in [initial-setup](references/initial-setup.md); ask the user if any missing skills should be added. |
| **A6** | 🕹️👑 | **Confirm understanding:** summarize your understanding of the user's requirements and ask for confirmation before proceeding to the designer's discovery interview. This prevents misunderstandings that compound through the entire run. |
| **A7** | 🕹️👑 | 🕹️ asks 👑 whether 👑 wants to review `SPEC.md` and `PLAN.md` before the loop starts, and say that it is recommended to do so, especially for complex projects. If 👑 declines, mention that the workflow will continue to the end unsupervised, and that they'll be alerted once a PR is ready. |
| **A8** | 🕹️👑 | 🕹️ infers `target_branch` from 👑's current branch; checks its status and if there are uncommited changes; checks if it exists in remote and is current (same commit hash, didn't diverge). If any issues are found, 🕹️ alerts 👑 and explains that, since a PR will be created, a target branch in remote that is up-to-date must be chosen, but the current branch has issues (lists which); 🕹️ asks for another branch or offers to help prepare the current branch (e.g. push to remote if not there). Finally, 🕹️ defines `target_branch` and creates feature branch `ralph/<8-char-uuid>` off it. |
| **A9** | 🕹️ | Write `NEXT_PROMPT1.md` based on `agents/designer.md` for the 🎨 designer agent, including the workdocs path, feature branch, and any extra notes; launch 🎨 via `claude -p "$(cat workdocs/NEXT_PROMPT1.md)" --model <designer_model> --dangerously-skip-permissions`. |
| **A10** | 🎨 | Confirm you are on the correct feature branch (`ralph/<8-char-uuid>`). |
| **A11** | 🎨🕹️ | Discovery: 🎨 and 🕹️ interview each other until problem and requirements are clear. Don't stop to ask 👑 anything; if a gap is identified, try to infer from 👑's intent and choose the best option. Consider security implications. |
| **A12** | 🎨 | Explore existing artifacts (repos, docs, web); if new relevant information sources are found, follow and read them. Author `SPEC.md`, `PLAN.md`, and seed `TAKEAWAYS.md`. Every success criterion must be programmatically verifiable; see [specification-guide](references/specification-guide.md). Iterate as needed; commit frequently. |
| **A13** | 🕹️🎨 | 🕹️ reviews `SPEC.md` and `PLAN.md`; iterate with 🎨 until satisfied, via multi-turn `--continue` calls. |
| **A14** | 🎨 | Verify project-specific tooling works. Write `SETUP.md` covering only project-specific setup — tools, services, env vars, test fixtures. This step follows the design phase because only then are the required project tools known. Do not include general harness or workspace setup in `SETUP.md`. |
| **A15** | 🎨 | Self-review all workdocs for consistency and minimal redundancy. |
| **A16** | 🎨👑🕹️ | **Optional**: If in A7 👑 asked to review `SPEC.md` and `PLAN.md`, pause and alert user, asking them to review. 🕹️ facilitates message passing between 👑 and 🎨. Iterate until satisfied. Before proceeding, mention to 👑 that the workflow will continue to the end unsupervised, and that they'll be alerted once a PR is ready. |
| **A17** | 🕹️ | Proceed directly to the Phase B loop without exiting or alerting 👑. |

### Phase B — Implementation (the Ralph Loop)

Phase B is **the Ralph loop**. Each iteration is a fresh agent session with clean context. The 🕹️ orchestrator agent manages session lifecycle; the 🔨 coder agent implements tasks within each session.

**Context compaction is NOT a substitute for the loop.** It is lossy, unpredictable, and not under agent control. The Ralph loop works by exiting and restarting with fresh context. Workdocs are the external memory. `NEXT_PROMPT<X>.md` is the handoff. Never rely on compaction to continue working past your assigned task.

| Step | Role | What to do |
|------|------|------------|
| **B1** | 🕹️ | Fill `agents/coder.md` template with current context and any extra notes. Write as `workdocs/NEXT_PROMPT<X>.md` (incrementing X – don't overwrite existing files). Spawn coder via `claude -p "$(cat workdocs/NEXT_PROMPT<X>.md)" --model <coder_model> --dangerously-skip-permissions`. |
| **B2** | 🔨 | Read this skill and workdocs (`SPEC.md`, `PLAN.md`, `SETUP.md`, `TAKEAWAYS.md`). Confirm you are on the correct feature branch (`ralph/<8-char-uuid>`). Identify progress from `PLAN.md` (tasks ✅ vs ⬜). |
| **B3** | 🔨 | Choose **ONE** unclaimed task (⬜) with satisfied dependencies. Heuristics (non-ordered): higher downstream impact, higher uncertainty, higher number of dependent tasks, easier to complete. Implement exactly this one task and exit — do not pick up additional tasks. If you believe two tasks must be combined, escalate to the orchestrator; do not decide unilaterally. Mark 🔧 in `PLAN.md` and **commit immediately — before any implementation begins**. This gives the orchestrator real-time visibility into which task is in progress. |
| **B4** | 🔨 | Verify tooling for this task: run the relevant setup steps from `SETUP.md`, confirm tools are reachable and return expected output. Restart and re-verify if needed. |
| **B5** | 🔨 | Implement; add tests and update `TAKEAWAYS.md` as you go; commit frequently. Ensure all tests pass after each logical change. Follow the [testing-guide](references/testing-guide.md). |
| **B6** | 🔨 | Self-review all commits added for this task, like an external reviewer would. Check for logic errors, missing edge cases, incomplete tests, and style issues. Fix anything found; re-test after fixes. |
| **B7** | 🔨 | Simulated UAT (user acceptance testing): test every flow and edge case as an end user. Follow the [verification-guide](references/verification-guide.md). |
| **B8** | 🔨 | Review and tighten task success criteria in `PLAN.md`; strengthen with new knowledge, do NOT weaken. Then do a full `PLAN.md` review: scan the entire file for obvious inconsistencies introduced by your changes (stale task names, references to renamed files, outdated success criteria); fix only clear, unambiguous errors. Note any findings in `TAKEAWAYS.md`. |
| **B9** | 🔨 | Confirm all success criteria pass; re-test from scratch. Do NOT proceed without full confidence. |
| **B10** | 🔨 | Self-reflect; update `TAKEAWAYS.md` with learnings, deviations, and workflow observations. |
| **B11** | 🔨 | Update workdocs: mark task ✅ in `PLAN.md` and append a brief session summary to the task entry (e.g., "Session: completed cleanly" or "Session: required two fix iterations") — convey how the session went, not a detailed reflection (that goes in TAKEAWAYS.md). Update `SPEC.md`, `SETUP.md`, `TAKEAWAYS.md` as needed. Commit. |
| **B12** | 🔨 | **Exit**: commit all work, tell 🕹️ the task is complete, and exit. |
| **B13** | 🕹️ | When agent exits: read `PLAN.md`. If tasks remain → back to **B1**. If all tasks done → proceed to Phase C (**C1**). If agent reported being stuck → investigate, adjust workdocs, then **B1**. |

### Phase C — Verification & Ship

Phase C may span one or more sessions, using the same `NEXT_PROMPT<X>.md` handoff pattern as Phase B.

| Step | Role | What to do |
|------|------|------------|
| **C1** | 🕹️ | Fill `agents/verifier.md` template with current context and any extra notes. Write as `workdocs/NEXT_PROMPT<X>.md` (increment `<X>` – don't overwrite previous files). Spawn verifier via `claude -p "$(cat workdocs/NEXT_PROMPT<X>.md)" --model <verifier_model> --dangerously-skip-permissions`. |
| **C2** | 🔍 | Read this skill and workdocs (`SPEC.md`, `PLAN.md`, `SETUP.md`, `TAKEAWAYS.md`). Review all code changes (full diff from base to current). |
| **C3** | 🔍 | Holistic verification: PR-style commit review; simulated UAT across all flows and edge cases; security review. Follow the [testing-guide](references/testing-guide.md) and [verification-guide](references/verification-guide.md). Pre-existing security issues: document in PR description; fixing optional. |
| **C4** | 🔍 | For each issue found: assess severity. Fix; re-verify after each fix. Repeat C2–C3 until clean or `max_fix_iterations`. If context is approaching limit → commit, tell 🕹️ to restart from **C1**; orchestrator writes a new `NEXT_PROMPT<X>.md` and spawns a fresh verifier. |
| **C5** | 🔍 | Check if `target_branch` has diverged since workflow start. If unchanged: proceed. If diverged: merge target into feature branch, resolve conflicts, re-run verification (back to C2). If conflicts are too complex: escalate to 🕹️/👑. |
| **C6** | 🔍 | Push feature branch to remote. Open draft PR from feature branch → `target_branch`. Write a full PR description (summary, changes, test plan, security assessment) according to [pr-description-guide](references/pr-description-guide.md). |
| **C7** | 🔍 | **AI reviewer loop (Claude Code CLI):** Spawn a fresh reviewer session: `claude -p "$(cat workdocs/NEXT_PROMPT_REVIEWER<X>.md)" --model <ai_reviewer_model> --dangerously-skip-permissions`. Write `NEXT_PROMPT_REVIEWER<X>.md` with: full PR diff (from `git diff <target_branch>...HEAD`), the task list from `PLAN.md`, the success criteria from `SPEC.md`, and instructions to (1) act as a thorough code reviewer, (2) post review comments directly on the PR via `gh pr review <number> --comment -b "..."` and `gh api repos/{owner}/{repo}/pulls/{number}/comments`, and (3) exit when done. After the reviewer exits: read all new PR comments (`gh api repos/{owner}/{repo}/pulls/{number}/comments` and `gh api repos/{owner}/{repo}/issues/{number}/comments`); for each comment, decide whether to fix the issue (use your judgment), fix those warranted, reply explaining the fix or reason for skipping, and resolve the thread. Re-spawn the reviewer after fixes and repeat. Stop when the reviewer reports no new issues, or after `ai_reviewer_max_iterations` (default: 5) iterations. **Always attempt this step; never skip it.** |
| **C8** | 🔍 | **Wrap up**: Update `TAKEAWAYS.md` and push. Update PR description. Add 👑 as reviewer on the draft PR. Alert 🕹️ that you're done and exit. |
| **C9** | 🕹️ | **Self-reflect**: Read `TAKEAWAYS.md`; read agent transcripts to study the executed workflow; analyze Phase C findings vs what Phase B should have caught; review workdocs for accuracy; review `USER_PROMPT.md` and note unaddressed items. If any issues, come up with general workflow improvement suggestions (must not be project-specific) that would have prevented or mitigated the issues. |
| **C10** | 🕹️ | Archive workdocs: move `workdocs/` to `workdocs_archive/<YYYYMMDD_HHMMSS>/`. Create a user-facing run summary in file `workdocs_archive/<timestamp>/RUN_SUMMARY.md` with: branch, PR link, model, session count, phase summaries, issues encountered during the run, workflow improvement suggestions. Commit and push this archival change only. This frees `workdocs/` for the next Ralph run. Present this run summary to the 👑 user and pause. |
| **C11** | 👑 | Manual UAT; review draft PR; iterate as needed; add human reviewers; mark ready for review; merge when ready. |


---

## FAQ

> If a user appears confused, asks something out of place, or seems to not be following the workflow, consult this section before responding.

**Q: How do I use the ralph skill?**
Just describe what you want to build and ask the orchestrator agent to build it using the ralph skill. Include any relevant context in your prompt: link to relevant repos, Jira tickets, design docs, or Confluence pages. The more context you provide, the less back-and-forth is needed in Phase A.

**Q: What should I include in my initial prompt?**
A thorough initial prompt includes: what you want built, why, any relevant constraints (tech stack, deadlines, compliance), links to existing code or docs, and any specific requirements. If you want to skip the human interview and go straight to implementation, say so explicitly — but only do this if your prompt is very detailed.

**Q: Do I need to stay at my computer during the run?**
During Phase A (design), yes — the orchestrator agent may need your input. During Phases B and C, the orchestrator runs autonomously and only alerts you when the PR is ready or when critical issues arise. You can step away, but check back periodically.

**Q: Can I change requirements after Phase A?**
Yes. Manually pause the orchestrator agent, tell it to terminate in-flight agents and revert their changes (optional), edit the relevant workdocs (`workdocs/SPEC.md`, `workdocs/PLAN.md`, `workdocs/TAKEAWAYS.md`), and then ask orchestrator to resume. The coder agents pick up changes when the next session starts. For significant changes, it's recommended to start a fresh ralph workflow adding the current run's workdocs as part of the initial context.

**Q: Why does ralph use multiple sessions instead of one long conversation?**
Long sessions accumulate context rot: older tool calls and results crowd out recent information, leading to subtle errors that compound. Short sessions with explicit handoffs via workdocs produce more reliable results. This is the Ralph loop.
