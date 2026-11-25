# Nudgebee automation API

Every call is one JSON-RPC request with the action name as `method`:

```bash
curl -sS "$NB_URL/api/rpc" \
  -H 'content-type: application/json' \
  -H "Authorization: Bearer $NB_API_TOKEN" \
  --data '{"jsonrpc":"2.0","id":1,"method":"workflow_list_taskdefinitions","params":{}}'
```

`scripts/nb <method> '<params-json>'` does exactly this. Params can be sent flat
(as below) or wrapped as `{"request": {...}}`; both are accepted.

The full list of actions, with argument and return types, is in the public API
reference (GraphQL view of the same actions): see `docs-map.md` → API reference.

## Discover

| Need | Call | Params |
|---|---|---|
| Accounts (ids for everything else) | `accounts_list` | `{"limit": 50}` |
| Integrations (Slack, SSH, DB, ticketing…) | `integrations_list` | see API reference |
| Every task type + `input_schema` / `output_schema` / `runtime_notes` | `workflow_list_taskdefinitions` | `{}` |
| Built-in templates | `workflow_list_template`, `workflow_get_template` | see API reference |
| Existing automations | `workflow_list` | `{"name": "...", "status": "ACTIVE", "tags": "layer:child,job:cpu", "limit": 50}` — `tags` is a comma-separated string; an object is ignored |
| One automation | `workflow_get` | `{"account_id": "...", "id": "..."}` |
| Real events to design/test against | `events_list` | see API reference (filter by account, source, event type, time) |

Tip: `workflow_list_taskdefinitions` is large. Save it to a file once and search it
(`jq '.tasks[] | select(.name|test("aws"))'`) instead of reading it whole.

## Check (nothing runs)

| Check | Call | Params |
|---|---|---|
| Definition is valid | `workflow_validate` | `{"account_id": "...", "workflow": {"name": "...", "definition": {...}}}` |
| Would this event fire this trigger? | `workflow_check_trigger_match` | `{"account_id": "...", "trigger_type": "event", "params": {<trigger params>}, "payload": {<event>}}` → `{matched, reason, filter, error}` |

## Create and change

| Do | Call | Params |
|---|---|---|
| Create | `workflow_create` | `{"account_id": "...", "workflow": {"name": "...", "description": "...", "tags": {...}, "definition": {...}}}` → `{id}`. Version 1 is **live** at once, status **PAUSED**: triggers don't fire, manual runs and calls from a parent still work |
| Turn triggers on / off | `workflow_resume` / `workflow_pause` | `{"account_id": "...", "id": "..."}` — resume only when the user says so |
| Update the draft | `workflow_update` | `{"account_id": "...", "id": "...", "workflow": {...}}` |
| Publish the draft as a new version | `workflows_create_version` | `{"account_id": "...", "id": "...", "set_live": true, "status": "<current status>"}` — `status` becomes the automation's status: omitted = PAUSED, which silently pauses an ACTIVE automation. Pass its current status; pass `ACTIVE` only with the user's OK. Refused when the draft equals the live version |
| Make an older version live (rollback) | `workflows_update_live_version` | `{"account_id": "...", "id": "...", "version_number": N}` |
| List versions | `workflow_list_versions` | `{"account_id": "...", "id": "..."}` |
| Delete | `workflow_delete` | `{"account_id": "...", "id": "..."}` — ask first |

The automation belongs to `account_id`. Its steps can still act on other accounts
through their own `account_id` param (see `patterns.md`).

## Run and inspect

| Do | Call | Params |
|---|---|---|
| Run the live version | `workflow_execute` | `{"account_id": "...", "id": "...", "inputs": {...}}` |
| Run the current draft | `workflow_execute` | same + `"use_draft_definition": true` |
| Run as if an event had fired it | `workflow_execute` | same, with `"inputs": {"event": {<full event payload>}}`. Passing only `event_id` gives the run `Inputs.event = {id}` and nothing else — build the payload as in `data-shapes.md` |
| Dry run (see Safety in SKILL.md) | `workflow_dryrun_execute` | `{"account_id": "...", "definition": {...}, "inputs": {...}, "name": "..."}` → `{dryrun_id, execution_id}` |
| Run one task on its own | `workflow_execute_task` | `{"account_id": "...", "task_type": "...", "params": {...}}` |
| List runs | `workflow_list_executions` | `{"account_id": "...", "id": "<workflow id>", "limit": 20, "status": "FAILED"}` |
| One run, with every step's input/output/error | `workflow_get_execution` | `{"account_id": "...", "workflow_id": "...", "id": "<execution id>"}` |
| Runs started by an event | `workflow_list_executions_for_event` | `{"account_id": "...", "event_id": "..."}` |
| Re-run a run | `workflow_replay_execution` | `{"account_id": "...", "workflow_id": "...", "execution_id": "...", "inputs": {...}}` |
| Cancel | `workflow_cancel_execution` | `{"account_id": "...", "id": "<workflow id>", "execution_id": "..."}` |
| Answer an approval | `workflow_complete_approval` | `{"account_id", "workflow_id", "execution_id", "task_id", "status": "approve"/"reject", "comments"}` |

`workflow_execute`, `workflow_replay_execution` and `workflow_execute_task` really
run. `workflow_execute_task` with a read-only task type is the safest way to try a
step's params against a real resource.

## Errors

| Symptom | Likely cause |
|---|---|
| 401 | Token missing, mistyped (keep the `sk-nb-` prefix), expired or revoked |
| 403 / "unauthorized" | Token's user lacks the right on that account or module, or the token was limited to other modules |
| `account_id is required` | Most workflow actions need the account the automation belongs to |
| `action '…' not supported` | Wrong method name — check the API reference |
| Validation errors from `workflow_validate` / `workflow_create` | Read each one; they name the task id and field |
| HTTP 200 but `{"status": "FAILED", "error": ...}` | `workflow_execute_task` and `workflow_dryrun_execute` report failures in the body — always check `status` |
