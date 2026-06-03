# Automation Studio — API Capabilities Reference

What works and what doesn't when managing **Automation Studio** (and the SQL activities inside it) via the SFMC REST/SOAP APIs. Every entry below was verified against the live tenant by reading the actual result (SOAP `<OverallStatus>` / REST read-back), **not** just the HTTP status code.

Tenant: `mcfl8qnsz4yp5b6zc2q3d11y0qty` · Stack S7 · EID 7281705
Last verified: 2026-06-01

---

## TL;DR

- You **can build** an automation end-to-end via API: create the automation shell → create a SQL Query activity → wire it in as a step. And you can manage the DEs it touches.
- You **cannot run** an automation via API (Run Once is UI-only — type is immutable).
- You **cannot create** a SQL Update activity against a **sendable** DE via API (requires a temporary un-sendable workaround, or the UI).

---

## ✅ Works via API (verified)

| Task | How | Notes |
|---|---|---|
| Create an automation (shell) | REST `POST /automation/v1/automations` | Returns real id/key, status Ready. |
| Create a SQL Query activity | REST `POST /automation/v1/queries` | Works **only against a non-sendable target DE**. Requires `categoryId` + `targetId` (the DE ObjectID), `targetKey`, `queryText`, `targetUpdateTypeId`. |
| Wire a query into an automation as a step | REST `PATCH /automation/v1/automations/{id}` with `steps[].activities[]` (`objectTypeId: 300` = SQL Query, `activityObjectId` = queryDefinitionId) | Read-back confirms the step + activity. |
| Read a query activity | REST `GET /automation/v1/queries/{id}` or `GET /automation/v1/queries` (list) | Reliable. Best way to learn the exact JSON schema — mirror an existing item. |
| Read an automation | REST `GET /automation/v1/automations/{id}` | Reliable by id. Shows steps, activities (with `id`, `activityObjectId`, `objectTypeId`), target DEs, row counts. |
| List automations | SOAP `Retrieve` `Program` (Properties: Name, CustomerKey, ObjectID) | More reliable than REST list. Paginated / non-deterministic ordering — don't assume completeness from one call. |
| DE prerequisites | REST `/data/v1/customobjects` (create/read), SOAP `Update` (add fields), REST `PATCH` (toggle sendable) | All work. |

### Working SQL Query create payload (REST)
```json
POST /automation/v1/queries
{
  "name": "MCP Test SQL",
  "key": "<guid>",
  "description": "...",
  "queryText": "SELECT SubscriberKey, EmailAddress, FirstName, LastName, CreatedDate, Rate FROM [MCP TEST]",
  "targetName": "MCP TEST",
  "targetKey": "5C7C2D77-38EB-454D-9DCD-6AFFF046DF39",
  "targetId": "3a183376-6a58-f111-a5bd-5cba2c7ae570",   // DE ObjectID (same as /customobjects/{id})
  "targetDescription": "MCP TEST DE",
  "targetUpdateTypeId": 1,                                // 0=Overwrite, 1=Update, 2=Append
  "categoryId": 644899                                    // a Query Activities folder id
}
```

### Wiring a query into an automation step (REST)
```json
PATCH /automation/v1/automations/{automationId}
{
  "name": "MCP Test",
  "steps": [
    { "name": "Step 1", "step": 1,
      "activities": [
        { "name": "MCP Test SQL",
          "activityObjectId": "<queryDefinitionId>",
          "objectTypeId": 300,
          "displayOrder": 1 } ] } ]
}
```

---

## ❌ Cannot be done via API (verified blockers)

