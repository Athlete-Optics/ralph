---
name: verification-guide
description: Verification guidelines for agentic coding workflows
---

# Verification Guide

Guidelines for verifying code changes in agentic coding workflows. Generic — usable with or without the Ralph workflow.

---

## Verify Before Committing

Always run these before committing:

- **Tests**: the full relevant test suite (not just your new tests).
- **Linting**: language-specific linters configured for the project.
- **Type checking**: if the project uses static types, run the type checker.
- **Build**: if applicable, verify the project builds without errors.

Never commit code that doesn't pass its own tests. If a pre-existing test fails and it's unrelated to your changes, document it — but do not commit code that introduces new failures.

---

## Self-Review

Review your own changes as an external reviewer would. Before considering your work done:

- **Correctness**: does the code do what it's supposed to? Are there off-by-one errors, null/nil handling gaps, or missed error paths?
- **Edge cases**: what inputs break it? Empty collections, zero values, Unicode, very large inputs, concurrent access.
- **Error handling**: are errors caught, logged, and propagated correctly? Are error messages informative without leaking internals?
- **Security**: authentication/authorization checks present? Input validated? No secrets in logs or error messages? No SQL injection, XSS, or path traversal?
- **Naming quality**: do names communicate intent? Would a reader understand the code without comments?
- **Unnecessary complexity**: is there a simpler way? Are abstractions justified by actual reuse or do they add cognitive overhead?
- **Diff review**: read the actual diff (not just the files). Ensure no debug code, commented-out blocks, or unintended changes made it in.

---

## End-to-End Testing

After implementation, test the complete user journey:

- Think like an end user, not a developer. Navigate the UI, call the API, run the CLI — whatever the user does.
- Test the **happy path** first, then **error paths** (bad input, missing permissions, network failure).
- Test **cross-feature interactions**: does your change break existing features? Does it work with the features it depends on?
- For UI: test on different screen sizes, with keyboard navigation, with assistive technology if relevant.
- For APIs: test with real HTTP clients (curl, Postman, or automated tests), not just unit test mocks.
- For CLIs: test with actual terminal invocations, including edge cases (no arguments, invalid flags, piped input).

---

## Critical Review Checklist

| Area | What to check |
|------|--------------|
| **Backwards compatibility** | Does this break existing APIs, CLI flags, config formats, database schemas, or file formats? |
| **Database migrations** | Are migrations reversible? Do they handle existing data correctly? Performance impact on large tables? |
| **API contract changes** | Are changes additive (new fields) or breaking (removed/renamed fields)? Is versioning needed? |
| **Performance regressions** | Any new N+1 queries, unbounded loops, or memory allocations? Any hot paths that got slower? |
| **Security** | Auth checks on new endpoints? Input validation? Secrets management? CSRF/XSS protection? |
| **Error messages** | Informative without leaking internal details (stack traces, SQL queries, file paths, secrets)? |
| **Logging** | Sufficient for debugging without being noisy? No sensitive data logged? Appropriate log levels? |

---

## Fix-and-Reverify

When issues are found during verification:

1. **Fix the issue.** Do not document it as a known issue unless it's truly out of scope.
2. **Re-verify from scratch.** Do not assume a fix didn't break something else. Re-run the full test suite and re-check the areas you reviewed.
3. **All issues must be addressed** — even minor or cosmetic ones. Small issues compound and signal low quality to reviewers.
4. **Track iterations.** If you've fixed and re-verified more than `max_fix_iterations` times, the problem may be structural. Stop and escalate.

---

## Escalation

If you cannot fix an issue after the maximum number of iterations:

- **Stop.** Do not ship broken code. Do not weaken tests. Do not mark the issue as "known" and move on.
- **Escalate** to the appropriate party (orchestrator, user, team lead) with:
  - What the issue is (symptoms, not just error messages).
  - What you tried (approaches, not just "I tried fixing it").
  - What you think the root cause is (even if uncertain).
  - What would be needed to fix it (tools, access, design changes).

---

## Workdocs Accuracy

After completing work, verify that documentation reflects reality:

- **Specs** accurately describe the final behavior (not just the intended behavior from before implementation).
- **Plans** have correct status markers and completion records.
- **Takeaways** include any learnings from verification that future agents should know.
- **Setup docs** are still accurate (new tools, changed commands, updated requirements).

Update any workdocs that have drifted from the code.
