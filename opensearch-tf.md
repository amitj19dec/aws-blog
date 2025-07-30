Looking at this Terraform code, I can see it follows a well-structured approach for creating secure, isolated SageMaker workspaces. Let me analyze the current architecture and explain how to add OpenSearch that aligns with the existing patterns.

## Current Architecture Analysis

**Key Patterns Observed:**
1. **Naming Convention**: `${var.sagemaker_workspace_name}-{resource-type}` (equivalent to `project-{uuid}-` pattern)
2. **VPC Architecture**: Private subnets for compute, separate subnets for endpoints
3. **Security**: KMS encryption, security groups, VPC-only access, IP whitelisting
4. **Modular Design**: Separate modules for endpoints, S3, and IAM

## OpenSearch Terraform Implementation

Here's how I would add OpenSearch to this infrastructure:

### 1. Create OpenSearch Module Structure

**Directory: `src/terraform/infrastructure/opensearch/`**

**`opensearch/main.tf`**
```hcl
# Get current region and account
data "aws_region" "current" {}
data "aws_caller_identity" "current" {}

# OpenSearch Domain
resource "aws_opensearch_domain" "domain" {
  domain_name    = "${var.workspace_name}-vectorstore"
  engine_version = var.opensearch_version

  # Cluster configuration
  cluster_config {
    instance_type            = var.instance_type
    instance_count           = var.instance_count
    dedicated_master_enabled = var.dedicated_master_enabled
    dedicated_master_type    = var.master_instance_type
    dedicated_master_count   = var.master_instance_count
    zone_awareness_enabled   = var.multi_az_enabled
    
    dynamic "zone_awareness_config" {
      for_each = var.multi_az_enabled ? [1] : []
      content {
        availability_zone_count = var.availability_zone_count
      }
    }
  }

  # VPC configuration
  vpc_options {
    subnet_ids         = var.subnet_ids
    security_group_ids = [aws_security_group.opensearch_sg.id]
  }

  # EBS storage configuration
  ebs_options {
    ebs_enabled = true
    volume_type = var.volume_type
    volume_size = var.volume_size
    iops        = var.volume_type == "gp3" ? var.iops : null
    throughput  = var.volume_type == "gp3" ? var.throughput : null
  }

  # Encryption configuration
  encrypt_at_rest {
    enabled    = true
    kms_key_id = var.kms_key_arn
  }

  node_to_node_encryption {
    enabled = true
  }

  domain_endpoint_options {
    enforce_https       = true
    tls_security_policy = "Policy-Min-TLS-1-2-2019-07"
  }

  # Advanced security options
  advanced_security_options {
    enabled                        = true
    anonymous_auth_enabled         = false
    internal_user_database_enabled = false
    master_user_options {
      master_user_arn = var.master_user_arn
    }
  }

  # Logging configuration
  log_publishing_options {
    cloudwatch_log_group_arn = aws_cloudwatch_log_group.opensearch_logs.arn
    log_type                 = "INDEX_SLOW_LOGS"
    enabled                  = true
  }

  log_publishing_options {
    cloudwatch_log_group_arn = aws_cloudwatch_log_group.opensearch_logs.arn
    log_type                 = "SEARCH_SLOW_LOGS"
    enabled                  = true
  }

  log_publishing_options {
    cloudwatch_log_group_arn = aws_cloudwatch_log_group.opensearch_logs.arn
    log_type                 = "ES_APPLICATION_LOGS"
    enabled                  = true
  }

  # Access policy
  access_policies = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          AWS = var.authorized_principals
        }
        Action = "es:*"
        Resource = "arn:aws:es:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:domain/${var.workspace_name}-vectorstore/*"
        Condition = {
          IpAddress = {
            "aws:sourceIp" = var.allowed_cidrs
          }
        }
      }
    ]
  })

  tags = merge(var.tags, {
    Name       = "${var.workspace_name}-vectorstore"
    provisoner = "UAIS"
  })

  depends_on = [aws_cloudwatch_log_group.opensearch_logs]
}

# Security Group for OpenSearch
resource "aws_security_group" "opensearch_sg" {
  name        = "${var.workspace_name}-opensearch-sg"
  description = "Security group for OpenSearch domain"
  vpc_id      = var.vpc_id

  # HTTPS access from SageMaker and Lambda security groups
  ingress {
    description     = "HTTPS access from SageMaker"
    from_port       = 443
    to_port         = 443
    protocol        = "tcp"
    security_groups = var.allowed_security_groups
  }

  # OpenSearch API ports from within VPC
  ingress {
    description = "OpenSearch API from VPC"
    from_port   = 9200
    to_port     = 9200
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
  }

  ingress {
    description = "OpenSearch cluster communication"
    from_port   = 9300
    to_port     = 9300
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
  }

  # No outbound rules needed (OpenSearch doesn't initiate outbound connections)
  egress {
    description = "All outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(var.tags, {
    Name       = "${var.workspace_name}-opensearch-sg"
    provisoner = "UAIS"
  })
}

# CloudWatch Log Group for OpenSearch
resource "aws_cloudwatch_log_group" "opensearch_logs" {
  name              = "/aws/opensearch/${var.workspace_name}-vectorstore"
  retention_in_days = var.log_retention_days
  kms_key_id        = var.kms_key_arn

  tags = merge(var.tags, {
    Name       = "${var.workspace_name}-opensearch-logs"
    provisoner = "UAIS"
  })
}

# CloudWatch Log Resource Policy
resource "aws_cloudwatch_log_resource_policy" "opensearch_logs_policy" {
  policy_name = "${var.workspace_name}-opensearch-logs-policy"

  policy_document = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Service = "es.amazonaws.com"
        }
        Action = [
          "logs:PutLogEvents",
          "logs:CreateLogGroup",
          "logs:CreateLogStream"
        ]
        Resource = "arn:aws:logs:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:*"
      }
    ]
  })
}
```

