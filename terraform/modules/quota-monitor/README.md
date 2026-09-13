# quota-monitor module

Terraform port of `connect-quota-monitor-cfn.yaml` (no longer present in this repo). Deploys the enhanced
Amazon Connect Service Quota Monitor: a scheduled Lambda function that
discovers Connect instances, checks quota utilization against CloudWatch and
Service Quotas, and sends consolidated alerts through SNS.

## What it deploys

- Lambda function (`lambda_function.main`, Python 3.13) packaged directly from
  the repo root's `lambda_function.py` and `quota_definitions.json` via
  `data.archive_file`. Reserved concurrency of 1, a dead-letter queue, and
  KMS-encrypted environment variables.
- SQS dead-letter queue for failed invocations.
- EventBridge rule on `var.schedule_expression` (default `rate(1 hour)`) that
  triggers the Lambda, plus the invoke permission.
- CloudWatch log group with `var.log_retention_days` retention.
- Two CloudWatch alarms (Lambda errors, Lambda duration) that publish to the
  alert SNS topic.
- SNS topic (KMS-encrypted) for alerts, with an optional email subscription
  when `var.notification_email` is set.
- Three KMS keys (SNS, DynamoDB, Lambda), each with a rotation-enabled key and
  an alias.
- DynamoDB table for metrics/report storage when `var.use_dynamodb` is true,
  with a GSI on `instance_id`, TTL, point-in-time recovery, and KMS
  encryption.
- Two S3 buckets: metrics/reports and access logs, each with versioning,
  AES256 encryption, and a public access block.
- IAM execution role scoped to the Connect discovery/list APIs, Service
  Quotas, CloudWatch, SNS, SQS, KMS, and (conditionally) S3/DynamoDB storage.
- Optional security group and Lambda VPC config when `var.vpc_id` is set.

## Parity notes (deliberate differences from the CFN template)

1. **No placeholder+deploy.sh step.** The CFN template ships a bootstrap
   `ZipFile` inline stub and relies on `deploy.sh` (no longer present in this
   repo) to upload the real code to
   S3 and call `UpdateFunctionCode` after stack create. Terraform's
   `data.archive_file` bundles the real `lambda_function.py` and
   `quota_definitions.json` directly into the deployment zip at plan time, so
   a single `terraform apply` produces a fully working function. The CFN
   template's deployment bucket and its `DEPLOYMENT_METHOD`/`DEPLOYMENT_BUCKET`
   environment variables served that upload path and are not created by this
   module.
2. **DynamoDB table name and IAM scope now point at the table this module
   actually creates.** The CFN template's `DynamoDBTableName` parameter
   (default `ConnectQuotaMonitor`) is used for both the `DYNAMODB_TABLE`
   environment variable and the `DynamoDBStoragePolicy` IAM resource ARN, but
   the table CloudFormation actually creates is named
   `${AWS::StackName}-metrics` (a different, unrelated name). Unless someone
   manually renames the parameter to match, the deployed role can never
   read/write the table the stack creates. This port ties `DYNAMODB_TABLE`
   and the IAM policy to the real `aws_dynamodb_table.metrics` ARN and name so
   DynamoDB storage works without extra configuration.
3. **DynamoDB GSI has no provisioned throughput.** The CFN template sets
   `BillingMode: PAY_PER_REQUEST` on the table but also sets
   `ProvisionedThroughput` (5 read/5 write) on the `InstanceIdIndex` GSI. The
   DynamoDB API rejects `ProvisionedThroughput` on any index when the table
   billing mode is `PAY_PER_REQUEST`, so the CFN template as written would
   fail at table-create time. This port omits read/write capacity on the GSI
   so the table actually creates successfully.
4. **Lambda runtime is python3.13**, matching the CFN template's runtime
   parameter default.

## Inputs

| Name | Type | Default | Description |
|------|------|---------|-------------|
| `name_prefix` | string | (required) | Prefix for every resource name. |
| `notification_email` | string | `""` | Email subscribed to the alert SNS topic. Empty skips the subscription. |
| `threshold_percentage` | number | `80` | Quota utilization alert threshold (1-99). |
| `schedule_expression` | string | `"rate(1 hour)"` | EventBridge schedule for the monitor Lambda. |
| `use_dynamodb` | bool | `true` | Create the DynamoDB table and its IAM/KMS resources. |
| `use_s3_storage` | bool | `true` | Create the metrics and access-log S3 buckets and their IAM policy. |
| `lambda_timeout` | number | see note below | Lambda timeout in seconds (60-900). |
| `lambda_memory` | number | `512` | Lambda memory in MB (256-10240). |
| `log_retention_days` | number | `30` | CloudWatch Logs retention. |
| `vpc_id` | string | `""` | Optional VPC ID for the Lambda. |
| `subnet_ids` | list(string) | `[]` | Required when `vpc_id` is set. |
| `force_destroy` | bool | `false` | Allow `terraform destroy` to delete non-empty S3 buckets. |
| `repo_root` | string | (required) | Absolute path to the directory containing `lambda_function.py` and `quota_definitions.json`. |
| `tags` | map(string) | `{}` | Tags merged onto every resource. |

This module's own default for `lambda_timeout` is never used when you deploy
through the root Terraform config in this repository, because
`terraform/main.tf` always passes `var.lambda_timeout` into the module
explicitly. The effective value comes from the root `terraform/variables.tf`
(default `600`), or from whatever `terraform.tfvars` sets, since
`terraform.tfvars.example` pins it to `300`. Check `terraform.tfvars` if you
need to know the value a given deployment is actually using.

## Outputs

See `outputs.tf`: `lambda_function_name`, `lambda_function_arn`,
`sns_topic_arn`, `s3_bucket_name`, `dynamodb_table_name`,
`dead_letter_queue_url`, `dead_letter_queue_arn`.

## IAM

The execution role's inline policy grants read-only List/Describe/Get access
only to the services the monitor actually calls: Connect, Connect Cases,
Connect Campaigns, Service Quotas (`ListServiceQuotas` and `ListServices` only,
`GetServiceQuota` is intentionally omitted since the monitor batches applied
limits per service instead of one call per quota), CloudWatch, and STS. These use `Resource: "*"` because they are
account-wide discovery APIs (you cannot scope `ListInstances` to a specific
instance ARN before you have called it). `sns:Publish` is scoped to the
module's own SNS topic, SQS access is scoped to the module's own
dead-letter queue, and KMS access is scoped to the module's own keys.
