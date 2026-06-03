# MCP Onboarding Campaign — Build Summary & UI Runbook

**Brand:** Northbridge Financial (fictional Financial Services)
**Built:** 2026-06-03 · via SFMC REST API · Business Unit EID 7281705
**Status:** All components built & verified. Two actions remain — both **UI-only** (see Part B).

---

## Part A — What was built (all verified live)

### Phase 1 — Seed data (MCP TEST Data Extension)
5 synthetic FS customer records inserted (verified via SOAP Retrieve). `example.com` addresses = guaranteed non-real.

| SubscriberKey | First | Last | Email | Rate |
|---|---|---|---|---|
| NB-1001 | Aisha | Rahman | aisha.rahman@example.com | 3.25% |
| NB-1002 | Daniel | Okafor | daniel.okafor@example.com | 4.10% |
| NB-1003 | Mei Lin | Chen | meilin.chen@example.com | 2.95% |
| NB-1004 | Carlos | Mendez | carlos.mendez@example.com | 5.00% |
| NB-1005 | Sophie | Turner | sophie.turner@example.com | 3.75% |

### Phase 2 — Automation Studio filter
- **Target DE:** `MCP Onboarding Audience` — id `d5d78e2a-585f-f111-a5bd-5cba2c7ae570`, key `1d2c097d-9eb1-4920-9253-8db91f6812d7` (**non-sendable** by design — see note 1).
- **SQL activity:** `MCP Onboarding - Filter Engaged` — id `90e03fa0-d9e5-4719-8198-5d5bd8b4ded6`. Overwrite. Logic: select MCP TEST records in the onboarding cohort (`SubscriberKey LIKE 'NB-%'`) **and** that did **NOT open or click** in the last 30 days → write to the audience DE. (The `NB-%` guard keeps stray/test rows like `TEST_0001` out of the audience on every Run Once.)
- **Automation:** `MCP Onboarding Prep` — id `afa2f5fa-4a61-494f-a93b-08ec3bfdf1ca`, status **Ready**, Step 1 = the SQL activity (verified via read-back).

### Phase 3 — Email assets (Content Builder → folder `MCP Onboarding`, id 950113)
| Email | Asset id | Role |
|---|---|---|
| NB Onboarding - 1 Welcome | 1461876 | Touchpoint 1 (all entrants) |
| NB Onboarding - 2 Getting Started | 1461877 | Engaged branch |
| NB Onboarding - 3 Nudge | 1461878 | Not-engaged branch |

All three: original copy, navy/gold Northbridge brand, `FirstName` AMPScript personalization with `"there"` fallback.

### Phase 3 — Journey (Draft)
`MCP Onboarding Journey` — id `34132867-c39a-49e4-a1ff-ffbbf5a9a1dc`, key `mcp-onboarding-d6901d66`.
Entry event definition: `DEAudience-e38dc154-c959-4a50-acd4-a6a2511e7d41`.

```
[Entry: Data Extension = MCP Onboarding Audience]
   │
   ▼
1 Welcome ──► Wait 2 days ──► Engagement Split: clicked Welcome's "Activate" link?
                                  ├─ YES (engaged) ───────► 2 Getting Started ─► end
                                  └─ NO (not engaged) ─► Wait 2 days ─► 3 Nudge ─► end
```
Journey-level `metaData.dataSource = "ContactsModel"` is set (this is what makes split fields resolve in the builder).

---

## Part B — Actions you must do in the UI (cannot be done via API)

> **Why:** the SFMC API can *build* automations and journeys but cannot *run* an automation ("Run Once" is UI-only) or *activate/publish* a journey. These are platform limitations, verified on this tenant.

### Step 1 — Run the automation to populate the audience
1. **Automation Studio** → open **MCP Onboarding Prep**.
2. Click **Run Once** (top-right) → confirm. *(Or attach a Schedule if you want it recurring.)*
3. Wait for the green success tick on Step 1.

