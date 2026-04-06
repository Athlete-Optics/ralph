---
name: specification-guide
description: Guidelines for writing SPEC.md in a Ralph workflow run. Use when authoring or updating the specification and plan workdocs.
---

# Ralph Specification Guide

Guidelines for writing `SPEC.md` — the main source of truth for what is being built and how. The designer authors it in Phase A; coders may update it cautiously in Phase B under scope-of-impact rules.

A good specification is **complete enough** that a coder with no prior context can implement a task from `PLAN.md` without needing to ask clarifying questions, yet **concise enough** to stay within the workdocs token budget. Favor precision over length.

---

## Recommended Sections

Include all sections that apply to the project. Skip sections that are genuinely irrelevant, but err on the side of including them — an empty section heading signals that the topic was considered.

**Problem description**
- What problem is being solved, and for whom
- Current state: what exists today, what's broken or missing
- Why it matters: business motivation, user pain, technical debt

**Requirements**
- Functional requirements: what the software must do
- Non-functional requirements: performance, reliability, accessibility, compliance
- Security requirements: authentication, authorization, data handling, secrets management, attack surface considerations
- Constraints: budget, timeline, tech stack, backwards compatibility

**Assumptions**
- Explicit assumptions about the environment, users, data, existing systems, or third-party services
- Flag any assumptions that are uncertain or unverified — coders need to know what might not hold

**Existing architecture, features, and code structure** (if relevant)
- Key files, modules, services, and their relationships — only what's relevant to the project
- Existing features and capabilities that the project builds on or modifies
- Do not describe existing code exhaustively; focus on what coders will interact with or modify

**Proposed architecture, features, and code structure**
- High-level design: components, data flow, API contracts
- New or modified features and their expected behavior
- Key technical decisions and their rationale
- New dependencies and systems being introduced

**Deployment strategy**
- How changes will be rolled out (e.g. feature flags, staged rollout, blue/green)
- Migration steps if applicable
- Rollback plan

**Testing strategy**
- Which test types apply (see [testing-guide](testing-guide.md) for the full list): unit, functional, property-based, data-driven, golden file, integration, end-to-end, UI, security
- The appropriate mix for this project — not every project needs every type
- Required tooling for running tests (e.g. browser automation for UI, specific test frameworks)

**Success criteria**
- What must be true for the project to be considered done
- Every criterion must be **programmatically verifiable** — state exactly what tooling or command confirms it
- Success criteria commands must use only tools listed in `SETUP.md`; do not rely on tools that may not be installed
- Cover both individual-task criteria (also listed per task in `PLAN.md`) and whole-project criteria

**Coding guidelines** (only if non-trivial)
- Only include project-specific conventions that deviate from standard practices
- Do not reiterate common best practices — keep the token budget for what matters

---

## Phase B: Cautious SPEC.md Updates

Coders may need to update `SPEC.md` during Phase B when reality diverges from the design. Follow these scope-of-impact rules:

- **Own task only** (clarifying a requirement, correcting a detail that only affects your task) — proceed; update the spec and note it in `TAKEAWAYS.md`.
- **Adjacent tasks** (a change that affects neighboring tasks or dependencies) — proceed with caution; note the change and reasoning in `TAKEAWAYS.md` so other coders are aware.
- **Overall direction** (a requirement is fundamentally wrong, the architecture fails, a core assumption is invalid) — do NOT silently rewrite; add a note to `TAKEAWAYS.md` describing the issue, mark yourself stuck, and surface it to the orchestrator.

The key principle: changes that only affect your own task are safe. Changes that ripple outward require escalation proportional to the blast radius.

---

## Writing Success Criteria

Success criteria are the most important part of `SPEC.md`. They are the contract between the designer and the coder, and the checklist for the verifier.

**Rules:**
1. Every criterion must be expressed as a command or script that exits 0 on success and non-zero on failure.
2. Use only tools listed in `SETUP.md` — do not assume tools are available if they are not explicitly listed.
3. Cover the happy path, edge cases, and negative cases where feasible.
4. Keep criteria tight — a criterion that always passes regardless of implementation is useless.
5. Do not duplicate criteria across tasks unnecessarily; whole-project criteria belong in the final section.

**Common patterns:**
```bash
# File existence
test -f path/to/file && echo "ok"

# Pattern match (ripgrep)
rg "expected pattern" path/to/file

# Negative match
! rg "should not appear" path/to/file

# Structural check (Python)
python3 -c "
import re
text = open('path/to/file').read()
# ... assertion ...
print('PASS')
"

# Pre-commit hooks pass
pre-commit run --files path/to/file
```
