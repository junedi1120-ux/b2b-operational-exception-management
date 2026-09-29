# Phase 2 observed status

Last checked: 2026-09-25. No credentials or tenant identifiers are included. All data is simulated.

## Result: Exception Detection Gate passed (2026-09-25)

Two scheduled detection flows were built in the `Project C North America Spike` Developer environment. Each rule-matched source row creates one `Pending` row in `Operational Exception Spike`, and the existing Phase 1 flow then classifies it:

`Ops Case` / `Onboarding` → scheduled detection flow → duplicate check → `Operational Exception Spike` row created → `Project C - Phase 1 - Classify Note` → AI fields written back.

| # | Gate test | Simulated input | Observed |
| --- | --- | --- | --- |
| 1 | SLA positive creates an exception | `SIM-C-101`: opened 2026-09-22, no first response, `Open` | `EXC-SLA-SIM-C-101` created, `SLA_BREACH`, `Pending` |
| 2 | SLA negative creates nothing | `SIM-C-102`: opened 2026-09-22, first response recorded, `Open` | No exception; the list step returned one case only |
| 3 | Onboarding positive creates an exception | `SIM-O-201`: target go-live 2026-09-20, `In Progress` | `EXC-ONB-SIM-O-201` created, `ONBOARDING_DELAY`, `Pending` |
| 4 | Onboarding negative creates nothing | `SIM-O-202`: target go-live 2026-09-20, `Completed` | No exception; the list step returned one onboarding only |
| 5 | Re-run does not duplicate | Both flows run again with the same data | Condition `false` in both flows; create step skipped; one exception per source remained |
| — | New exceptions receive AI triage | Rows created by tests 1 and 3 | `AI Issue Category` and `AI Summary` filled by the Phase 1 flow (both `CUSTOMER_INPUT_PENDING`) |

This shows that the rules, the duplicate check, and the AI handoff worked on four simulated rows. It is not evidence of accuracy, time saved, or any business outcome. The AI category is a provisional label, not a verified root cause.

## Implemented design

| Item | Implementation |
| --- | --- |
| Source tables | `Ops Case` (Case Ref, Opened At, First Response At, Status, Handoff Notes); `Onboarding` (Onboarding Ref, Target Go-live Date, Status, Handoff Notes) |
| Exception table | Phase 1 `Operational Exception Spike`, with Source Type, Source ID, Exception Type, and Review Status added. `Case Ref` holds the Exception ID. Handoff Notes is kept as the AI input. |
| Flow A | `Project C - Phase 2 - Detect SLA Breach`, daily recurrence. Filter: no first response, status `Open`, opened more than 24 hours ago. |
| Flow B | `Project C - Phase 2 - Detect Onboarding Delay`, daily recurrence. Filter: status not `Completed`, target go-live date before today. |
| Duplicate prevention | Before creating a row, each flow looks for an exception with the same Source ID, the same Exception Type, and Review Status `Pending`. It creates a row only when none is found. |
| Open definition | For Phase 2 duplicate prevention, `Pending` means open. Whether Accepted / Edited / Rejected count as closed is decided in Phase 3. |
| Status and type columns | Single-line text, not Choice, to keep the flow filters simple |

## Accepted implementation differences

| Plan | Implemented | Reason |
| --- | --- | --- |
| Onboarding ID | `Onboarding Ref` | `Onboarding ID` conflicted with the table's system key name. The meaning is unchanged. |
| `Opened At < utcNow() - 24h` | `Microsoft.Dynamics.CRM.OlderThanXHours(PropertyName='crc93_openedat',PropertyValue=24)` | Built-in Dataverse query function with the same 24-hour meaning |
| — | Power Platform UI switched to English; flow steps renamed in English | Makes screenshots usable for recruiter-facing material |

## Evidence

| File | Shows |
| --- | --- |
| [phase-2-flow-a-rerun-no-duplicate.png](evidence/phase-2-flow-a-rerun-no-duplicate.png) | Flow A re-run: condition `false`, create skipped |
| [phase-2-flow-a-exception-table.png](evidence/phase-2-flow-a-exception-table.png) | `EXC-SLA-SIM-C-101` row with Handoff Notes and AI fields |
| [phase-2-flow-a-exception-ai-fields.png](evidence/phase-2-flow-a-exception-ai-fields.png) | AI fields and source fields for the SLA exception |
| [phase-2-flow-a-exception-source-fields.png](evidence/phase-2-flow-a-exception-source-fields.png) | Source Type, Source ID, Exception Type, Review Status |
| [phase-2-flow-b-first-run-create.png](evidence/phase-2-flow-b-first-run-create.png) | Flow B first run: condition `true`, exception created |
| [phase-2-flow-b-rerun-no-duplicate.png](evidence/phase-2-flow-b-rerun-no-duplicate.png) | Flow B re-run: condition `false`, create skipped |
| [phase-2-exception-table-both-rules.png](evidence/phase-2-exception-table-both-rules.png) | Both exceptions in one table view |

The four Flow A screenshots were taken with the Chinese UI. They will be retaken in English in Phase 4.

## Time

Phase 2: about 1.5 hours (approximate). Cumulative: about 6.5 hours (approximate).
