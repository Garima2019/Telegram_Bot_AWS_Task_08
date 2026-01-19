# Deployment Checklist

Use this checklist when deploying the refactored infrastructure.

## Pre-Deployment

### 1. Prerequisites ✅
- [ ] AWS CLI installed and configured
- [ ] Terraform >= 1.7.0 installed
- [ ] Telegram Bot Token obtained from @BotFather
- [ ] AWS account with appropriate permissions

### 2. Remote State Setup ✅
```bash
# Set your bucket name
export STATE_BUCKET="your-terraform-state-bucket"
export STATE_REGION="us-east-1"

# Create S3 bucket
aws s3 mb s3://$STATE_BUCKET --region $STATE_REGION

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket $STATE_BUCKET \
  --versioning-configuration Status=Enabled

# Enable encryption
aws s3api put-bucket-encryption \
  --bucket $STATE_BUCKET \
  --server-side-encryption-configuration \
  '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'

# Create DynamoDB lock table
aws dynamodb create-table \
  --table-name terraform-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region $STATE_REGION
```

- [ ] S3 bucket created
- [ ] Versioning enabled
- [ ] Encryption enabled
- [ ] DynamoDB table created

### 3. Configuration ✅
```bash
# Copy example config
cp terraform.tfvars.example terraform.tfvars

# Edit configuration
nano terraform.tfvars
```

**Required variables to set**:
- [ ] `telegram_bot_token`
- [ ] `s3_bucket_name` (must be globally unique)
- [ ] `aws_region`
- [ ] `environment` (dev/staging/prod)

**Optional but recommended**:
- [ ] `team`
- [ ] `cost_center`
- [ ] `openai_api_key` (if using AI features)
- [ ] `gemini_api_key` (if using AI features)

### 4. Backend Configuration ✅

Edit `backend.tf`:
- [ ] Update `bucket` name with your state bucket
- [ ] Verify `region` matches your state bucket region
- [ ] Confirm `key` path is appropriate

## Deployment

### 5. Initialize Terraform ✅
```bash
terraform init
```

**Expected output**:
- [ ] Backend successfully configured
- [ ] Providers downloaded
- [ ] Modules initialized
- [ ] No errors

### 6. Validate Configuration ✅
```bash
terraform validate
```

- [ ] Configuration is valid
- [ ] No syntax errors

### 7. Format Check ✅
```bash
terraform fmt -check -recursive
```

- [ ] Code is properly formatted (or run `terraform fmt -recursive` to fix)

### 8. Plan Review ✅
```bash
terraform plan -out=tfplan
```

**Review the plan for**:
- [ ] Correct number of resources to create
- [ ] S3 bucket name is unique
- [ ] DynamoDB table name is correct
- [ ] Lambda function configuration is correct
- [ ] IAM policies are scoped (no wildcards)
- [ ] Tags are applied to all resources
- [ ] No unexpected deletions

**Expected resource counts**:
- Lambda: 1 function, 1 role, 3 policies, 1 log group
- API Gateway: 1 API, 1 stage, 1 integration, 1 route, 1 permission
- DynamoDB: 1 table
- S3: 1 bucket + 5 configurations
- **Total**: ~20-25 resources

### 9. Apply Configuration ✅
```bash
terraform apply tfplan
```

- [ ] All resources created successfully
- [ ] No errors in output
- [ ] Outputs displayed correctly

### 10. Verify Outputs ✅
```bash
# View all outputs
terraform output

# Get specific outputs
terraform output webhook_url
terraform output s3_bucket_name
terraform output dynamodb_table_name
```

- [ ] webhook_url is displayed
- [ ] All ARNs are present
- [ ] No sensitive data exposed (tokens hidden)

## Post-Deployment

### 11. Configure Telegram Webhook ✅
```bash
# Get webhook URL
WEBHOOK_URL=$(terraform output -raw webhook_url)

# Set webhook (replace YOUR_BOT_TOKEN)
curl -X POST "https://api.telegram.org/botYOUR_BOT_TOKEN/setWebhook" \
  -d "url=$WEBHOOK_URL"

# Verify webhook
curl "https://api.telegram.org/botYOUR_BOT_TOKEN/getWebhookInfo"
```

