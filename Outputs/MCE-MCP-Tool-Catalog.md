# MCE MCP Server — Tool Catalog

The complete tool set exposed by Salesforce's **Marketing Cloud Engagement (MCE) MCP server** (`salesforce-mcp` / `mcp-server-sfmc`) — ~100 tools across 9 categories. Useful as a capability map: every tool wraps a standard MCE REST/SOAP operation.

**Source:** <https://developer.salesforce.com/docs/marketing/mce-mcp/references/mce-mcp-tools/mce-mcp-tools.html> (captured 2026-06-11). Per-category pages follow `mce-mcp-tools-<category>.html`.

**Tool classifications** — each tool's doc page also lists required **Permission Scopes** (a missing scope yields an `API Permission Failed` error):
- **Read-only** — cannot modify objects.
- **Destructive** — can change/overwrite data irreversibly.
- **Asynchronous** — starts a job; needs a separate status check.
- **Open-world** — handles non-rigid / free-form schema data.
- **Executable / Run** — triggers execution (run automation/query, send/publish).

> **Note on run vs. build:** the catalog includes *run / publish / send* tools (`sfmc_run_automation`, `sfmc_run_sql_query`, `sfmc_publish_journey`, `sfmc_send_*`). When working through the **raw REST API** directly, automation Run-Once / journey activation behave as UI-only; the MCP server's wrapper tools may reach a working execution path, so prefer them for run/activate operations when available.

---

## Automations (14)
| Tool | Does | Class |
|---|---|---|
| `sfmc_create_automation` | Create an Automation Studio automation | Write |
| `sfmc_update_automation` | Update an automation | Destructive |
| `sfmc_get_automation` | Retrieve an automation by ID | Read-only |
| `sfmc_get_automations` | List all automations | Read-only |
| `sfmc_get_automation_categories` | List automation folders | Read-only |
| `sfmc_get_automation_instance` | Retrieve a specific execution instance | Read-only |
| `sfmc_run_automation` | Run an automation immediately | Executable |
| `sfmc_run_automation_activities` | Run specific activities in an automation | Executable |
| `sfmc_create_sql_query` | Create a SQL Query activity | Destructive |
| `sfmc_update_sql_query` | Update a SQL Query activity | Destructive |
| `sfmc_get_sql_query` | Retrieve a SQL Query activity by ID | Read-only |
| `sfmc_get_sql_queries` | List all SQL Query activities | Read-only |
| `sfmc_run_sql_query` | Execute a SQL Query activity | Executable |
| `sfmc_validate_sql_query` | Validate SQL query text | Read-only |

## Data Extensions (14)
| Tool | Does | Class |
|---|---|---|
| `sfmc_create_data_extension` | Create a new data extension | Write |
| `sfmc_update_data_extension` | Modify DE properties | Destructive |
| `sfmc_delete_data_extension` | Remove a DE entirely | Destructive |
| `sfmc_clear_data_extension_data` | Remove all records from a DE | Destructive |
| `sfmc_create_data_extension_field_async` | Add fields to an existing DE | Async |
| `sfmc_update_data_extension_field_async` | Modify fields | Destructive · Async |
| `sfmc_get_data_extension` | Fetch a DE by identifier | Read-only |
| `sfmc_get_data_extensions` | Search/list data extensions | Read-only |
| `sfmc_get_data_extensions_by_category` | DEs within a specific folder | Read-only |
| `sfmc_get_data_extension_fields` | Field definitions for a DE | Read-only |
| `sfmc_get_data_extension_folders` | List DE folders | Read-only |
| `sfmc_get_data_extension_link` | Direct UI link to a DE | Read-only |
| `sfmc_retrieve_data_extension_record` | Get a single row | Read-only |
| `sfmc_upsert_data_extension_record` | Insert or update a single row | Write |

## Journeys (36)
| Tool | Does | Class |
|---|---|---|
| `sfmc_create_journey` | Route to the right journey-creation tool | — |
| `sfmc_create_journey_builder_journey` | Create a journey | Write |
| `sfmc_update_journey` | Modify journey content | Destructive |
| `sfmc_delete_journey` | Delete a journey by ID/key | Destructive |
| `sfmc_get_journey` | Retrieve all steps/config for a journey | Read-only |
| `sfmc_get_journeys` | List journeys in the account | Read-only |
| `sfmc_get_journey_versions` | List all versions of a journey | Read-only |
| `sfmc_get_journey_link` | Direct UI link to a journey | Read-only |
| `sfmc_get_journey_publish_status` | Check publication status | Read-only |
| `sfmc_publish_journey` | Publish a journey version | Async |
| `sfmc_republish_journey_content` | Refresh email content w/o new version | Write |
| `sfmc_pause_journey` | Pause a running journey | Write |
| `sfmc_resume_journey` | Resume a paused journey | Write |
| `sfmc_stop_journey` | Permanently stop a running journey | Destructive |
| `sfmc_insert_contacts_into_journey_async` | Batch-insert contacts into a journey | Async |
| `sfmc_insert_contacts_into_journey_status` | Status of a batch insertion | Read-only |
| `sfmc_exit_contact_from_journey` | Remove a contact from an active journey | Write |
| `sfmc_exit_contact_from_journey_status` | Status of a contact-removal request | Read-only |
| `sfmc_fire_journey_event` | Trigger events / signal wait activities | Write |
| `sfmc_create_event_definition` | Define a journey-trigger event | Write |
| `sfmc_update_event_definition` | Modify an event definition | Destructive |
| `sfmc_delete_event_definition` | Delete an event definition | Destructive |
| `sfmc_get_event_definition` | Retrieve an event definition | Read-only |
| `sfmc_get_event_definitions` | List event definitions | Read-only |
| `sfmc_api_event_trigger` | Build an API event entry trigger | Write |
| `sfmc_data_extension_trigger` | Build a DE entry trigger | Write |
| `sfmc_email_activity` | Construct an email activity | Write |
| `sfmc_sms_activity` | Construct an SMS activity | Write |
| `sfmc_wait_activity` | Construct a wait activity | Write |
| `sfmc_decision_split_activity` | Construct a decision split | Write |
| `sfmc_engagement_decision_activity` | Construct an engagement decision split | Write |
| `sfmc_random_split_activity` | Construct a random split (control holdout) | Write |
| `sfmc_einstein_sto_activity` | Construct an Einstein Send-Time-Optimization activity | Write |
| `sfmc_einstein_engagement_frequency_activity` | Construct an Einstein Engagement Frequency activity | Write |
| `sfmc_refresh_transactional_email` | Refresh content for a transactional send def | Write |
| `sfmc_republish_triggered_send` | Refresh content for a classic triggered send | Write |

