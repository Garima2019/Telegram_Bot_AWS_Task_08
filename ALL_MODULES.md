# All Module Code - Copy Reference

This file contains ALL module code. Copy each section to the corresponding file.

---

## 📁 modules/s3/main.tf

```hcl
############################################
# S3 BUCKET FOR FILE STORAGE
############################################

resource "aws_s3_bucket" "this" {
  bucket = var.bucket_name

  # Allow force destroy only in dev
  force_destroy = var.environment == "dev" ? true : false

  tags = merge(
    var.common_tags,
    {
      Name = var.bucket_name
      Type = "file-storage"
    }
  )
}

############################################
# PUBLIC ACCESS BLOCK
############################################

resource "aws_s3_bucket_public_access_block" "this" {
  bucket = aws_s3_bucket.this.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

############################################
# SERVER-SIDE ENCRYPTION
############################################

resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
    bucket_key_enabled = true
  }
}

############################################
# VERSIONING
############################################

resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id

  versioning_configuration {
    status = var.enable_versioning ? "Enabled" : "Suspended"
  }
}

############################################
# LIFECYCLE CONFIGURATION
############################################

resource "aws_s3_bucket_lifecycle_configuration" "this" {
  bucket = aws_s3_bucket.this.id

  # Expire old versions
  rule {
    id     = "expire-noncurrent-versions"
    status = "Enabled"

    noncurrent_version_expiration {
      noncurrent_days = var.noncurrent_version_days
    }
  }

  # Transition to Intelligent-Tiering (prod only)
  dynamic "rule" {
    for_each = var.enable_intelligent_tiering ? [1] : []
    
    content {
      id     = "intelligent-tiering"
      status = "Enabled"

      transition {
        days          = 30
        storage_class = "INTELLIGENT_TIERING"
      }
    }
  }

  # Clean up incomplete multipart uploads
  rule {
    id     = "abort-incomplete-multipart-uploads"
    status = "Enabled"

    abort_incomplete_multipart_upload {
      days_after_initiation = 7
    }
  }
}
```

---

## 📁 modules/s3/variables.tf

```hcl
variable "bucket_name" {
  type        = string
  description = "Name of the S3 bucket"
}

variable "environment" {
  type        = string
  description = "Environment name (dev, staging, prod)"
}

variable "common_tags" {
  type        = map(string)
  description = "Common tags to apply to all resources"
  default     = {}
}

variable "enable_versioning" {
  type        = bool
  description = "Enable S3 bucket versioning"
  default     = true
}

variable "noncurrent_version_days" {
  type        = number
  description = "Days to retain noncurrent versions"
  default     = 90
}

variable "enable_intelligent_tiering" {
  type        = bool
  description = "Enable transition to Intelligent-Tiering storage class"
  default     = false
}
```

---

## 📁 modules/s3/outputs.tf

```hcl
output "bucket_name" {
  description = "Name of the S3 bucket"
  value       = aws_s3_bucket.this.id
}

output "bucket_arn" {
  description = "ARN of the S3 bucket"
  value       = aws_s3_bucket.this.arn
}

output "bucket_domain_name" {
  description = "Bucket domain name"
  value       = aws_s3_bucket.this.bucket_domain_name
}

output "bucket_regional_domain_name" {
  description = "Bucket regional domain name"
  value       = aws_s3_bucket.this.bucket_regional_domain_name
}
```

---

## 📁 modules/dynamodb/main.tf

