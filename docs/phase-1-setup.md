# Phase 1: Power Platform setup and one-row proof

This guide reflects the setup that passed the Technical Feasibility Gate on 2026-09-24 (see [phase-1-status.md](phase-1-status.md)). All data is simulated.

## Pass condition

One simulated row in Dataverse → row-created trigger in Power Automate → `Project C OpenAI Spike` Custom Connector → OpenAI structured JSON → the same row's `AI Issue Category` and `AI Summary` populated. Record the run result only after viewing the flow run and rereading the row.

## 0. Environment prerequisites

- Use a **Developer environment** with Dataverse, created under a work or school account you control. Microsoft says the Power Apps Developer Plan supports Dataverse, cloud flows, and custom connectors for development and testing. Do not use a production environment. [Developer Plan](https://learn.microsoft.com/en-us/power-platform/developer/plan)
- **Region matters.** In an Asia-region environment, both the Gemini API and the OpenAI API rejected the connector's calls with location errors. The passing setup used a Developer environment created with the **North America** region selected (assigned to Canada). [Environment regions](https://learn.microsoft.com/en-us/power-platform/admin/regions-overview) · [OpenAI supported countries](https://help.openai.com/en/articles/5347006-openai-api-supported-countries-and-territories)
- If creating a table shows “You don't have permission to create here,” assign your user the **System Customizer** security role for that environment in the Power Platform admin center.
- An **OpenAI API key** with API billing enabled. API billing is separate from a ChatGPT subscription; setting a monthly usage limit is recommended. [OpenAI API billing](https://help.openai.com/en/articles/9039756)

## 1. Create the table

In [Power Apps](https://make.powerapps.com/), select the Developer environment, then **Tables → New table**. Create `Operational Exception Spike` with primary name `Case Ref`. Add exactly the three other columns in [schema.md](../dataverse/schema.md): `Handoff Notes`, `AI Issue Category`, `AI Summary`. [Microsoft table instructions](https://learn.microsoft.com/en-us/power-apps/maker/data-platform/create-custom-entity)

Do not add the test row yet. A row created before the flow is enabled does not exercise the trigger.

## 2. Import the Custom Connector and create its connection

In the same environment, open **Custom connectors → New custom connector → Import an OpenAPI file**, select [openai-chat-completions.swagger.json](../connector/openai-chat-completions.swagger.json), and name it `Project C OpenAI Spike`. Check:

- Host `api.openai.com`, base URL `/v1`, one action `ClassifyNotesOpenAI` (`POST /chat/completions`, shown as *Classify handoff notes*).
- Security: API key in the `Authorization` header.

Save the connector. When creating the connection, enter the API key value as **`Bearer <your key>`** — the word `Bearer`, one space, then the key. Omitting the prefix makes the connection fail. Enter the key only in the Power Platform connection, never in the repo, chat, or a flow variable. [Import instructions](https://learn.microsoft.com/en-us/connectors/custom-connectors/define-openapi-definition)

## 3. Create the cloud flow

In [Power Automate](https://make.powerautomate.com/), in the same environment, choose **Create from blank** (automated cloud flow) and name it `Project C - Phase 1 - Classify Note`.

1. **Trigger — Microsoft Dataverse, “When a row is added, modified or deleted.”** Change type **Create** (Chinese UI: 建立), table **Operational Exception Spikes**, scope **Organization**. Create-only prevents the later writeback from triggering the flow again. [Trigger reference](https://learn.microsoft.com/en-us/power-automate/dataverse/create-update-delete-trigger)
2. **Action — `Project C OpenAI Spike` › Classify handoff notes.** (Filter the action list by *Custom* if it does not appear.) Rename the step `ClassifyNotes`; the later expression depends on this exact name. Set:
   - `model`: `gpt-4o-mini`
   - `messages` item 1 — `role`: `system`; `content`: the system instruction in [openai-request.example.json](../samples/openai-request.example.json)
   - `messages` item 2 — `role`: `user`; `content`: the trigger's **Handoff Notes** dynamic value
   - Under advanced parameters, `response_format.type`: `json_schema`; `response_format.json_schema`: the `json_schema` object from the same example file (name `exception_classification`, `strict: true`, the four-value `issue_category` enum, and `summary`)
3. **Action — Data Operation › Parse JSON.** Content (as an expression):

   ```text
   body('ClassifyNotes')?['choices']?[0]?['message']?['content']
   ```

   The model returns the structured result as a JSON **string** in the first choice's message content, so it must be parsed before mapping. Schema:

   ```json
   {"type":"object","properties":{"issue_category":{"type":"string"},"summary":{"type":"string"}},"required":["issue_category","summary"]}
   ```

4. **Action — Microsoft Dataverse › Update a row.** Table **Operational Exception Spikes**; Row ID: the trigger's row identifier (the dynamic value named after the table). Under advanced parameters set `AI Issue Category` = Parse JSON **issue_category** and `AI Summary` = Parse JSON **summary**. Leave every other field empty.
5. Save. The flow is turned on after saving.

If the OpenAI call fails, Parse JSON and Update a row are skipped, so the AI columns stay blank rather than receiving partial output. [Structured Outputs](https://developers.openai.com/api/docs/guides/structured-outputs) · [Power Automate expressions](https://learn.microsoft.com/en-us/power-automate/expression-cookbook)

## 4. Run the one-row test

After the flow is saved and on, open the table in Power Apps, choose **Edit → New row → New row using form** (Chinese UI: 新增列 › 使用表單新增資料列), and enter both values from [one-case.json](../samples/one-case.json) **before saving**. Do not enter AI values manually.

Do **not** type the test row directly into the table grid: the grid creates the row as soon as the first cell (Case Ref) is committed, so the Create trigger fires with an empty Handoff Notes value and OpenAI returns HTTP 400 (`expected a string, got null`). Adding the note afterward is an update and does not re-trigger the flow.

Then open the flow's **28-day run history** and confirm the run succeeded, and reopen the row to confirm both AI columns are populated. The category must be one of the four fixed values; the summary must be one or two factual sentences without a root-cause claim.

Record the observed result in [phase-1-status.md](phase-1-status.md) before calling the gate passed. A connector-only test or a locally valid JSON file does not satisfy the end-to-end gate.

## Earlier Gemini attempt

The spike first targeted the Gemini API free tier via [gemini-generate-content.swagger.json](../connector/gemini-generate-content.swagger.json) and [llm-request.example.json](../samples/llm-request.example.json). From the Asia-region environment it returned location errors before any model response, so the passing setup uses OpenAI. Those files are kept as a record of the attempt; see the status file for details.