## Content (9)
| Tool | Does | Class |
|---|---|---|
| `sfmc_create_content_builder_asset` | Create a Content Builder asset | Destructive |
| `sfmc_create_email` | Create an HTML email asset | Destructive |
| `sfmc_create_email_template` | Create an email template asset | Destructive |
| `sfmc_create_sms` | Create an SMS message asset | Destructive |
| `sfmc_update_content_builder_asset` | Update a Content Builder asset | Destructive |
| `sfmc_get_content_builder_asset` | Retrieve an asset by ID | Read-only |
| `sfmc_get_content_assets` | Query assets with simple filters | Read-only |
| `sfmc_search_content_builder_assets` | Query assets with advanced queries | Read-only |
| `sfmc_get_content_categories` | List Content Builder folders | Read-only |

## Contacts (10)
| Tool | Does | Class |
|---|---|---|
| `sfmc_update_contact_attributes` | Update subscriber attributes (profile or DE) | Destructive |
| `sfmc_retrieve_contact_status` | Retrieve a contact's status | Read-only |
| `sfmc_get_contact_key_by_email_address` | Contact keys by email address | Read-only |
| `sfmc_get_email_subscription_status` | Email subscription status of a subscriber | Read-only |
| `sfmc_get_push_opt_in_status_by_subscriber_key` | Push subscription status | Read-only |
| `sfmc_get_list_subscribers` | All members of a classic list | Read-only |
| `sfmc_get_attribute_set_definition` | A specific attribute set definition | Read-only |
| `sfmc_get_attribute_set_definitions` | List attribute set definitions | Read-only |
| `sfmc_get_attribute_set_by_name` | Data from an attribute set by name | Read-only |
| `sfmc_search_attributes` | Search across all attribute sets | Read-only |

## Email (5)
| Tool | Does | Class |
|---|---|---|
| `sfmc_create_email_send_definition` | Create an email send definition (transactional) | Destructive |
| `sfmc_create_triggered_send_definition` | Create a triggered send definition | Destructive |
| `sfmc_send_transactional_email` | Send a transactional email | Async |
| `sfmc_get_transactional_send_status` | Status of a transactional send request | Read-only |
| `sfmc_get_triggered_send_summary` | Sends/opens/clicks/bounces for triggered sends | Read-only |

## SMS (8) — MobileConnect
| Tool | Does | Class |
|---|---|---|
| `sfmc_create_sms_definition` | Create a transactional SMS message definition | Destructive |
| `sfmc_create_sms_send_definition` | Create an SMS send definition (transactional) | Destructive |
| `sfmc_create_mobileconnect_keyword` | Create a MobileConnect keyword | Destructive |
| `sfmc_send_outbound_sms_message` | Send a transactional SMS immediately | Open-world |
| `sfmc_get_sms_definition` | Retrieve an SMS definition by key | Read-only |
| `sfmc_get_sms_definitions` | List transactional SMS definitions | Read-only |
| `sfmc_get_sms_subscription_status` | Subscription status for an SMS definition | Read-only |
| `sfmc_get_mobileconnect_codes` | MobileConnect short/long code configs | Read-only |

## Push (1) — MobilePush
| Tool | Does | Class |
|---|---|---|
| `sfmc_send_push_notification` | Send push notifications immediately | Destructive · Open-world |

## Utilities (5)
| Tool | Does | Class |
|---|---|---|
| `sfmc_describe_object` | Describe the properties of an object | Read-only |
| `sfmc_get_lists` | All subscriber lists | Read-only |
| `sfmc_get_send_classifications` | All send classifications | Read-only |
| `sfmc_get_sender_profiles` | All sender profiles | Read-only |
| `sfmc_get_timezones` | Available time zones | Read-only |

> SMS (MobileConnect) and Push (MobilePush) require those channels to be provisioned; without provisioning the API returns 403.
