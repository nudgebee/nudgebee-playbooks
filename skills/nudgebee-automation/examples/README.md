# Examples

A parent and a child that were created and run end to end through the API:

- `describe-alarm-aws.json` — child. Inputs `account_id`, `alarm_name`, `region`;
  one read-only AWS CLI step on the account it is given.
- `on-cloudwatch-alarm.json` — parent. Event trigger listening to several accounts
  (replace `<ACCOUNT_ID_n>`), filtered to RDS CloudWatch alarms; passes
  `Inputs.event.cloud_account_id`, `Inputs.event.aggregation_key` and
  `Inputs.event.labels.aws_region` to the child, then prints the result.

Create the child first, then the parent:

```bash
scripts/nb workflow_create "{\"account_id\":\"$ACCOUNT\",\"workflow\":$(cat examples/describe-alarm-aws.json)}"
scripts/nb workflow_create "{\"account_id\":\"$ACCOUNT\",\"workflow\":$(cat examples/on-cloudwatch-alarm.json)}"
```

Both start paused. Change the filter to your alarm before resuming the parent.
