# SFMC MCP POC — Persistent Memory

This file is auto-loaded by Claude Code at the start of every session in this directory.
It captures hard-won knowledge so each session doesn't have to rediscover it.

See `SFMC-MCP-SESSION-NOTES.md` for full details. Quick reference below.

---

## Environment

- **MCP server**: `salesforce-mcp` (HTTP transport, project-scoped)
- **MCP endpoint**: `https://<MCP_HOST>/t/<TENANT>/c/<MCP_ROUTING_ID>/api/mcp`
- **REST base**: `https://<TENANT>.rest.marketingcloudapis.com`
- **Stack**: S7 (`<SOAP_HOST>`), Enterprise ID `<EID>`
- **Auth**: OAuth 2.0 Authorization Code + PKCE (public package, no client secret)
- **Token TTL**: ~18 min — refresh script runs every 15 min via Windows Scheduled Task `\ClaudeCode\SFMC-MCP-TokenRefresh`
- **Refresh script**: `C:\Users\harsh.f.patel\.claude\sfmc-refresh.ps1` (OAuth callback port `54322`)
- **Claude Code exe**: `C:\Users\harsh.f.patel\AppData\Roaming\Claude\claude-code\2.1.142\claude.exe`

If the token is expired (401 from REST API), run:
```powershell
powershell -ExecutionPolicy Bypass -File "$env:USERPROFILE\.claude\sfmc-refresh.ps1"
```

---

## Critical SFMC REST API Quirks — `/data/v1/customobjects`

**Always apply these when creating Data Extensions via REST.**

### 1. Use `length`, NOT `maxLength`
```json
{ "name": "SubscriberKey", "type": "Text", "length": 100, ... }
```
Wrong property → misleading error: `"A text field with no length specified cannot be used as a primary key"`.

### 2. EmailAddress type REQUIRES explicit `length`
Error `"Length is invalid for field EmailAddress because it does not have a data type of Text"` means the OPPOSITE — length is **required**, not forbidden.
```json
{ "name": "EmailAddress", "type": "EmailAddress", "length": 254 }   // ✅
{ "name": "EmailAddress", "type": "EmailAddress" }                  // ❌ fails
{ "type": "Email" }                                                 // ❌ "fieldType is invalid"
```

### 3. Sendable config — two-step, string form in PATCH
POST with `sendableSubscriberField` as an object causes a JSON deserialization error. Workaround:

**Step A — Create with `isSendable: false`:**
```json
POST /data/v1/customobjects
{ "name": "MyDE", "categoryId": <id>, "isSendable": false, "fields": [...] }
```

**Step B — PATCH to enable sendable (both fields as strings):**
```json
PATCH /data/v1/customobjects/{id}
{ "isSendable": true, "sendableCustomObjectField": "SubscriberKey", "sendableSubscriberField": "Subscriber Key" }
```

### 4. Required field properties (every field, all booleans)
`isTemplateField`, `isHidden`, `isOverridable`, `isInheritable`, `isReadOnly`, `mustOverride`, `isNullable` — plus `name`, `type`, `ordinal`, and `length` (when applicable).

### 5. categoryId is required
Use `/email/v1/categories/{id}` to look up folders. Walk with `parentCatId` to build a path.

---

## Reference DEs in this Account

| Name | ID | Folder |
|---|---|---|
| `TestSubscribers` | `<REDACTED-ID>` | Data Extensions › Swetha Test (catId <REDACTED-ID>) |
| `MCP TEST` | `<REDACTED-ID>` | Data Extensions (root, catId <REDACTED-ID>) |

Both: `SubscriberKey`(Text,100,PK) · `EmailAddress`(254) · `FirstName`(Text,50) · `LastName`(Text,50) · `CreatedDate`(Date) · Sendable: `SubscriberKey → Subscriber Key`.

---

## Useful Endpoints

| Purpose | Endpoint |
|---|---|
| List/create DEs | `/data/v1/customobjects` |
| DE field list (GET) | `/data/v1/customobjects/{id}/fields` |
| Folder details | `/email/v1/categories/{id}` |
| Automations | `/automation/v1/automations` (status `Ready` = active) |
| **Add/edit DE fields** | SOAP `Update` on `https://{tenant}.soap.marketingcloudapis.com/Service.asmx` — REST has no field-mutate endpoint |

---

## Adding Fields to Existing DEs (SOAP required)

REST returns 404 or silently no-ops. Use SOAP `Update`:
- Element names: `FieldType` (not `type`), `MaxLength` (not `length`), PascalCase booleans
- Auth: bearer token in `<fueloauth xmlns="http://exacttarget.com">…</fueloauth>` header
- See `~/.claude/CLAUDE.md` for full XML template.

---

## House Rules

- Never put credentials, bearer tokens, or client secrets into files in this repo or commit them.
- If a REST call returns 401, refresh the token before retrying — don't loop.
- Prefer two-step (create + PATCH) for sendable DEs over single POST — the POST-with-sendable path is broken for `sendableSubscriberField`.
- When unsure about a property name, GET an existing DE and inspect the response — the API often accepts only the canonical name returned by GET.
