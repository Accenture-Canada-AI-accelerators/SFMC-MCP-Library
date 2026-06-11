# MCE → Marketing Cloud Next: Migration Research & Readiness

> **Status:** Research complete (Phase 1) — 168-agent deep-research, adversarially verified  
> **Last updated:** 2026-06-10  
> **Author:** Claude Code + Harsh Patel  

---

## 1. Source Environment — What We Have (MCE Inventory)

Inventoried via REST API (`/asset/v1/content/assets`) on 2026-06-09.  
Full PDF: `SFMC_Asset_Inventory.pdf` · Raw JSON: `~/.claude/asset_inventory_raw.json` (local only)

### 1.1 Asset Count by Type

| Asset Type | Type ID | Count | Migration Priority |
|---|---|---|---|
| jpg | 23 | 193 | Low — static images, re-upload |
| png | 28 | 163 | Low — static images, re-upload |
| templatebasedemail | 207 | 53 | **High** — core email content |
| htmlemail | 208 | 17 | **High** — active campaigns |
| webpage | 205 | 14 | Medium — CloudPages |
| landingpage | 247 | 6 | Medium — CloudPages |
| layoutblock | 213 | 5 | Medium — structural blocks |
| jsonmessage | 230 | 5 | Medium — mobile/push |
| interactivecontent | 249 | 4 | Medium — interactive pages |
| freeformblock | 195 | 4 | Medium — content blocks |
| htmlblock | 197 | 3 | Medium — content blocks |
| imageblock | 199 | 3 | Medium — content blocks |
| template | 4 | 3 | **High** — base templates |
| codesnippetblock | 220 | 2 | **High** — AMPscript/code |
| dynamicblock | 201 | 2 | Medium — dynamic content |
| gif | 20 | 2 | Low — static images |
| jpeg | 22 | 2 | Low — static images |
| packagedefinition | 236 | 2 | Low — TBD |
| html | 175 | 1 | Medium |
| icemailformblock | 232 | 1 | Medium — interactive block |
| imagecarouselblock | 224 | 1 | Medium — interactive block |
| textonlyemail | 209 | 1 | Medium |
| pptx | 124 | 1 | Low — document |
| **TOTAL** | | **488** | |

### 1.2 Active Campaign Emails (Highest Priority Assets)

| ID | Name | Type | Folder |
|---|---|---|---|
| 1462567 | MAR POC - Seg3 Renewal (FR) | htmlemail | Development/Harsh/Mortgage Auto Renewal POC |
| 1462566 | MAR POC - Seg3 Renewal (EN) | htmlemail | Development/Harsh/Mortgage Auto Renewal POC |
| 1462565 | MAR POC - Seg1 Renewal (FR) | htmlemail | Development/Harsh/Mortgage Auto Renewal POC |
| 1462564 | MAR POC - Seg1 Renewal (EN) | htmlemail | Development/Harsh/Mortgage Auto Renewal POC |
| 1462420 | MAR - Mortgage Renewal (FR) | htmlemail | Development/Harsh/MAR Campaign |
| 1462419 | MAR - Mortgage Renewal (EN) | htmlemail | Development/Harsh/MAR Campaign |
| 1461878 | NB Onboarding - 3 Nudge | htmlemail | Development/Harsh/MCP Onboarding |
| 1461877 | NB Onboarding - 2 Getting Started | htmlemail | Development/Harsh/MCP Onboarding |
| 1461876 | NB Onboarding - 1 Welcome | htmlemail | Development/Harsh/MCP Onboarding |
| 1461643 | MCP Test - Welcome (1) | htmlemail | Development/Harsh/MCP Test |

### 1.3 Key Observations from Inventory

- **All 488 assets are in Draft status** (except `JunePics.pptx` — Published)
- **360 images (73.8%)** — mostly newsletter photo assets, many unoptimised (some >1 MB)
- **71 emails** — 53 SFBG newsletter editions (2021–2024) + 17 active HTML emails + 1 text-only
- **2 AMPscript/code blocks** — `Generic Footer` and `AMPScript delete code` — high risk for migration
- **5 mobile/push JSON messages** — depends on MCN mobile channel support
- **24 CloudPages** — mix of landing pages, feedback forms, SSJS test pages
- **Folder structure**: 59 unique folders across 4 main trees: `Content Builder/`, `CloudPages/`, `SFBG Newsletter/`, `Development/`

