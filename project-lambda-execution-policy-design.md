# Project Lambda Execution Policy Design

## Overview

This document outlines the design and implementation of the `project-{uuid}-lambda-role` execution policy for the UHC Ask AI Platform. This role enables ML scientists to build business logic within strict project boundaries while maintaining security isolation and governance.

## Architecture Principles

### Core Design Philosophy
- **Isolation by Design**: Every Lambda function operates within project-specific resource boundaries
- **Least Privilege Access**: Minimal permissions required for business logic implementation
- **Network Containment**: All operations constrained within project VPC
- **Auditability**: Complete traceability of all resource access and modifications

### Shared Responsibility Model
- **Engineering Team**: Provisions infrastructure, IAM roles, and network components
- **ML Scientists**: Implement business logic using pre-provisioned execution role
- **Governance**: Automated compliance through naming conventions and resource policies

---

## Lambda Execution Role Structure

### Role Configuration
```yaml
Role Name: project-{uuid}-lambda-role
Trust Policy: lambda.amazonaws.com
Policy Type: Inline policy (no managed policies)
Policy Name: project-{uuid}-lambda-execution-policy
```

### Trust Policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

---

## Execution Policy Definition

### 1. Foundation Permissions

#### VPC Networking
```json
{
  "Sid": "VPCNetworking",
  "Effect": "Allow",
  "Action": [
    "ec2:CreateNetworkInterface",
    "ec2:DescribeNetworkInterfaces",
    "ec2:DeleteNetworkInterface",
    "ec2:AttachNetworkInterface",
    "ec2:DetachNetworkInterface"
  ],
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "ec2:vpc": "arn:aws:ec2:*:*:vpc/project-${uuid}-vpc"
    }
  }
}
```

#### CloudWatch Logging
```json
{
  "Sid": "CloudWatchLogs",
  "Effect": "Allow",
  "Action": [
    "logs:CreateLogGroup",
    "logs:CreateLogStream",
    "logs:PutLogEvents",
    "logs:DescribeLogGroups",
    "logs:DescribeLogStreams"
  ],
  "Resource": [
    "arn:aws:logs:*:*:log-group:/aws/lambda/project-${uuid}-*",
    "arn:aws:logs:*:*:log-group:/aws/lambda/project-${uuid}-*:*"
  ]
}
```

### 2. Data Services Access

#### Amazon S3
```json
{
  "Sid": "S3Access",
  "Effect": "Allow",
  "Action": [
    "s3:GetObject",
    "s3:PutObject",
    "s3:DeleteObject",
    "s3:ListBucket",
    "s3:GetBucketLocation",
    "s3:GetObjectVersion"
  ],
  "Resource": [
    "arn:aws:s3:::project-${uuid}-*",
    "arn:aws:s3:::project-${uuid}-*/*"
  ]
}
```

#### Amazon OpenSearch
```json
{
  "Sid": "OpenSearchAccess",
  "Effect": "Allow",
  "Action": [
    "es:ESHttpGet",
    "es:ESHttpPost",
    "es:ESHttpPut",
    "es:ESHttpDelete",
    "es:ESHttpHead"
  ],
  "Resource": "arn:aws:es:*:*:domain/project-${uuid}-*/*"
}
```

#### DynamoDB
```json
{
  "Sid": "DynamoDBAccess",
  "Effect": "Allow",
  "Action": [
    "dynamodb:GetItem",
    "dynamodb:PutItem",
    "dynamodb:UpdateItem",
    "dynamodb:DeleteItem",
    "dynamodb:Query",
    "dynamodb:Scan",
    "dynamodb:BatchGetItem",
    "dynamodb:BatchWriteItem",
    "dynamodb:DescribeTable"
  ],
  "Resource": [
    "arn:aws:dynamodb:*:*:table/project-${uuid}-*",
    "arn:aws:dynamodb:*:*:table/project-${uuid}-*/index/*"
  ]
}
```

### 3. AI/ML Services Access

#### Amazon Bedrock
```json
{
  "Sid": "BedrockAccess",
  "Effect": "Allow",
  "Action": [
    "bedrock:InvokeModel",
    "bedrock:GetKnowledgeBase",
    "bedrock:RetrieveAndGenerate",
    "bedrock:Retrieve",
    "bedrock:ListKnowledgeBases"
  ],
  "Resource": "*",
  "Condition": {
    "StringLike": {
      "bedrock:knowledgeBaseId": "project-${uuid}-*"
    }
  }
}
```