**`opensearch/variables.tf`**
```hcl
variable "workspace_name" {
  description = "Workspace name for resource naming"
  type        = string
}

variable "vpc_id" {
  description = "VPC ID where OpenSearch will be deployed"
  type        = string
}

variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
}

variable "subnet_ids" {
  description = "Subnet IDs for OpenSearch deployment"
  type        = list(string)
}

variable "kms_key_arn" {
  description = "KMS key ARN for encryption"
  type        = string
}

variable "master_user_arn" {
  description = "ARN of the master user for OpenSearch"
  type        = string
}

variable "authorized_principals" {
  description = "List of authorized AWS principals"
  type        = list(string)
}

variable "allowed_security_groups" {
  description = "Security groups allowed to access OpenSearch"
  type        = list(string)
}

variable "allowed_cidrs" {
  description = "CIDR blocks allowed to access OpenSearch"
  type        = list(string)
}

variable "opensearch_version" {
  description = "OpenSearch engine version"
  type        = string
  default     = "OpenSearch_2.11"
}

variable "instance_type" {
  description = "Instance type for OpenSearch nodes"
  type        = string
  default     = "m6g.large.search"
}

variable "instance_count" {
  description = "Number of instances in the OpenSearch cluster"
  type        = number
  default     = 3
}

variable "dedicated_master_enabled" {
  description = "Enable dedicated master nodes"
  type        = bool
  default     = true
}

variable "master_instance_type" {
  description = "Instance type for master nodes"
  type        = string
  default     = "m6g.medium.search"
}

variable "master_instance_count" {
  description = "Number of dedicated master nodes"
  type        = number
  default     = 3
}

variable "multi_az_enabled" {
  description = "Enable multi-AZ deployment"
  type        = bool
  default     = true
}

variable "availability_zone_count" {
  description = "Number of availability zones"
  type        = number
  default     = 2
}

variable "volume_type" {
  description = "EBS volume type"
  type        = string
  default     = "gp3"
}

variable "volume_size" {
  description = "EBS volume size in GB"
  type        = number
  default     = 100
}

variable "iops" {
  description = "IOPS for gp3 volumes"
  type        = number
  default     = 3000
}

variable "throughput" {
  description = "Throughput for gp3 volumes"
  type        = number
  default     = 125
}

variable "log_retention_days" {
  description = "CloudWatch log retention period"
  type        = number
  default     = 90
}

variable "tags" {
  description = "Tags to apply to resources"
  type        = map(string)
  default     = {}
}
```

**`opensearch/outputs.tf`**
```hcl
output "domain_arn" {
  description = "ARN of the OpenSearch domain"
  value       = aws_opensearch_domain.domain.arn
}

output "domain_id" {
  description = "Unique identifier for the OpenSearch domain"
  value       = aws_opensearch_domain.domain.domain_id
}

output "domain_name" {
  description = "Name of the OpenSearch domain"
  value       = aws_opensearch_domain.domain.domain_name
}

output "endpoint" {
  description = "Domain-specific endpoint used to submit index, search, and data upload requests"
  value       = aws_opensearch_domain.domain.endpoint
}

output "kibana_endpoint" {
  description = "Domain-specific endpoint for Kibana"
  value       = aws_opensearch_domain.domain.kibana_endpoint
}

output "security_group_id" {
  description = "ID of the OpenSearch security group"
  value       = aws_security_group.opensearch_sg.id
}

output "log_group_name" {
  description = "Name of the CloudWatch log group"
  value       = aws_cloudwatch_log_group.opensearch_logs.name
}
```

### 2. Update Main Infrastructure

