# Architecture — Programmatic SFMC Campaign Build

How the *Northbridge Onboarding* campaign was designed and assembled in Salesforce Marketing Cloud entirely through APIs (2026-06-03). Diagrams use **Mermaid** (GitHub renders them automatically). All identifiers here are object *names* only — no tenant, EID, or object IDs.

Companion docs: `CAMPAIGN-DESIGN-NOTES.md` (rationale + lessons), `MCP-Onboarding-Campaign-Runbook.md` (operational steps), `SFMC-Campaign-Brief-Template.xlsx` (intake template).

---

## 1. Build-time architecture (layers)

The agent orchestrates; an auth/integration layer (kept outside the repo) brokers tokens and HTTP; the SFMC APIs front the platform objects.

```mermaid
flowchart TB
    subgraph ORCH["Orchestration — Claude Code (agent)"]
        A["brief → clarify → plan → build → verify → hand off"]
    end

    subgraph AUTH["Integration + Auth — lives outside the repo"]
        R["sfmc-refresh.ps1<br/>OAuth Auth-Code + PKCE → mcpt- bridge token"]
        H["sfmc.py<br/>unwrap 4-segment REST JWT · GET/POST/PATCH · SOAP"]
        R --> H
    end

    subgraph APIS["SFMC APIs"]
        REST["REST API<br/>/data · /automation · /asset · /interaction"]
        SOAP["SOAP API<br/>Retrieve = read rows · Perform = run (blocked)"]
    end

    subgraph PLAT["SFMC Platform — single Business Unit"]
        DE["Data Extensions"]
        AS["Automation Studio"]
        CB["Content Builder"]
        JB["Journey Builder"]
    end

    A -->|"local tool calls"| R
    H -->|"HTTPS · Bearer"| REST
    H -->|"HTTPS · Bearer"| SOAP
    REST --> PLAT
    SOAP --> PLAT
```

**Why auth sits outside the repo:** the live token and instance endpoints stay off a public repository while the in-repo docs stay generic.

---

## 2. Build sequence (method + the API/UI boundary)

Each object is created, then **read back and verified**, before moving on. The final two steps cross a hard platform boundary: the API can *build* but not *run/activate*.

```mermaid
flowchart TB
    REQ["Requirements<br/>(Excel brief / prompt)"] --> PLAN["Plan — map each item to an SFMC object<br/>ask only blocking questions"]

    PLAN --> P1
    subgraph P1["Phase 1 · Data"]
        P1a["POST rowset → MCP TEST"] --> P1v{{"verify: SOAP Retrieve"}}
    end

    P1 --> P2
    subgraph P2["Phase 2 · Audience + Filter"]
        P2a["POST → Audience DE (non-sendable)"] --> P2b["POST → SQL query activity"]
        P2b --> P2c["POST + PATCH → automation, wire as Step 1"] --> P2v{{"verify: GET read-back"}}
    end

    P2 --> P3
    subgraph P3["Phase 3 · Content + Journey"]
        P3a["POST → 3 email assets"] --> P3b["POST → DE-entry event definition"]
        P3b --> P3c["POST → Draft journey"] --> P3v{{"verify: GET ?extras=all"}}
    end

    P3 --> RB["Runbook"]
    RB --> BD{"API / UI boundary"}
    BD -->|"UI only"| U1["Run Once — automation"]
    BD -->|"UI only"| U2["Activate — journey"]
```

**Pattern:** `build → read-back verify → proceed`. For unfamiliar JSON (e.g. the engagement split) the rule was *mirror an existing live object, don't guess*.

---

## 3. Runtime data flow (campaign once activated)

```mermaid
flowchart TB
    SRC["MCP TEST<br/>(source DE)"]
    SRC -->|"SQL filter: exclude opened/clicked 30d + NB-% cohort<br/>Automation · Overwrite · Run Once (UI)"| AUD["MCP Onboarding Audience<br/>(non-sendable DE)"]
    AUD -->|"DE-entry event definition → journey trigger"| ENTRY([Journey entry])

    subgraph JNY["Journey · metaData.dataSource = ContactsModel"]
        ENTRY --> J1["1 · Welcome email"]
        J1 --> W1["wait 2 days"]
        W1 --> DEC{"Engagement Split<br/>clicked Welcome CTA?"}
        DEC -->|"Yes — engaged"| J2["2 · Getting Started email"]
        DEC -->|"No"| W2["wait 2 days"]
        W2 --> J3["3 · Nudge email"]
        J2 --> EX1([exit])
        J3 --> EX2([exit])
    end
```

---

## 4. Key architectural decisions

| Decision | Reason |
|---|---|
| Auth + tooling **outside** the repo | keeps the live token and endpoints off a public repo |
| **Non-sendable** audience DE | a sendable SQL target throws *"could not build exclusion text"* |
| `metaData.dataSource = "ContactsModel"` on the journey | makes decision/engagement-split fields resolve |
| **REST-direct** rather than MCP tools | MCP tools bind at session start; they could not load mid-session |
| **Build via API, run/activate via UI** | platform limitation — the handoff (runbook) was designed around it |
| Cohort guard (`NB-%`) in the SQL | keeps stray/test rows out of the audience on every Overwrite |
| Read-back verification after every write | SOAP returns HTTP 200 even on failure; trust the read, not the status code |

---

## 5. Component inventory

| Layer | Component | Role |
|---|---|---|
| Orchestration | Claude Code agent | plans + drives the build |
| Auth | `sfmc-refresh.ps1` | OAuth PKCE refresh → bridge token |
| Auth | `sfmc.py` | derive REST JWT + thin REST/SOAP client |
| API | REST `/data` `/automation` `/asset` `/interaction` | create + read objects |
| API | SOAP `Retrieve` / `Perform` | read DE rows; run attempt |
| Platform | Data Extensions | source + filtered audience |
| Platform | Automation Studio | SQL filter automation |
| Platform | Content Builder | 3 onboarding emails |
| Platform | Journey Builder | Draft journey + entry event |
