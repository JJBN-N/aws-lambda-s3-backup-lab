# AWS Serverless Data Backup Pipeline
AWS Lambda + DynamoDB + Scheduled S3 Backups


A beginner–intermediate AWS project that fetches sample users from JSONPlaceholder, stores them in DynamoDB, and exports timestamped JSON backups to a private S3 bucket every 2 minutes.

## Architecture

```text
JSONPlaceholder API
        |
        v
Lambda 1: import-jsonplaceholder-users
        |  DynamoDB PutItem
        v
DynamoDB: jsonplaceholder-users
        |  DynamoDB Scan
        v
Lambda 2: export-dynamodb-to-s3 <--- EventBridge Scheduler
        |                              rate(2 minutes)
        |  S3 PutObject
        v
Private S3 bucket: backups/YYYY-MM-DD/users-....json

Both Lambda functions -> CloudWatch Logs and Metrics
```

**Trigger flow:** Run Lambda 1 manually to import users. EventBridge Scheduler invokes Lambda 2 every 2 minutes. Uploading a ZIP file to S3 does not trigger either function.

## AWS resources

- DynamoDB table: `jsonplaceholder-users`
  - Partition key: `id` (Number)
- Lambda 1: `import-jsonplaceholder-users`
- Lambda 2: `export-dynamodb-to-s3`
- Private S3 bucket for JSON exports
- EventBridge schedule: `dynamodb-backup-every-2min`
- CloudWatch for logs and metrics

Use the **same AWS Region** for all resources.

## Prerequisites

- AWS account with permission to create IAM roles and policies, DynamoDB tables, Lambda functions, S3 buckets, and EventBridge schedules.
- The extracted assignment folder and a web browser. AWS CLI is not required.
- Python 3.12 or another supported Lambda Python runtime.

Keep the Lambda functions outside a custom VPC for this public API exercise unless internet routing is configured. Never commit AWS credentials, secrets, account-specific values, or personal data to Git.

## Project files

```text
.
├── README.md
├── AWS_Lamda_DynamoDB_S3_Backup_Lab_Report.md
├── lambda_zips/
│   ├── lambda-import-users.zip
│   └── lambda-backup-to-s3.zip
├── lambda_src/
│   ├── import_users/lambda_function.py
│   └── backup_users/lambda_function.py
└── iam_policies/
    ├── lambda-import-putitem-policy.json
    └── lambda-backup-export-policy.json
```

Upload the **inner function ZIPs** shown above to their matching Lambda functions—not the outer assignment ZIP. Both functions use this handler:

```text
lambda_function.lambda_handler
```

## Setup and deployment

### 1. Create DynamoDB and S3

1. In your chosen Region, create a DynamoDB table named `jsonplaceholder-users`.
2. Set `id` as the partition key with type **Number**. Do not add a sort key.
3. Create a globally unique S3 bucket in the same Region.
4. Keep **Block all public access** and default encryption enabled. No `backups/` folder needs to be created; S3 creates the prefix when an object is uploaded.

### 2. Deploy Lambda 1: import users

1. Create a Python Lambda function named `import-jsonplaceholder-users`.
2. Create or use an execution role with `AWSLambdaBasicExecutionRole` for CloudWatch logging.
3. Attach the policy in `iam_policies/lambda-import-putitem-policy.json`. Replace `REGION` and `ACCOUNT_ID` so its resource points to your DynamoDB table ARN. Grant only `dynamodb:PutItem` on that table.
4. Set environment variable:
   - `TABLE_NAME=jsonplaceholder-users`
5. Upload `lambda_zips/lambda-import-users.zip`.
6. Set the handler to `lambda_function.lambda_handler`.
7. Use 128 MB memory and a 30-second timeout.
8. Test with an empty JSON event: `{}`.

The function fetches the 10 sample users from `https://jsonplaceholder.typicode.com/users` and writes them to DynamoDB with `PutItem`.

### 3. Deploy Lambda 2: export to S3

1. Create a Python Lambda function named `export-dynamodb-to-s3`.
2. Give its execution role `AWSLambdaBasicExecutionRole` and the permissions in `iam_policies/lambda-backup-export-policy.json`.
3. Replace `REGION`, `ACCOUNT_ID`, and `YOUR_UNIQUE_BUCKET_NAME` in the policy. Grant only:
   - `dynamodb:Scan` on the demo table.
   - `s3:PutObject` under the bucket's `backups/` prefix.
