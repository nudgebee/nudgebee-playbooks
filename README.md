# Nudgebee playbooks

Nudgebee workflow definitions for cross-platform service operations, arranged the
way an Ansible playbook repository is: generic orchestrators on top, per-platform
implementations underneath, and a catalogue that decides which implementation a
given service gets.

These are **Nudgebee workflows**, not Ansible playbooks. They run on the Nudgebee
workflow engine. `ansible-playbook` will not read them.

## Layout

```
workflows/                       orchestrators — platform-agnostic
  wf-000-catalogue-lookup.yml    service name -> platform, target, connection details
  wf-003-restart.yml             restart, with Slack approval
  wf-010-diagnose.yml            diagnose, read-only

plugins/                         per-platform implementations
  kubernetes/{restart,diagnose}.yml
  linux/{restart,diagnose}.yml
  cloud/{restart,diagnose}.yml   AWS

catalogues/
  demo.json                      service -> platform overrides + plugin index
```

An orchestrator never names a platform. It calls the catalogue lookup, switches on
the platform that comes back, and calls the matching plugin. Adding a platform means
adding `plugins/<platform>/{restart,diagnose}.yml` and one `case` in each
orchestrator's `core.switch` — no other file changes.

## How WF-003 runs

```
wf-000-catalogue-lookup      resolve the service (knowledge graph + catalogue overrides)
        |
   core.approval             Slack, 2h timeout, approve/reject
        |
   core.switch               on platform
     |     |     |
   k8s  linux  cloud         plugins/<platform>/restart.yml
        |
 wf-010-diagnose             post-restart health check
        |
   llm.summary               plain-language result
        |
 notifications.im            Slack
```

Rejection short-circuits: `notify_rejected` fires and nothing is touched.

## File format

Each file is one workflow in the canonical Nudgebee YAML shape:

```yaml
name: demo-restart-k8s
description: ...
definition:
  triggers: [...]
  inputs: [...]
  tasks: [...]
  output: {...}
```

The `definition` block is byte-equivalent to what the engine stores, so these files
can be imported back into a Nudgebee account as-is. `layout` keys are canvas
coordinates for the workflow editor and have no effect on execution.

## Configuration

Secrets and account-specific identifiers are not in this repo. The workflows
reference them as `{{ Configs.* }}`, resolved from the Nudgebee account config at
run time:

| Config key | Used by | What it is |
|---|---|---|
| `tenant_id` | wf-000 | Tenant whose knowledge graph is queried |
| `kg_db_integration_id` | wf-000 | Postgres integration holding the knowledge graph |
| `linux_ssh_integration_id` | wf-000, catalogues/demo.json | SSH integration for Linux targets |
| `slack_warroom_channel` | wf-003 | Slack channel for approval and result messages |

`aws_account` and `region` are not configs — they come out of the catalogue lookup
per service.

## Task types used

| Type | Where |
|---|---|
| `core.call-workflow` | orchestrators calling the lookup and the plugins |
| `core.switch` | platform dispatch |
| `core.approval` | Slack approve/reject gate on WF-003 |
| `dbms.query` | knowledge graph lookup |
| `data.transform` | JSONata resolution of discovery + overrides |
| `cloud.k8s.cli` | kubernetes plugins |
| `cloud.aws.cli` | cloud plugins |
| `integrations.ssh` | linux plugins |
| `llm.summary` | plain-language interpretation |
| `notifications.im` | Slack |

## Safety

- WF-010 and every `diagnose` plugin are read-only.
- WF-003 cannot mutate anything without an explicit Slack approval — the switch is
  gated on `Tasks.approval.output.status == 'approve'`.
- Restarts are in-place (`rollout restart`, `systemctl restart`, `reboot-instances`).
  Nothing here deletes, scales or replaces a resource.

## Claude skill: build automations with Claude

`skills/nudgebee-automation/` is a [Claude Code](https://docs.claude.com/en/docs/claude-code)
skill that designs, builds, tests and debugs Nudgebee automations through the
Nudgebee API, using the patterns in this repository (parent + reusable children,
one automation across many accounts, approval before any change).

It reads the live task catalogue, triggers and your real events from the API, reads
the public docs for feature behaviour and UI steps, and checks every automation
(validate + trigger match against real events) before saving it.

Install in Claude Code (one command):

```
/plugin install nudgebee-automation --marketplace nudgebee/nudgebee-playbooks
```

On older Claude Code versions: `/plugin marketplace add nudgebee/nudgebee-playbooks`,
then `/plugin install nudgebee-automation@nudgebee`. Without plugins, copy the folder
instead:

```bash
git clone https://github.com/nudgebee/nudgebee-playbooks
mkdir -p ~/.claude/skills
cp -r nudgebee-playbooks/skills/nudgebee-automation ~/.claude/skills/
```

Create an API token in Nudgebee (avatar menu → **API Tokens** → type **Platform**), then:

```bash
export NB_API_TOKEN=sk-nb-...
export NB_URL=https://app.nudgebee.com   # or your own domain
```

Ask Claude, for example: *"Build a Nudgebee automation that diagnoses EC2 CPU alarms
across all my AWS accounts and offers a restart with Slack approval."*
