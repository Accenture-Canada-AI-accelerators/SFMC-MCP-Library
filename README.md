# SFMC-MCP-Library

# SFMC MCP Integration — Test Cases & Use Cases

This document provides a comprehensive list of test scenarios to validate the SFMC MCP server integration with Marketing Cloud Engagement.

---

## Test Case Categories

### 1. Data Extension (DE) CRUD Operations

#### 1.1 Create Data Extension
- **Scenario**: Create a new DE with mixed field types
- **Test DE name**: `TEST_CRM_Leads`
- **Fields**:
  - `LeadID` (Text, 50, Primary Key)
  - `Email` (EmailAddress, 254)
  - `Company` (Text, 100)
  - `LeadScore` (Number)
  - `CreatedDate` (Date)
  - `LastActivity` (Date)
- **Expected**: DE created in root "Data Extensions" folder, all 6 fields present
- **Validation**: GET `/data/v1/customobjects/{id}/fields` returns all fields

#### 1.2 Retrieve Data Extension
- **Scenario**: Fetch metadata of existing DE
- **Test DE**: `TestSubscribers` (ID: `866ead53-5a58-f111-a5bd-5cba2c7ae570`)
- **Expected**: Full DE object with `fieldCount=5`, `isSendable=true`, `sendableCustomObjectField="SubscriberKey"`
- **Validation**: Correct folder path in categoryId 723391

#### 1.3 Update Data Extension Metadata
- **Scenario**: Enable sendable on an existing non-sendable DE
- **Test DE**: Create `TEST_EmailList` with `isSendable: false`, then PATCH to enable
- **Expected**: PATCH returns `isSendable: true`, mapped to "Email" field
- **Validation**: Two-step flow (create + PATCH with string-form `sendableSubscriberField`)

#### 1.4 List Data Extensions
- **Scenario**: Retrieve all DEs in account with pagination
- **Expected**: 
  - Total: ~10-15 DEs found
  - Both `TestSubscribers` and `MCP TEST` appear in list
  - Support `$pagesize=20` pagination
- **Validation**: Count matches manual SFMC count

#### 1.5 Delete Data Extension
- **Scenario**: Create a temporary DE and delete it
- **Test DE name**: `TEMP_TestDelete`
- **Expected**: 204 or 200 response, DE no longer exists on second GET
- **Validation**: GET returns 404 after delete

---

### 2. Data Extension Field Operations

#### 2.1 Add Field to Existing DE
- **Scenario**: Add a new field to `MCP TEST` using SOAP API
- **Field**: `PhoneNumber` (Text, 20)
- **Expected**: Field added as ordinal 7
- **Validation**: 
  - SOAP UpdateResponse shows `<StatusCode>OK</StatusCode>`
  - GET `/fields` returns field in list
- **Note**: REST field-add endpoint doesn't exist — SOAP required

#### 2.2 Update Field on Existing DE
- **Scenario**: Modify field properties (e.g., `isRequired` or `isNullable`)
- **Test Field**: Modify `Rate` field on `MCP TEST`
- **Expected**: SOAP Update applies changes
- **Validation**: GET `/fields` reflects updated properties

#### 2.3 List Fields on DE
- **Scenario**: Retrieve all fields from `TestSubscribers`
- **Expected**: 5 fields with correct types and lengths:
  - SubscriberKey (Text, 100, PK)
  - EmailAddress (EmailAddress or Text, 254)
  - FirstName (Text, 50)
  - LastName (Text, 50)
  - CreatedDate (Date)
- **Validation**: Field properties match creation spec

#### 2.4 Field Type Edge Cases
- **Scenario**: Test native EmailAddress type with explicit length
- **Expected**: `type: "EmailAddress"` + `length: 254` succeeds
- **Negative test**: Same without `length` should fail with misleading error
- **Validation**: Error message says "Length is invalid…"

---

### 3. Sendable Data Extension Configuration

#### 3.1 Configure Sendable DE from Scratch
- **Scenario**: Create a marketing DE and configure sendable mapping
- **Test DE**: `TEST_Subscribers_Marketing`
- **Fields**: SubscriberKey (PK), Email, FirstName, CompanyName
- **Steps**:
  1. POST with `isSendable: false`
  2. PATCH with `sendableCustomObjectField` and `sendableSubscriberField` (both strings)
- **Expected**: Two-step succeeds; single POST with sendable fails
- **Validation**: Final DE is sendable, SubscriberKey → Subscriber Key mapped

#### 3.2 Verify Sendable Configuration
- **Scenario**: GET an existing sendable DE
- **Test DE**: `MCP TEST`
- **Expected**: Response includes:
  - `isSendable: true`
  - `sendableCustomObjectField: "SubscriberKey"`
  - `sendableSubscriberField: "Subscriber Key"`
- **Validation**: All three fields present and correct

---

### 4. Folder Navigation & Organization