```hcl
############################################
# DYNAMODB TABLE
############################################

resource "aws_dynamodb_table" "this" {
  name         = var.table_name
  billing_mode = var.billing_mode

  hash_key  = "user_id"
  range_key = "sort_key"

  # Provisioned capacity (only if billing_mode = "PROVISIONED")
  read_capacity  = var.billing_mode == "PROVISIONED" ? var.read_capacity : null
  write_capacity = var.billing_mode == "PROVISIONED" ? var.write_capacity : null

  attribute {
    name = "user_id"
    type = "S"
  }

  attribute {
    name = "sort_key"
    type = "S"
  }

  # Enable point-in-time recovery for production
  point_in_time_recovery {
    enabled = var.enable_point_in_time_recovery
  }

  # Server-side encryption
  server_side_encryption {
    enabled = true
  }

  # TTL (optional - can be used to auto-expire old data)
  ttl {
    attribute_name = "ttl"
    enabled        = var.enable_ttl
  }

  tags = merge(
    var.common_tags,
    {
      Name = var.table_name
      Type = "message-storage"
    }
  )
}

############################################
# AUTO-SCALING (for PROVISIONED mode)
############################################

resource "aws_appautoscaling_target" "read" {
  count = var.billing_mode == "PROVISIONED" && var.enable_autoscaling ? 1 : 0

  max_capacity       = var.autoscaling_read_max_capacity
  min_capacity       = var.read_capacity
  resource_id        = "table/${aws_dynamodb_table.this.name}"
  scalable_dimension = "dynamodb:table:ReadCapacityUnits"
  service_namespace  = "dynamodb"
}

resource "aws_appautoscaling_policy" "read" {
  count = var.billing_mode == "PROVISIONED" && var.enable_autoscaling ? 1 : 0

  name               = "${var.table_name}-read-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.read[0].resource_id
  scalable_dimension = aws_appautoscaling_target.read[0].scalable_dimension
  service_namespace  = aws_appautoscaling_target.read[0].service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "DynamoDBReadCapacityUtilization"
    }
    target_value = var.autoscaling_read_target
  }
}

resource "aws_appautoscaling_target" "write" {
  count = var.billing_mode == "PROVISIONED" && var.enable_autoscaling ? 1 : 0

  max_capacity       = var.autoscaling_write_max_capacity
  min_capacity       = var.write_capacity
  resource_id        = "table/${aws_dynamodb_table.this.name}"
  scalable_dimension = "dynamodb:table:WriteCapacityUnits"
  service_namespace  = "dynamodb"
}

resource "aws_appautoscaling_policy" "write" {
  count = var.billing_mode == "PROVISIONED" && var.enable_autoscaling ? 1 : 0

  name               = "${var.table_name}-write-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.write[0].resource_id
  scalable_dimension = aws_appautoscaling_target.write[0].scalable_dimension
  service_namespace  = aws_appautoscaling_target.write[0].service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "DynamoDBWriteCapacityUtilization"
    }
    target_value = var.autoscaling_write_target
  }
}
```

---

## 📁 modules/dynamodb/variables.tf

```hcl
variable "table_name" {
  type        = string
  description = "Name of the DynamoDB table"
}

variable "environment" {
  type        = string
  description = "Environment name (dev, staging, prod)"
}

variable "common_tags" {
  type        = map(string)
  description = "Common tags to apply to all resources"
  default     = {}
}

variable "billing_mode" {
  type        = string
  description = "DynamoDB billing mode (PROVISIONED or PAY_PER_REQUEST)"
  default     = "PAY_PER_REQUEST"

  validation {
    condition     = contains(["PROVISIONED", "PAY_PER_REQUEST"], var.billing_mode)
    error_message = "Billing mode must be PROVISIONED or PAY_PER_REQUEST"
  }
}

variable "read_capacity" {
  type        = number
  description = "Read capacity units (only for PROVISIONED mode)"
  default     = 5
}

variable "write_capacity" {
  type        = number
  description = "Write capacity units (only for PROVISIONED mode)"
  default     = 5
}

variable "enable_point_in_time_recovery" {
  type        = bool
  description = "Enable point-in-time recovery (recommended for prod)"
  default     = false
}

variable "enable_ttl" {
  type        = bool
  description = "Enable TTL for automatic data expiration"
  default     = false
}

variable "enable_autoscaling" {
  type        = bool
  description = "Enable auto-scaling (only for PROVISIONED mode)"
  default     = false
}

variable "autoscaling_read_max_capacity" {
  type        = number
  description = "Maximum read capacity for auto-scaling"
  default     = 100
}

variable "autoscaling_write_max_capacity" {
  type        = number
  description = "Maximum write capacity for auto-scaling"
  default     = 100
}

variable "autoscaling_read_target" {
  type        = number
  description = "Target utilization for read capacity"
  default     = 70
}

variable "autoscaling_write_target" {
  type        = number
  description = "Target utilization for write capacity"
  default     = 70
}
```