#### Amazon SageMaker
```json
{
  "Sid": "SageMakerInference",
  "Effect": "Allow",
  "Action": [
    "sagemaker:InvokeEndpoint",
    "sagemaker:DescribeEndpoint",
    "sagemaker:ListEndpoints"
  ],
  "Resource": "arn:aws:sagemaker:*:*:endpoint/project-${uuid}-*"
}
```

### 4. Graph Database and Cache Access

#### Amazon Neptune
```json
{
  "Sid": "NeptuneAccess",
  "Effect": "Allow",
  "Action": [
    "neptune-db:connect",
    "neptune-db:ReadDataViaQuery",
    "neptune-db:WriteDataViaQuery",
    "neptune-db:DeleteDataViaQuery"
  ],
  "Resource": "arn:aws:neptune-db:*:*:cluster/project-${uuid}-*/database/*"
}
```

#### ElastiCache
```json
{
  "Sid": "ElastiCacheAccess",
  "Effect": "Allow",
  "Action": [
    "elasticache:DescribeCacheClusters",
    "elasticache:DescribeReplicationGroups"
  ],
  "Resource": "arn:aws:elasticache:*:*:replicationgroup/project-${uuid}-*"
}
```

### 5. Configuration and Secrets Management

#### AWS Systems Manager Parameter Store
```json
{
  "Sid": "ParameterStoreAccess",
  "Effect": "Allow",
  "Action": [
    "ssm:GetParameter",
    "ssm:GetParameters",
    "ssm:GetParametersByPath"
  ],
  "Resource": "arn:aws:ssm:*:*:parameter/project-${uuid}/*"
}
```

#### AWS Secrets Manager
```json
{
  "Sid": "SecretsManagerAccess",
  "Effect": "Allow",
  "Action": [
    "secretsmanager:GetSecretValue",
    "secretsmanager:DescribeSecret"
  ],
  "Resource": "arn:aws:secretsmanager:*:*:secret:project-${uuid}-*"
}
```

#### AWS KMS
```json
{
  "Sid": "KMSAccess",
  "Effect": "Allow",
  "Action": [
    "kms:Decrypt",
    "kms:GenerateDataKey"
  ],
  "Resource": "arn:aws:kms:*:*:key/*",
  "Condition": {
    "StringLike": {
      "kms:ViaService": [
        "s3.*.amazonaws.com",
        "secretsmanager.*.amazonaws.com",
        "ssm.*.amazonaws.com",
        "dynamodb.*.amazonaws.com"
      ]
    }
  }
}
```

### 6. Messaging and Event Services

#### Amazon SQS
```json
{
  "Sid": "SQSAccess",
  "Effect": "Allow",
  "Action": [
    "sqs:SendMessage",
    "sqs:ReceiveMessage",
    "sqs:DeleteMessage",
    "sqs:GetQueueAttributes",
    "sqs:GetQueueUrl"
  ],
  "Resource": "arn:aws:sqs:*:*:project-${uuid}-*"
}
```

#### Amazon SNS
```json
{
  "Sid": "SNSAccess",
  "Effect": "Allow",
  "Action": [
    "sns:Publish",
    "sns:GetTopicAttributes"
  ],
  "Resource": "arn:aws:sns:*:*:project-${uuid}-*"
}
```

### 7. Monitoring and Observability

#### CloudWatch Metrics
```json
{
  "Sid": "CloudWatchMetrics",
  "Effect": "Allow",
  "Action": [
    "cloudwatch:PutMetricData"
  ],
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "cloudwatch:namespace": "project-${uuid}/application"
    }
  }
}
```

#### AWS X-Ray
```json
{
  "Sid": "XRayTracing",
  "Effect": "Allow",
  "Action": [
    "xray:PutTraceSegments",
    "xray:PutTelemetryRecords"
  ],
  "Resource": "*"
}
```

---

## Security Boundaries and Constraints

### Network Isolation
- **VPC Deployment**: All Lambda functions must deploy within `project-{uuid}-vpc`
- **Subnet Restrictions**: Functions limited to engineering-provided private subnets
- **Security Groups**: Managed exclusively by engineering team
- **Internet Access**: Only through NAT Gateway or VPC endpoints

### Resource Scope Enforcement
- **Naming Convention**: All accessible resources follow `project-{uuid}-*` pattern
- **Cross-Project Isolation**: Zero access to other project resources
- **Service Boundaries**: Limited to specific AWS services within project scope

### Compliance Requirements
- **Tagging**: All Lambda functions auto-tagged with project metadata
- **Audit Logging**: CloudTrail captures all resource access
- **Cost Allocation**: Resource usage tracked per project for billing

