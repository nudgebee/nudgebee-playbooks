# Debugging

Start from evidence: the run (`workflow_get_execution`) or, if there is no run, the
trigger. Read the docs page for the symptom (`docs-map.md`) before guessing.

## It did not run

1. Is it `ACTIVE`, and is the **live** version the one you think? (`workflow_get`,
   `workflow_list_versions`). Edits to a draft do nothing until published.
2. Take the event that should have started it (`events_list`) and run
   `workflow_check_trigger_match` with the live trigger and that event. The
   `reason` says which part failed.
3. Common causes: the event came from an account not in the trigger's account list;
   the filter compares a field that is missing or spelled differently on the real
   event; wrong lifecycle phase (`params.on`, default `event.created`); filter
   rendered to something other than `true`.
4. `workflow_list_executions_for_event` with the event id shows whether a run did
   start and failed early.
5. Docs: "Workflow did not trigger".

## It ran and failed

1. `workflow_list_executions` (`status: FAILED`) → `workflow_get_execution`.
2. Find the first failed step; read its rendered input, output and error.
3. Typical causes:
   - **Template rendered empty** — a referenced field isn't there. Compare the
     template with the actual output of the earlier step in the same execution.
   - **Permission error** — triggered runs act as the automation's owner; check
     that user's rights on the account the step targeted.
   - **Wrong account** — the step had no `account_id`, so it ran on the
     automation's own account instead of the event's.
   - **Timeout** — the default step timeout is short; set `timeout` on slow steps
     and on approvals.
   - **Integration missing** on that account (Slack, SSH, DB…).
4. Fix in the draft, `workflow_validate`, then replay the failed run
   (`workflow_replay_execution`) if it is safe to repeat.

## Engine behaviour that surprises people

- A `data.transform` result is under `.output.data`, not `.output`; so is a cloud
  CLI step's command output.
- A child called with `core.call-workflow` returns its `definition.output` as a
  JSON **string** under `Tasks.<id>.output.output` (plus `run_id`). Keep child
  outputs small and flat, and parse before reading fields.
- `workflow_validate` and `workflow_create` refuse a parent whose
  `core.call-workflow` names a child that doesn't exist yet — create children first.
- A new automation is live and PAUSED: it can be run by hand and called by a parent,
  but its own triggers don't fire until `workflow_resume`.
- Steps named in a `core.switch` case's `next` must not have their own
  `depends_on`; the switch decides when they run. Chained switches are fragile —
  prefer one switch plus `if` conditions.
- Inside trigger filters the event is `event`; inside step params it is
  `Inputs.event`. A webhook body is `webhook_payload` in filters and
  `Inputs.webhook_json` in steps.
- `definition.version` is `"v1"`.
- Secrets are masked in inputs, logs and errors, but **not** in task outputs —
  don't echo them.
- Schedules use the live version's cron.

## Before telling the user it works

Show them the evidence: the execution id, the steps that ran, the output that
proves the job was done (or the trigger-match result for events it should and
should not catch). Say plainly what was not tested.