| Task | Blocker (exact error / behavior) |
|---|---|
| **Run / trigger an automation ("Run Once")** | The hard wall. `POST /automation/v1/automations/{id}/actions/start` demands, in sequence: `Steps` → `Activity.Id` → `IsIncludedInRun=true` → then fails with **"The selected automation type: unspecified is not valid to be used in run once."** SOAP `Perform`/`start` → generic `PerformOptionStart` exception. |
| **Set/change an automation's `type`** | `PATCH` with `type: "scheduled"` (and even with a full paused `schedule` object) returns "ok" but **silently discards** the change — `type` stays `unspecified` on read-back. UI sets the type; API cannot. |
| **SQL Update query targeting a *sendable* DE** | `POST /automation/v1/queries` → **"Could not build exclusion text for unknown subscriber field type _Subscriber Key."** Only worked around by temporarily setting the DE non-sendable. |
| **Overwrite query reading + writing the *same* DE** | **"Update Type must be 'Update' or the target data extension cannot appear in the query."** Same-source-and-target requires Update. |
| **Inline SQL inside an Automation object** (SOAP `<Activities><SQLStatement>…`) | Returns HTTP 200 but is **silently discarded** — no real activity is created. (Source of an earlier false "success" — never use this pattern.) |
| **Create SQL activity via SOAP `QueryDefinition`** | Generic `CreateQueryDefinition` exception (ErrorID). Use the REST `/automation/v1/queries` endpoint instead. |

### Workaround used for sendable-DE Update (with caveats)
To create an Update SQL activity whose target is a sendable DE:
1. Capture baseline sendable config (`isSendable`, `sendableCustomObjectField`, `sendableSubscriberField`).
2. `PATCH` the DE `isSendable: false`.
3. `POST /automation/v1/queries` (Update) — now succeeds.
4. Wire into the automation step.
5. `PATCH` the DE back to sendable (re-apply the two sendable fields as **strings**).

Caveats: mutates a production DE; may briefly affect anything using it (e.g. a journey entry source). **Save-time only** — this does not prove the Update runs cleanly at execution time against the (restored) sendable DE.

---

## ⚠️ Cross-cutting API gotchas (these caused most of the debugging pain)

