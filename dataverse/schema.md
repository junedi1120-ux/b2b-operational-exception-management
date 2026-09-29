# Phase 1 Dataverse schema

Create **one** custom table: `Operational Exception Spike` (suggested logical name `pc_operationalexceptionspike`; the actual publisher prefix may differ). Use the automatically generated Dataverse row ID as the flow's update key.

| Column display name | Suggested schema name | Type | Required | Purpose |
| --- | --- | --- | --- | --- |
| Case Ref | `pc_case_ref` | Primary name, single line of text | Yes | Identify the simulated test case. |
| Handoff Notes | `pc_handoff_notes` | Multiple lines of text | Yes for the test row | Only text sent to the LLM. |
| AI Issue Category | `pc_ai_issue_category` | Single line of text | No | Suggested fixed category from the LLM. |
| AI Summary | `pc_ai_summary` | Multiple lines of text | No | One or two factual sentences from the LLM. |

The primary key, created/modified timestamps, and other system fields are supplied by Dataverse. The `pc_` prefix is illustrative; record the actual logical names shown in your environment and use those in Power Automate.

Fixed category values for this spike:

1. `CUSTOMER_INPUT_PENDING`
2. `INTERNAL_HANDOFF`
3. `TECHNICAL_BLOCKER`
4. `OTHER_UNCLEAR`

These describe what the note says. They do not establish a true root cause. For Phase 1, the category is text because the OpenAI structured-output schema (strict JSON schema with an enum) enforces the allowed values and the flow only writes after parsing. A Dataverse Choice column would add setup work without changing the one-row feasibility result.

Do not add `final_category` or review-status fields in this spike; those belong to the human-review MVP after the feasibility gate. The deterministic SLA and onboarding fields likewise belong to the later rules stage.
