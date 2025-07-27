# ML Scientist IAM Permissions Summary

## Overview
This document outlines the IAM permissions granted to ML Scientists within the project-based resource isolation model. All permissions are scoped to resources following the `project-{uuid}-` naming convention.

## 1. Lambda Function Permissions

### ✅ **Allowed Actions:**
- Create, update, delete Lambda functions
- Configure function settings (memory, timeout, environment variables)
- Manage function versions and aliases
- Create and manage event source mappings
- Configure triggers from approved sources

### 🔒 **Mandatory Conditions:**
- **Function Name:** Must follow `project-{uuid}-*` pattern
- **Execution Role:** Must use `project-{uuid}-lambda-role` only
- **VPC Configuration:** Must deploy in `project-{uuid}-vpc`
- **Subnets:** Must use project-specific private subnets only
- **Security Groups:** Must use project-specific security groups

### 📋 **Event Source Restrictions:**
- Can only connect to resources within project scope:
  - S3 buckets: `project-{uuid}-*`
  - DynamoDB tables: `project-{uuid}-*`
  - SQS queues: `project-{uuid}-*`
  - API Gateway: `project-{uuid}-*`

## 2. Step Functions Permissions

### ✅ **Allowed Actions:**
- Create, update, delete state machines
- Start and stop executions
- View execution history and logs

### 🔒 **Mandatory Conditions:**
- **State Machine Name:** Must follow `project-{uuid}-*` pattern
- **Lambda Integration:** Can only invoke project-scoped Lambda functions
- **IAM Role:** Must use project-specific execution role

## 3. DynamoDB Permissions

### ✅ **Allowed Actions:**
- Create, update, delete tables
- Configure table settings (indexes, streams, etc.)
- Read and write data operations
- Manage table backups and point-in-time recovery

### 🔒 **Mandatory Conditions:**
- **Table Name:** Must follow `project-{uuid}-*` pattern
- **Regional Scope:** Limited to approved AWS regions

## 4. Amazon Lex Permissions

### ✅ **Allowed Actions:**
- Create and manage Lex bots
- Configure intents, slots, and slot types
- Build and test bots
- Deploy bot versions and aliases

### 🔒 **Mandatory Conditions:**
- **Bot Name:** Must follow `project-{uuid}-*` pattern
- **Integration:** Can only integrate with project-scoped Lambda functions

## 5. Parameter Store & Secrets Manager Access

### ✅ **Allowed Actions:**
- **Parameter Store:**
  - Get, put, delete parameters under `/project-{uuid}/` hierarchy
  - Create parameter hierarchies for application configuration
  
- **Secrets Manager:**
  - Create, retrieve, update secrets with `project-{uuid}-*` prefix
  - Manage secret versions and rotation

### 🔒 **Restrictions:**
- Cannot access parameters/secrets outside project scope
- Must use project-specific KMS keys for encryption

## 6. Access to Engineering-Provisioned Resources

### 📖 **Read-Only Access:**
- **S3 Buckets:** `project-{uuid}-*`
  - GetObject, PutObject, DeleteObject permissions
  - ListBucket for project buckets only

- **OpenSearch:** `project-{uuid}-*`
  - Search, index, and manage documents
  - Create and manage indexes
  - Configure mappings and settings

- **Neptune:** `project-{uuid}-*`
  - Query and update graph database
  - Manage graph data and relationships

- **ElastiCache:** `project-{uuid}-*`
  - Read and write cache data
  - Manage cache keys and values

- **Bedrock Knowledge Base:** `project-{uuid}-*`
  - Query knowledge base
  - Retrieve and rank documents

- **SageMaker:**
  - Create and manage endpoints
  - Deploy models to project-scoped endpoints
  - Access project-specific SageMaker domains

## 7. Network & Security Constraints

### 🚫 **Enforced Restrictions:**
- **No Internet Access:** All Lambda functions must be in private subnets
- **VPC Boundaries:** Cannot access resources outside project VPC
- **Security Groups:** Must use project-specific security groups
- **VPC Endpoints:** External API calls must go through approved VPC endpoints

## 8. Tagging Requirements

### 🏷️ **Auto-Applied Tags:**
- **Project:** `project-{uuid}`
- **Owner:** `ml-team`
- **Environment:** `${environment}` (dev/staging/prod)
- **CostCenter:** `${cost-center}`

### 📋 **Manual Tagging:**
- ML Scientists can add additional tags for resource organization
- Cannot override or remove engineering-applied tags

## 9. Denied Actions

### ❌ **Explicitly Prohibited:**
- Creating resources outside project naming convention
- Using IAM roles other than project-specific roles
- Accessing cross-project resources
- Deploying functions outside project VPC
- Creating internet-facing resources
- Modifying engineering-provisioned infrastructure
- Using external Lambda layers not approved by Engineering

## 10. Monitoring & Compliance

### 📊 **Automatic Monitoring:**
- All actions logged via CloudTrail
- Resource creation/modification alerts
- Cost allocation by project tags
- Compliance violations automatically flagged

### 🔍 **Audit Requirements:**
- Regular review of resource utilization
- Compliance with security policies
- Cost optimization recommendations

## 11. Support & Escalation

### 🆘 **Engineering Team Support:**
- Infrastructure issues or limitations
- VPC connectivity problems
- Performance optimization of managed services
- Security group modifications

### 🔧 **Self-Service Capabilities:**
- Application logic development
- Business rule implementation
- Data processing workflows
- Model integration and deployment

---

**Last Updated:** July 26, 2025  
**Document Version:** 1.0  
**Approval:** Engineering Team & ML Leadership
