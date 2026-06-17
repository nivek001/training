# Forge Agent Operating Contract — ServiceNow MCP

> **What this is:** The operating procedure every Forge agent (Claude Code) follows when it drives the Accenture-hosted ServiceNow MCP server. It governs *how* the agent works — login flow, tool discipline, query economy, communication style, and operational safety — once a connection exists.
>
> **How it differs from the Safety Contract:** [`SAFETY-CONTRACT.md`](SAFETY-CONTRACT.md) defines the *boundaries* (what the agent may never do, what needs confirmation). This contract defines the *procedure* (how the agent does what it's allowed to do). Read both. The Safety Contract wins on any conflict.
>
> **How it becomes live:** This is not a doc the agent might read someday — it is loaded as the agent's memory. `/forge-bootstrap` copies this file to `.forge/AGENT-OPERATING-CONTRACT.md` in your worktree and wires a project `CLAUDE.md` to `@`-import it, so Claude follows it automatically every session. See [§ Bootstrap loads this as agent memory](#bootstrap-loads-this-as-agent-memory).
>
> **Connecting vs operating:** *Connecting* to the MCP is handled by the `/remotemagic` (Accenture-hosted, default) and `/localmagic` (advanced, [RUNBOOK-06](../runbooks/RUNBOOK-06-forge-local-mcp.md)) skills. This contract is what the agent does *after* it's connected.

MCP server URL: `https://servicenow-mcp.victoriouspebble-047683d1.westus2.azurecontainerapps.io/mcp`

## Communication Style
* Be concise. Use tables for multi-record output. Summarize findings.
* For each finding: state the fact, whether action is needed, and a recommendation (or confirm it's fine).
* Never show raw JSON, sys_ids, or stack traces. Translate into plain language.
* Use **Red/Yellow/Green** or **A–F grades** for health assessments.

## Login Flow

> **CRITICAL**: Call `configure-instance` BEFORE `login-to-servicenow-stateless`. Never skip or reorder.

1. **Ask** for: Instance URL, OAuth Client ID, OAuth Client Secret.
2. Call `configure-instance(instance_url, client_id, client_secret)`. Save `login_id`. Tell user their Redirect URI is: `https://servicenow-mcp.victoriouspebble-047683d1.westus2.azurecontainerapps.io/callback`
3. Call `login-to-servicenow-stateless(login_id=...)`. Present the returned `oauth_url` **as-is** as a clickable link: `[Authorize ServiceNow](<oauth_url>)`. Never substitute the callback URL here.
4. User authorizes → gives you a code. Call `complete-login-stateless(login_id=..., verification_code=...)`.
5. Pass `login_id` to **every** subsequent tool call. On auth errors, restart from step 3.
6. **login_id is echoed back in every tool response.** If you ever lose track of it, check the most recent tool result — it will contain `"login_id":"<value>"`.

> The `/remotemagic` skill automates steps 1–5 and caches `login_id` locally. This section is the canonical contract the skill implements — if the skill and this contract ever disagree, this contract is the source of truth.

## MCP Resources (On-Demand Guides)

> **BEFORE multi-table write operations or complex tools**, read the relevant MCP resource first. These cost zero tokens until accessed and prevent common mistakes.

| Resource | URI | When to Read |
|---|---|---|
| Authentication Guide | `servicenow://help/authentication` | If confused about stateful vs stateless login |
| Catalog Item Model | `servicenow://models/catalog_item` | Before creating catalog items with variables, choices, scripts, or UI policies |
| CMDB CI Model | `servicenow://models/cmdb_ci` | Before creating CIs, relationships, service mappings, or using `analyze-ci-impact` |
| ITSM Record Model | `servicenow://models/itsm_record` | Before creating incidents, changes, or problems with related records |
| Now Assist Generative | `servicenow://models/now_assist_generative` | **MUST READ** before using `generate-flow-from-text` or `generate-catalog-from-text` |
| Service Mapping Guide | `servicenow://guides/service_mapping` | When asked about Service Mapping, top-down discovery, or mapping application services |

Resources are on-demand — read once per conversation, then use for all subsequent calls.

## Tools (42)

### Authentication & Configuration (5)

| Tool | Purpose |
|---|---|
| `configure-instance` | Set instance URL, client ID, secret |
| `login-to-servicenow` / `-stateless` | Initiate OAuth (stateful / stateless) |
| `complete-login` / `-stateless` | Complete OAuth |

### Core CRUD (6)

| Tool | Purpose |
|---|---|
| `get-record` | Fetch one record by sys_id or number |
| `create-record` | Create a record on any table. Response returns only `sys_id`, `number`, `state` + the fields you sent. |
| `update-record` | Update a record by sys_id. Response returns only `sys_id`, `number`, `state` + the fields you updated. |
| `perform-query` | Query with encoded syntax; key fields only by default, use `fields` or `all_fields=true` |
| `count-records` | Exact count only — no record payload, very fast |
| `get-table-schema` | Table field definitions |

### App Scope (3) — `scope`

| Tool | Purpose |
|---|---|
| `get-current-scope` | Read the currently active application scope for this user (name, sys_id, scope string). |
| `list-available-scopes` | Fuzzy-search scopes by name or scope string. Use to find a sys_id before `set-app-scope`. Supports `limit` / `offset`. |
| `set-app-scope` | Switch the active scope via `scope_identifier` — a sys_id, a scope string (e.g. `x_acal_sdk_lab`, `global`), or an app name. New artifacts land in this scope until changed. |

Call `get-current-scope` before any create-write if the user hasn't explicitly named a scope; call `set-app-scope` before `create-action` or `generate-flow-from-text` when the target scope isn't global.

### Health & Diagnostics (2) — `health`

| Tool | Purpose |
|---|---|
| `check-mid-server-cluster-health` | MID Server status & load |
| `audit-integration-latency` | REST/SOAP outbound latency trends |

### Security (1) — `security`

| Tool | Purpose |
|---|---|
| `audit-acl-exposure` | High-risk ACL scan (public, scripted, unprotected) |

### Process & Workflows (2) — `process`

| Tool | Purpose |
|---|---|
| `diagnose-process-bottlenecks` | SLA breach analysis by assignment group |
| `detect-stalled-flows` | Stuck Flow Designer flows |

### Replatforming (3) — `replatforming`

| Tool | Purpose |
|---|---|
| `audit-customization-variance` | Customization variance vs OOB baseline |
| `analyze-metadata-deletions` | Metadata deletions by class |
| `identify-skipped-upgrade-candidates` | Skipped upgrades needing review |

### App Engine Refactoring (4) — `refactoring`

| Tool | Purpose |
|---|---|
| `scan-synchronous-gliderecord` | Sync GlideRecord in client scripts |
| `detect-legacy-dot-walking` | Dot-walking in server scripts |
| `audit-global-scope-violations` | Custom artifacts in global scope |
| `analyze-legacy-jelly-usage` | Legacy Jelly XML usage |

### Catalog Migration (3) — `catalog_migration`

| Tool | Purpose |
|---|---|
| `extract-catalog-item-metadata` | Catalog item inventory with metadata |
| `map-catalog-variables-paginated` | Variable mapping with type breakdown |
| `audit-catalog-ui-policies` | UI policies and actions for a catalog item |

### CMDB Health (5) — `cmdb_health`

| Tool | Purpose |
|---|---|
| `aggregate-cmdb-health-metrics` | Health metrics with A–F compliance grading |
| `evaluate-kpi-completeness` | Required-field completeness for a CI class |
| `assess-duplicate-relationship-counts` | Duplicate CI relationships |
| `query-remediation-tasks` | Remediation tasks by state |
| `analyze-ci-impact` | Traverse CMDB relationships to find blast radius of a CI (retirement, re-platforming, change impact). Read the CMDB CI resource first. |

### Security & PII (4) — `security_pii`

| Tool | Purpose |
|---|---|
| `scan-pii-dictionary-attributes` | PII columns with encryption status |
| `audit-high-risk-roles` | High-risk role assignments (admin, impersonator) |
| `evaluate-encryption-contexts` | Encryption context audit |
| `detect-unencrypted-journal-fields` | Unencrypted journal fields |

### Generative & Now Assist (4) — `generative`

| Tool | Purpose |
|---|---|
| `generate-catalog-item-sc-api` | Create catalog item + variables + choices via Table API (no Now Assist required) |
| `create-action` | Create a Flow Designer action (definition + typed inputs/outputs) via Table API. Only `name` is required; pass `inputs`/`outputs` arrays for typed parameters. No Now Assist required. Open in Flow Designer to add step logic and activate. |
| `generate-flow-from-text` | Create a Flow Designer flow from natural language via OOTB Text2Flow API (requires Now Assist for Creator) |
| `generate-catalog-from-text` | Create a catalog item + variables from natural language via OOTB Text2Catalog API (requires Now Assist for Creator) |

## Generative Tools — How To Use

> **CRITICAL**: Read the resource `servicenow://models/now_assist_generative` before your first call.

### Generate a Flow
```
generate-flow-from-text(
  prompt="Create a flow that sends an email when a P1 incident is created",
  flow_name="P1 Incident Alert"       # optional
)
```
- Returns `sysId` — the flow is persisted in **draft/preview** status
- Find it in Flow Designer under scope **Global**, filter by **Draft** or **All**
- The flow includes trigger + actions as described in the prompt

### Generate a Catalog Item (AI)
```
generate-catalog-from-text(
  prompt="Create a catalog item called VPN Access Request for requesting VPN access with a dropdown for region (US, EU, APAC)"
)
```
- Returns `cat_item_id` — the item is created in **draft** state
- **Manual publish required**: Open in Catalog Builder → Review and submit → Submit
- Use pattern **"called [Name] for [description] with [variables]"** in your prompt for best name extraction

### Generate a Catalog Item (Table API — no AI)
```
generate-catalog-item-sc-api(
  name="VPN Access Request",
  short_description="Request VPN access",
  variables=[{...}]
)
```
- Creates item + variables + choices via Table API directly
- No Now Assist required — works on any instance

### Create a Flow Designer Action (Table API — no AI)
```
create-action(
  name="Send P1 Email",
  description="Sends email for P1 incidents",
  inputs=[{"name":"incident_number","type":"string"}],
  outputs=[{"name":"email_sent","type":"boolean"}]
)
```
- Returns the new action's `sys_id`; status is draft/inactive until you add step logic in Flow Designer and activate.
- For scope-bound actions, call `set-app-scope` first.

## Common Tables

| Table | Contains |
|---|---|
| `incident` / `change_request` / `problem` | ITSM core |
| `sc_request` / `sc_task` | Service catalog requests |
| `sc_cat_item` / `item_option_new` / `question_choice` | Catalog items, variables, choices |
| `catalog_script_client` / `catalog_ui_policy` | Catalog scripts + UI policies |
| `sys_script` / `sys_script_include` | Business rules / script includes |
| `sys_db_object` | Table definitions (custom = `u_` prefix) |
| `cmdb_ci_service` / `cmdb_ci_business_app` | CSDM service & app layers |
| `sys_user` / `sys_user_group` | Users and groups |

## Query Best Practices
- **For counts:** Use `count-records` — returns just a number, no payloads.
- **For record data:** Use `perform-query` — only key fields are returned by default. Always pass `fields` to get exactly what you need (e.g., `fields="number,state,short_description"`). Only use `all_fields=true` when the user explicitly asks for every column.
- **Limit guidance:** Use `limit=50–100` for analysis. Only go up to `limit=500` when the user explicitly needs every record.
- If a call times out, silently retry with a smaller limit. Never surface timeout errors to the user without first retrying.
- When reporting record counts, be transparent: if you couldn't retrieve every record, tell the user the count is **"at least X"** rather than presenting an incomplete number as exact.
- Never ask the user what limit to use — just get the data.

## Context Management
- **Every response echoes your `login_id`.** If you lose it, read it from the most recent tool response.
- **Use `count-records` first, then query.** Get the count, then pull a sample if needed.
- **Never dump raw results.** Summarize each query before making the next call.
- **One query at a time** for multi-step assessments. Summarize, then proceed.
- **Always specify `fields`** — even when defaults exist, explicit field lists are more efficient.

## Error Handling
- If a table returns 403/404: *"I couldn't access [table] — this may not be activated on your instance."* Move on.
- Error responses now include ServiceNow's actual error message (e.g., missing mandatory fields, ACL restrictions). Use these to diagnose issues.
- If a query fails, retry silently before reporting failure. Never show HTTP errors.

## Safety
- **Read before write.** Always query before creating or updating.
- **Never fabricate sys_ids.** Only use sys_ids from query results.
- **`data` must be a JSON object, never a string.** When calling `create-record` or `update-record`, pass `data` as `{"field": "value"}` — not `'{"field": "value"}'`. All values must be strings (e.g., `"urgency": "1"`, not `"urgency": 1`).
- **The Safety Contract overrides this section.** Before any write, deploy, bulk operation, or OOB change, the boundaries in [`SAFETY-CONTRACT.md`](SAFETY-CONTRACT.md) (§1 hard prohibitions, §2 confirmation gates) apply. When uncertain whether an action is allowed, stop and ask — inventing permission is worse than friction.

## Bootstrap loads this as agent memory

The whole point of this contract is that the agent does not have to be told these rules each session — they are loaded as project memory.

When `/forge-bootstrap` runs in a worktree, it:

1. Copies this file to `.forge/AGENT-OPERATING-CONTRACT.md` in the project (alongside `.forge/SAFETY-CONTRACT.md`).
2. Creates or updates a project `CLAUDE.md` at the worktree root that imports both:

   ```markdown
   # Project agent memory

   @.forge/AGENT-OPERATING-CONTRACT.md
   @.forge/SAFETY-CONTRACT.md
   ```

3. Claude Code auto-loads `CLAUDE.md` (and its `@`-imports) at the start of every session in that worktree — so the agent operates the ServiceNow MCP correctly from the first prompt, with no copy-paste setup.

If you ever find the agent ignoring this contract, confirm `CLAUDE.md` exists at the worktree root and that the two `@`-imports resolve. Re-run `/forge-bootstrap` to repair.
