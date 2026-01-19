# Telegram Bot Infrastructure (Terraform)

Production-ready AWS infrastructure for a Telegram bot with file storage, message persistence, and AI integration.

## 🏗️ Architecture

```
┌─────────────┐
│  Telegram   │
│   Webhook   │
└──────┬──────┘
       │
       ▼
┌─────────────────────┐
│   API Gateway       │
│   (HTTP API)        │
│   - Throttling      │
│   - Logging         │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐      ┌──────────────┐
│   Lambda Function   │─────▶│  DynamoDB    │
│   - Python 3.11     │      │  - Messages  │
│   - 512MB RAM       │      │  - Metadata  │
│   - 60s timeout     │      └──────────────┘
└─────────┬───────────┘
          │
          ▼
     ┌──────────┐
     │    S3    │
     │  Bucket  │
     │  - Files │
     │  - Media │
     └──────────┘
```

## 📁 Module Structure

```
terraform-telegram-bot/
├── main.tf                    # Root module - orchestrates everything
├── variables.tf               # Root-level variables
├── outputs.tf                 # Root-level outputs
├── terraform.tfvars           # telegram_bot_token 
├── handler.py                 # Lambda function code
└── README.md
```
- Variables used everywhere. No hardcoded ARNs or names.
- Outputs are excellent:
  - webhook URL
  - ARNs
  - names
  - log groups
- Remote state:
  - S3 backend
  - DynamoDB locking
  - Encryption + versioning documented


## 🚀 Quick Start

### Prerequisites

