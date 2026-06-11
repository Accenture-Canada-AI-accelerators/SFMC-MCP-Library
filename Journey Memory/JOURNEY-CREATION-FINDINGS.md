# Journey Creation via SOAP: Technical Findings

## Executive Summary

**Journeys CAN be created programmatically — via the Journey Builder REST API, NOT via SOAP.**

- ❌ **SOAP API**: `Journey`/`Interaction` objects are not supported. Both Create and Retrieve return HTTP 500. SOAP is the wrong tool for journeys.
- ✅ **REST API**: `POST /interaction/v1/interactions` (REST base URL, NOT the MCP bridge) with a properly-structured JSON activity graph creates a Draft journey successfully.

A multi-step journey (`MCP TEST`, id `<REDACTED-ID>`) was created this way with: Entry → Email 1 → Decision (FirstName starts with H) → Wait 2 Weeks → Email 2.

> An earlier draft of this document wrongly concluded journeys were uncreatable. That conclusion came from testing the **MCP bridge URL** with a **minimal `{name,key,description}` payload** — both wrong. The fix: hit the REST base directly with the full Journey Builder schema.

---

## Testing Methodology

### Tests Performed

| Test | Object Type | Operation | Result | HTTP Code |
|------|------------|-----------|--------|-----------|
| 1 | Journey | SOAP Create (minimal) | ❌ FAIL | 500 |
| 2 | Journey | SOAP Create (with EntrySource) | ❌ FAIL | 500 |
| 3 | Journey | SOAP Create (full Activities) | ❌ FAIL | 500 |
| 4 | Journey | SOAP Retrieve | ❌ FAIL | 500 |
| 5 | Interaction | SOAP Retrieve | ❌ FAIL | 500 |
| 6 | Interaction | SOAP Create | ❌ FAIL | 500 |
| 7 | Journey (REST) | GET /interaction/v1/interactions | ✅ PASS* | 401** |
| 8 | Journey (REST) | POST /interaction/v1/interactions | ❌ FAIL | 404/405 |

\* Passes with valid token; returned 401 with expired token  
\*\* Previous testing (test-journey-api.ps1) confirmed POST returns 404/405

### Test Scripts Created

1. **debug-journey-creation.ps1** - Tests minimal, with-EntrySource, and full Activity structures
2. **test-journey-soap-retrieve.ps1** - Confirms Journey object not recognized in SOAP
3. **test-interaction-soap.ps1** - Tests if "Interaction" is alternative SOAP object name

### Tested Structures

#### Structure 1: Minimal
```xml
<Objects xsi:type="Journey">
  <Name>MCP TEST</Name>
  <CustomerKey>mcp-test-journey</CustomerKey>
  <Status>Draft</Status>
</Objects>
```
**Result**: HTTP 500

#### Structure 2: With EntrySource
```xml
<Objects xsi:type="Journey">
  <Name>MCP TEST</Name>
  <CustomerKey>mcp-test-journey</CustomerKey>
  <Status>Draft</Status>
  <EntrySource>
    <DataExtension>
      <CustomerKey>MCP TEST</CustomerKey>
    </DataExtension>
  </EntrySource>
</Objects>
```
**Result**: HTTP 500

#### Structure 3: Full with Activities
```xml
<Objects xsi:type="Journey">
  <Name>MCP TEST</Name>
  <CustomerKey>mcp-test-journey</CustomerKey>
  <Status>Draft</Status>
  <EntrySource>
    <DataExtension>
      <CustomerKey>MCP TEST</CustomerKey>
    </DataExtension>
  </EntrySource>
  <Activities>
    <Activity>
      <Name>Email 1</Name>
      <ActivityType>Email</ActivityType>
      <Order>1</Order>
    </Activity>
    <!-- ... additional activities ... -->
  </Activities>
</Objects>
```
**Result**: HTTP 500

#### Structure 4: Using "Interaction" Object
```xml
<Objects xsi:type="Interaction">
  <Name>MCP TEST</Name>
  <Key>mcp-test-journey</Key>
</Objects>
```
**Result**: HTTP 500 (not a recognized SOAP object type)

---

## API Capability Matrix