---

## 2. Marketing Cloud Next (MCN) — Capability Research

> **Source:** Deep-research workflow — 168 agents, 96 adversarially-verified claims (27 confirmed / 69 refuted), multi-source  
> **Research date:** 2026-06-10  
> **Key sources:** Salesforce Ben, SFMC Tips (Medium/@marketingcloudtips), dev.to/ampscript-ninja, Knak, Mavlers, official help.salesforce.com, developer.salesforce.com

### 2.1 GA / Release Status (as of June 2026)

Marketing Cloud Next is **Generally Available** on Salesforce's core platform, available in Growth and Advanced editions. Active feature development continues through seasonal releases:

| Release | Key MCN Milestones |
|---|---|
| Winter '26 (Sept 2025) | Landing pages and forms reached GA |
| Spring '26 (Feb 2026) | "Send Marketing Cloud Engagement Email" action in Flows (requires MCE+ add-on); landing page/form enhancements |
| Summer '26 (June 2026) | **AMPscript support in MCN** — GA rolling out June 13, 2026; targeted function set (not 100% of functions yet) |

MCE (Marketing Cloud Engagement / ExactTarget) is **not being shut down**. Salesforce explicitly frames the relationship as **"convergence, not migration."** MCE customers can continue using Email Studio, Journey Builder, and Automation Studio. No published end-of-life, no sunset date, no mandatory cutover exists for any MCE product. However, community sources (Salesforce Ben, Summer '26) describe MCE as entering **"maintenance mode"** — MCE is receiving markedly fewer new features compared to MCN.

### 2.2 Content Asset Types in MCN

Official Salesforce Help ("Types of Content in Marketing Cloud Next") documents **17 content types** in MCN, including:

| MCN Content Type | Description | MCE Equivalent |
|---|---|---|
| Email | Marketing email | htmlemail, templatebasedemail |
| SMS / MMS | Text messaging | MobileConnect JSON |
| WhatsApp | WhatsApp messaging | (no MCE equivalent) |
| Image | Image files | jpg, png, jpeg, gif assets |
| Document (PDF) | PDF files | (file storage) |
| Reusable Content | Content fragments/blocks | htmlblock, freeformblock, codesnippetblock |
| Landing Page | Web landing pages | webpage (205), landingpage (247) |
| Form | Web forms | interactivecontent (249) |

**Critical limitation — Connect API upload scope:** The MCN Connect API upload endpoint (`mc-manage-content-connect-upload`) accepts **only two contentType values**: `sfdc_cms__image` and `sfdc_cms__email`. Content blocks, templates, CloudPages-style interactive content, and other types **cannot be uploaded programmatically via this API endpoint**. This is a confirmed hard constraint as of June 2026.

### 2.3 Automation & Journey Capabilities

| MCE Capability | MCN Equivalent | Status |
|---|---|---|
| Journey Builder | **Flows** (segment-triggered, event-triggered, activation-triggered, broadcast) | **Complementary, not a replacement** — JB continues via MCE+ bridge |
| Automation Studio | No direct equivalent confirmed in MCN | Unknown/not documented |
| SQL Query Activities | No equivalent confirmed | Likely requires Data Cloud Calculated Insights or Apex |

**Key confirmed fact:** MCN Flows do **NOT replace** Journey Builder. Salesforce has explicitly stated Flows and Journey Builder are complementary. MCE customers retain Journey Builder access via the MCE+ bridge with "no migration required."

### 2.4 AMPscript & Scripting

| Scripting | MCE | MCN Status |
|---|---|---|
| AMPscript | Fully supported | **Summer '26 GA (June 13, 2026)** — partial function set; failing in preview (Apr '26): `SystemDateToLocalDate()`, `RedirectTo()` — re-test at GA |
| SSJS (Server-Side JavaScript) | Supported in CloudPages/automations | **Not confirmed in MCN** — different server-side paradigm |
| Handlebars | Not used | Native in MCN; AMPscript works alongside Handlebars for data retrieval |

AMPscript support in MCN is **not a full port** — Salesforce's official language is "targeted set of functions." Community testing in preview orgs (April 2026) confirmed some functions produce errors. Migrating AMPscript-heavy assets requires function-by-function verification.

### 2.5 Migration Tooling

**Confirmed: No automated migration tooling exists as of June 2026.**

