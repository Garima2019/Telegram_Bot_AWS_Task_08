# Changelog

All notable changes to this Terraform infrastructure will be documented in this file.

## [2.0.0] - 2026-01-14

### 🎯 Major Refactor - Class 8 Improvements

#### Added
- **Module Structure**: Introduced separate modules for Lambda, API Gateway, DynamoDB, and S3
- **Remote State**: Added S3 backend configuration with DynamoDB state locking
- **Least-Privilege IAM**: Implemented resource-scoped permissions for Lambda execution role
  - S3 permissions limited to specific bucket and enforce encryption
  - DynamoDB permissions scoped to specific table and indexes
  - Removed all wildcard actions and resources
- **Comprehensive Tagging**: Added consistent tags across all resources
  - Project, Environment, ManagedBy, Team, CostCenter
  - Applied via default_tags at provider level
- **Enhanced Outputs**: Added detailed outputs for all major resources and deployment info
- **Variable Validation**: Added validation rules for critical variables
- **Logging Configuration**: Structured JSON logging for Lambda and API Gateway
- **Throttling**: Added API Gateway throttling protection
- **Auto-scaling**: DynamoDB auto-scaling support for provisioned capacity mode
- **Security Hardening**:
  - S3 bucket versioning and lifecycle management
  - Point-in-time recovery for DynamoDB (prod)
  - Server-side encryption enforcement
  - Public access blocking for S3

#### Changed
- **Breaking**: Root module now orchestrates child modules instead of creating resources directly
- **Breaking**: Variable names standardized (e.g., `s3_bucket_name` remains consistent)
- **Breaking**: IAM roles now have resource-scoped permissions (may require policy updates)
- **Improved**: CloudWatch log groups now created explicitly with retention policies
- **Improved**: API Gateway stage includes detailed access logging in JSON format

#### Removed
- Monolithic `main.tf` in favor of modular structure
- Overly permissive IAM policies with wildcards
- Hardcoded ARNs and resource names in policies

#### Documentation
- Added comprehensive README with deployment instructions
- Module-level documentation for all modules
- terraform.tfvars.example with all configurable options
- Prerequisites and setup guide for remote state

### Migration Guide from 1.x to 2.0

1. **Backup your state file**:
   ```bash
   cp terraform.tfstate terraform.tfstate.backup
   ```

2. **Set up remote state backend**:
   - Create S3 bucket for state
   - Create DynamoDB table for locking
   - Update `backend.tf` with your bucket name

3. **Update variable values**:
   - Copy `terraform.tfvars.example` to `terraform.tfvars`
   - Update with your configuration

4. **Initialize new backend**:
   ```bash
   terraform init -migrate-state
   ```

5. **Review changes**:
   ```bash
   terraform plan
   ```

6. **Apply changes**:
   ```bash
   terraform apply
   ```

---

## [1.0.0] - Initial Release

### Added
- Basic Lambda function for Telegram bot
- DynamoDB table for message storage
- S3 bucket for file storage
- API Gateway HTTP API
- Basic IAM roles and policies