# Production implementation plan

## Purpose and inputs

This plan verifies that the approved Prototype result can be consumed by a Production workflow and that two independent implementation units can run in parallel before integration and PR review.

Inputs are:

- `e2e-workflow-proof.txt` (approved Prototype proof)
- `.ai-workflow/artifacts/e2e-live-product-prototype-handoff-e2e_live_production_9f73010e-v1.md`
- approved Production specification, SHA-256 `41623F0C6961A3956D975FF46489D221963846B1383F2CF39D4F460F833A70EB`
- approved Production Issue, SHA-256 `658E792CB5D28B22951F835B6332D59CECC5880CC4019CF87F148CC8415153FF`
- planning base revision `52a8270`
- orchestrator generation `1`

No child task may alter `e2e-workflow-proof.txt`. The working tree is intentionally dirty, so child tasks must never start from bare `52a8270`; the approved plan and manifest will define a later local-only execution baseline.

## Execution units

### Unit A: `production-proof-a`

Purpose: create the deterministic Production proof A from the approved Prototype input.

Dependencies: none. Owned files: `production-proof-a.txt`. The child implementation task may modify only that file. The orchestration layer writes `.ai-workflow/runs/e2e_live_production_9f73010e/unit-a-result.json` after the child commit is known.

The exact content is:

```text
production workflow proof
task_id=e2e_live_production_9f73010e
unit=A
prototype_result=e2e-workflow-proof.txt
prototype_task_id=e2e_live_9f73010e
prototype_base_revision=52a8270
```

Tests: exact-content assertion, Prototype identity/base assertion, scope-escape assertion, child-result schema validation, artifact SHA-256 assertion, and `git diff --check`.

Acceptance: status `SUCCEEDED`, zero blocking findings in local review, schema-valid result, artifact digest matches the file, and returned generation/base equal the dispatch fence.

### Unit B: `production-proof-b`

Purpose: create the same deterministic Production proof independently for unit B.

Dependencies: none. Owned files: `production-proof-b.txt`. The child implementation task may modify only that file. The orchestration layer writes `.ai-workflow/runs/e2e_live_production_9f73010e/unit-b-result.json` after the child commit is known.

The exact content is identical to Unit A except the final unit line is `unit=B`.

Tests and acceptance are identical to Unit A, with scope disjointness checked against Unit A. A and B must be simultaneously runnable and must not observe or depend on one another.

### Integration: `production-integration`

Dependencies: `production-proof-a` and `production-proof-b`. The integration task may write only `.ai-workflow/runs/e2e_live_production_9f73010e/integration-result.json` and `.ai-workflow/artifacts/e2e-live-product-local-pr-draft-e2e_live_production_9f73010e-v1.md`; it must not rewrite either child output.

Integration tests assert the original Prototype proof remains exact, A and B are exact and distinct, both reference this Production task and Prototype source, both child results validate against the managed schema, base/generation fencing matches, scopes are disjoint, no rejected/stale child is integrated, no unexpected implementation files exist, and `git diff --check` passes. On success, generate a local PR draft from the managed template.

## Stable execution-base rule

After independent plan review has completed and a human has approved the final plan, create one local-only execution baseline commit containing the already-approved artifacts, final plan, final manifest, review evidence, and plan-approval receipt. Do not push it. Record its hash and dispatch both children from that exact hash and generation. Reject any child with a different base or generation. This preparation does not authorize publication or merge.

## Review and gates

The plan and manifest must pass schema, dependency, DAG, unique-ID, disjoint-scope, digest, and cleanliness checks before review. The independent plan-review conversation receives repository artifacts and evidence, not planner history. Retain that reviewer conversation across rounds and perform at least three qualifying rounds, even if the first approves. Adopted changes require manifest digest recomputation and another review round. Any unresolved Critical/High blocking finding stops progression.

### Canonical approval digest

`approval_digest` is an uppercase hexadecimal SHA-256. It is calculated over the UTF-8 bytes of the following canonical input, using LF separators and exactly one final LF:

```text
production_spec_sha256=<approved Production specification SHA-256>
production_issue_sha256=<approved Production Issue SHA-256>
planning_base_revision=<planning base revision>
implementation_plan_sha256=<SHA-256 of the exact plan file bytes>
manifest_body_sha256=<SHA-256 of the canonical JSON manifest with the approval_digest member omitted>
```

The manifest body is canonical JSON encoded with UTF-8, LF line endings, stable property order as stored in the manifest, and one final LF. The digest output is uppercase hexadecimal. Any change to the approved spec, Issue, plan, manifest body, or planning base invalidates the digest and requires recomputation before another review round.

### Fail-closed child result acceptance

Schema validity alone is insufficient. For each child, the orchestrator must reject the result unless all of the following are true: `unit_id` equals the expected unit; `status` is `SUCCEEDED`; `commit` is non-null and exists; the child commit's diff from the dispatch base modifies exactly the child-owned proof file; `base_revision` and `generation` equal the approved dispatch fence; `local_review.reviewed_revision` equals `commit`; `local_review.blocking_count` is zero; `local_review.disposition` is `approved`; every mandatory test is explicitly named and marked passed; and `artifact_digest` equals the SHA-256 of the child output. After integration, the integrated output digest must still equal the child digest. Missing, empty, unrecognized, or inconsistent test evidence fails closed.

### Review-round protocol and persistence

For every qualifying round, persist a review record under `.ai-workflow/reviews/` containing the round number, the immutable reviewed input set (`reviewed_plan_sha256`, `reviewed_manifest_sha256`, `reviewed_approval_digest`), reviewer conversation/project identity, findings, and an explicit `adopted`, `rejected`, or `human_decision` disposition with rationale for every finding. If findings are adopted, also record the immutable resulting output set (`resulting_plan_sha256`, `resulting_manifest_sha256`, `resulting_approval_digest`). Apply adopted changes, recompute affected hashes and `approval_digest`, rerun all planning-artifact validation, and independently verify the revised artifacts through MCP before incrementing the qualifying-round counter. Then submit the revised artifacts to the same retained reviewer conversation. After each round update `plan_review_iteration`, `qualifying_plan_review_iteration`, `review_history`, retained-review fields in `review_context` (using the actual conversation/project identity), the state revision, and synchronized timestamps. H4 is reachable only when the qualifying counter is at least 3 and the latest qualifying round is `APPROVE`.

If a reviewed artifact's exact full-file digest cannot be recovered from durable evidence, preserve that review record as historical but mark it non-qualifying; never invent a digest. Restart the qualifying sequence from the first later round whose reviewed input tuple is complete.

After the qualifying review loop, hold H4 for explicit human approval bound to the final plan digest, manifest digest, approval digest, review artifact, and planning base. Until H4, do not create the execution baseline, child outputs/results, or implementation state.

After implementation and integration, submit the local PR draft to the Production ChatGPT Project for PR review through MCP. Store the review and findings locally. Critical/High findings must be fixed and re-reviewed; discretionary findings remain human decisions. Publication and merge remain separate human gates and are outside this E2E test.
