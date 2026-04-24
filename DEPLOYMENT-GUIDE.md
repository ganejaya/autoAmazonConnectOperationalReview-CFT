# Deployment Guide — CloudFormation

Complete guide for deploying, updating, and managing the Amazon Connect Operational Review tool using CloudFormation.

## Prerequisites

- AWS account with permissions to create Lambda functions, IAM roles, and CloudWatch Log Groups
- An existing Amazon Connect instance (you'll need the instance ARN)
- An existing S3 bucket for reports
- AWS CLI (optional, for CLI-based deployment)

## Initial Deployment

### Option A: AWS Console

1. Open the [CloudFormation Console](https://console.aws.amazon.com/cloudformation)
2. Click **Create stack** → **With new resources (standard)**
3. Select **Upload a template file** and upload `CFT-AmazonConnectOperationsReview.yml`
4. Click **Next**
5. Fill in the parameters:

| Parameter | Description | Example |
|---|---|---|
| Stack name | A name for this deployment | `ConnectOpsReview` |
| AmazonConnectInstanceARN | Your Connect instance ARN | `arn:aws:connect:us-east-1:123456789012:instance/aaa-bbb-ccc` |
| AmazonConnectCloudWatchLogGroup | Connect CloudWatch Log Group name | `/aws/connect/my-instance` |
| AmazonS3ForReports | Existing S3 bucket name for reports | `my-connect-ops-reports` |

6. Click **Next** through the options page (defaults are fine)
7. On the review page, check **I acknowledge that AWS CloudFormation might create IAM resources**
8. Click **Submit**

The stack takes about 1-2 minutes to create.

### Option B: AWS CLI

```bash
aws cloudformation create-stack \
  --stack-name ConnectOpsReview \
  --template-body file://CFT-AmazonConnectOperationsReview.yml \
  --parameters \
    ParameterKey=AmazonConnectInstanceARN,ParameterValue="arn:aws:connect:us-east-1:123456789012:instance/aaa-bbb-ccc" \
    ParameterKey=AmazonConnectCloudWatchLogGroup,ParameterValue="/aws/connect/my-instance" \
    ParameterKey=AmazonS3ForReports,ParameterValue="my-connect-ops-reports" \
  --capabilities CAPABILITY_IAM \
  --region us-east-1
```

Wait for completion:

```bash
aws cloudformation wait stack-create-complete --stack-name ConnectOpsReview
```

### Verify

```bash
# Check stack status
aws cloudformation describe-stacks --stack-name ConnectOpsReview --query 'Stacks[0].StackStatus'

# Get outputs
aws cloudformation describe-stacks --stack-name ConnectOpsReview --query 'Stacks[0].Outputs'

# Test the Lambda function
aws lambda get-function --function-name amazonConnectOperationalReview-auto
```

---

## Updating Lambda Code

When the Lambda function code is updated in the code repo, there are two paths depending on whether you have CI/CD configured.

### With CI/CD (automated)

1. A push to `lambda_function.py` in the code repo triggers a GitHub Actions workflow
2. The workflow embeds the updated code into the CFT template's `ZipFile` block and creates a PR
3. Review and merge the PR
4. Update the deployed stack with the new template:

**Console:**
1. Open CloudFormation Console → select your stack
2. Click **Update**
3. Select **Replace current template** → upload the updated `CFT-AmazonConnectOperationsReview.yml`
4. Click through — parameters stay the same
5. Review and submit

**CLI:**

```bash
aws cloudformation update-stack \
  --stack-name ConnectOpsReview \
  --template-body file://CFT-AmazonConnectOperationsReview.yml \
  --parameters \
    ParameterKey=AmazonConnectInstanceARN,UsePreviousValue=true \
    ParameterKey=AmazonConnectCloudWatchLogGroup,UsePreviousValue=true \
    ParameterKey=AmazonS3ForReports,UsePreviousValue=true \
  --capabilities CAPABILITY_IAM

aws cloudformation wait stack-update-complete --stack-name ConnectOpsReview
```

### Without CI/CD (manual)

1. Pull the latest template from this repo (or download it)
2. Update the stack using the Console or CLI steps above

### Using Change Sets (recommended for production)

Change sets let you preview what CloudFormation will change before applying:

```bash
# Create a change set
aws cloudformation create-change-set \
  --stack-name ConnectOpsReview \
  --change-set-name lambda-code-update \
  --template-body file://CFT-AmazonConnectOperationsReview.yml \
  --parameters \
    ParameterKey=AmazonConnectInstanceARN,UsePreviousValue=true \
    ParameterKey=AmazonConnectCloudWatchLogGroup,UsePreviousValue=true \
    ParameterKey=AmazonS3ForReports,UsePreviousValue=true \
  --capabilities CAPABILITY_IAM

# Review what will change
aws cloudformation describe-change-set \
  --stack-name ConnectOpsReview \
  --change-set-name lambda-code-update

# Apply if satisfied
aws cloudformation execute-change-set \
  --stack-name ConnectOpsReview \
  --change-set-name lambda-code-update
```

---

## Updating Infrastructure Configuration

If you need to change infrastructure settings (timeout, memory, IAM policies, log retention, etc.):

### Changing parameter values

Update the stack with new parameter values:

```bash
aws cloudformation update-stack \
  --stack-name ConnectOpsReview \
  --template-body file://CFT-AmazonConnectOperationsReview.yml \
  --parameters \
    ParameterKey=AmazonConnectInstanceARN,ParameterValue="arn:aws:connect:us-east-1:123456789012:instance/new-instance-id" \
    ParameterKey=AmazonConnectCloudWatchLogGroup,UsePreviousValue=true \
    ParameterKey=AmazonS3ForReports,UsePreviousValue=true \
  --capabilities CAPABILITY_IAM
```

### Changing resource definitions

Edit `CFT-AmazonConnectOperationsReview.yml` directly (e.g., changing Lambda timeout, adding an IAM policy), then update the stack with the modified template.

**Important**: Do not edit the `ZipFile: |` block directly — that's managed by automation. Edit `lambda_function.py` in the code repo instead.

### What to review before updating

- Use change sets (see above) to preview changes before applying
- Pay attention to resources marked as **Replacement** — these will be deleted and recreated, which can cause brief downtime
- IAM role changes are typically in-place updates (no downtime)
- Lambda function code updates are in-place (no downtime)

---

## Rollback

### Automatic rollback

CloudFormation automatically rolls back if a stack update fails. You don't need to do anything — the stack returns to its previous state.

### Manual rollback

If a successful update causes issues at runtime:

**Console:**
1. Open CloudFormation Console → select your stack
2. Click **Update** → **Roll back**
3. Or update with the previous version of the template

**CLI:**

```bash
# Re-deploy the previous template version
git checkout HEAD~1 -- CFT-AmazonConnectOperationsReview.yml

aws cloudformation update-stack \
  --stack-name ConnectOpsReview \
  --template-body file://CFT-AmazonConnectOperationsReview.yml \
  --parameters \
    ParameterKey=AmazonConnectInstanceARN,UsePreviousValue=true \
    ParameterKey=AmazonConnectCloudWatchLogGroup,UsePreviousValue=true \
    ParameterKey=AmazonS3ForReports,UsePreviousValue=true \
  --capabilities CAPABILITY_IAM
```

### Emergency: revert Lambda code via AWS Console

If you need to fix the Lambda immediately without going through CloudFormation:

1. Open the Lambda Console → find `amazonConnectOperationalReview-auto`
2. Edit the code directly or upload a known-good zip

Be aware this creates **drift** — CloudFormation thinks the stack is in one state but the actual resource differs. To reconcile, update the stack with the correct template afterward.

To check for drift:

```bash
aws cloudformation detect-stack-drift --stack-name ConnectOpsReview
aws cloudformation describe-stack-drift-detection-status --stack-drift-detection-id <id>
```

---

## Teardown

### Console

1. Open CloudFormation Console → select your stack
2. Click **Delete**
3. Confirm

### CLI

```bash
aws cloudformation delete-stack --stack-name ConnectOpsReview
aws cloudformation wait stack-delete-complete --stack-name ConnectOpsReview
```

This removes the Lambda function, IAM role, policies, and CloudWatch Log Group. It does not remove:
- The S3 reporting bucket (not created by this template)
- The Connect instance

---

## Common Operations Reference

| Task | Command |
|---|---|
| Check stack status | `aws cloudformation describe-stacks --stack-name ConnectOpsReview` |
| List stack resources | `aws cloudformation list-stack-resources --stack-name ConnectOpsReview` |
| Preview changes | Create a change set (see above) |
| Update stack | `aws cloudformation update-stack ...` |
| View outputs | `aws cloudformation describe-stacks ... --query 'Stacks[0].Outputs'` |
| Validate template | `aws cloudformation validate-template --template-body file://CFT-AmazonConnectOperationsReview.yml` |
| Check for drift | `aws cloudformation detect-stack-drift --stack-name ConnectOpsReview` |
| Delete stack | `aws cloudformation delete-stack --stack-name ConnectOpsReview` |