#### 4.1 Walk Folder Tree
- **Scenario**: Navigate from `TestSubscribers` (catId 723391) to root
- **Expected**:
  - 723391 → "Swetha Test"
  - 723391.parentCatId = 644897 → "Data Extensions"
  - 644897.parentCatId = 0 (root)
- **Validation**: Path = "Data Extensions › Swetha Test"
- **Endpoint**: `/email/v1/categories/{id}`

#### 4.2 List All DE Folders
- **Scenario**: Retrieve all Data Extension category folders
- **Expected**: At least 2 folders: "Data Extensions" (root) and "Swetha Test"
- **Validation**: Filter by `categorytype eq 'dataextension'` works

#### 4.3 Find DE by Category
- **Scenario**: List all DEs in "Swetha Test" folder
- **Expected**: `TestSubscribers` appears in that folder
- **Validation**: categoryId 723391 contains TestSubscribers

---

### 5. Automation & Journey Querying

#### 5.1 List All Automations
- **Scenario**: Fetch automation inventory
- **Expected**: 
  - Total: ~9 automations
  - 5 with status "Ready" (active)
  - 4 with status "InactiveTrigger"
- **Validation**: Count matches manual SFMC check
- **Endpoint**: `/automation/v1/automations`

#### 5.2 List All Journeys (Interactions)
- **Scenario**: Fetch journey inventory
- **Expected**:
  - Total: ~14 journeys
  - 3 with status "Published" (active)
  - 10 in "Draft"
  - 1 "Stopped"
- **Validation**: All journey names and versions match SFMC UI
- **Endpoint**: `/interaction/v1/interactions?extras=all`

#### 5.3 Filter Active Automations
- **Scenario**: Query only active automations
- **Filter**: `status == 'Ready'`
- **Expected**: 5 automations returned
- **Validation**: No InactiveTrigger or other statuses

#### 5.4 Filter Active Journeys
- **Scenario**: Query only published journeys
- **Filter**: `status == 'Published'`
- **Expected**: 3 journeys returned
- **Validation**: Names: "New Journey - c", "New Journey - January 26 2022 9.36 PM", "NG test"

---

### 6. API Error Handling & Resilience

#### 6.1 Token Expiry & Refresh
- **Scenario**: Simulate token expiry (18 min), trigger refresh
- **Steps**:
  1. Wait or manually expire token
  2. Call REST API → expect 401
  3. Run `powershell -ExecutionPolicy Bypass -File ~/.claude/sfmc-refresh.ps1`
  4. Retry API call with new token
- **Expected**: Refresh succeeds, new token works, no manual re-auth needed
- **Validation**: Token payload shows new `access_token`

#### 6.2 Misleading Error Messages
- **Scenario**: Trigger known misleading errors and verify workarounds
- **Test cases**:
  - Use `maxLength` instead of `length` → "A text field with no length specified cannot be used as a primary key"
  - Omit `length` on EmailAddress → Same error
  - Use `type: "Email"` (short form) → "fieldType is invalid: Email"
  - Use object form for `sendableSubscriberField` → "JSON Deserialization Exception"
- **Expected**: All errors triggered as documented
- **Validation**: Workarounds (use `length`, use string form) succeed

#### 6.3 404 on Non-Existent Resource
- **Scenario**: GET a DE that doesn't exist
- **Test ID**: `00000000-0000-0000-0000-000000000000`
- **Expected**: 404 with proper error message
- **Validation**: No 500 errors or crashes

#### 6.4 Invalid Field Configuration
- **Scenario**: Create DE with missing required boolean properties
- **Expected**: 400 error — "isTemplateField is required for Field"
- **Validation**: Error message is clear and actionable

#### 6.5 Port Already In Use
- **Scenario**: OAuth refresh with stale port 54321 reservation
- **Expected**: Script gracefully falls back to port 54322
- **Validation**: Refresh succeeds on alternate port

---

### 7. SOAP vs REST API Differences

#### 7.1 Property Name Mapping
- **Scenario**: Compare field property names between REST (POST/PATCH) and SOAP
- **REST**: `type`, `length`, `isPrimaryKey`
- **SOAP**: `FieldType`, `MaxLength`, `IsPrimaryKey`
- **Test**: Create identical field via REST, update it via SOAP
- **Expected**: All properties apply correctly despite name differences
- **Validation**: GET response shows updated field

#### 7.2 Bearer Token in SOAP
- **Scenario**: Authenticate to SOAP with bearer token via `<fueloauth>` header
- **Expected**: Token accepted, no WSS/basic auth needed
- **Validation**: SOAP response includes `<StatusCode>OK</StatusCode>`