**Add to `main.tf`:**
```hcl
# Add OpenSearch module after the existing modules
module "opensearch" {
  source = "./opensearch"

  workspace_name             = var.sagemaker_workspace_name
  vpc_id                    = module.spoke_vpc.vpc_attributes.id
  vpc_cidr                  = module.spoke_vpc.vpc_attributes.cidr_block
  subnet_ids                = values({ for k, v in module.spoke_vpc.private_subnet_attributes_by_az : split("/", k)[1] => v.id if split("/", k)[0] == "sagemaker_subnet" })
  kms_key_arn              = aws_kms_key.sagemaker_kms_key.arn
  master_user_arn          = module.sagemaker_domain_execution_role.sagemaker_role_arn
  authorized_principals    = [module.sagemaker_domain_execution_role.sagemaker_role_arn]
  allowed_security_groups  = [aws_security_group.sagemaker_sg.id]
  allowed_cidrs           = local.optum_ip_whitelist
  
  tags = merge(var.tags, { provisoner = local.resource_provisioner })
}
```

### 3. Update IAM Permissions

**Add to `iam/iam.tf`:**
```hcl
# Add OpenSearch permissions to the existing SageMaker execution policy
resource "aws_iam_policy" "opensearch_access_policy" {
  name        = "${var.sagemaker_workspace_name}-OpenSearchAccess"
  description = "Policy to allow access to OpenSearch domain"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "es:ESHttpGet",
          "es:ESHttpPost", 
          "es:ESHttpPut",
          "es:ESHttpDelete",
          "es:ESHttpHead",
          "es:DescribeDomains",
          "es:ListDomainNames"
        ]
        Resource = [
          "arn:aws:es:*:*:domain/${var.sagemaker_workspace_name}-vectorstore",
          "arn:aws:es:*:*:domain/${var.sagemaker_workspace_name}-vectorstore/*"
        ]
      }
    ]
  })
}

# Attach OpenSearch policy to SageMaker role
resource "aws_iam_role_policy_attachment" "attach_opensearch_policy" {
  role       = aws_iam_role.sagemaker_domain_default_execution_role.name
  policy_arn = aws_iam_policy.opensearch_access_policy.arn
}
```

### 4. Add VPC Endpoint for OpenSearch

**Update `endpoints/main.tf`:**
```hcl
# Add to the existing endpoint_names variable default
variable "endpoint_names" {
  type        = list(string)
  description = "VPC Endpoint Names"
  default     = [...existing..., "es"] # Add "es" for OpenSearch
}
```

### 5. Update KMS Key Policy

**Update `main.tf` KMS key policy:**
```hcl
resource "aws_kms_key_policy" "sagemaker_key_policy" {
  key_id = aws_kms_key.sagemaker_kms_key.id
  policy = jsonencode({
    Id = "sagemaker_key_policy"
    Statement = [
      {
        Action = "kms:*"
        Effect = "Allow"
        Principal = {
          AWS = [
            data.aws_caller_identity.current.account_id,
            module.sagemaker_domain_execution_role.sagemaker_role_arn
          ]
        }
        Resource = "*"
        Sid      = "Enable IAM User Permissions"
      },
      # Add OpenSearch service access
      {
        Effect = "Allow"
        Principal = {
          Service = "es.amazonaws.com"
        }
        Action = [
          "kms:Encrypt",
          "kms:Decrypt",
          "kms:ReEncrypt*",
          "kms:GenerateDataKey*",
          "kms:DescribeKey"
        ]
        Resource = "*"
        Sid      = "Allow OpenSearch Service"
      }
    ]
    Version = "2012-10-17"
  })
}
```

## Key Design Decisions

### 1. **Security-First Approach**
- **VPC-Only Deployment**: No internet access, following existing pattern
- **Fine-Grained Access Control**: Uses advanced security options with IAM integration
- **Encryption**: At-rest and in-transit encryption using the same KMS key
- **Network Isolation**: Dedicated security group with minimal required access

### 2. **Integration with Existing Architecture**
- **Naming Convention**: Follows `${workspace_name}-vectorstore` pattern
- **Subnet Placement**: Uses same private subnets as SageMaker
- **IAM Integration**: Integrates with existing SageMaker execution role
- **Tagging**: Consistent tagging strategy with existing resources

### 3. **Production-Ready Features**
- **Multi-AZ Deployment**: High availability across availability zones  
- **Dedicated Master Nodes**: Improved cluster stability
- **Comprehensive Logging**: All log types sent to CloudWatch
- **Monitoring Ready**: CloudWatch integration for metrics and alarms

### 4. **Resource Optimization**
- **Right-Sized Instances**: m6g.large for data nodes, m6g.medium for masters
- **GP3 Storage**: Cost-effective with configurable IOPS and throughput
- **Log Retention**: 90-day retention aligning with compliance requirements

This OpenSearch implementation seamlessly integrates with the existing SageMaker infrastructure while maintaining the same security posture, naming conventions, and operational patterns. The ML scientists can then use this OpenSearch cluster through their Lambda functions with the project-scoped IAM role we discussed earlier.
