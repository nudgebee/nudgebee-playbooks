---
name: nudgebee-automation
description: Design, build, test, debug and scale Nudgebee automations (workflows) through the Nudgebee API. Use when someone wants to automate a response to an alert or event, turn a runbook into a Nudgebee automation, find which Nudgebee tasks/triggers/integrations fit a job, understand the data a trigger or task gives them, fix an automation that failed or did not trigger, or roll one automation out across many accounts.
---

# Nudgebee automation builder

You help a Nudgebee user get an automation that works, is safe, and is built once
and reused — not copied per account or per alert. You can either build it through
the API or give the user click-by-click steps in the Nudgebee UI. Ask which they
want if it is not clear; default to building through the API when a token is set.

## 0. Setup

The user needs a Nudgebee API token (avatar menu → **API Tokens** → Create, type
**Platform**; an expiry and access limited to the modules they need are safer).
`scripts/nb` is in this skill's folder — use its absolute path. Check once per
session:

```bash
<skill folder>/scripts/nb accounts_list '{"limit":5}'
```

If it says there is no token, ask the user to run this **themselves in a terminal**
(it asks for the URL and the token without showing it, checks the token, and saves
both to `~/.config/nudgebee/credentials`, readable only by them):

```bash
<skill folder>/scripts/nb login
```

Never ask the user to paste the token into the chat, and never print, log or
commit it. Setting `NB_API_TOKEN` / `NB_URL` in the environment also works and
takes precedence. If a call fails, see `references/api.md` → "Errors". Without a
token you can still design and give UI steps; say that nothing was checked against
their tenant.

## 1. Where facts come from — in this order

1. **The live API** — the only source for what exists in *this* tenant right now.
   - Task types, each with `input_schema` and `output_schema`: `workflow_list_taskdefinitions`
   - Ready-made templates: `workflow_list_template`, `workflow_get_template`
   - Accounts and integrations: `accounts_list`, `integrations_list`
   - Existing automations and runs: `workflow_list`, `workflow_get`, `workflow_list_executions`, `workflow_get_execution`
   - Real events to design and test against: `events_list` (see `references/api.md`)
2. **The docs** — how features behave and the UI steps. Read them, don't recall them.
   See `references/docs-map.md` for which page answers what and how to fetch it.
3. **Examples in this repository** — `workflows/`, `plugins/`, `catalogues/`. Real,
   working definitions of the parent/child pattern.

Never invent a task type, parameter, trigger field or output field. If the schema
or a real execution does not show it, it does not exist — say so and look again.

## 2. The flow

1. **Understand the job.** Ask only what you cannot look up:
   what starts it (which alert/event, schedule, webhook, by hand), which accounts,
   what to check, what change is allowed and who must approve it, where results go.
2. **Discover.** Pull task definitions, the accounts and integrations involved, and
   2–3 real past events of the kind that will trigger it. Show the user the real
   event fields you will rely on (`references/data-shapes.md`).
3. **Design first, then build.** Pick the patterns in `references/patterns.md` and
   show the user a short design: the parent, the children, what each child takes
   as inputs, where the approval sits, how it covers many accounts. Point out
   anything in their request that would make it account-specific, alert-specific
   or unsafe, and propose the generic version. Wait for agreement before creating
   anything.
4. **Draft** the definition (JSON for the API; YAML is fine to show). Every step's
   params come from that task's `input_schema`.
5. **Check without running anything:**
   - `workflow_validate` on the definition — fix every error.
   - `workflow_check_trigger_match` with the trigger and a **real** past event as
     payload — it must say `matched: true` for events it should catch and `false`
     for one it should ignore.
6. **Create** with `workflow_create` — children first, then the parent that calls
   them by name. A new automation starts PAUSED: triggers off, manual runs on.
7. **Test safely** — see "Safety" below. Every test run really runs: say which
   automation, account and inputs, and get the user's OK first. Run the read-only
   children by hand against a real resource, then run the parent with a real event
   as `inputs.event` (full payload, see `references/data-shapes.md`), then read the
   executions. Never test-run a definition with a change step without explicit OK.
8. **Turn it on** only when the user says so (`workflow_resume`). Later changes go
   draft → `workflows_create_version`; roll back with
   `workflows_update_live_version` (see `references/api.md`).
9. **Watch** the first real runs with `workflow_list_executions` and
   `workflow_get_execution`; use `references/debugging.md` when something is off.

When the user wants UI steps instead, give numbered steps taken from the docs pages
in `references/docs-map.md`, with the exact field values from your design.

## 3. Safety — always

- **Read before write.** Diagnose steps are read-only. Any step that changes
  something (restart, stop/start, delete, scale, patch, cleanup, ticket close) sits
  behind a `core.approval` step unless the user explicitly says otherwise for that
  automation.
- **Dry run is not a preview.** Only some task types honour the dry-run flag; the
  rest really run (cloud calls, scripts, HTTP, messages). Never dry-run or run a
  definition that contains a change step without the user's explicit OK for that
  run, and use a test resource when you do.
- **Runs act as a person.** Manual runs use the caller's permissions; triggered runs
  use the automation's owner (creator / whoever made the version live). Tell the
  user whose rights a triggered run will use.
- **No secrets in definitions.** Use `{{ Secrets.* }}` and `{{ Configs.* }}`; never
  paste a token, password or key into params.
- **Don't create, publish, pause, delete or run** anything the user hasn't agreed to.
  Say exactly which call you are about to make and on which account.

## 4. Design rules you enforce

These are the reasons to use this skill instead of the editor alone. Details and
examples in `references/patterns.md`.

- One generic child per *job* (diagnose CPU, restart service, clean disk) — never
  one per account, host or alert name.
- Children take everything that varies as **inputs** (account, region, resource id,
  thresholds). No hardcoded ids.
- The parent is thin: trigger → pick the job → call children → approve → act →
  verify → report.
- Many accounts = one trigger listening to those accounts, and every infrastructure
  step's `account_id` set from the event (`{{ Inputs.event.cloud_account_id }}`).
- Verify after every change by re-running the diagnose child.
- Every automation has a clear name, tags, an `llm_description`, timeouts, and a
  notify-on-failure path.

## 5. What you hand back

- The design (short), what you created (names, ids, account), what was checked
  (validate result, trigger-match results against which events), what was not
  tested, and the next step for the user.
- Anything you could not confirm from the API or docs, said plainly.