**Expected response**:
```json
{
  "ok": true,
  "result": {
    "url": "https://your-api-url/webhook",
    "has_custom_certificate": false,
    "pending_update_count": 0
  }
}
```

- [ ] Webhook set successfully
- [ ] URL matches terraform output
- [ ] No pending updates or errors

### 12. Test Bot Functionality ✅

Send test messages to your bot:

- [ ] `/start` - Bot responds with welcome message
- [ ] `/help` - Bot shows command list
- [ ] Send a text message - Bot saves to DynamoDB
- [ ] Send a photo - Bot uploads to S3
- [ ] `/stats` - Bot shows statistics
- [ ] `/latest` - Bot retrieves last message

### 13. Verify AWS Resources ✅

**Lambda**:
```bash
aws lambda get-function --function-name $(terraform output -raw lambda_function_name)
```
- [ ] Function exists and is active
- [ ] Environment variables set correctly
- [ ] Memory and timeout configured

**DynamoDB**:
```bash
aws dynamodb describe-table --table-name $(terraform output -raw dynamodb_table_name)
```
- [ ] Table exists and is active
- [ ] Correct key schema
- [ ] Encryption enabled

**S3**:
```bash
aws s3 ls $(terraform output -raw s3_bucket_name)
```
- [ ] Bucket exists and is accessible
- [ ] Public access blocked

**CloudWatch Logs**:
```bash
# Lambda logs
aws logs tail /aws/lambda/$(terraform output -raw lambda_function_name) --follow

# API Gateway logs
aws logs describe-log-groups --log-group-name-prefix /aws/apigateway
```
- [ ] Log groups created
- [ ] Logs are being written
- [ ] Retention configured

### 14. Security Validation ✅

**IAM Policy Check**:
```bash
# Get Lambda role ARN
ROLE_ARN=$(terraform output -raw lambda_role_arn)

# List attached policies
aws iam list-attached-role-policies --role-name $(basename $ROLE_ARN)

# Review inline policies
aws iam list-role-policies --role-name $(basename $ROLE_ARN)
```

Verify:
- [ ] No wildcard resources in S3 policy
- [ ] No wildcard resources in DynamoDB policy
- [ ] No wildcard actions (specific actions only)
- [ ] Policies follow least-privilege

**S3 Security**:
```bash
# Check public access block
aws s3api get-public-access-block \
  --bucket $(terraform output -raw s3_bucket_name)

# Check encryption
aws s3api get-bucket-encryption \
  --bucket $(terraform output -raw s3_bucket_name)
```

- [ ] All public access blocked
- [ ] Encryption enabled (AES256)

### 15. Documentation ✅

- [ ] Document webhook URL (securely)
- [ ] Update team documentation with deployment date
- [ ] Record any customizations made
- [ ] Save terraform.tfvars in secure location (DO NOT COMMIT)
- [ ] Update runbook with any environment-specific notes

## Monitoring Setup (Class 9)

These will be addressed in the next class:

- [ ] Set up CloudWatch alarms for Lambda errors
- [ ] Configure API Gateway throttling alarms
- [ ] Create DynamoDB capacity alarms
- [ ] Set up SNS notifications
- [ ] Create CloudWatch dashboard

## Rollback Procedure

If deployment fails:

```bash
# Destroy all resources
terraform destroy

# Or destroy specific modules
terraform destroy -target=module.lambda
terraform destroy -target=module.api_gateway

# Restore from previous state (if needed)
terraform state pull > current.tfstate
cp terraform.tfstate.backup terraform.tfstate
terraform state push terraform.tfstate.backup
```

## Success Criteria

✅ **Deployment is successful if**:

1. All Terraform resources created without errors
2. Telegram webhook configured and responding
3. Bot responds to test messages
4. Files upload to S3 successfully
5. Messages stored in DynamoDB
6. CloudWatch logs are being written
7. No IAM wildcard policies
8. All resources properly tagged
9. Security configurations in place
10. Documentation updated

---

**Deployment Date**: _______________  
**Deployed By**: _______________  
**Environment**: _______________  
**Any Issues**: _______________