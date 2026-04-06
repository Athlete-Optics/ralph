---
name: testing-guide
description: Testing guidelines for agentic coding workflows
---

# Testing Guide

Guidelines for writing and running tests in agentic coding workflows. Generic — usable with or without the Ralph workflow.

---

## Test Types

| Type | When to use | Example |
|------|------------|---------|
| **Unit** | Isolated logic, pure functions, data transformations | `test_calculate_discount_applies_percentage` |
| **Functional** | Feature-level behavior with real (non-mocked) components | `test_login_flow_creates_session` |
| **Property-based** | Functions with broad input domains, invariants that must hold for any input | `test_serialize_then_deserialize_is_identity` |
| **Data-driven** | Same logic, many input/output combinations | `test_tax_calculation` parameterized across jurisdictions |
| **Golden file** | Output that should match a known-good snapshot (HTML, reports, configs) | `test_report_output_matches_golden` |
| **Integration** | Components interacting with external systems (DB, APIs, file I/O) | `test_create_user_writes_to_database` |
| **End-to-end** | Full user journeys through the running system | `test_checkout_flow_from_cart_to_confirmation` |
| **UI / browser** | Visual correctness, interaction flows, responsive behavior | `test_modal_closes_on_escape_key` |
| **Security** | Auth bypass, injection, privilege escalation, data leakage | `test_unauthenticated_user_gets_401` |
| **Performance / load** | Response time, throughput, resource usage under load | `test_api_responds_under_200ms_at_100rps` |

Not every project needs every type. The specification should define the appropriate mix.

---

## Preferences

- **Automated over manual.** Every test should run without human intervention.
- **Fast over slow.** Prefer unit and functional tests; use integration/E2E only where cheaper tests can't verify the behavior.
- **Deterministic over flaky.** If a test depends on timing, network, or randomness, make it deterministic (fixed seeds, mocked clocks, stubbed network). If a flaky test cannot be fixed, document it and quarantine it.
- **Behaviors over implementation.** Test what the code does, not how it does it. Tests should survive refactors.
- **Mock at boundaries.** Mock external systems (HTTP, DB, file I/O), not internal modules. Internal mocking creates brittle tests.
- **Fixtures and factories over hardcoded data.** Generate test data programmatically. Hardcoded data drifts from reality and obscures intent.
- **One assertion per concept.** A test can have multiple assertions, but they should verify a single logical concept. If a test name needs "and", consider splitting.

---

## Fix-and-Reverify Loop

When tests fail:

1. **Fix the code** (not the test) unless the test itself is incorrect.
2. **Re-run ALL tests**, not just the failing one — a fix can break something else.
3. **Repeat** until the full suite is clean or `max_fix_iterations` is reached.
4. **Never skip or weaken a test** to make it pass. If a test is wrong, fix the test and document why.
5. **All issues must be fixed** — even minor or cosmetic. Small issues compound.

If the loop exhausts `max_fix_iterations`, stop and escalate rather than shipping broken code.

---

## Coverage Targets

- Strive for high coverage of **new and modified code**. 100% path coverage is not the goal.
- Focus on: critical paths, edge cases, error handling, boundary conditions.
- Coverage metrics are a signal, not a target. A test suite with 80% coverage that tests meaningful behaviors is better than 95% coverage that tests getters and setters.
- When coverage gaps exist, prioritize: error paths > edge cases > happy paths (happy paths are usually covered first).

---

## Test Naming

Test names should describe the behavior being verified, not the implementation.

| Good | Bad |
|------|-----|
| `test_user_with_expired_token_gets_403` | `test_check_token` |
| `test_empty_cart_shows_empty_state` | `test_cart_render` |
| `test_concurrent_writes_are_serialized` | `test_mutex` |
| `test_csv_export_includes_header_row` | `test_export` |

Pattern: `test_<subject>_<condition>_<expected_outcome>`

---

## Pre-existing Failures

If pre-existing tests fail when you run the suite:

1. **Investigate** whether your changes caused the failure. Check git blame, recent commits, and the test's history.
2. If **your changes caused it**: fix it — this is a regression.
3. If **pre-existing and unrelated**: document in the PR description under a "Known pre-existing failures" section. Do not block your work on them, but do not delete or skip them either.
4. If **unclear**: err on the side of investigating. A test that "always fails" may be a canary for a real issue.