1. **AWS Account** with appropriate permissions
2. **Terraform** >= 1.7.0 ([Install](https://www.terraform.io/downloads))
3. **AWS CLI** configured with credentials
4. **Telegram Bot Token** from [@BotFather](https://t.me/botfather)

### Step 1: Set Up Remote State

Create S3 bucket and DynamoDB table for Terraform state:

```bash
# Create S3 bucket for state
aws s3 mb s3://your-terraform-state-bucket --region us-east-1

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket your-terraform-state-bucket \
  --versioning-configuration Status=Enabled

# Enable encryption
aws s3api put-bucket-encryption \
  --bucket your-terraform-state-bucket \
  --server-side-encryption-configuration \
  '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'

# Create DynamoDB table for state locking
aws dynamodb create-table \
  --table-name terraform-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

### Step 2: Configure Variables

```bash
# Copy example config
cp terraform.tfvars.example terraform.tfvars

# Edit with your values
nano terraform.tfvars
```

Required variables:
- `telegram_bot_token` - Get from @BotFather
- `s3_bucket_name` - Must be globally unique
- `aws_region` - AWS region to deploy

Optional variables:
- `openai_api_key` - For AI features
- `gemini_api_key` - Alternative AI provider

### Step 3: Update Backend Configuration

Edit `backend.tf` and update the bucket name:

```hcl
terraform {
  backend "s3" {
    bucket = "your-terraform-state-bucket"  # Change this
    key    = "telegram-bot/terraform.tfstate"
    region = "us-east-1"
  }
}
```

### Step 4: Deploy

```bash
# Initialize Terraform and migrate state
terraform init

# Review changes
terraform plan

# Deploy infrastructure
terraform apply

# Note the webhook_url from outputs
```

### Step 5: Configure Telegram Webhook

```bash
# Get the webhook URL from outputs
WEBHOOK_URL=$(terraform output -raw webhook_url)

# Set the webhook (replace YOUR_BOT_TOKEN)
curl -X POST "https://api.telegram.org/botYOUR_BOT_TOKEN/setWebhook" \
  -d "url=$WEBHOOK_URL"

# Verify webhook is set
curl "https://api.telegram.org/botYOUR_BOT_TOKEN/getWebhookInfo"
```

## 🔧 Configuration

### Environment-Specific Configurations

**Development**:
```hcl
environment = "dev"
lambda_memory_size = 512
api_log_retention_days = 7
s3_noncurrent_version_days = 30
```

**Production**:
```hcl
environment = "prod"
lambda_memory_size = 1024
api_log_retention_days = 30
s3_noncurrent_version_days = 365

# Enable additional features
module "dynamodb" {
  enable_point_in_time_recovery = true
  billing_mode = "PROVISIONED"
  enable_autoscaling = true
}
```

### Tagging Strategy

All resources are tagged with:
- `Project` - telegram-bot
- `Environment` - dev/staging/prod
- `ManagedBy` - terraform
- `Team` - Configurable (default: platform)
- `CostCenter` - Configurable (default: engineering)

## 🔒 Security Features

### IAM - Least Privilege

Lambda execution role has **scoped permissions**:

✅ **DynamoDB**:
- Actions: PutItem, GetItem, UpdateItem, DeleteItem, Query, Scan
- Resource: Specific table ARN only

✅ **S3**:
- Actions: PutObject, GetObject, DeleteObject
- Resource: Specific bucket ARN only
- Condition: Encryption enforced via condition

❌ **No wildcards** in actions or resources

### S3 Security

- ✅ Public access blocked
- ✅ Encryption at rest (AES256)
- ✅ Versioning enabled
- ✅ Lifecycle policies for cost optimization
- ✅ Access logging (can be enabled)

### DynamoDB Security

- ✅ Encryption at rest
- ✅ Point-in-time recovery (prod)
- ✅ Scoped IAM permissions
- ✅ Auto-scaling (optional)

### API Gateway

- ✅ Throttling (100 burst, 50 RPS)
- ✅ CloudWatch logging
- ✅ JSON-formatted access logs

## 📊 Monitoring & Logging

### CloudWatch Log Groups

All components log to CloudWatch:

```bash
# Lambda logs
/aws/lambda/telegram-bot-dev-handler

# API Gateway logs
/aws/apigateway/telegram-bot-dev-api
```

### Viewing Logs

```bash
# Recent Lambda logs
aws logs tail /aws/lambda/telegram-bot-dev-handler --follow

# API Gateway access logs
aws logs tail /aws/apigateway/telegram-bot-dev-api --follow

# Filter for errors
aws logs filter-log-events \
  --log-group-name /aws/lambda/telegram-bot-dev-handler \
  --filter-pattern "ERROR"
```

## 🎯 Module Usage

### Using Individual Modules

You can use modules independently in other projects:

```hcl
module "telegram_lambda" {
  source = "./modules/lambda"
  
  function_name         = "my-bot-handler"
  handler_file          = "./handler.py"
  s3_bucket_arn        = aws_s3_bucket.files.arn
  dynamodb_table_arn   = aws_dynamodb_table.messages.arn
  
  environment_variables = {
    API_KEY = var.api_key
  }
}
```

## 🔄 Maintenance

### Updating Infrastructure

```bash
# Pull latest changes
git pull

# Review changes
terraform plan

# Apply updates
terraform apply
```

### Destroying Infrastructure

```bash
# Destroy all resources
terraform destroy

# Destroy specific module
terraform destroy -target=module.lambda
```

⚠️ **Warning**: This will delete:
- All messages in DynamoDB
- All files in S3 (if force_destroy = true)
- All Lambda functions and logs

### State Management

```bash
# List resources in state
terraform state list

# Show specific resource
terraform state show module.lambda.aws_lambda_function.this

# Move resource between modules
terraform state mv \
  aws_lambda_function.old \
  module.lambda.aws_lambda_function.this
```

## 📈 Scaling Considerations

### Lambda

Current: 512MB RAM, 60s timeout

Scale up if needed:
```hcl
lambda_memory_size = 1024  # Increases CPU proportionally
lambda_timeout = 120       # For longer operations
```

### DynamoDB

**Development**: PAY_PER_REQUEST (on-demand)
**Production**: PROVISIONED with auto-scaling

```hcl
module "dynamodb" {
  billing_mode           = "PROVISIONED"
  read_capacity          = 5
  write_capacity         = 5
  enable_autoscaling     = true
  autoscaling_read_max   = 100
  autoscaling_write_max  = 100
}
```

### API Gateway

Throttling limits can be adjusted:
```hcl
throttling_burst_limit = 200  # Burst capacity
throttling_rate_limit  = 100  # Sustained RPS
```

## 🐛 Troubleshooting

### Common Issues

**Issue**: Terraform init fails with backend error
```bash
# Solution: Verify S3 bucket exists and you have access
aws s3 ls s3://your-terraform-state-bucket
```

**Issue**: Lambda function fails with permission errors
```bash
# Solution: Check IAM policy attachments
terraform state show module.lambda.aws_iam_role_policy.dynamodb_access
```

**Issue**: Webhook not working
```bash
# Solution: Verify API Gateway URL and Lambda permission
curl -X POST $(terraform output -raw webhook_url) -d '{}'
```

**Issue**: S3 bucket name already exists
```bash
# Solution: Choose a globally unique bucket name in terraform.tfvars
s3_bucket_name = "my-unique-bot-files-12345"
```

### Debug Mode

```bash
# Enable Terraform debug logging
export TF_LOG=DEBUG
terraform plan

# Enable Lambda debug logging
# Add to terraform.tfvars:
environment_variables = {
  LOG_LEVEL = "DEBUG"
}
```

## 📚 Additional Resources

- [Terraform AWS Provider Docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [AWS Lambda Best Practices](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html)
- [DynamoDB Best Practices](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/best-practices.html)
- [Telegram Bot API](https://core.telegram.org/bots/api)

## 🤝 Contributing

1. Create a feature branch
2. Make your changes
3. Update CHANGELOG.md
4. Submit a pull request

## 📄 License

This project was developed for an MSc Cloud Solutions course and implements a Telegram bot using AWS Lambda, DynamoDB for chat data storage, and Amazon S3.

---

**Questions?** Open an issue or contact the platform team.
