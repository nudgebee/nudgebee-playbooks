# Data shapes

How to find out what data a trigger or step gives you. Always confirm on real data
from the user's tenant; field availability depends on the event source.

## The definition

```jsonc
{
  "name": "on-cpu-alarm",                 // unique per account
  "description": "...",
  "tags": {"layer": "parent", "job": "cpu"},
  "definition": {
    "version": "v1",
    "llm_description": "When to use this automation",
    "inputs":   [{"id": "instance_id", "type": "string", "required": true, "description": "..."}],
    "triggers": [{"type": "event", "params": {"filter": "{{ ... }}", "accounts": ["<account id>"]}}],
    "tasks": [
      {"id": "diagnose", "type": "core.call-workflow",
       "params": {"workflow_name": "diagnose-cpu-aws",
                  "inputs": {"account_id": "{{ Inputs.event.cloud_account_id }}"}},
       "timeout": "10m"}
    ],
    "output":  {"summary": "{{ Tasks.diagnose.output }}"},
    "timeout": "1h"
  }
}
```

Task fields: `id`, `type`, `params`, `depends_on`, `if`, `timeout`, `matrix`,
`set_vars`, `set_state`, `failure_policy`, `hooks`, `disabled`, and `tasks` for
group / loop bodies.

Trigger types: `manual`, `schedule`, `webhook`, `event`, `optimization`. Params per
type are in the docs page "Configuring Triggers".

## A task's inputs and outputs

`workflow_list_taskdefinitions` → `tasks[]`, each with `name`, `description`,
`input_schema`, `output_schema`, `runtime_notes`, `aliases`. Use the schema for
params; use `output_schema` to know what `{{ Tasks.<id>.output.* }}` will contain.
When the schema is loose, run the step once (`workflow_execute_task`, read-only
types only) and read the real output.

## The event a trigger receives

Documented fields (docs: "Configuring Triggers" → Event Payload Schema): `id`,
`cloud_account_id`, `event_type`, `source`, `cluster`, `subject_namespace`,
`subject_name`, `priority`, `status`, `labels`. Cloud alarms usually carry resource
id, region and metric details in `labels`.

Get the truth for a given alert:

1. Best: an execution of any event-triggered automation that ran on that kind of
   event — `workflow_get_execution` → `inputs.event` is exactly what a trigger gets.
2. Otherwise `events_list` filtered to that alert (e.g.
   `{"where": {"source": {"_eq": "..."}, "account_id": {"_eq": "..."}}, "limit": 3,
   "order_by": [{"column": "created_at", "order": "desc"}]}`). Its rows are **not**
   trigger payloads yet: `labels` is a JSON *string* (parse it into an object), and
   `event_type` (= `aggregation_key`) and `lifecycle_phase` are missing. A filter on
   `event.labels.*` silently fails to match an unconverted row.
3. Show the user the fields that matter (account, resource id, region, alert name,
   priority, labels).
4. Use exactly those names in the filter (`event.*`) and in steps (`Inputs.event.*`).
5. Prove it with `workflow_check_trigger_match`: the converted events as `payload`
   must give `matched: true`, and a copy with the key field changed must give
   `false`.

Seen on a cloud-alarm event: `source`, `aggregation_key` (the alarm name),
`cloud_account_id`, `cloud_resource_id`, `subject_name`, `priority`, and in
`labels` things like `aws_region`, `service_name`, `resource_id`, `resource_arn`,
the metric name and resource tags (`tag.<key>`). Check your own events — sources
differ.

## Template variables

`Inputs.*`, `Inputs.event.*`, `Inputs.webhook_json.*`, `Tasks['id'].output.*`,
`Configs.*`, `Secrets.*`, `Vars.*`, `State.*`, `Matrix.*`, `LoopItem.*`, date/time
helpers. Full table: docs "Workflow Builder" → Variable Reference Cheatsheet.