---

## Implementation Guidelines

### Lambda Function Requirements
```python
# Required environment variables
ENVIRONMENT_VARIABLES = {
    'PROJECT_ID': 'project-{uuid}',
    'VPC_ID': 'project-{uuid}-vpc',
    'OPENSEARCH_ENDPOINT': 'project-{uuid}-vectorstore.region.es.amazonaws.com',
    'S3_BUCKET': 'project-{uuid}-documents',
    'NEPTUNE_ENDPOINT': 'project-{uuid}-knowledge-graph.cluster-xyz.region.neptune.amazonaws.com'
}

# Required Lambda configuration
LAMBDA_CONFIG = {
    'Role': 'arn:aws:iam::account:role/project-{uuid}-lambda-role',
    'VpcConfig': {
        'SubnetIds': ['subnet-private-1a', 'subnet-private-1b'],
        'SecurityGroupIds': ['sg-project-{uuid}-lambda']
    },
    'TracingConfig': {'Mode': 'Active'},
    'Environment': {'Variables': ENVIRONMENT_VARIABLES}
}
```

### Service Integration Patterns
```python
# Example: OpenSearch client initialization
import boto3
from opensearchpy import OpenSearch, RequestsHttpConnection
from aws_requests_auth.aws_auth import AWSRequestsAuth

def get_opensearch_client():
    host = os.environ['OPENSEARCH_ENDPOINT']
    region = os.environ['AWS_REGION']
    service = 'es'
    credentials = boto3.Session().get_credentials()
    awsauth = AWSRequestsAuth(credentials, region, service)
    
    client = OpenSearch(
        hosts=[{'host': host, 'port': 443}],
        http_auth=awsauth,
        use_ssl=True,
        verify_certs=True,
        connection_class=RequestsHttpConnection
    )
    return client
```

### Error Handling and Monitoring
```python
import json
import logging
from datetime import datetime

# Configure logging
logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    try:
        # Business logic implementation
        result = process_request(event)
        
        # Custom metrics
        put_custom_metric('ProcessingSuccess', 1)
        
        return {
            'statusCode': 200,
            'body': json.dumps(result)
        }
    
    except Exception as e:
        logger.error(f"Processing failed: {str(e)}")
        put_custom_metric('ProcessingError', 1)
        
        return {
            'statusCode': 500,
            'body': json.dumps({'error': 'Processing failed'})
        }

def put_custom_metric(metric_name, value):
    cloudwatch = boto3.client('cloudwatch')
    cloudwatch.put_metric_data(
        Namespace=f"project-{os.environ['PROJECT_ID']}/application",
        MetricData=[
            {
                'MetricName': metric_name,
                'Value': value,
                'Timestamp': datetime.utcnow()
            }
        ]
    )
```

---

## Governance and Compliance

### Automated Compliance Checks
- **AWS Config Rules**: Validate naming conventions and resource configurations
- **Cost Budgets**: Monitor spending per project with automated alerts
- **Access Analyzer**: Ensure no unintended external access to project resources

### ML Scientist Responsibilities
1. **Follow naming conventions** for all created resources
2. **Use provided execution role** exclusively for Lambda functions
3. **Implement proper error handling** and logging
4. **Monitor resource usage** and optimize for cost
5. **Document business logic** and integration patterns

### Engineering Team Oversight
1. **Regular policy reviews** for security compliance
2. **Resource utilization monitoring** across all projects
3. **Access pattern analysis** for potential security issues
4. **Infrastructure updates** and maintenance windows

---

## Troubleshooting Guide

### Common Permission Issues
- **VPC Access Denied**: Verify Lambda is deployed in correct VPC and subnets
- **OpenSearch 403 Errors**: Check resource-based policies and security groups
- **S3 Access Issues**: Validate bucket naming and IAM policy resource ARNs
- **Parameter Store Access**: Confirm parameter paths follow `/project-{uuid}/` structure

### Monitoring and Debugging
- **CloudWatch Logs**: Check `/aws/lambda/project-{uuid}-*` log groups
- **X-Ray Traces**: Analyze request flow and performance bottlenecks
- **VPC Flow Logs**: Investigate network connectivity issues
- **CloudTrail**: Audit API calls and permission denied events

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-07-26 | Initial policy design and documentation |

---

## Contact and Support

For questions regarding this policy or implementation support:
- **Engineering Team**: engineering-team@company.com
- **ML Platform Team**: ml-platform@company.com
- **Security Team**: security@company.com
