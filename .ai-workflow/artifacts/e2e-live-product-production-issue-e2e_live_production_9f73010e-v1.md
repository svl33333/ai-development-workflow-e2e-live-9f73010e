# Production E2E Issue

## Problem

The initial E2E verified only the Prototype-side C2C execution. The Production Project and parallel implementation-task behavior still require verification.

## Goal

Use the approved Prototype handoff to verify Production planning, two independent implementation tasks, integration, tests, and PR review.

## Scope

- Preserve `e2e-workflow-proof.txt`.
- Unit A owns only `production-proof-a.txt` and its evidence record.
- Unit B owns only `production-proof-b.txt` and its evidence record.
- Integrate both successful units and review the integrated result.

## Acceptance

- The plan manifest declares `max_parallel_codex_tasks: 2`.
- Unit A and Unit B have no dependencies and disjoint change scopes.
- Both units start from the approved base/generation and pass their own tests and local review.
- The integrated result passes all unit and integration checks.
- The Production ChatGPT Project independently reviews the final local PR draft.
- Publication and merge require separate human approval and are not performed by this test.

## Rollback

Reject any child with failed tests, scope escape, stale base/generation, invalid result schema, or a blocking review finding. Do not integrate rejected children.