### Step 2 — Verify the audience populated
1. **Email Studio → Email → Subscribers → Data Extensions** → open **MCP Onboarding Audience**.
2. Check **Records** — expect all 5 NB-#### contacts (none have opened/clicked yet, so none are excluded). Spot-check `EmailAddress` / `FirstName` are present.

### Step 3 — Review & activate the journey
1. **Journey Builder** → open **MCP Onboarding Journey** (Draft).
2. Confirm the **Entry Source** tile shows **MCP Onboarding Audience** and set the **entry/contact evaluation** options you want (re-entry is currently *OnceAndDone*).
3. Open each **Email** tile and confirm the **Send Classification**, **Sender Profile**, and **Delivery Profile** (I left these for you — JB requires a Send Classification before activation; assets + subject/preheader are already wired).
4. Open the **Engagement Split** tile — see note 2.
5. Click **Validate**, then **Activate**.

---

## Notes, decisions & options

**1. Why the audience DE is non-sendable.** A SQL activity that targets a *sendable* DE throws *"could not build exclusion text for unknown subscriber field type Subscriber Key."* Non-sendable avoids this and still works as a journey entry source (the journey resolves email via the contact model + the DE's `EmailAddress`).

**2. Engagement Split metric.** I mirrored a known-good split: `statsTypeId 3` = **clicked**, evaluating the Welcome email's *Activate* link (`https://www.example.com/activate`). To branch on **opened** instead, open the tile in the UI and switch the metric to *Opened* (or keep click-based and update the tracked URL to your real CTA). Verify the tile reads correctly before activating.

**3. Exclusion rule — you chose "opened/clicked OR active".** The built SQL implements the reliable, always-runnable half: **opened OR clicked in the last 30 days** (against the `_Open` / `_Click` system data views, which always exist). To *also* exclude contacts **currently active** in the MCP Test journey, your account must expose Journey Builder data views (`_JourneyActivity` / `_Journey`). If it does, swap the SQL for the augmented version below and re-save the activity:

```sql
SELECT m.SubscriberKey, m.EmailAddress, m.FirstName, m.LastName, m.Rate, m.CreatedDate
FROM [MCP TEST] m
WHERE NOT EXISTS (SELECT 1 FROM [_Open]  o WHERE o.SubscriberKey = m.SubscriberKey AND o.EventDate > DATEADD(DAY,-30,GETDATE()))
  AND NOT EXISTS (SELECT 1 FROM [_Click] c WHERE c.SubscriberKey = m.SubscriberKey AND c.EventDate > DATEADD(DAY,-30,GETDATE()))
  AND NOT EXISTS (
      SELECT 1 FROM [_JourneyActivity] ja
        JOIN [_Journey] j ON ja.VersionID = j.VersionID
      WHERE ja.SubscriberKey = m.SubscriberKey
        AND j.JourneyName = 'MCP Test'
  );
```
*(Note: engagement here is account-wide open/click. To scope strictly to MCP Test sends, join `_Open`/`_Click` → `_Sent` → the journey's send JobIDs.)*

**4. Scope check (token-efficiency).** All writes used the live REST API directly (the `salesforce-mcp` MCP tools could not load mid-session); same endpoints, same result.

---

## Reference IDs
| Object | id / key |
|---|---|
| MCP TEST DE | `3a183376-6a58-f111-a5bd-5cba2c7ae570` / `5C7C2D77-38EB-454D-9DCD-6AFFF046DF39` |
| MCP Onboarding Audience DE | `d5d78e2a-585f-f111-a5bd-5cba2c7ae570` / `1d2c097d-9eb1-4920-9253-8db91f6812d7` |
| SQL activity | `90e03fa0-d9e5-4719-8198-5d5bd8b4ded6` |
| Automation | `afa2f5fa-4a61-494f-a93b-08ec3bfdf1ca` |
| Email assets | Welcome 1461876 · Getting Started 1461877 · Nudge 1461878 |
| Journey | `34132867-c39a-49e4-a1ff-ffbbf5a9a1dc` / `mcp-onboarding-d6901d66` |
| Entry event def | `DEAudience-e38dc154-c959-4a50-acd4-a6a2511e7d41` |
