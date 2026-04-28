# Amazon Connect Automated Operational Review

An automated operational health assessment tool for Amazon Connect instances. Generates a comprehensive HTML report covering security, resilience, capacity, observability, and cost considerations.

## Overview

This solution deploys a Lambda function that analyzes your Amazon Connect instance and produces a detailed HTML report uploaded to S3. The report includes an executive summary with pass/fail/warn checks across multiple pillars.

## Report Sections

| Pillar | Checks |
|--------|--------|
| **Security** | Identity Management (SAML), S3 Data Encryption, Streaming Encryption |
| **Resilience** | Multi-AZ Architecture, Global Resiliency (ACGR), Carrier Diversity |
| **Operational Excellence** | API Throttling, Contact Flow Logging, KVS Retention Period |
| **Capacity Analysis** | Instance Resource Limits, Concurrency Limits, Account Level API Limits |
| **Observability** | CloudWatch Alarm Validation (Connect & KDS), Missed Calls Analysis |
| **Cost** | Phone Number Mix Analysis, Channel Usage Breakdown |

## Additional Features

- Data Storage & Streaming configuration audit
- KMS encryption validation for S3 and streaming destinations
- Kinesis Data Streams alarm validation against AWS best practices
- Account-level API rate quota review with modified quota highlighting
- Color-coded capacity utilization (Green/Orange/Red)
- Executive summary with clickable anchors to each section

## Deployment

### Prerequisites

- An Amazon Connect instance
- An S3 bucket for report output
- AWS CLI configured with appropriate permissions

### Deploy via CloudFormation

```bash
aws cloudformation deploy \
  --template-file CFT-AmazonConnectOperationsReview.yml \
  --stack-name amazon-connect-ops-review \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides \
    AmazonConnectInstanceARN=arn:aws:connect:<region>:<account-id>:instance/<instance-id> \
    AmazonConnectCloudWatchLogGroup=/aws/connect/<instance-name> \
    AmazonS3ForReports=<your-s3-bucket-name>
```

### Parameters

| Parameter | Description |
|-----------|-------------|
| `AmazonConnectInstanceARN` | ARN of your Amazon Connect instance |
| `AmazonConnectCloudWatchLogGroup` | CloudWatch Log Group for the Connect instance |
| `AmazonS3ForReports` | S3 bucket name for uploading reports |
| `Boto3LayerArn` | (Optional) Lambda Layer ARN with newer boto3 for DataTables, Notifications, Workspaces metrics |

### Optional: Boto3 Lambda Layer

Some newer Connect APIs (ListDataTables, ListNotifications, ListWorkspaces, SearchEmailAddresses) require a newer boto3 version than the Lambda runtime provides. To enable these metrics:

```bash
mkdir -p python
pip install boto3 -t python/
zip -r boto3-layer.zip python/
```

Upload as a Lambda Layer in the AWS Console, then provide the Layer ARN in the `Boto3LayerArn` parameter.

## IAM Permissions

The CloudFormation template creates a Lambda execution role with:

- `AWSLambdaBasicExecutionRole` — CloudWatch Logs
- `AmazonConnectReadOnlyAccess` — Connect instance read access
- `AWSCloudTrail_ReadOnlyAccess` — API throttling analysis
- `ServiceQuotasReadOnlyAccess` — Quota limit retrieval
- `CloudWatchReadOnlyAccess` — Metrics and alarms
- `mobiletargeting:PhoneNumberValidate` — Carrier diversity analysis
- `s3:PutObject` — Report upload to S3
- `connect:SearchEmailAddresses`, `connect:ListDataTables`, `connect:ListNotifications`, `connect:ListWorkspaces` — Additional Connect APIs

## Execution

The Lambda function can be invoked manually or on a schedule. It has a 15-minute timeout and 256MB memory allocation.

```bash
aws lambda invoke \
  --function-name amazonConnectOperationalReview-auto \
  --payload '{}' \
  output.json
```

The HTML report is uploaded to: `s3://<bucket>/connect-review_<instance-alias>_<region>_<timestamp>.html`

## Report Lookback Period

The report analyzes the last 30 days of CloudWatch metrics data by default for:
- Concurrent call/chat/email/task peak usage
- Missed calls trends
- API throttling events via CloudTrail

## Architecture

```
Lambda Function
  ├── Amazon Connect APIs (instance config, flows, phone numbers, storage)
  ├── Service Quotas API (instance & account-level limits)
  ├── CloudWatch API (metrics, alarms)
  ├── CloudTrail API (API throttling events)
  ├── Pinpoint API (phone number carrier validation)
  └── S3 (report upload)
```

# CloudFormation Template

This CloudFormation template, named CFT-AmazonConnectOperationalReview.yml, is designed to deploy the AWS resources needed to run an automated operational review for an Amazon Connect instance.
The template provisions an IAM Role with specific permissions and an AWS Lambda function to execute the review logic, along with a CloudWatch Log Group for logging.


## Parameters (User Inputs)

These values must be provided by the user when creating the stack, allowing the template to be reusable in different AWS accounts or environments.
 

AmazonConnectInstanceARN - The ARN of the target Amazon Connect instance. Passed as an environment variable to the Lambda function.


AmazonConnectCloudWatchLogGroup - The ARN of the Amazon Connect CloudWatch Log Group. Passed as an environment variable to the Lambda function.

AmazonS3ForReports - The name of an existing S3 bucket where the operational review reports will be uploaded. Used to define an S3 PutObject permission in the IAM role and passed as an environment variable to the Lambda function.


## Resources (Infrastructure Components)

This is the core section defining the three AWS resources provisioned by the template.
 

1. IAM Role (AmazonConnectOperationalReviewLambdaExecutionRole)

This role grants the Lambda function the necessary permissions to operate.
 

Trust Policy: Allows the AWS Lambda service (lambda.amazonaws.com) to assume the role.
 

Managed Policies: Attaches standard AWS policies for broad, read-only access and basic Lambda execution:
AWSLambdaBasicExecutionRole (for writing logs to CloudWatch)
AmazonConnectReadOnlyAccess
AWSCloudTrail_ReadOnlyAccess
ServiceQuotasReadOnlyAccess
CloudWatchReadOnlyAccess

Inline Policies: Attaches two custom policies:
PinpointPhoneNumberValidateReadOnlyAccess: Allows the action mobiletargeting:PhoneNumberValidate, which is likely used by the review logic to validate phone number formats.
PutObjectToS3-Review: Grants write permission (s3:PutObject) to the specific S3 bucket provided in the AmazonS3ForReports parameter, allowing the Lambda function to upload the final review report.
 

2. Lambda Function (LambdaFunctionAmazonConnectOperationalReviewauto)

This is the code execution environment for the operational review logic.

FunctionName: Set to a fixed value: "amazonConnectOperationalReview-auto".

Role: References the ARN of the IAM Role defined above using the intrinsic function !GetAtt.

Code Source: The deployment package (amazonConnectOperationalReview-auto.zip) is fetched from a globally shared S3 location (operations-review-code-share), indicating this is a pre-built solution from AWS.

Environment Variables: The input parameters are passed directly to the Lambda function's environment:
CONNECT_INSTANCE_ARN
CONNECT_CW_LOG_GROUP
S3_REPORTING_BUCKET
 

3. CloudWatch Log Group (LambdaLogGroup)

LogGroupName: Matches the expected log group for the Lambda function: "/aws/lambda/amazonConnectOperationalReview-auto".
 

RetentionInDays: Sets log retention to 30 days.
 



## Authors

- [@ganejaya](https://www.github.com/ganejaya)


