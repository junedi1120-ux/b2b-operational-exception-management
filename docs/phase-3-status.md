# Phase 3 observed status

Last checked: 2026-09-29. No credentials or tenant identifiers are included. All data is simulated.

## Result: Human Review Gate passed (2026-09-29)

Reviewers work directly on the Dataverse main form of `Operational Exception Spike`. No custom UI was built.

`Operational Exception Spike` row (`Pending`, AI fields filled) → reviewer opens the main form → sets `Review Status`, `Final Category`, `Review Note` → saves. AI fields stay unchanged.

| # | Test | Simulated input | Observed |
| --- | --- | --- | --- |
| 1 | Accepted (SLA) | `EXC-SLA-SIM-C-101`, AI `CUSTOMER_INPUT_PENDING` | `Accepted`; Final Category = AI category; note recorded; AI fields unchanged |
| 2 | Accepted (Onboarding) | `EXC-ONB-SIM-O-201`, AI `CUSTOMER_INPUT_PENDING` | `Accepted`; Final Category = AI category |
| 3 | Edited | `SIM-C-103` → `EXC-SLA-SIM-C-103`, AI `CUSTOMER_INPUT_PENDING` | `Edited`; Final Category `INTERNAL_HANDOFF`; note explains the correction |
| 4 | Rejected | `SIM-C-104` → `EXC-SLA-SIM-C-104`, AI `INTERNAL_HANDOFF` | `Rejected`; Final Category blank; note records a false positive (source status stale) |
| 5 | Flow A regression after review | `SIM-C-101` still matches the SLA rule; its exception is `Accepted` | Condition `false`; create skipped; no new exception |
| 6 | Flow B regression after review | `SIM-O-201` still matches the onboarding rule; its exception is not `Pending` | Condition `false`; create skipped; no new exception |
| 7 | No re-creation after Edited / Rejected | Flow A run again with `SIM-C-103` and `SIM-C-104` still matching the rule | Re-run at 2026-09-29 18:50 listed 3 cases; for `SIM-C-104` (Rejected) the condition was `false` and create was skipped. Table afterwards: 6 rows (4 exceptions + 2 Phase 1 rows), one exception per source |
| — | AI fields protected on the form | `AI Issue Category`, `AI Summary` | Read-only (lock icon) on the main form |

Tests 3 and 4 are controlled simulated test cases. They were designed to exercise the Edited and Rejected paths. The plan allowed one retry if the AI matched the intended category; no retry was needed.

This shows that the three review paths and the duplicate rule work on simulated rows. It is not evidence of AI accuracy, time saved, or any business outcome. The AI category is a provisional label, not a verified root cause.

## Review rules (confirmed by ChatGPT, 2026-09-29)

| Review Status | Final Category | Review Note |
| --- | --- | --- |
| `Accepted` | Same as `AI Issue Category` | Optional |
| `Edited` | Reviewer's corrected category | Required |
| `Rejected` | Blank | Required |

`Rejected` means the exception or AI triage should not produce a valid final category. If only the category is wrong, use `Edited`.

## Implemented design

| Item | Implementation |
| --- | --- |
| New columns | `Final Category` (`crc93_finalcategory`), `Review Note` (`crc93_reviewnote`); both single line of text; Review Note max length raised above the default 100 |
| Review Status values | `Pending` / `Accepted` / `Edited` / `Rejected` (text column, not Choice) |
| Form | Main form shows Final Category, Review Note, Review Status; AI Issue Category and AI Summary set to read-only |
| Duplicate prevention (changed) | One exception per `Source ID + Exception Type`, regardless of Review Status. The Review Status filter was removed from `Check Existing SLA Exception` and `Check Existing Onboarding Exception`. |
| Condition step | Renamed `If No Pending Exception` → `If No Existing Exception` in both flows. The check steps were not renamed. |

## Known limits (accepted as MVP limits at the gate)

- Read-only applies to the form only. The AI fields can still be edited in the table grid. A production setup would use security roles.
- The review rules are a convention. Nothing enforces Final Category = AI category on Accepted, or a required note on Edited / Rejected.
- One exception per source means a second, independent event on the same source is not detected. Accepted as MVP scope (no episode / version field).

## Observations (not changes)

- `EXC-SLA-SIM-C-103`: AI Summary says the file is expected "from the customer", which the handoff note does not state. The prompt asks the model not to infer absent facts.
- `EXC-SLA-SIM-C-104`: AI Summary correctly says the case is resolved, but the fixed category list has no "not an issue" value, so the model returned `INTERNAL_HANDOFF`.

Decision (2026-09-29): record both, change nothing. No prompt tuning and no new category. The Rejected path is the mechanism for cases outside the taxonomy. These are not accuracy metrics.

## Evidence

| File | Shows |
| --- | --- |
| [phase-3-review-accepted-c101.png](evidence/phase-3-review-accepted-c101.png) | Accepted review on `EXC-SLA-SIM-C-101`; AI fields locked |
| [phase-3-flow-a-regression-no-duplicate.png](evidence/phase-3-flow-a-regression-no-duplicate.png) | Flow A after rule change: condition `false`, create skipped |
| [phase-3-flow-b-regression-no-duplicate.png](evidence/phase-3-flow-b-regression-no-duplicate.png) | Flow B after rule change: condition `false`, create skipped |
| [phase-3-edited-c103-before-review.png](evidence/phase-3-edited-c103-before-review.png) | `EXC-SLA-SIM-C-103` before review (`Pending`, AI fields filled) |
| [phase-3-review-edited-c103.png](evidence/phase-3-review-edited-c103.png) | Edited review on `EXC-SLA-SIM-C-103` |
| [phase-3-review-rejected-c104.png](evidence/phase-3-review-rejected-c104.png) | Rejected review on `EXC-SLA-SIM-C-104` |
| [phase-3-exception-table-all-review-paths.png](evidence/phase-3-exception-table-all-review-paths.png) | Table view: Exception Type, Review Status, Final Category for all rows |
| [phase-3-flow-a-rerun-c104-check.png](evidence/phase-3-flow-a-rerun-c104-check.png) | Flow A re-run, iteration 3 of 3: duplicate check for `SIM-C-104` |
| [phase-3-flow-a-rerun-c104-no-duplicate.png](evidence/phase-3-flow-a-rerun-c104-no-duplicate.png) | Same iteration: condition `false`, create skipped |
| [phase-3-exception-table-after-rerun.png](evidence/phase-3-exception-table-after-rerun.png) | Table after the re-run: Case Ref, AI fields, Review Status; 6 rows, no duplicates |

Form screenshots show a Chinese form shell (tab and system labels). `phase-3-exception-table-all-review-paths.png` does not show Case Ref; `phase-3-exception-table-after-rerun.png` does. Recruiter-facing screenshots will be retaken in Phase 4.

## Time

Phase 3: about 1 hour (approximate; screenshot uploads span 18:04–18:57 on 2026-09-29, plus short setup before the first screenshot and review time outside this session). Cumulative: about 7.5 hours (approximate).