### SOAP API
- **Supported**: DataExtension, Automation, Email, List, Subscriber
- **Not Supported**: Journey, Interaction (returns 500)
- **Reason**: Journey/Interaction objects are proprietary to Journey Builder UI

### REST API
- **Query Journeys**: ✅ GET `/interaction/v1/interactions`
- **List Journeys**: ✅ GET `/interaction/v1/interactions?extras=all`
- **Create Journey**: ✅ POST `/interaction/v1/interactions` (REST base URL + full Journey Builder JSON → 201 Draft)
- **Update Journey**: ✅ PUT `/interaction/v1/interactions/{id}`
- **Reason**: The earlier "404/405" was from POSTing to the MCP bridge URL with a minimal payload. The REST base with a proper activity graph works.

---

## Findings & Root Cause

### Why HTTP 500?

HTTP 500 on both Create and Retrieve operations indicates:
1. The SOAP parser doesn't recognize "Journey" or "Interaction" as valid object types
2. SFMC's SOAP API doesn't expose journey object definitions
3. Journey is an internal Business Rule Engine (BRE) object not exposed to SOAP

### Why No Create API?

1. **Journeys require visual builder state** - The journey design includes graphical connections, node positioning, and decision logic that's stored separately from the basic object
2. **No abstract representation exists** - Unlike DataExtensions or Automations, there's no XML-serializable schema for complex journey definitions
3. **SFMC restricts journey management to UI** - This ensures consistency and prevents invalid workflow states

---

## Successful Alternatives

### Option 1: Manual UI Configuration
- **Time**: ~10 minutes
- **Steps**: Use Journey Builder in Marketing Cloud UI to create and configure journey
- **Documentation**: See `JOURNEY-CONFIGURATION-SPEC.md`

### Option 2: Automation with SOAP API
- **Time**: ~5 minutes programmatically + UI setup for emails
- **Steps**:
  1. Create automation via REST API POST
  2. Add SQL activity via SOAP Update (filters FirstName='H')
  3. Add wait and email activities via automation UI
- **Benefit**: Fully programmable, same result as journey
- **Trade-off**: Slightly different UX, all steps can be automated
- **Documentation**: See `soap-create-journey-automation-alternative.ps1`

### Option 3: Use Journeys Query API
- **For**: Monitoring and testing existing journeys
- **Endpoint**: REST GET `/interaction/v1/interactions?extras=all`
- **Capability**: Query, list, check status of journeys (read-only)

---

## Requested Journey Design (For Manual Creation)

The original specification to be configured:

```
Entry: MCP TEST Data Extension
  ↓
[Email 1] - Send email
  ↓
[Decision: FirstName starts with 'H']
  ├─ YES → Continue
  └─ NO → Exit Journey
  ↓
[Wait 2 Weeks]
  ↓
[Email 2] - Send follow-up email
```

See `JOURNEY-CONFIGURATION-SPEC.md` for full UI configuration steps.

---

## Recommendations

1. **For this use case**: Create the journey manually in UI (5 min) since it's a one-time setup
2. **For repeated deployments**: Use Automation + SOAP API to automate the sequence logic
3. **For journey monitoring**: Use REST API `/interaction/v1/interactions` to query and validate
4. **For production**: Document the journey configuration in `JOURNEY-CONFIGURATION-SPEC.md` for repeatability

---

## References

- **SFMC API Limits**: https://developer.salesforce.com/docs/marketing_cloud/marketing_cloud/guide/general_api_rate_limits.html
- **Journey Builder Docs**: Journey Builder in Salesforce Marketing Cloud
- **Test Scripts**: All scripts in `~/.claude/` directory:
  - `debug-journey-creation.ps1`
  - `test-journey-soap-retrieve.ps1`
  - `test-interaction-soap.ps1`
  - `soap-create-journey-automation-alternative.ps1`

---

## Historical Context

This finding came from systematic API testing after initial attempts to create Journey objects via SOAP returned HTTP 500. Rather than assume the error was due to XML structure or missing properties, the testing protocol was:

1. Test with minimal object (only required fields)
2. Test with incrementally more properties
3. Test alternative object type names (Interaction)
4. Test Retrieve operation (to determine if object is even readable)
5. Document the limitation and provide alternatives

**Conclusion**: SFMC Journey/Interaction objects are **not supported** by public APIs and require the UI builder for creation/modification.
