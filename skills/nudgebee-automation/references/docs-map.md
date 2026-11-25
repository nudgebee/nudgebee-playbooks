# Docs map

Public docs: https://docs.nudgebee.com/docs/ — source markdown is public at
`https://raw.githubusercontent.com/nudgebee/nudgebee-docs/main/doc-server/docs/<path>.md`.
Fetch the raw markdown (cleaner than the HTML). Page list:
`https://docs.nudgebee.com/sitemap.xml`. Read the page; don't answer from memory —
the docs change with the product.

## Automations (`features/workflow-builder/`)

| Question | Page |
|---|---|
| What the builder is, UI steps, variables cheatsheet, limits, whose permissions a run uses | `index` |
| Which tool to use: event playbook vs automation vs auto-optimize | `automation-architecture-guide` |
| Trigger types, event filters, listening to several accounts, lifecycle phases, event payload | `triggers` |
| Event-triggered automation, step by step | `create-event-automation` |
| Optimization-triggered automation and its payload | `create-optimization-automation` |
| How tasks reference each other; task categories | `workflow-tasks-reference` |
| Task families | `core-tasks`, `ai-tasks`, `cloud-cli-tasks`, `kubernetes-tasks`, `scripting-tasks`, `database-tasks`, `observability-tasks`, `notification-tasks`, `ticket-tasks`, `integration-tasks`, `source-control-tasks`, `cicd-tasks`, `data-tasks`, `event-tasks`, `network-tasks`, `message-queue-tasks`, `crypto-tasks` |
| Validate, dry run and what it does / doesn't skip | `workflow-dry-run-validation` |
| Approvals | `workflow-operations-approvals` |
| Drafts, versions, rollback | `workflow-versioning` |
| It did not trigger | `workflow-did-not-trigger` |

## Integrations (`integrations/`)

`index` lists them. Folders: `Notifications`, `Tickets`, `Observability`,
`Databases`, `In-Memory`, `Messaging Queue`, `Servers`, `Code Repository`, `CICD`,
`Webhooks`, `MCP`, `LLM`, `Docs`, `Authentication`. When a step needs an
integration (Slack, SSH, a database, a ticketing tool), read that integration's page
for setup, then check `integrations_list` to see whether it is already connected on
the account.

## API (`api-docs/`)

| Question | Page |
|---|---|
| Tokens: create, scope, expiry, how to send | `api-tokens` |
| Every action with argument and return types | `index` (large — search it for the action name) |