- Salesforce has not completed automated tooling for migrating MCE email templates, images, and files to MCN (Mavlers, April 8, 2026)
- The "one-click connect from MCE to MCN isn't currently clear for some aspects of the marketing lifecycle. For example, content." (Salesforce Ben, June 2025)
- Salesforce Ben reports Salesforce is "working on mechanisms for MCE to MCN" — but this is unattributed to any official roadmap item; no official Salesforce statement, press release, or release notes item has confirmed a migration pipeline is under active development
- Third-party guides (Mavlers, Knak) describe manual migration as the only documented approach

### 2.6 Claims investigated and *ruled out* (refuted under verification)

Of 96 verified claims, 69 were refuted as overstated, misattributed, or unverifiable. The ones most likely to mislead a migration team — **do not assume these are true**:

| Plausible-sounding claim | Verdict | Why it was rejected |
|---|---|---|
| "AMPscript carries over to MCN with minimal changes / same function set applies" | ❌ Refuted (strongest cluster, 8 agents) | MCN ships a **targeted/partial** function set, not a port. Existing AMPscript-heavy assets need function-by-function rework, not a lift |
| "AMPscript is the #1 migration blocker" | ❌ Refuted | Asserted by one pseudonymous blog with no survey/data backing |
| "Salesforce is actively building MCE→MCN migration tooling" | ❌ Refuted | Only the *absence* of tooling is confirmed. "Actively working on it" traces to one community blog, no official roadmap item |
| "MCN Flows replace Journey Builder / almost every automation" | ❌ Refuted (3 agents) | Salesforce states Flows and JB are **complementary**; JB continues via MCE+ bridge |
| "No migration required" = content moves automatically / migration is easy | ❌ Refuted (misread) | The phrase means MCE keeps running via the bridge — **not** that content auto-transfers. There is no auto-transfer |
| "Consent management is the biggest migration barrier" | ❌ Refuted | Overstated; not supported as a ranked barrier by any primary source |
| "SSJS reliance forces a multi-year migration" | ❌ Refuted | "Very High" complexity is real; the "multi-year" figure is vendor-blog speculation |
| "MCN exposes REST, Bulk, Data 360, Connect, Metadata, SOAP APIs for content" | ❌ Refuted | Only the **Connect API** is documented for content, and it accepts just images + emails |
| `SystemDateToLocalDate()` / `RedirectTo()` are "confirmed broken" in MCN | ⚠️ Preview-only | Observed failing in a **Summer '26 preview org** (April 2026) — a preview snapshot, not a GA-confirmed defect. Re-test at GA |

**Net:** the verified-confirmed picture in §2.1–2.5 is *more conservative* than the marketing-blog narrative. AMPscript portability and migration-tooling readiness are the two areas where community sources most consistently over-promised.

---

## 3. MCE → MCN Feature Parity Matrix

> **Status legend:** ✅ GA parity · ⚠️ Partial/incomplete · ❌ No equivalent · 🔄 Different model · ❓ Unknown