---

## 📁 modules/dynamodb/outputs.tf

```hcl
output "table_name" {
  description = "Name of the DynamoDB table"
  value       = aws_dynamodb_table.this.name
}

output "table_arn" {
  description = "ARN of the DynamoDB table"
  value       = aws_dynamodb_table.this.arn
}

output "table_id" {
  description = "ID of the DynamoDB table"
  value       = aws_dynamodb_table.this.id
}

output "table_stream_arn" {
  description = "ARN of the DynamoDB table stream"
  value       = aws_dynamodb_table.this.stream_arn
}
```

---

## 📁 modules/lambda/main.tf

```hcl
############################################
# LAMBDA FUNCTION PACKAGING
############################################

data "archive_file" "lambda_zip" {
  type        = "zip"
  source_file = var.handler_file
  output_path = "${path.module}/lambda.zip"
}

############################################
# LAMBDA FUNCTION
############################################

resource "aws_lambda_function" "this" {
  function_name = var.function_name
  role          = aws_iam_role.this.arn

  handler = "handler.lambda_handler"
  runtime = var.runtime

  filename         = data.archive_file.lambda_zip.output_path
  source_code_hash = data.archive_file.lambda_zip.output_base64sha256

  memory_size = var.memory_size
  timeout     = var.timeout

  environment {
    variables = var.environment_variables
  }

  # Logging configuration
  logging_config {
    log_format = "JSON"
    log_group  = aws_cloudwatch_log_group.lambda.name
  }

  # Reserved concurrent executions (optional)
  reserved_concurrent_executions = var.reserved_concurrent_executions

  tags = merge(
    var.common_tags,
    {
      Name = var.function_name
      Type = "telegram-bot-handler"
    }
  )

  depends_on = [
    aws_iam_role_policy_attachment.lambda_basic,
    aws_iam_role_policy.dynamodb_access,
    aws_iam_role_policy.s3_access,
    aws_cloudwatch_log_group.lambda
  ]
}

############################################
# CLOUDWATCH LOG GROUP
############################################

resource "aws_cloudwatch_log_group" "lambda" {
  name              = "/aws/lambda/${var.function_name}"
  retention_in_days = var.log_retention_days

  tags = var.common_tags
}
```

---

## 📁 modules/lambda/iam.tf

```hcl
############################################
# LAMBDA EXECUTION ROLE
############################################

resource "aws_iam_role" "this" {
  name = "${var.function_name}-execution-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Service = "lambda.amazonaws.com"
      }
      Action = "sts:AssumeRole"
    }]
  })

  tags = var.common_tags
}

############################################
# BASIC LAMBDA EXECUTION PERMISSIONS
############################################

resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.this.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

############################################
# LEAST-PRIVILEGE DYNAMODB ACCESS
############################################

resource "aws_iam_role_policy" "dynamodb_access" {
  name = "${var.function_name}-dynamodb-policy"
  role = aws_iam_role.this.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "DynamoDBTableAccess"
        Effect = "Allow"
        Action = [
          "dynamodb:PutItem",
          "dynamodb:GetItem",
          "dynamodb:UpdateItem",
          "dynamodb:DeleteItem",
          "dynamodb:Query",
          "dynamodb:Scan"
        ]
        Resource = [
          var.dynamodb_table_arn,
          "${var.dynamodb_table_arn}/index/*"  # For any GSIs/LSIs
        ]
      }
    ]
  })
}

############################################
# LEAST-PRIVILEGE S3 ACCESS
############################################

resource "aws_iam_role_policy" "s3_access" {
  name = "${var.function_name}-s3-policy"
  role = aws_iam_role.this.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "S3ObjectOperations"
        Effect = "Allow"
        Action = [
          "s3:PutObject",
          "s3:GetObject",
          "s3:DeleteObject"
        ]
        Resource = "${var.s3_bucket_arn}/*"
        Condition = {
          StringEquals = {
            "s3:x-amz-server-side-encryption" = "AES256"
          }
        }
      },
      {
        Sid    = "S3BucketListAccess"
        Effect = "Allow"
        Action = [
          "s3:ListBucket"
        ]
        Resource = var.s3_bucket_arn
        Condition = {
          StringLike = {
            "s3:prefix" = "*"
          }
        }
      }
    ]
  })
}

############################################
# OPTIONAL: X-RAY TRACING
############################################

resource "aws_iam_role_policy_attachment" "xray" {
  count = var.enable_xray ? 1 : 0

  role       = aws_iam_role.this.name
  policy_arn = "arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess"
}
```