#### 7.3 SOAP Field Update Without REST Alternative
- **Scenario**: Add field via SOAP (REST endpoint doesn't exist)
- **Expected**: SOAP succeeds where REST `/fields` POST returns 404
- **Validation**: Field appears in subsequent GET

---

### 8. MCP-Specific Integration Tests

#### 8.1 Token Refresh Automation
- **Scenario**: Validate Windows Scheduled Task runs every 15 minutes
- **Test**: Check task history in Task Scheduler
- **Expected**: Task `\ClaudeCode\SFMC-MCP-TokenRefresh` shows recent runs
- **Validation**: Token is kept fresh without manual intervention

#### 8.2 MCP Server Status
- **Scenario**: Check MCP server connection status
- **Command**: `claude.exe mcp list`
- **Expected**: `salesforce-mcp` shows "Connected ✅"
- **Validation**: Bearer token header is present and valid

#### 8.3 MCP Config Persistence
- **Scenario**: Verify MCP config persists in `~/.claude.json`
- **Expected**: `mcpServers.salesforce-mcp` entry exists with:
  - `type: "http"`
  - `url: "https://mai-mce-mcp-cdp1.sfdc-yfeipo.svc.sfdcfc.net/..."`
  - `headers.Authorization: "Bearer eyJ..."`
- **Validation**: Config format matches project requirements

#### 8.4 Project-Scoped vs Global Configuration
- **Scenario**: Verify MCP is project-scoped, not global
- **Expected**: MCP server only available when Claude Code opens `/MC MCP POC` directory
- **Validation**: Different project directory doesn't auto-load `salesforce-mcp`

---

### 9. Data Validation & Consistency

#### 9.1 Field Count After Operations
- **Scenario**: Create DE with 5 fields, add 1 field, verify count
- **Expected**: Final `fieldCount=6`
- **Validation**: Both REST list and SOAP response agree

#### 9.2 Sendable Field Mapping Consistency
- **Scenario**: Create sendable DE, GET it multiple times
- **Expected**: `sendableCustomObjectField` and `sendableSubscriberField` always match
- **Validation**: No silent changes or inconsistencies

#### 9.3 Folder Hierarchy Integrity
- **Scenario**: Create DE in subfolder, verify parentCatId chain is correct
- **Expected**: Walking parentCatId always leads to root (catId with no parent)
- **Validation**: No orphaned folders or circular references

---

### 10. Performance & Scale Tests

#### 10.1 Large Field List
- **Scenario**: Retrieve DE with 20+ fields
- **Test DE**: Create `TEST_LargeDE` with 20 fields
- **Expected**: GET `/fields` returns all fields, no truncation
- **Validation**: Response includes `fieldCount=20`

#### 10.2 Pagination
- **Scenario**: List DEs with `$pagesize=5`
- **Expected**: Multiple pages returned, pagination works
- **Validation**: `$page=1` has 5 items, `$page=2` has next 5 (up to total count)

#### 10.3 Large Query Results
- **Scenario**: List 100+ automations or journeys
- **Expected**: All items retrieved via pagination
- **Validation**: No truncation, all items counted

---

## Test Execution Checklist

### Pre-Execution
- [ ] Token is fresh (run refresh if needed)
- [ ] MCP server shows "Connected" (`claude.exe mcp list`)
- [ ] Test DEs don't already exist (or are acceptable to overwrite)
- [ ] Network connectivity to SFMC confirmed

### During Execution
- [ ] Record HTTP status codes for each call
- [ ] Capture response headers (e.g., RateLimit-Remaining)
- [ ] Note any warnings or deprecations
- [ ] Screenshot UI for manual verification of created objects

### Post-Execution
- [ ] All test DEs deleted (cleanup)
- [ ] Verify no orphaned objects in SFMC
- [ ] Confirm token still valid
- [ ] Document any bugs or unexpected behaviors in `/TEST_RESULTS.md`

---

## Known Limitations & Documented Quirks

| Issue | Workaround | Test Case |
|-------|-----------|-----------|
| REST field-add endpoint returns 404 | Use SOAP API for field mutations | 2.1, 7.3 |
| `sendableSubscriberField` as object fails | Use string form in PATCH, not object | 3.1, 6.2 |
| EmailAddress type without explicit `length` fails | Always include `length: 254` on EmailAddress fields | 2.4, 6.2 |
| `maxLength` vs `length` property name | Use `length` in REST (SOAP uses `MaxLength`) | 7.1 |
| SFMC error messages say the opposite | Try variations; don't trust error message literally | 6.2 |
| Port 54321 can get stuck | Use alternate port 54322 in refresh script | 6.5 |

---

## Success Criteria

✅ **All test cases pass** if:
1. All CRUD operations on DEs complete successfully
2. Field add/update works via SOAP API
3. Sendable DE configuration succeeds via two-step flow
4. Automation & journey queries return expected counts
5. Token refresh completes without manual auth
6. Error handling triggers correct workarounds
7. No orphaned objects left in SFMC after cleanup
8. All API quirks are confirmed and documented

---

## Next Steps After Testing

1. Document any additional SFMC API quirks discovered
2. Update `CLAUDE.md` with new findings
3. Create utility scripts for common MCP operations
4. Set up monitoring for MCP connection health
5. Plan integration with Claude AI for automated tasks
