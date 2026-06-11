# Campaign Design Notes & Lessons — Northbridge Onboarding (POC)

A worked, end-to-end onboarding campaign built programmatically in SFMC via the REST/SOAP APIs, plus the lessons that make the *next* one faster. Operational detail and all object IDs live in **`MCP-Onboarding-Campaign-Runbook.md`**; this file is the design rationale + reusable knowledge.

Brand: *Northbridge Financial* (fictional Financial Services). Channel: email only. Built 2026-06-03.

---

## 1. What was built

| Phase | Component | Notes |
|---|---|---|
| 1 | 5 synthetic FS records → `MCP TEST` DE | `example.com` addresses (guaranteed non-real); verified via SOAP Retrieve |
| 2 | `MCP Onboarding Audience` DE (non-sendable) | journey entry target |
| 2 | `MCP Onboarding - Filter Engaged` SQL + `MCP Onboarding Prep` automation | Overwrite; cohort + engagement filter |
| 3 | 3 Content Builder emails (Welcome / Getting Started / Nudge) | original copy, `FirstName` AMPScript w/ fallback |
| 3 | `MCP Onboarding Journey` (Draft) | DE entry → email → wait → engagement split → branch emails |

**Journey flow**
```
Entry (MCP Onboarding Audience)
  → Welcome → Wait 2d → Engagement Split (clicked Welcome CTA?)
        ├─ engaged ────► Getting Started
        └─ not engaged ─► Wait 2d → Nudge
```

---

## 2. Design decisions (and why)

- **Non-sendable audience DE.** A SQL activity targeting a *sendable* DE fails with *"could not build exclusion text for unknown subscriber field type Subscriber Key."* A non-sendable target avoids it and still works as a journey entry source (email resolves via the contact model + the DE's `EmailAddress`).
- **DE-entry event definition + `metaData.dataSource = "ContactsModel"`.** Both are required for a Data Extension entry source whose fields resolve in decision tiles. Without `ContactsModel`, split tiles open but render blank.
- **Engagement split = click-based on the Welcome CTA.** Mirrored a known-good `ENGAGEMENTDECISION` tile (`statsTypeId 3` + `engagementUrls`) for reliability; switch to "Opened" in the UI if preferred.
- **`SubscriberKey LIKE 'NB-%'` cohort guard** in the SQL keeps stray/test rows out of the audience on every Overwrite run.
- **`OnceAndDone` entry mode** — onboarding should fire once per contact.

---

## 3. Lessons learned — API capabilities vs. limits

**Build via API ✅ / Run via API ❌.** You can fully *build* automations and journeys through the API, but two things are **UI-only** on this tenant:
- **Running an automation** ("Run Once") — `POST …/automations/{id}/actions/start` ends in *"automation type: unspecified is not valid to be used in run once."* Even SOAP `Perform` (start) on the underlying `QueryDefinition` returns `InvalidRequest` (`Schedule::Start`). Both run-paths are UI-only.
- **Activating / publishing a journey** — API creates Draft only.

**Other verified gotchas**
- **SQL Update / exclusion against a *sendable* DE is blocked** → target a non-sendable DE.
- **Journey `description` rejects** any of `& < > " ' /` (errorcode 121072 — e.g. `w/` fails).
- **DE rows:** insert via `POST /hub/v1/dataevents/key:{deKey}/rowset`; **read back via SOAP `Retrieve`** (`DataExtensionObject[Name]`) — the REST `/data/v1/customobjects/{id}/rowset` GET is **404**.
- **SOAP returns HTTP 200 even on failure** — always parse `<OverallStatus>`.
- **"Mirror, don't guess"** — for any JB activity JSON (email, wait, split), copy the shape from an existing live journey rather than constructing blind.

---

## 4. Token / connectivity mechanics

- The MCP OAuth refresh writes an **`mcpt-` bridge token** that is valid for the MCP server but **401s against the raw REST API**. The REST-usable platform JWT is the **4-segment** token embedded inside that wrapper (`header.payload.signature.encrypted`) — all four segments are required (a 3-segment slice parses but 401s).
- **Token TTL ≈ 18 min.** Refresh immediately before a build and batch all writes inside the window.
- **MCP tools bind at session start.** A mid-session reconnect does not surface them, so REST-direct (same endpoints) is the reliable in-session path.
- The refresh script must resolve the **current** `claude.exe` version dynamically (a hard-coded version path silently breaks persistence on every CLI update).

---

## 5. Reusable assets produced

- **`SFMC-Campaign-Brief-Template.xlsx`** — fill-in brief that maps 1:1 to these objects (dropdowns, required markers, worked Northbridge example).
- **`MCP-Onboarding-Campaign-Runbook.md`** — operational record + UI run/activate steps + all object IDs.
- **`AUTOMATION-API-CAPABILITIES.md`** — the full verified Automation Studio API reference.
- Verified JB JSON shapes (entry trigger, `EMAILV2`, `WAIT`, `ENGAGEMENTDECISION`, journey envelope) — skip recon next time.

---

## 6. Repeatable workflow

1. **Brief** — fill `SFMC-Campaign-Brief-Template.xlsx`.
2. **Map** — translate each tab to SFMC objects; confirm before building.
3. **Build + verify** — DEs → data → SQL/automation → emails → journey, each read-back-verified.
4. **Hand off** — runbook with the two UI-only steps (Run Once + Activate).
