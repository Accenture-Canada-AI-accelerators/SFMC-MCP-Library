# Journey Configuration Specification: MCP TEST

## Overview
This document describes the `MCP TEST` journey design. It was created programmatically via the **Journey Builder REST API** (`POST /interaction/v1/interactions`) — journey id `<REDACTED-ID>`, status `Draft`. Note: SOAP does NOT support journeys (HTTP 500); REST is the correct path. The UI steps below remain useful for assigning real email assets before publishing. See `JOURNEY-CREATION-FINDINGS.md` for the API details.

## Design Specification

### Journey Metadata
- **Name**: MCP TEST
- **Key**: mcp-test-journey
- **Status**: Draft (ready for publishing after configuration)
- **Entry Source**: MCP TEST Data Extension

### Journey Workflow Structure

```
Entry (MCP TEST DE)
     ↓
[Step 1] Email 1 (Email activity)
     ↓
[Step 2] Filter FirstName H (Decision activity)
     ├─ YES → Continue to Step 3
     └─ NO → Exit Journey
     ↓
[Step 3] Wait 2 Weeks (Wait activity)
     ↓
[Step 4] Email 2 (Email activity)
     ↓
End Journey
```

### Detailed Activity Configuration

#### Step 1: Email 1
- **Type**: Email Activity
- **Name**: Email 1
- **Purpose**: Initial email to segment
- **Configuration**: (Requires email asset selection in UI)

#### Step 2: Filter FirstName H (Decision)
- **Type**: Decision Activity
- **Name**: Filter FirstName H
- **Condition Type**: Attribute-based
- **Condition**:
  - **Attribute**: FirstName
  - **Operator**: Starts With
  - **Value**: H
- **Paths**:
  - **YES (Matches)**: Continue to Step 3 (Wait 2 Weeks)
  - **NO (Does Not Match)**: Exit Journey (stop engagement)

#### Step 3: Wait 2 Weeks
- **Type**: Wait Activity
- **Name**: Wait 2 Weeks
- **Duration**: 2 Weeks
- **Trigger**: After Previous Activity (Email 1)

#### Step 4: Email 2
- **Type**: Email Activity
- **Name**: Email 2
- **Purpose**: Follow-up email after 2-week wait
- **Configuration**: (Requires email asset selection in UI)

---

## UI Configuration Steps

### To Create This Journey in Marketing Cloud Engagement:

1. **Navigate** to: Email Studio → Journey Builder
2. **Create New Journey**:
   - Name: `MCP TEST`
   - Description: `Journey: Email1 decision FirstName H, wait 2 weeks, Email2`
   - Click Create

3. **Configure Entry Source**:
   - Select "Data Extension"
   - Choose: `MCP TEST`
   - Click Save

4. **Add Activities** in order:
   - Click "+" → Add Activity
   - Select "Email" → Choose Email 1 asset
   - Connect to next step

5. **Add Decision Activity**:
   - Click "+" → Add Activity → Decision
   - Set Criteria:
     - Attribute: `FirstName`
     - Operator: `Starts With`
     - Value: `H`
   - Connect YES path to Wait activity
   - Connect NO path to Exit

6. **Add Wait Activity**:
   - Click "+" → Add Activity → Wait
   - Duration: `2` + `Weeks`
   - Save

7. **Add Final Email**:
   - Click "+" → Add Activity → Email
   - Choose Email 2 asset
   - Connect to End

8. **Review & Publish**:
   - Check all connections are valid
   - Click "Start Journey" or keep as Draft

---

## API Capability Summary

| Operation | REST API | SOAP API | Support |
|-----------|----------|----------|---------|
| List Journeys | ✅ GET /interaction/v1/interactions | ❌ 500 | REST only |
| Create Journey | ❌ 404/405 | ❌ 500 | **Not Supported** |
| Update Journey | ❌ 404/405 | ❌ 500 | **Not Supported** |
| Delete Journey | ❌ 404/405 | ❌ 500 | **Not Supported** |
| Query Journey | ✅ GET /interaction/v1/interactions | ❌ 500 | REST only |

---

## Alternative: Create Automation Instead

If you need to automate the email sequence programmatically, you can use **Automations** instead, which ARE fully supported via SOAP API:

✅ Automations support full SOAP CRUD including Activities configuration
✅ Can implement similar logic with SQL activities, decision splits, wait steps
✅ Can send emails via automation workflow

See `SOAP-AUTOMATION-EXAMPLE.md` for automation-based alternative.

---

## Reference

- Marketing Cloud Documentation: https://developer.salesforce.com/docs/marketing_cloud/marketing_cloud/guide/
- Journey Builder User Guide: Journey Builder in Salesforce Marketing Cloud
- Entry Source must be a Data Extension with valid subscriber fields
- Both Email assets must exist and be active before activating journey
