# Ralph — Agentic Coding Workflow

Ralph is an unsupervised agentic coding workflow built for Claude Code. It orchestrates multiple short, focused agent sessions — each with fresh context — connected by **workdocs** as external memory and a **handoff file** (`NEXT_PROMPT<X>.md`) between them. Inspired by the [Ralph Wiggum Loop](https://ghuntley.com/specs/) (Geoffrey Huntley, 2025).

Two core principles:
- **Avoid context rot** through deliberate session management
- **Prevent error propagation** through agent self-verification and back pressure

---

## Workflow

```mermaid
flowchart TD
    U([👑 User]) -->|Describe what to build| A1

    subgraph PhaseA ["Phase A — Preparation"]
        A1[A1–A6: Orchestrator interviews User\nSetup & context check] -->|Interview done| A7
        A7[A7: Orchestrator spawns 🎨 Designer] --> A8
        A8[A8–A15: Designer explores, writes\nSPEC · PLAN · SETUP] --> A16
        A16{User reviews\nSPEC & PLAN?}
        A16 -->|Approved| A17[A17: Proceed to Phase B]
        A16 -->|Changes needed| A8
    end

    subgraph PhaseB ["Phase B — Implementation Loop (Ralph Loop)"]
        B1[🕹️ Spawn 🔨 Coder\nWrite NEXT_PROMPT] --> B2
        B2[🔨 Read workdocs\nPick ONE task ⬜ → 🔧] --> B3
        B3[Implement · Test · Self-review\nSimulated UAT] --> B4
        B4[Mark task ✅\nUpdate workdocs\nCommit & Exit] --> B5
        B5{Tasks\nremaining?}
        B5 -->|Yes| B1
        B5 -->|Stuck| B6[Orchestrator adjusts\nworkdocs, retries]
        B6 --> B1
    end

    subgraph PhaseC ["Phase C — Verification & Ship"]
        C1[🕹️ Spawn 🔍 Verifier\nWrite NEXT_PROMPT] --> C2
        C2[Full diff review\nHolistic UAT · Security] --> C3
        C3{Issues\nfound?}
        C3 -->|Fix & re-verify| C2
        C3 -->|Clean| C4
        C4[Push branch\nOpen Draft PR] --> C5
        C5[🤖 AI Reviewer loop\nPost inline comments] --> C6
        C6[Fix comments\nUpdate PR description\nAlert User] --> C7
        C7[Archive workdocs\nPresent RUN_SUMMARY]
    end

    A17 --> B5
    B5 -->|No tasks left| C1
    C7 --> End([👑 User reviews\nMerges PR])

    style PhaseA fill:#1e3a5f,stroke:#4a90d9,color:#e8f4fd
    style PhaseB fill:#1a3a1a,stroke:#4caf50,color:#e8f5e9
    style PhaseC fill:#3a1a1a,stroke:#ef5350,color:#fde8e8
    style U fill:#4a4a00,stroke:#ffc107,color:#fff9c4
    style End fill:#4a4a00,stroke:#ffc107,color:#fff9c4
```

---

## Roles

| Emoji | Role | Responsibility |
|-------|------|----------------|
| 👑 | **User** | Approves design, reviews PR, merges |
| 🕹️ | **Orchestrator** | Manages end-to-end workflow; spawns agents; owns loop control |
| 🎨 | **Designer** | Explores codebase, writes SPEC / PLAN / SETUP |
| 🔨 | **Coder** | Implements one task per session; self-verifies; exits |
| 🔍 | **Verifier** | Holistic verification; opens and iterates on the PR |

---

## Workdocs (External Memory)

Each fresh agent session re-reads these files from scratch — no reliance on context compaction.

| File | Purpose |
|------|---------|
| `USER_PROMPT.md` | Original user request, verbatim. Write-once. |
| `SPEC.md` | Full spec: requirements, architecture, verifiable success criteria |
| `PLAN.md` | Task list with `⬜ / 🔧 / ✅` status markers |
| `SETUP.md` | Project-specific tooling, env vars, test fixtures |
| `TAKEAWAYS.md` | Learnings, deviations, workflow observations |
| `NEXT_PROMPT<X>.md` | Handoff file written before each spawned agent |

---

## Key Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `designer_model` | `claude-opus-4-6` | Model for the designer agent |
| `coder_model` | `claude-sonnet-4-6` | Model for each coder agent |
| `verifier_model` | `claude-opus-4-6` | Model for the verifier agent |
| `max_fix_iterations` | `10` | Max fix/re-verify cycles before escalating |
| `max_stuck_retries` | `3` | Max retries when an agent makes no progress |
| `ai_reviewer_max_iterations` | `5` | Max AI reviewer feedback cycles in Phase C |

---

## Usage

Install via [Claude Code](https://claude.ai/code) and point to this repo's `skills/` directory. Then, in any session:

> "Build [feature] using the ralph workflow."

The orchestrator agent handles everything from there — interviewing you during Phase A, running the implementation loop autonomously in Phase B, and alerting you when the PR is ready for review in Phase C.

---

## Philosophy

> Context rot is the enemy of reliable agentic systems. The Ralph loop exits deliberately, hands off via structured workdocs, and starts fresh — every single time.

See [`skills/ralph-workflow/SKILL.md`](skills/ralph-workflow/SKILL.md) for the full specification.