4. Set environment variables:
   - `TABLE_NAME=jsonplaceholder-users`
   - `BUCKET_NAME=<your-private-bucket-name>`
   - `BACKUP_PREFIX=backups`
5. Upload `lambda_zips/lambda-backup-to-s3.zip`.
6. Set the handler to `lambda_function.lambda_handler`.
7. Use 128 MB memory and a 30-second timeout.
8. Test with `{}`.

The export function scans the table (including scan pagination), converts DynamoDB `Decimal` values to JSON-compatible values, and uploads a JSON array. Each run creates a unique object key using the UTC timestamp and Lambda request ID, so previous exports are not overwritten.

### 4. Schedule automatic exports

In **Amazon EventBridge Scheduler**, create a recurring rate-based schedule:

- Name: `dynamodb-backup-every-2min`
- Schedule expression: `rate(2 minutes)`
- Flexible time window: Off
- Target: Lambda function `export-dynamodb-to-s3`
- Input: `{}`
- Execution role: create a Scheduler role that can invoke the target Lambda

The Scheduler execution role is separate from the Lambda function's execution role. Use the same Region as the other resources.

## Verify the pipeline

1. Run Lambda 1 and confirm that approximately 10 users appear in DynamoDB.
2. Run Lambda 2 manually and confirm a JSON file appears under `backups/YYYY-MM-DD/` in S3.
3. Run Lambda 2 again and confirm a second, uniquely named file is created.
4. Enable the schedule and confirm at least two scheduled invocations create additional S3 files.
5. Check each Lambda's CloudWatch log group and metrics (`Invocations`, `Errors`, and `Duration`).

Running the import function more than once does not create duplicate users: `PutItem` replaces an item with the same partition-key value (`id`).

## IAM and security

- Lambda 1 needs `dynamodb:PutItem` to write imported users.
- Lambda 2 needs `dynamodb:Scan` to read table records and `s3:PutObject` to create backup objects.
- EventBridge Scheduler needs permission to invoke Lambda 2 through its own execution role.
- Keep permissions scoped to the required table and S3 prefix. Do not use `AdministratorAccess` for this lab.
- Do not hardcode AWS access keys. Boto3 uses the Lambda execution role credentials.
- Keep the S3 bucket private and Block Public Access enabled.

## Limitations

This project creates JSON exports from a DynamoDB table scan. It is a learning exercise, **not a production-grade backup**:

- A table scan is not a transactionally consistent point-in-time snapshot.
- The JSON export does not preserve table configuration, indexes, alarms, or access policies.
- Large scans may exceed Lambda time or memory limits and consume read capacity.
- A 2-minute schedule can create about 30 files per hour (720 per day), increasing storage and request costs.

For production disaster recovery, consider DynamoDB Point-in-Time Recovery (PITR) and managed DynamoDB Export to S3. Set retention and recovery objectives according to the workload.


## Cost and cleanup

AWS may charge for DynamoDB requests, Lambda execution, S3 storage and requests, EventBridge Scheduler, and CloudWatch logs. Pricing and free-tier eligibility vary. A 2-minute schedule can create thousands of objects per month.

After testing:

1. Disable and delete the EventBridge schedule.
2. Delete both Lambda functions.
3. Delete the DynamoDB table if its data is no longer needed.
4. Empty the S3 bucket (including versions, if versioning is enabled), then delete it.
5. Remove lab-only CloudWatch log groups, IAM roles, and policies if no longer needed.

## Official references

- [Deploy Python Lambda functions with ZIP archives](https://docs.aws.amazon.com/lambda/latest/dg/python-package.html)
- [Python Lambda handlers](https://docs.aws.amazon.com/lambda/latest/dg/python-handler.html)
- [EventBridge Scheduler invoking Lambda](https://docs.aws.amazon.com/lambda/latest/dg/with-eventbridge-scheduler.html)
- [DynamoDB Scan with Boto3](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/dynamodb/table/scan.html)
- [Lambda execution roles](https://docs.aws.amazon.com/lambda/latest/dg/lambda-intro-execution-role.html)
- [Amazon S3 pricing](https://aws.amazon.com/s3/pricing/)
- [Amazon DynamoDB pricing](https://aws.amazon.com/dynamodb/pricing/)