---

## 📁 modules/lambda/variables.tf

```hcl
variable "function_name" {
  type        = string
  description = "Name of the Lambda function"
}

variable "handler_file" {
  type        = string
  description = "Path to the Python handler file"
}

variable "runtime" {
  type        = string
  description = "Lambda runtime"
  default     = "python3.11"
}

variable "memory_size" {
  type        = number
  description = "Memory size in MB"
  default     = 512
}

variable "timeout" {
  type        = number
  description = "Timeout in seconds"
  default     = 60
}

variable "environment_variables" {
  type        = map(string)
  description = "Environment variables for the Lambda function"
  default     = {}
}

variable "common_tags" {
  type        = map(string)
  description = "Common tags to apply to all resources"
  default     = {}
}

variable "log_retention_days" {
  type        = number
  description = "CloudWatch log retention in days"
  default     = 7
}

variable "reserved_concurrent_executions" {
  type        = number
  description = "Reserved concurrent executions (-1 for unreserved)"
  default     = -1
}

variable "enable_xray" {
  type        = bool
  description = "Enable AWS X-Ray tracing"
  default     = false
}

# IAM permissions
variable "s3_bucket_arn" {
  type        = string
  description = "ARN of the S3 bucket for scoped permissions"
}

variable "dynamodb_table_arn" {
  type        = string
  description = "ARN of the DynamoDB table for scoped permissions"
}
```

---

## 📁 modules/lambda/outputs.tf

```hcl
output "function_name" {
  description = "Name of the Lambda function"
  value       = aws_lambda_function.this.function_name
}

output "function_arn" {
  description = "ARN of the Lambda function"
  value       = aws_lambda_function.this.arn
}

output "invoke_arn" {
  description = "Invoke ARN of the Lambda function"
  value       = aws_lambda_function.this.invoke_arn
}

output "role_arn" {
  description = "ARN of the Lambda execution role"
  value       = aws_iam_role.this.arn
}

output "role_name" {
  description = "Name of the Lambda execution role"
  value       = aws_iam_role.this.name
}

output "log_group_name" {
  description = "Name of the CloudWatch log group"
  value       = aws_cloudwatch_log_group.lambda.name
}
```

---

## 📁 modules/api_gateway/main.tf

