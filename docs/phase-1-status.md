# Phase 1 observed status

Last checked: 2026-09-24. No credentials or tenant identifiers are included. All data is simulated.

## Result: Technical Feasibility Gate passed (2026-09-24)

The Phase 1 pass condition was met once, end to end, in the `Project C North America Spike` Developer environment (Canada region):

`SIM-C-002` created in Dataverse → row-created trigger → `Project C OpenAI Spike` Custom Connector (`gpt-4o-mini`, fixed JSON schema) → Parse JSON → the same row updated.

| Item | Observed |
| --- | --- |
| Flow | `Project C - Phase 1 - Classify Note`, built in Power Automate by the project owner with step-by-step guidance |
| Trigger | Microsoft Dataverse, change type *Create* only, scope Organization (Create only, so the writeback does not re-trigger the flow) |
| Test row | `SIM-C-002`, created through the table's form with Case Ref and Handoff Notes entered before saving |
| Run | Succeeded, duration about 4 seconds |
| AI Issue Category written back | `CUSTOMER_INPUT_PENDING` (one of the four fixed values) |
| AI Summary written back | “The implementation team is awaiting field mapping confirmation from the customer before proceeding with the final setup handoff.” |

Evidence: [run history](evidence/phase-1-run-history.png) · [SIM-C-002 row after writeback](evidence/phase-1-sim-c-002-writeback.png)

One earlier run failed. `SIM-C-001` was entered directly in the table grid; the row was created as soon as Case Ref was committed, before Handoff Notes was filled in. The Create-only trigger therefore sent an empty note, and OpenAI returned HTTP 400 (`Invalid value for 'content': expected a string, got null`). Handoff Notes added afterward counted as an update and did not re-trigger the flow. Finding: with this trigger design, a row must be written with its note in a single save.

This proves technical feasibility for one simulated row only. It is not evidence of accuracy, time saved, or any business outcome. The category is a provisional label, not a verified root cause.

The sections below record how the gate was reached, including the blocked attempts.

## Independent Developer environment (current attempt)

The project owner resumed Phase 1 with a separate Microsoft work account and Developer environment named `Andy 的環境`. In this environment, the `Operational Exception Spike` Dataverse table was created and its `Case Ref`, `Handoff Notes`, `AI Issue Category`, and `AI Summary` columns were visible afterward. No test row has been inserted yet, because the flow must be enabled before the row-created trigger can be tested.

Power Automate initially reused the QUT sign-in. The account owner switched to the new work account, and `Andy 的環境` became selectable. The `Project C Gemini Spike` Custom Connector was imported and saved with one `ClassifyNotes` action and `x-goog-api-key` header authentication. The account owner entered the Gemini key directly in Power Automate, and the connection appeared in the connector test page.

Two tests used only the simulated `SIM-C-001` handoff note and the fixed classification schema:

| Endpoint | Observed result |
| --- | --- |
| `POST /v1beta/interactions` | HTTP 400: `This API is not available in your current location.` |
| `POST /v1beta/models/gemini-3.5-flash-lite:generateContent` | HTTP 400: `User location is not supported for the API use.` |

The connector was updated to the simpler `generateContent` endpoint, and the repo definition and request example match that saved version. Both endpoints were rejected before a model response was returned. Flow creation and the end-to-end row run were not attempted after this external API block. **The Technical Feasibility Gate has not passed.** Do not report an AI category, summary, or business outcome from this attempt. Google lists supported regions, but the observed API response does not identify which part of this Power Automate-to-Gemini path Google classified as unsupported. [Google region list](https://ai.google.dev/gemini-api/docs/available-regions)

An OpenAI Chat Completions connector definition and one request example were prepared as a narrow fallback. Both JSON files passed local syntax checks. `Project C OpenAI Spike` was then imported into the same `Andy 的環境` Developer environment. Its saved definition showed `POST https://api.openai.com/v1/chat/completions` and an `Authorization` header API-key field. The owner created the connection by entering the key in Power Platform; the key was not recorded in the repo or chat. One connector test with the same simulated note and fixed JSON schema returned HTTP 403, `unsupported_country_region_territory` / `Country, region, or territory not supported`. No model output was returned. Billing status was not confirmed. This does not change the gate status.

The Power Platform admin center initially opened under an older QUT sign-in and therefore could not display this independent environment. No QUT environment content was created or changed during that check. In a separate sign-in under Andy, the admin environment list showed `Andy 的環境` as a Developer environment in the **Asia** region. The connector response carried an OpenAI edge header ending `HKG`, but that alone does not establish the exact outbound IP.

To test the region hypothesis within the existing Power Platform architecture, a second Developer environment named `Project C North America Spike` was created with Dataverse requested and the North America macro region selected. The admin list assigned it to **Canada** and later showed **Ready** with Dataverse. After the owner explicitly authorized connector creation and Phase 1 testing in this environment, the same OpenAI OpenAPI file was imported and `Project C OpenAI Spike` was saved. Its test page shows one `ClassifyNotesOpenAI` operation. The owner corrected the connection's `Authorization` value to include the `Bearer ` prefix without sharing the key. One connector test using only the simulated `SIM-C-001` note returned **HTTP 200** with schema validation successful. The response contained `issue_category: CUSTOMER_INPUT_PENDING` and the summary: “The implementation team is awaiting confirmation of the requested field mapping from the customer before proceeding with the final setup handoff.” This confirms the connector can call the LLM API from this environment.

Creating a Dataverse table in this environment then showed “You don't have permission to create here.” In the Power Platform admin center, Andy's environment user was enabled but had no directly assigned security role; user diagnostics showed read/write access mode and an active license. Refreshing that user did not change the table permission. The owner then authorized granting Andy the `System Customizer` role in this environment; after that, the `Operational Exception Spike` table and its four Phase 1 columns were created and verified by the owner. The flow and one-row test that followed are recorded in the result section above. The original Asia environment and its Dataverse table remain in place but are no longer used for Phase 1.

## QUT Developer environment (earlier blocked attempt)

| Step | Observation |
| --- | --- |
| Local repo | Minimal schema, connector definition, request example, and one simulated case created. JSON syntax and local consistency checks passed. |
| Developer environment | Created; admin center showed **Ready** and **Dataverse: Yes**. |
| Custom Connector definition | `Project C Gemini Spike` imported and saved with one `ClassifyNotes` action. |
| Custom Connector connection | **Blocked.** Connection creation showed a tenant Data Loss Prevention (DLP) policy error. No Gemini key was entered. |
| Dataverse table | **Blocked.** The maker portal said the account lacked permission to create a table in the selected developer environment and offered to switch environments. The switch was cancelled. |
| Flow and one-row run | Not created or run. No category or summary has been written back to Dataverse. |

This earlier QUT attempt was stopped by the project owner's decision. A saved connector definition and local JSON checks do not meet the required end-to-end proof. Do not work around an organization's DLP policy in its tenant.