- **SOAP returns HTTP 200 even on failure.** The real outcome is in `<OverallStatus>` / `<StatusMessage>` (and per-result `<StatusCode>`). Always parse the body; never trust the HTTP code alone.
- **OAuth token TTL ~18 min.** An expired token causes **empty-body HTTP 500** on SOAP (looks like a schema error but isn't). Refresh before any multi-step operation.
- **REST automation *list* is unreliable** here (`GET /automation/v1/automations` returned 0). Read by id, or use SOAP `Program` for names.
- **Field lists:** `GET /data/v1/customobjects/{id}/fields` works (parse the `.fields[]` array). SOAP `DataExtensionField` retrieve **500s** when you include certain properties (e.g. `CategoryID`) or a filter.
- **Mirror, don't guess:** for query activities, `GET` an existing one to copy the exact field shape (required `categoryId`, `targetId`, update-type ids) rather than guessing the schema.

---

## Related: Journeys (for context)
- **SOAP does not support journeys** (`Journey`/`Interaction` → HTTP 500 on Create and Retrieve).
- **REST Journey Builder API works** for creation (`POST /interaction/v1/interactions`, full activity graph) — Draft only; activation/run is effectively UI-territory.
- See `JOURNEY-CREATION-FINDINGS.md`.

### Journey entry source = Data Extension (verified working via API)
A proper "Data Extension" entry tile needs a backing **event definition**, then the trigger references it:
1. `POST /interaction/v1/eventDefinitions` with `type:"EmailAudience"`, `eventDefinitionKey:"DEAudience-{guid}"`, `dataExtensionId`, `dataExtensionName`, `sourceApplicationExtensionId:"97e942ee-6914-4d3d-9e52-37ecb71f79ed"` (standard DE-entry app id), `arguments:{serializedObjectType:3, eventDefinitionKey, dataExtensionId, criteria:"", useHighWatermark:false}`.
2. Trigger: `type:"EmailAudience"`, `metaData.title:"Data Extension"`, `iconUrl:"/images/icon-data-extension.svg"`, `entrySourceGroupConfigUrl:"jb:///data/entry/audience/entrysourcegroupconfig.json"`, and `metaData.eventDefinitionId`/`eventDefinitionKey` pointing at the event definition.
- `entryMode` is a top-level enum **string** — valid tokens (from existing journeys): `NotSet`, `OnceAndDone`, `MultipleEntries`, `SingleEntryAcrossAllVersions`. There is **no** "re-entry only after exit" token; invalid values throw `JSON Deserialization Exception (10004)`. Set it via PUT (GET→modify→PUT round-trips cleanly).

### Decision Split (MultiCriteriaDecision) — how to make it OPEN in the UI
This took several iterations. An API-built decision split only opens in the UI when it matches the real structure **exactly**:
- `type` = **`MULTICRITERIADECISION`** (not `MultiCriteriaDecision`), key like `MULTICRITERIADECISIONV2-1`
- `metaData.isConfigured` = `true`, `configurationArguments.schemaVersionId` = `252`
- Criteria lives in **`configurationArguments.criteria`** as an object keyed by outcome key → a **`FilterDefinition`** XML string (NOT in `outcomes[].arguments`, NOT `RuleEnvelope`):
  `<FilterDefinition><ConditionSet Operator="AND" ConditionSetName="Individual Filter Grouping"><Condition IsEphemeralAttribute="true" Key="Event.{eventDefinitionKey}.{FieldName}" Operator="BeginsWith" UiMetaData="{}"><Value><![CDATA[H]]></Value></Condition></ConditionSet></FilterDefinition>`
- The field is referenced via the **entry event**: `Key="Event.DEAudience-{key}.FirstName"` — using the journey's own entry event-definition key.
- Outcomes: `default_path_1` (+ optional guid-keyed middle branches) each with `metaData.label`, `criteriaDescription`, `invalid:false`.
- **Remainder/"otherwise" → exit is NOT expressible via API:** an outcome with `next:""` is rejected (`errorcode 121000`). Real UI splits always wire the remainder to a real activity. For exit, **omit the remainder outcome** — non-matchers exit implicitly.

**KNOWN LIMITATION (last mile) — confirmed by diagnostic:** even with the correct structure, the decision tile *opens* but the **rule contents do not render** in the builder (condition row is blank). Diagnostics:
- Ruled OUT the operator: switching to a known-good operator (`Equal`, mirroring a working decision exactly) still rendered blank.
- No schema-registration API: `eventDefinitions/{key}/schema`, `interactions/{id}/schema`, `contacts/v1/schema/dataExtensions/{id}` all 404.
- Root cause: the field (`Event.DEAudience-{key}.Field`) only resolves once the entry DE's attributes are **registered**, which happens when the journey/entry event is **published** (working event defs carry `interactionCount` + `automationId`; an API-built Draft does not). Publishing is blocked by placeholder emails (no real assets).

**RESOLVED (2026-06-01) — it IS fully API-doable.** Diffing my API-built decision against the same tile after the user configured it in the UI showed the activity JSON was **byte-for-byte identical** — my structure was always correct. The missing piece was a **journey-level flag**, not publishing:
```json
"metaData": { "dataSource": "ContactsModel", "disableEndWait": true }
```
- Before: journey `metaData` was `{}` → tile opens but condition renders **blank** (field unresolved).
- After setting `dataSource:"ContactsModel"` → the field resolves and the rule displays.

So the earlier "needs publish / UI-only" conclusion was WRONG. **To build a fully-working decision split via API: produce the correct activity structure AND set the journey's `metaData.dataSource = "ContactsModel"`.** No publishing or UI field-picker required. Full recipe in memory: `journey-decision-split-recipe.md`.

---

## Reference IDs (this tenant)
- **MCP Test** automation — id `cb5b294b-5a07-4f52-8a06-4c32cb72cb94`, key `510e4095-700a-4bdf-8f2d-0f9776850cc1` (has SQL activity in Step 1; not runnable via API)
- **MCP Test SQL** query activity — id `b79260a1-9c81-4508-bac7-9d5276d497e5` (Update on MCP TEST)
- **MCP TEST** DE — id `3a183376-6a58-f111-a5bd-5cba2c7ae570`, key `5C7C2D77-38EB-454D-9DCD-6AFFF046DF39`, sendable (SubscriberKey → "Subscriber Key")
- Query Activities folder categoryId: `644899`