```hcl
############################################
# API GATEWAY HTTP API
############################################

resource "aws_apigatewayv2_api" "this" {
  name          = var.api_name
  protocol_type = "HTTP"

  # CORS configuration (optional)
  cors_configuration {
    allow_origins = var.cors_allow_origins
    allow_methods = var.cors_allow_methods
    allow_headers = var.cors_allow_headers
    max_age       = var.cors_max_age
  }

  tags = merge(
    var.common_tags,
    {
      Name = var.api_name
      Type = "telegram-webhook"
    }
  )
}

############################################
# LAMBDA INTEGRATION
############################################

resource "aws_apigatewayv2_integration" "lambda" {
  api_id                 = aws_apigatewayv2_api.this.id
  integration_type       = "AWS_PROXY"
  integration_method     = "POST"
  integration_uri        = var.lambda_invoke_arn
  payload_format_version = "2.0"
  timeout_milliseconds   = var.integration_timeout_ms
}

############################################
# ROUTE
############################################

resource "aws_apigatewayv2_route" "webhook" {
  api_id    = aws_apigatewayv2_api.this.id
  route_key = "POST /webhook"
  target    = "integrations/${aws_apigatewayv2_integration.lambda.id}"
}

############################################
# CLOUDWATCH LOG GROUP
############################################

resource "aws_cloudwatch_log_group" "api_logs" {
  name              = "/aws/apigateway/${var.api_name}"
  retention_in_days = var.log_retention_days

  tags = var.common_tags
}

############################################
# STAGE WITH LOGGING
############################################

resource "aws_apigatewayv2_stage" "prod" {
  api_id      = aws_apigatewayv2_api.this.id
  name        = var.stage_name
  auto_deploy = true

  access_log_settings {
    destination_arn = aws_cloudwatch_log_group.api_logs.arn
    format = jsonencode({
      requestId              = "$context.requestId"
      ip                     = "$context.identity.sourceIp"
      requestTime            = "$context.requestTime"
      httpMethod             = "$context.httpMethod"
      routeKey               = "$context.routeKey"
      status                 = "$context.status"
      protocol               = "$context.protocol"
      responseLength         = "$context.responseLength"
      integrationErrorMessage = "$context.integrationErrorMessage"
      errorMessage           = "$context.error.message"
      errorMessageString     = "$context.error.messageString"
    })
  }

  # Throttling settings
  default_route_settings {
    throttling_burst_limit = var.throttling_burst_limit
    throttling_rate_limit  = var.throttling_rate_limit
  }

  tags = var.common_tags

  depends_on = [aws_cloudwatch_log_group.api_logs]
}

############################################
# LAMBDA PERMISSION
############################################

resource "aws_lambda_permission" "api_gateway" {
  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = var.lambda_function_name
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_apigatewayv2_api.this.execution_arn}/*/*"
}
```

---

## 📁 modules/api_gateway/variables.tf

```hcl
variable "api_name" {
  type        = string
  description = "Name of the API Gateway"
}

variable "lambda_invoke_arn" {
  type        = string
  description = "Invoke ARN of the Lambda function"
}

variable "lambda_function_name" {
  type        = string
  description = "Name of the Lambda function"
}

variable "common_tags" {
  type        = map(string)
  description = "Common tags to apply to all resources"
  default     = {}
}

variable "stage_name" {
  type        = string
  description = "Name of the API Gateway stage"
  default     = "prod"
}

variable "log_retention_days" {
  type        = number
  description = "CloudWatch log retention in days"
  default     = 7
}

variable "integration_timeout_ms" {
  type        = number
  description = "Integration timeout in milliseconds"
  default     = 30000
}

variable "throttling_burst_limit" {
  type        = number
  description = "Throttling burst limit"
  default     = 100
}

variable "throttling_rate_limit" {
  type        = number
  description = "Throttling rate limit (requests per second)"
  default     = 50
}

# CORS configuration
variable "cors_allow_origins" {
  type        = list(string)
  description = "CORS allowed origins"
  default     = ["*"]
}

variable "cors_allow_methods" {
  type        = list(string)
  description = "CORS allowed methods"
  default     = ["POST"]
}

variable "cors_allow_headers" {
  type        = list(string)
  description = "CORS allowed headers"
  default     = ["content-type"]
}

variable "cors_max_age" {
  type        = number
  description = "CORS max age in seconds"
  default     = 300
}
```

---

## 📁 modules/api_gateway/outputs.tf

```hcl
output "api_id" {
  description = "ID of the API Gateway"
  value       = aws_apigatewayv2_api.this.id
}

output "api_endpoint" {
  description = "Base URL of the API Gateway"
  value       = aws_apigatewayv2_stage.prod.invoke_url
}

output "webhook_url" {
  description = "Full webhook URL for Telegram"
  value       = "${aws_apigatewayv2_stage.prod.invoke_url}/webhook"
}

output "api_execution_arn" {
  description = "Execution ARN of the API Gateway"
  value       = aws_apigatewayv2_api.this.execution_arn
}

output "log_group_name" {
  description = "Name of the CloudWatch log group"
  value       = aws_cloudwatch_log_group.api_logs.name
}
```

---

## ✅ Quick Copy Instructions

1. Create the file structure (use PowerShell commands from QUICK_START.md)
2. Copy each section above into the corresponding file
3. Save all files
4. Run `terraform init`

**That's it!** All module code is now in one place for easy reference.