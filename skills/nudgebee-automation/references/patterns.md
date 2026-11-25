# Design patterns

Use these to turn "an automation for alert X on host Y in account Z" into something
built once and reused. Working examples of patterns 1–3 are in this repository
(`workflows/`, `plugins/`, `catalogues/`).

## 1. Parent and children (orchestrator + plugins)

```
parent (thin)                    children (one per job, reusable)
─────────────                    ───────────────────────────────
trigger                          diagnose-cpu      inputs: account_id, region, instance_id
  → pick the job                 diagnose-disk     inputs: account_id, region, instance_id, volume
  → call child(ren) ───────────► restart-service   inputs: account_id, host, service
  → approval (if a change)       stop-start-vm     inputs: account_id, region, instance_id
  → call change child            …
  → call diagnose child again
  → notify / ticket
```

- Parent calls a child with `core.call-workflow`, by `workflow_name`, passing
  `inputs`. The child declares those as `definition.inputs` and has a `manual`
  trigger, so it can also be run by hand.
- A child does one job and returns a clear `output`. It never decides *whether*
  to act — the parent does.
- Children are where reuse happens: a new alert type usually needs a new parent
  (or a new case in one) and *no* new child.
- Read a child's result in the parent as `{{ Tasks.<step_id>.output.<field> }}`.

## 2. One automation, many accounts

- Event trigger: `params.accounts` = the list of accounts to listen to (UI:
  **Listen to accounts**). Empty = only the automation's own account.
- Every infrastructure step: `account_id: '{{ Inputs.event.cloud_account_id }}'`,
  and the parent passes it to children as an input.
- Region, resource id and names come from the event (`Inputs.event.labels.*`,
  `subject_name`, …) — check a real event for which fields carry them.
- The run's owner needs rights on every listed account; saving checks this.
- New accounts are **not** added automatically. Adding an account = adding it to
  the trigger's list. Tell the user.

## 3. Pick the job from the event (catalogue)

When one parent handles several alert kinds:

- Map alert → job with `core.switch` on a field of the event, or a
  `data.transform` that reads a small mapping kept in `{{ Configs.* }}`.
- Keep the mapping *data*, not logic: adding an alert kind should mean adding one
  entry, not editing steps.
- Prefer facts from the event and the platform (account, region, resource,
  labels) over lookups. Only keep in the mapping what discovery can't know
  (e.g. "this service runs as a systemd unit on host X").

## 4. Diagnose → approve → act → verify

1. Diagnose child (read-only): collect metrics, logs, recent changes, state.
2. Decide: often an AI step (`llm.*`) summarises the diagnosis and recommends; the
   automation still only *offers* the action.
3. `core.approval` with the diagnosis in the message, an explicit `timeout`, and
   approve/reject options. The change step has
   `if: "{{ Tasks.<approval>.output.status == 'approve' }}"`.
4. Change child.
5. Re-run the diagnose child; report before/after.
6. A reject or timeout path that notifies and changes nothing.

## 5. Noise, duplicates and loops

- Alerts repeat and flap. Before acting, check whether the problem is still there
  (diagnose first) and whether the same resource was acted on recently
  (`set_state` / `{{ State.* }}` as a cooldown key, e.g. `restart:<instance_id>`).
- If an alert exists at several severities, trigger on one (filter on priority or
  the alert name) so one problem runs once.
- Never let an automation's own action fire the trigger that started it.

## 6. Failure handling

- Set `timeout` on the automation and on slow steps. Retries: `retry_policy` on the
  definition, or a step's `failure_policy` (`retry` + `action`) — check the task
  definition and the docs for the exact fields, and only retry steps that are safe
  to repeat.
- Add a failure notification (hook or an `if` step on failure) so a broken run is
  seen, not silent.
- Make change steps idempotent: check state first, act only if needed.

## 7. Naming, tags, versions

- Names: `<job>-<platform>` for children (`diagnose-cpu-aws`), `<trigger>-<job>` for
  parents (`on-cpu-alarm`).
- Tags: `layer: parent|child`, `job: ...`, `platform: ...` — the list view filters on them.
- `llm_description`: when to use it, in one or two sentences.
- Change a live automation through a new draft → validate → publish. Roll back by
  making the previous version live.

## 8. Scale

- Fan out over a list with `core.foreach` or a task `matrix` (max 1,000
  combinations); keep per-item work in a child.
- One child per job across all accounts — the count of automations should grow with
  the number of *jobs*, not accounts, hosts or alert names.

## Anti-patterns — push back when you see them

| Request | Problem | Instead |
|---|---|---|
| A copy of the automation per account | N copies to fix every time | Pattern 2 |
| A copy per alert name or severity | Same job duplicated | One parent case → shared child |
| Ids, hostnames, regions typed into steps | Breaks on the next resource | Inputs from the event |
| Change step with no approval | Unreviewed change on every alert | Pattern 4 |
| "Test it with dry run" on a change step | Dry run can really run it | Run the read-only child; ask before any change |
| Looking up data from an internal database the automation shouldn't reach | Fragile, over-privileged | Event fields, platform APIs, Configs |
| One giant automation doing everything | Can't reuse or test parts | Parent + children |