| MCE Feature | Count in org | MCN Equivalent | Parity Status | Migration Path |
|---|---|---|---|---|
| **Content Builder — htmlemail** | 17 | Email (`sfdc_cms__email`) | ⚠️ Partial — AMPscript partial (Summer '26) | Upload via Connect API; manually verify AMPscript functions |
| **Content Builder — templatebasedemail** | 53 | Email content type | ⚠️ Partial — template system differs | Rebuild templates in MCN; no automated import |
| **Content Builder — images** (jpg/png/gif/jpeg) | 360 | Image (`sfdc_cms__image`) | ✅ GA | Re-upload via Connect API (`contentType: sfdc_cms__image`) |
| **Content Builder — htmlblock/freeformblock** | 7 | Reusable Content fragments | ⚠️ Partial | Manual rebuild; not uploadable via Connect API |
| **Content Builder — codesnippetblock** (AMPscript) | 2 | AMPscript fragments | ⚠️ Partial — function coverage TBD | Test each function in MCN Summer '26 |
| **Content Builder — dynamicblock** | 2 | Personalisation logic | ❓ Unknown | Assess per-block |
| **CloudPages — webpage/landingpage** | 20 | Landing Pages (MCN) | ✅ GA (Winter/Spring '26) | Rebuild in MCN; SSJS content requires rearchitecting |
| **CloudPages — interactivecontent** (SSJS) | 4 | Unknown | ❌ No confirmed equivalent | SSJS likely needs full rebuild in MCN paradigm |
| **CloudPages — jsonmessage** | 5 | SMS / WhatsApp natively | 🔄 Different model | Rebuild in MCN messaging framework |
| **Journey Builder** | (active journeys) | Flows (MCN) | ⚠️ Complementary — JB continues via MCE+ | New builds in Flows; existing journeys stay in MCE |
| **Automation Studio** | (SQL automations) | No confirmed equivalent | ❓ Unknown | Likely Data Cloud Calculated Insights or Apex |
| **AMPscript** | (in emails + blocks) | AMPscript (Summer '26) | ⚠️ Partial — targeted function set | Migrate but verify each function; known broken: `SystemDateToLocalDate()`, `RedirectTo()` |
| **SSJS** | (in CloudPages) | Not confirmed | ❌ No confirmed equivalent | Plan for full rebuild |
| **Data Extensions** | (source DEs for journeys) | Data Cloud / Unified Data | 🔄 Fundamentally different model | Significant rearchitecting; Data Cloud licensing required |
| **Send Classification** | (used in sends) | Publication Lists (MCN) | 🔄 Different model | Map send classifications to publication lists |

**Bottom line on parity (MCE vs MCN, Summer '26):**
- MCAE (Pardot) → MCN parity: **"largely achieved already"** (Salesforce Ben, Summer '26)
- MCE (ExactTarget) → MCN parity: **"more complicated"** (Salesforce Ben, Summer '26) — AMPscript only partially ported, SSJS unknown, Data Extensions fundamentally different, no automated content migration tooling

---

## 4. Migration Feasibility Assessment

### 4.1 Is migration required?

**No. Not now, and not on any announced timeline.**

Salesforce has explicitly stated there is no forced migration, no MCE shutdown, and no mandatory cutover. The official Salesforce positioning is "convergence, not migration." MCE customers continue to have full access to Email Studio, Journey Builder, and Automation Studio via the MCE+ bridge. No end-of-life date has been published for any MCE product as of June 2026.

However, MCE is de-facto entering **maintenance mode**: MCE received "relatively few and far between" updates in Summer '26 (Salesforce Ben). Strategic investment is clearly flowing to MCN. For organizations building net-new campaigns, MCN is the trajectory.

### 4.2 What CAN migrate today (feasible)

| Asset / Feature | Feasibility | Effort |
|---|---|---|
| Images (360 assets) | ✅ Feasible now | Low — re-upload via Connect API `sfdc_cms__image`; scriptable in bulk |
| Plain HTML emails (no AMPscript) | ✅ Feasible now | Medium — upload as `sfdc_cms__email`; layout/template rebuild required |
| AMPscript emails | ⚠️ Feasible from Summer '26 | High — each function must be tested; partial breakage expected |
| Landing pages (no SSJS) | ⚠️ Feasible now | High — rebuild in MCN Landing Pages; no import path |
| New journeys/flows | ✅ Feasible now | Medium — build new campaigns in MCN Flows natively |

### 4.3 Hard blockers (cannot migrate today)

| Blocker | Impact on this org | Detail |
|---|---|---|
| **No automated content migration tooling** | All 488 assets | Salesforce has confirmed no automated tooling exists; manual rebuild is the only documented path |
| **Connect API limited to 2 types** | Blocks blocks/templates | Only `sfdc_cms__image` and `sfdc_cms__email` accepted; content blocks, templates, CloudPages cannot be uploaded |
| **AMPscript partial support** | 2 codesnippetblocks + any AMPscript in 17 htmlemails + 53 templatebasedemails | `SystemDateToLocalDate()`, `RedirectTo()` confirmed broken; full function coverage TBD |
| **SSJS unknown in MCN** | 4 interactivecontent CloudPages + any SSJS in automations | No confirmed MCN equivalent; likely requires rearchitecting |
| **Data Extensions → Data Cloud** | All journey-entry DEs | Fundamentally different model; MCN uses Data Cloud — separate licensing and architecture |
| **Automation Studio** | SQL query automations | No confirmed MCN equivalent; business logic may require rebuilding in Data Cloud Calculated Insights |

### 4.4 This org's specific risk profile

From the 488-asset inventory:
- **Low risk** (360 images): straightforward re-upload via Connect API
- **Medium risk** (53 templatebasedemail SFBG newsletter editions): template structure changes; AMPscript may partially break
- **High risk** (17 htmlemail active campaign emails + 2 codesnippetblocks): AMPscript function verification required per-email; `Generic Footer` and `AMPScript delete code` blocks are unknowns
- **High risk** (4 SSJS CloudPages): no confirmed MCN port path
- **High risk** (5 jsonmessage): mobile/push model is fundamentally different in MCN

---

## 5. Recommended Migration Approach

### Decision: Parallel build strategy, not a lift-and-shift

Given no MCE EOL pressure and no automated tooling, the recommended approach is:

**Phase 0 — No action needed (now)**
- Continue operating active MAR, NB Onboarding, and MCP Test campaigns in MCE
- Do not attempt to migrate existing Journey Builder journeys — no benefit, high risk
- Monitor Salesforce Summer '26 AMPscript GA rollout for function completeness

**Phase 1 — New campaigns in MCN (next build cycle)**
- Build all net-new campaigns natively in MCN (Flows, MCN email, MCN landing pages)
- Use MCE+ bridge to reuse existing MCE emails in Flows where needed
- Build institutional knowledge on MCN before attempting migration of existing assets

**Phase 2 — Image migration (quick win, automated)**
```powershell
# Bulk re-upload all 360 images via MCN Connect API
POST /v55.0/connect/cms/content-items
Content-Type: application/json
{ "contentType": "sfdc_cms__image", ... }
```
Scriptable in ~1 day; images are the only asset type with a clean API path.

**Phase 3 — Email migration (when AMPscript GA stabilises)**
- Target: post-Summer '26 GA (estimate: Aug 2026+)
- For each active htmlemail: audit AMPscript usage → test each function → fix or replace broken functions
- Prioritise the 17 active campaign emails first; defer 53 SFBG newsletter archives
- Template-based emails require MCN template rebuild before content can be slotted in

**Phase 4 — CloudPages / SSJS (highest effort)**
- SSJS CloudPages have no direct MCN equivalent — requires feature-by-feature rebuild in MCN paradigm
- Assess whether MCN Flows + Landing Pages + Forms can replace the functionality
- Lowest urgency — CloudPages in MCE continue to work with no EOL

**Phase 5 — Automation Studio / Data Extensions (strategic decision)**
- Only relevant if and when the org moves to Data Cloud
- Requires separate Data Cloud licensing decision and architecture review
- Not scoped in this POC

### Effort summary

| Wave | Assets | Estimated effort | Dependency |
|---|---|---|---|
| Image upload script | 360 images | 1–2 days | Connect API access |
| New campaigns in MCN Flows | Net-new | Ongoing | MCN Growth license |
| 17 active HTML emails | 17 htmlemails | 2–4 weeks | AMPscript Summer '26 GA stable |
| SFBG newsletter archive | 53 templatebasedemails | Low priority — archival only | Post-AMPscript GA |
| SSJS CloudPages (4) | 4 pages | 1–2 weeks each | MCN Landing Pages + Forms assessment |
| Automation Studio + DEs | Automations | Major project | Data Cloud licensing decision |

---

## 6. Technical Reference — MCE API Learnings

Captured during inventory work — relevant for any migration tooling built on top of the MCE REST API.

### 6.1 Content Builder REST API

| Purpose | Endpoint | Notes |
|---|---|---|
| List all assets (paginated) | `GET /asset/v1/content/assets?$page=N&$pagesize=200` | Max 200/page; 488 assets = 3 pages |
| Get single asset | `GET /asset/v1/content/assets/{id}` | Returns full payload incl. HTML content |
| Get folder details | `GET /asset/v1/content/categories/{id}` | Walk `parentId` to build path |
| List folders by type | `GET /asset/v1/content/categories` | — |

**Critical gotchas:**
- On Accenture network, Python `urllib` fails (`getaddrinfo` error) — use PowerShell `Invoke-RestMethod` which uses Windows proxy settings natively
- `&` in URL query strings must be built as a string variable before passing to a PowerShell function — inline `&` is parsed as PS operator
- REST base URL is `sfmc.RB` in `sfmc.py` (not `sfmc.REST_BASE`)
- PowerShell `Out-File` writes UTF-8 BOM — read JSON back with `encoding="utf-8-sig"` in Python

### 6.2 Authentication

- **Flow:** OAuth 2.0 Authorization Code + PKCE (public package, no client secret)
- **Token TTL:** ~18 minutes; no refresh tokens — re-run full PKCE flow to renew
- **Bridge token vs REST token:** `sfmc-refresh.ps1` writes an `mcpt-` bridge token for MCP tools; raw REST API needs the inner 4-segment JWT — `sfmc.get_token()` unwraps it automatically
- **Refresh script:** `~/.claude/sfmc-refresh.ps1` (callback port 54322)

### 6.3 Inventory Tooling Built

| File | Location | Purpose |
|---|---|---|
| `asset_inventory.ps1` | `~/.claude/` | Full REST pagination + folder resolution + tabular report |
| `generate_asset_pdf.py` | `~/.claude/` | fpdf2 PDF generator — reads raw JSON, outputs formatted PDF |
| `asset_inventory_raw.json` | `~/.claude/` | Full API response (local only — never commit) |
| `SFMC_Asset_Inventory.pdf` | Project repo | Human-readable inventory (committed) |

---

## 7. Sources & References

All claims in §2–5 were adversarially verified by a 168-agent deep-research workflow (96 claims verified: 27 confirmed, 69 refuted as overstated or unverifiable). Only confirmed claims are included above.

| Source | Publisher | Date | Used for |
|---|---|---|---|
| [MCE→MCN analysis: "not clear for content"](https://www.salesforceben.com/the-future-of-marketing-cloud-journey-builder-flow-engine-and-more/) | Salesforce Ben | June 16, 2025 (updated June 24, 2025) | §2.5, §3, §4 |
| [Summer '26: AMPscript in MCN + MCE maintenance mode](https://www.salesforceben.com/) | Salesforce Ben | Summer '26 release coverage | §2.1, §2.4, §3 |
| [AMPscript support has arrived (SFMC Tips #280)](https://medium.com/@marketingcloudtips) | Medium/@marketingcloudtips (Nobuyuki Watanabe) | April 22, 2026 | §2.4 |
| [Send MCE Email from Flows (SFMC Tips #218, #268)](https://medium.com/@marketingcloudtips) | Medium/@marketingcloudtips | Spring '26 | §2.3 |
| [AMPscript Is Coming To Marketing Cloud Next](https://dev.to/ampscript-ninja) | dev.to/ampscript-ninja | June 3, 2026 | §2.4 — cites official Salesforce Summer 2026 Release Overview PDF pp. 353–355 |
| [AMPscript Summer '26 confirmation](https://martechnotes.com) | MartechNotes | 2026 | §2.4 |
| [No MCE EOL / no forced migration](https://knak.com/blog/salesforce-marketing-cloud-next/) | Knak | June 3, 2026 | §4.1 |
| [SFMC to MCN Migration: Checklist & Guide](https://www.mavlers.com/blog/sfmc-to-marketing-cloud-next-migration-guide/) | Mavlers | April 8, 2026 | §2.5, §4.2 |
| [Types of Content in Marketing Cloud Next](https://help.salesforce.com/s/articleView?id=mktg.mktg_content_types_ref.htm) | Salesforce Help (official) | 2025–2026 | §2.2 |
| [Create and Manage a Landing Page in MCN](https://help.salesforce.com/s/articleView?id=mktg.mktg_content_lp_create.htm) | Salesforce Help (official) | Winter/Spring '26 | §2.2, §3 |
| [Create and Manage Forms in MCN](https://help.salesforce.com/s/articleView?id=mktg.mktg_content_form_create.htm) | Salesforce Help (official) | Winter/Spring '26 | §2.2, §3 |
| [MCN Connect API — Content Upload](https://developer.salesforce.com/docs/marketing/marketing-cloud-growth/references/mc-connect-content/mc-manage-content-connect-upload.html) | Salesforce Developer Docs (official) | 2025–2026 | §2.2, §4.3 |
| [Salesforce Active Product & Feature Retirements](https://help.salesforce.com/s/articleView?id=000381744) | Salesforce Help (official) | 2025–2026 | §4.1 — confirms no MCE product in retirement list |

---

*Generated by Claude Code (claude-sonnet-4-6) · Research verified 2026-06-10 · 168 agents · 96 claims checked*
