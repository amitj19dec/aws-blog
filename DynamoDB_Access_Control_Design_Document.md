# DynamoDB Creation and Access Control Design Document

## Document Information
- **Document Version**: 1.0
- **Last Updated**: July 26, 2025
- **Owner**: Engineering Team
- **Stakeholders**: ML Engineering Team, Security Team, Platform Architecture
- **Classification**: Internal Use
- **Review Cycle**: Quarterly

---

## Table of Contents
1. [Executive Summary](#1-executive-summary)
2. [Architecture Overview](#2-architecture-overview)
3. [Security Model](#3-security-model)
4. [Network Isolation Strategy](#4-network-isolation-strategy)
5. [Automated Governance Implementation](#5-automated-governance-implementation)
6. [ML Scientist Operations](#6-ml-scientist-operations)
7. [Monitoring and Compliance](#7-monitoring-and-compliance)
8. [Operational Procedures](#8-operational-procedures)
9. [Troubleshooting Guide](#9-troubleshooting-guide)
10. [Implementation Timeline](#10-implementation-timeline)
11. [Appendices](#11-appendices)

---

## 1. Executive Summary

### 1.1 Purpose
This document defines the architecture, security controls, and operational procedures for DynamoDB table creation and access within the UHC Ask AI Platform's project-based isolation model. The design ensures complete data isolation between projects while enabling ML scientist self-service capabilities.

### 1.2 Key Principles
- **Defense in Depth**: Multiple security layers for comprehensive protection
- **Zero Trust Access**: All access explicitly verified at multiple control points
- **VPC-Only Operations**: Complete network isolation from internet traffic
- **Automated Governance**: EventBridge-driven policy enforcement
- **Project Isolation**: Bulletproof separation between project resources

### 1.3 Shared Responsibility Model
- **Engineering Team**: Infrastructure provisioning, network security, automated governance
- **ML Scientists**: Table creation, data operations, application logic within security boundaries
- **Security Team**: Policy compliance monitoring, incident response, audit oversight

### 1.4 Success Metrics
- **100%** VPC-only DynamoDB access compliance
- **< 30 seconds** automated policy attachment time
- **Zero** cross-project data access incidents
- **99.9%** table creation success rate with governance automation

---

## 2. Architecture Overview

### 2.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Project-{uuid} VPC                           │
│  ┌─────────────────┐    ┌──────────────────┐    ┌─────────────┐ │
│  │  ML Scientist   │    │   Lambda Func    │    │   DynamoDB  │ │
│  │     Role        │────│ project-{uuid}-* │────│ project-{...│ │
│  └─────────────────┘    └──────────────────┘    └─────────────┘ │
│           │                       │                     │       │
│           │              ┌─────────────────┐           │       │
│           └──────────────│ VPC Gateway     │───────────┘       │
│                          │ Endpoint        │                   │
│                          │ (DynamoDB)      │                   │
│                          └─────────────────┘                   │
└─────────────────────────────────────────────────────────────────┘
                                   │
┌─────────────────────────────────────────────────────────────────┐
│                    Governance Layer                             │
│  ┌─────────────────┐    ┌──────────────────┐    ┌─────────────┐ │
│  │   EventBridge   │────│ Governance       │────│  Resource   │ │
│  │   Rule          │    │ Lambda           │    │  Policy     │ │
│  └─────────────────┘    └──────────────────┘    └─────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Component Responsibilities

#### 2.2.1 Engineering Team Provisioned Components
- **VPC Infrastructure**: Network foundation for complete isolation
- **VPC Gateway Endpoint**: Private AWS backbone connectivity for DynamoDB
- **IAM Roles**: Base execution and user roles with project-scoped permissions
- **EventBridge Rules**: Automated governance trigger configuration
- **Governance Lambda**: Policy attachment and compliance enforcement
- **Monitoring Infrastructure**: CloudWatch dashboards, alarms, and audit logging

#### 2.2.2 ML Scientist Managed Components
- **DynamoDB Tables**: Application data storage following naming conventions
- **Lambda Functions**: Business logic implementation for data operations
- **Application Logic**: Query patterns, data processing, and integration workflows
- **Custom Metrics**: Application-specific monitoring and performance tracking

### 2.3 Data Flow Architecture

```
Table Creation Flow:
ML Scientist → CreateTable API → DynamoDB Table Created
                    ↓
CloudTrail Event → EventBridge Rule → Governance Lambda
                    ↓
Resource Policy Attached → Compliance Verified → Audit Logged

Data Access Flow:
Lambda Function → VPC Gateway Endpoint → DynamoDB Service
       ↓                   ↓                    ↓
IAM Policy Check → Endpoint Policy → Resource Policy Check
       ↓                   ↓                    ↓
    ALLOW             ALLOW              ALLOW = ACCESS GRANTED
```

---

## 3. Security Model

### 3.1 Multi-Layer Security Architecture

#### 3.1.1 Layer 1: Identity-Based Access Control (IAM)

**Lambda Execution Role Permissions**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DynamoDBVPCOnlyAccess",
      "Effect": "Allow",
      "Action": ["dynamodb:*"],
      "Resource": [
        "arn:aws:dynamodb:*:*:table/project-${uuid}-*",
        "arn:aws:dynamodb:*:*:table/project-${uuid}-*/index/*",
        "arn:aws:dynamodb:*:*:table/project-${uuid}-*/stream/*"
      ],
      "Condition": {
        "StringEquals": {
          "aws:SourceVpce": "vpce-project-${uuid}-dynamodb",
          "aws:SourceVpc": "vpc-project-${uuid}"
        }
      }
    },
    {
      "Sid": "DenyNonVPCAccess",
      "Effect": "Deny",
      "Action": "dynamodb:*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:SourceVpce": "vpce-project-${uuid}-dynamodb"
        }
      }
    }
  ]
}
```

**ML Scientist Role Permissions**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DynamoDBTableManagement",
      "Effect": "Allow",
      "Action": [
        "dynamodb:CreateTable",
        "dynamodb:UpdateTable",
        "dynamodb:DeleteTable",
        "dynamodb:DescribeTable",
        "dynamodb:ListTables",
        "dynamodb:TagResource",
        "dynamodb:UntagResource",
        "dynamodb:ListTagsOfResource",
        "dynamodb:CreateBackup",
        "dynamodb:DeleteBackup",
        "dynamodb:DescribeBackup",
        "dynamodb:ListBackups",
        "dynamodb:RestoreTableFromBackup",
        "dynamodb:RestoreTableToPointInTime"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "dynamodb:TableName": "project-${uuid}-*"
        },
        "StringEquals": {
          "aws:RequestedRegion": ["us-east-1", "us-west-2"],
          "aws:SourceVpce": "vpce-project-${uuid}-dynamodb"
        }
      }
    },
    {
      "Sid": "DenyNonProjectTables",
      "Effect": "Deny",
      "Action": "dynamodb:*",
      "Resource": "*",
      "Condition": {
        "StringNotLike": {
          "dynamodb:TableName": "project-${uuid}-*"
        }
      }
    }
  ]
}
```

#### 3.1.2 Layer 2: Network-Based Access Control

**VPC Gateway Endpoint Policy**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ProjectTableAccess",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "dynamodb:*",
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "dynamodb:tablename": "project-${uuid}-*"
        },
        "StringEquals": {
          "aws:PrincipalArn": [
            "arn:aws:iam::${account}:role/project-${uuid}-lambda-role",
            "arn:aws:iam::${account}:role/project-${uuid}-ml-scientist-role"
          ]
        }
      }
    },
    {
      "Sid": "DenyOtherProjects",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "dynamodb:*",
      "Resource": "*",
      "Condition": {
        "StringNotLike": {
          "dynamodb:tablename": "project-${uuid}-*"
        }
      }
    }
  ]
}
```

#### 3.1.3 Layer 3: Resource-Based Access Control

**Auto-Attached Table Resource Policy Template**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ProjectVPCOnlyAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::${account}:role/project-${uuid}-lambda-role",
          "arn:aws:iam::${account}:role/project-${uuid}-ml-scientist-role"
        ]
      },
      "Action": "dynamodb:*",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:SourceVpce": "vpce-project-${uuid}-dynamodb",
          "aws:SourceVpc": "vpc-project-${uuid}"
        }
      }
    },
    {
      "Sid": "DenyAllNonVPCAccess",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:SourceVpce": "vpce-project-${uuid}-dynamodb"
        }
      }
    },
    {
      "Sid": "DenyRootUser",
      "Effect": "Deny",
      "Principal": {
        "AWS": "arn:aws:iam::${account}:root"
      },
      "Action": "*",
      "Resource": "*"
    }
  ]
}
```

### 3.2 Encryption Strategy

#### 3.2.1 Encryption at Rest
```yaml
# Engineering Team Provisions
ProjectDynamoDBKMSKey:
  Type: AWS::KMS::Key
  Properties:
    Description: "DynamoDB encryption key for project-${uuid}"
    KeyPolicy:
      Version: '2012-10-17'
      Statement:
        - Sid: Enable IAM User Permissions
          Effect: Allow
          Principal:
            AWS: !Sub 'arn:aws:iam::${AWS::AccountId}:root'
          Action: 'kms:*'
          Resource: '*'
        - Sid: Allow Lambda Role Access
          Effect: Allow
          Principal:
            AWS: !GetAtt ProjectLambdaRole.Arn
          Action:
            - kms:Decrypt
            - kms:GenerateDataKey
          Resource: '*'
          Condition:
            StringEquals:
              'kms:ViaService': !Sub 'dynamodb.${AWS::Region}.amazonaws.com'

ProjectDynamoDBKMSAlias:
  Type: AWS::KMS::Alias
  Properties:
    AliasName: !Sub 'alias/project-${uuid}-dynamodb'
    TargetKeyId: !Ref ProjectDynamoDBKMSKey
```

---

## 4. Network Isolation Strategy

### 4.1 VPC Gateway Endpoint Architecture

#### 4.1.1 Network Topology
```
┌─────────────────────────────────────────────────────────────┐
│                 project-{uuid}-vpc (10.0.0.0/16)           │
│                                                             │
│  ┌─────────────────────┐    ┌─────────────────────┐        │
│  │  Private Subnet 1a  │    │  Private Subnet 1b  │        │
│  │   10.0.10.0/24      │    │   10.0.20.0/24      │        │
│  │                     │    │                     │        │
│  │  ┌───────────────┐  │    │  ┌───────────────┐  │        │
│  │  │ Lambda Funcs  │  │    │  │ Lambda Funcs  │  │        │
│  │  │ project-{...} │  │    │  │ project-{...} │  │        │
│  │  └───────────────┘  │    │  └───────────────┘  │        │
│  └─────────────────────┘    └─────────────────────┘        │
│            │                           │                   │
│            └─────────────┬─────────────┘                   │
│                          │                                 │
│                 ┌─────────────────┐                        │
│                 │ VPC Gateway     │                        │
│                 │ Endpoint        │                        │
│                 │ (DynamoDB)      │                        │
│                 └─────────────────┘                        │
│                          │                                 │
└──────────────────────────┼─────────────────────────────────┘
                           │
                ┌──────────────────┐
                │  AWS DynamoDB    │
                │  Service         │
                │ (Private Backbone)│
                └──────────────────┘
```

### 4.2 VPC Gateway Endpoint Configuration
```yaml
DynamoDBVPCEndpoint:
  Type: AWS::EC2::VPCEndpoint
  Properties:
    VpcId: !Ref ProjectVPC
    ServiceName: !Sub 'com.amazonaws.${AWS::Region}.dynamodb'
    VpcEndpointType: Gateway
    RouteTableIds: 
      - !Ref PrivateRouteTable1
      - !Ref PrivateRouteTable2
    PolicyDocument:
      Version: '2012-10-17'
      Statement:
        - Effect: Allow
          Principal: '*'
          Action: 'dynamodb:*'
          Resource: '*'
          Condition:
            StringLike:
              'dynamodb:tablename': !Sub 'project-${uuid}-*'
        - Effect: Deny
          Principal: '*'
          Action: 'dynamodb:*'
          Resource: '*'
          Condition:
            StringNotEquals:
              'aws:PrincipalArn': 
                - !GetAtt ProjectLambdaRole.Arn
                - !GetAtt MLScientistRole.Arn
```

### 4.3 Security Group Configuration
```yaml
LambdaSecurityGroup:
  Type: AWS::EC2::SecurityGroup
  Properties:
    GroupName: !Sub 'project-${uuid}-lambda-sg'
    GroupDescription: 'Security group for project Lambda functions'
    VpcId: !Ref ProjectVPC
    SecurityGroupEgress:
      # Allow HTTPS to other project services
      - IpProtocol: tcp
        FromPort: 443
        ToPort: 443
        DestinationSecurityGroupId: !Ref OpenSearchSecurityGroup
        Description: 'HTTPS to OpenSearch'
      - IpProtocol: tcp
        FromPort: 8182
        ToPort: 8182
        DestinationSecurityGroupId: !Ref NeptuneSecurityGroup
        Description: 'Neptune Graph DB access'
      # Deny all other outbound traffic (no internet access)
```

---

## 5. Automated Governance Implementation

### 5.1 EventBridge-Driven Policy Attachment

#### 5.1.1 EventBridge Rule Configuration
```yaml
DynamoDBGovernanceRule:
  Type: AWS::Events::Rule
  Properties:
    Name: !Sub 'project-${uuid}-dynamodb-governance'
    Description: 'Trigger governance Lambda when DynamoDB tables are created'
    EventPattern:
      source: ['aws.dynamodb']
      detail-type: ['AWS API Call via CloudTrail']
      detail:
        eventSource: ['dynamodb.amazonaws.com']
        eventName: ['CreateTable']
        requestParameters:
          tableName:
            - prefix: !Sub 'project-${uuid}-'
        responseElements:
          tableDescription:
            tableStatus: ['CREATING']
    State: ENABLED
    Targets:
      - Arn: !GetAtt DynamoDBGovernanceLambda.Arn
        Id: 'DynamoDBPolicyAttachment'
        RetryPolicy:
          MaximumRetryAttempts: 3
          MaximumEventAge: 300
```

#### 5.1.2 Governance Lambda Function
```python
import json
import boto3
import logging
import time
from botocore.exceptions import ClientError
from datetime import datetime

# Configure logging
logger = logging.getLogger()
logger.setLevel(logging.INFO)

class DynamoDBGovernanceHandler:
    def __init__(self):
        self.dynamodb_client = boto3.client('dynamodb')
        self.cloudwatch = boto3.client('cloudwatch')
        self.sns = boto3.client('sns')
        
    def lambda_handler(self, event, context):
        """Main Lambda handler for DynamoDB governance"""
        try:
            logger.info(f"Processing governance event: {json.dumps(event)}")
            
            # Extract table details from CloudTrail event
            table_details = self.extract_table_details(event)
            
            if not table_details:
                logger.error("Could not extract table details from event")
                return {'statusCode': 400, 'error': 'Invalid event structure'}
            
            # Validate naming convention
            project_uuid = self.validate_naming_convention(table_details['name'])
            
            if not project_uuid:
                return self.handle_non_compliant_table(table_details)
            
            # Wait for table to be active before attaching policy
            if self.wait_for_table_active(table_details['name']):
                # Generate and attach VPC-restrictive resource policy
                policy = self.generate_vpc_policy(project_uuid)
                success = self.attach_resource_policy(table_details['name'], policy)
                
                if success:
                    self.log_compliance_success(table_details, project_uuid)
                    self.send_notification(table_details, 'SUCCESS')
                    return {'statusCode': 200, 'compliance': 'enforced'}
                else:
                    self.handle_policy_attachment_failure(table_details)
                    return {'statusCode': 500, 'error': 'Policy attachment failed'}
            else:
                logger.error(f"Table {table_details['name']} did not become active")
                return {'statusCode': 500, 'error': 'Table activation timeout'}
                
        except Exception as e:
            logger.error(f"Governance handler error: {str(e)}")
            self.send_alert(f"Governance failure: {str(e)}")
            return {'statusCode': 500, 'error': str(e)}
    
    def generate_vpc_policy(self, project_uuid):
        """Generate VPC-restricted resource policy"""
        account_id = boto3.client('sts').get_caller_identity()['Account']
        region = boto3.Session().region_name
        
        policy = {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Sid": "ProjectVPCOnlyAccess",
                    "Effect": "Allow",
                    "Principal": {
                        "AWS": [
                            f"arn:aws:iam::{account_id}:role/project-{project_uuid}-lambda-role",
                            f"arn:aws:iam::{account_id}:role/project-{project_uuid}-ml-scientist-role"
                        ]
                    },
                    "Action": "dynamodb:*",
                    "Resource": "*",
                    "Condition": {
                        "StringEquals": {
                            "aws:SourceVpc": f"vpc-project-{project_uuid}",
                            "aws:SourceVpce": f"vpce-project-{project_uuid}-dynamodb"
                        }
                    }
                },
                {
                    "Sid": "DenyNonVPCAccess",
                    "Effect": "Deny",
                    "Principal": "*",
                    "Action": "*",
                    "Resource": "*",
                    "Condition": {
                        "StringNotEquals": {
                            "aws:SourceVpce": f"vpce-project-{project_uuid}-dynamodb"
                        }
                    }
                }
            ]
        }
        
        return policy
    
    def attach_resource_policy(self, table_name, policy, max_retries=3):
        """Attach resource policy to table with retry logic"""
        for attempt in range(max_retries):
            try:
                response = self.dynamodb_client.put_resource_policy(
                    ResourceArn=f"arn:aws:dynamodb:{boto3.Session().region_name}:{boto3.client('sts').get_caller_identity()['Account']}:table/{table_name}",
                    Policy=json.dumps(policy)
                )
                
                logger.info(f"Successfully attached policy to table {table_name}")
                self.put_custom_metric('PolicyAttachmentSuccess', 1, table_name)
                return True
                
            except ClientError as e:
                error_code = e.response['Error']['Code']
                
                if error_code == 'ResourceNotFoundException' and attempt < max_retries - 1:
                    logger.warning(f"Table {table_name} not found, retrying in {2 ** attempt} seconds")
                    time.sleep(2 ** attempt)
                    continue
                else:
                    logger.error(f"Failed to attach policy to {table_name}: {str(e)}")
                    self.put_custom_metric('PolicyAttachmentFailure', 1, table_name)
                    
                    if attempt == max_retries - 1:
                        self.send_alert(f"Failed to attach policy to {table_name} after {max_retries} attempts: {str(e)}")
                        return False
        
        return False

# Lambda entry point
def lambda_handler(event, context):
    handler = DynamoDBGovernanceHandler()
    return handler.lambda_handler(event, context)
```

---

## 6. ML Scientist Operations

### 6.1 Table Creation Patterns

#### 6.1.1 Standard Table Creation Function
```python
import boto3
import json
import os
import uuid
from datetime import datetime
from botocore.exceptions import ClientError

class ProjectDynamoDBManager:
    def __init__(self):
        self.project_id = os.environ['PROJECT_ID']  # project-{uuid}
        self.dynamodb_client = boto3.client('dynamodb')
        self.dynamodb_resource = boto3.resource('dynamodb')
        self.cloudwatch = boto3.client('cloudwatch')
        
        # Extract project UUID for KMS alias
        self.project_uuid = self.project_id.split('-')[1]
    
    def create_table(self, table_suffix, key_schema, attribute_definitions, 
                    billing_mode='PAY_PER_REQUEST', gsi_configs=None, 
                    stream_enabled=True, backup_enabled=True):
        """
        Create a DynamoDB table following project naming convention
        with comprehensive security and monitoring configuration
        """
        table_name = f"{self.project_id}-{table_suffix}"
        
        # Validate table name length
        if len(table_name) > 255:
            raise ValueError(f"Table name too long: {len(table_name)} characters (max 255)")
        
        table_config = {
            'TableName': table_name,
            'KeySchema': key_schema,
            'AttributeDefinitions': attribute_definitions,
            'BillingMode': billing_mode,
            'SSESpecification': {
                'Enabled': True,
                'SSEType': 'KMS',
                'KMSMasterKeyId': f'alias/project-{self.project_uuid}-dynamodb'
            },
            'Tags': [
                {'Key': 'Project', 'Value': self.project_id},
                {'Key': 'Owner', 'Value': 'ml-team'},
                {'Key': 'Environment', 'Value': os.environ.get('ENVIRONMENT', 'dev')},
                {'Key': 'CostCenter', 'Value': os.environ.get('COST_CENTER', 'ml-platform')},
                {'Key': 'CreatedBy', 'Value': os.environ.get('USER_NAME', 'lambda')},
                {'Key': 'CreatedAt', 'Value': datetime.utcnow().isoformat()}
            ]
        }
        
        # Add DynamoDB Streams if enabled
        if stream_enabled:
            table_config['StreamSpecification'] = {
                'StreamEnabled': True,
                'StreamViewType': 'NEW_AND_OLD_IMAGES'
            }
        
        try:
            # Create table
            response = self.dynamodb_client.create_table(**table_config)
            table_arn = response['TableDescription']['TableArn']
            
            # Wait for table to become active
            waiter = self.dynamodb_client.get_waiter('table_exists')
            waiter.wait(
                TableName=table_name,
                WaiterConfig={'Delay': 5, 'MaxAttempts': 24}  # 2 minute timeout
            )
            
            # Enable point-in-time recovery if backup enabled
            if backup_enabled:
                self.enable_point_in_time_recovery(table_name)
            
            # Put custom metric for monitoring
            self.put_custom_metric('TableCreated', 1, table_name)
            
            logger.info(f"Successfully created table {table_name}")
            
            return {
                'statusCode': 200,
                'body': json.dumps({
                    'message': f'Table {table_name} created successfully',
                    'tableArn': table_arn,
                    'tableName': table_name
                })
            }
            
        except ClientError as e:
            error_code = e.response['Error']['Code']
            
            if error_code == 'ResourceInUseException':
                return {
                    'statusCode': 409,
                    'body': json.dumps({
                        'message': f'Table {table_name} already exists'
                    })
                }
            elif error_code == 'LimitExceededException':
                return {
                    'statusCode': 429,
                    'body': json.dumps({
                        'message': 'Table creation rate limit exceeded, please retry'
                    })
                }
            else:
                logger.error(f"Error creating table {table_name}: {str(e)}")
                self.put_custom_metric('TableCreationError', 1, table_name)
                raise e
```

### 6.2 Secure Data Operations

#### 6.2.1 Session Management with Security Controls
```python
import boto3
import json
import uuid
import hashlib
from datetime import datetime, timedelta
from boto3.dynamodb.conditions import Key, Attr
from botocore.exceptions import ClientError

class SecureSessionManager:
    def __init__(self):
        self.project_id = os.environ['PROJECT_ID']
        self.table_name = f"{self.project_id}-user-sessions"
        self.dynamodb = boto3.resource('dynamodb')
        self.table = self.dynamodb.Table(self.table_name)
        self.cloudwatch = boto3.client('cloudwatch')
        
        # Security configuration
        self.session_timeout_hours = 24
        self.max_sessions_per_user = 5
        
    def create_session(self, user_id, session_data, client_ip=None):
        """Create a new user session with security controls"""
        
        # Validate input
        if not user_id or not isinstance(session_data, dict):
            raise ValueError("Invalid user_id or session_data")
        
        session_id = str(uuid.uuid4())
        timestamp = int(datetime.utcnow().timestamp())
        ttl = int((datetime.utcnow() + timedelta(hours=self.session_timeout_hours)).timestamp())
        
        # Create session item with security metadata
        session_item = {
            'session_id': session_id,
            'timestamp': timestamp,
            'user_id': user_id,
            'session_data': session_data,
            'ttl': ttl,
            'created_at': datetime.utcnow().isoformat(),
            'project_id': self.project_id,  # Security boundary marker
            'client_ip_hash': self.hash_ip(client_ip) if client_ip else None,
            'is_active': True,
            'access_count': 0,
            'last_accessed': datetime.utcnow().isoformat()
        }
        
        try:
            # Create session with condition to prevent duplicates
            response = self.table.put_item(
                Item=session_item,
                ConditionExpression='attribute_not_exists(session_id)'
            )
            
            # Log successful session creation
            self.log_security_event('SESSION_CREATED', {
                'session_id': session_id,
                'user_id': user_id,
                'client_ip_hash': session_item.get('client_ip_hash')
            })
            
            # Custom metric for monitoring
            self.put_custom_metric('SessionCreated', 1)
            
            return {
                'session_id': session_id,
                'expires_at': datetime.fromtimestamp(ttl).isoformat(),
                'status': 'created'
            }
            
        except ClientError as e:
            if e.response['Error']['Code'] == 'ConditionalCheckFailedException':
                raise ValueError("Session ID collision - please retry")
            else:
                logger.error(f"Error creating session: {str(e)}")
                self.put_custom_metric('SessionCreationError', 1)
                raise e
```

---

## 7. Monitoring and Compliance

### 7.1 CloudWatch Monitoring Strategy

#### 7.1.1 Infrastructure Metrics Dashboard
```yaml
DynamoDBMonitoringDashboard:
  Type: AWS::CloudWatch::Dashboard
  Properties:
    DashboardName: !Sub 'project-${uuid}-dynamodb-monitoring'
    DashboardBody: !Sub |
      {
        "widgets": [
          {
            "type": "metric",
            "x": 0,
            "y": 0,
            "width": 12,
            "height": 6,
            "properties": {
              "metrics": [
                [ "AWS/DynamoDB", "ConsumedReadCapacityUnits", "TableName", "project-${uuid}-user-sessions" ],
                [ ".", "ConsumedWriteCapacityUnits", ".", "." ],
                [ ".", "UserErrors", ".", "." ],
                [ ".", "SystemErrors", ".", "." ]
              ],
              "view": "timeSeries",
              "stacked": false,
              "region": "${AWS::Region}",
              "title": "DynamoDB Capacity and Errors",
              "period": 300
            }
          }
        ]
      }
```

### 7.2 Compliance Monitoring

#### 7.2.1 AWS Config Rules
```yaml
DynamoDBNamingComplianceRule:
  Type: AWS::Config::ConfigRule
  Properties:
    ConfigRuleName: !Sub 'project-${uuid}-dynamodb-naming-compliance'
    Description: 'Checks if DynamoDB tables follow project naming convention'
    Source:
      Owner: AWS
      SourceIdentifier: REQUIRED_TAGS
    InputParameters: !Sub |
      {
        "tag1Key": "Project",
        "tag1Value": "project-${uuid}",
        "tag2Key": "Owner",
        "tag2Value": "ml-team"
      }
    Scope:
      ComplianceResourceTypes:
        - AWS::DynamoDB::Table
```

---

## 8. Operational Procedures

### 8.1 Table Lifecycle Management

#### 8.1.1 Table Creation Workflow
```
1. ML Scientist Request
   ├── Define table requirements
   ├── Choose appropriate key schema
   └── Configure billing mode and capacity

2. Automated Validation
   ├── Validate naming convention
   ├── Check IAM permissions
   └── Verify VPC connectivity

3. Table Creation
   ├── Create table with encryption
   ├── Apply required tags
   └── Enable streams and backup

4. Governance Automation
   ├── EventBridge triggers governance Lambda
   ├── Attach VPC-restrictive resource policy
   └── Log compliance success

5. Verification
   ├── Validate VPC-only access
   ├── Test CRUD operations
   └── Confirm monitoring setup
```

---

## 9. Troubleshooting Guide

### 9.1 Common Issues and Solutions

#### 9.1.1 VPC Access Issues

**Issue: Lambda cannot access DynamoDB**
```
Symptoms:
- TimeoutException when calling DynamoDB
- No connectivity errors in Lambda logs
- VPC Flow Logs show rejected traffic

Diagnosis:
1. Check Lambda is in correct VPC and subnets
2. Verify VPC Gateway Endpoint exists for DynamoDB
3. Confirm route table associations
4. Validate security group rules

Solution:
1. Ensure Lambda uses project-specific VPC and private subnets
2. Verify VPC Gateway Endpoint is properly configured
3. Check route tables include VPC endpoint routes
4. Update security groups to allow HTTPS egress
```

---

## 10. Implementation Timeline

### 10.1 Phase 1: Foundation (Weeks 1-2)

#### Week 1: Core Infrastructure
- [ ] VPC and networking setup
- [ ] VPC Gateway Endpoint configuration
- [ ] Security groups and NACLs
- [ ] IAM roles and policies creation
- [ ] KMS keys for encryption

#### Week 2: Governance Framework
- [ ] EventBridge rules configuration
- [ ] Governance Lambda development
- [ ] CloudTrail setup for DynamoDB events
- [ ] SNS topics for alerts
- [ ] Initial testing framework

### 10.2 Phase 2: Security Implementation (Weeks 3-4)

#### Week 3: Access Controls
- [ ] Resource-based policy templates
- [ ] VPC endpoint policies
- [ ] Lambda execution role policies
- [ ] ML Scientist role policies
- [ ] Cross-project isolation testing

### 10.3 Phase 3: Operational Excellence (Weeks 5-6)

#### Week 5: Monitoring and Alerting
- [ ] CloudWatch dashboards
- [ ] Performance metrics
- [ ] Cost monitoring setup
- [ ] Backup automation
- [ ] Disaster recovery procedures

### 10.4 Phase 4: Testing and Validation (Weeks 7-8)

#### Week 7: Comprehensive Testing
- [ ] End-to-end functionality testing
- [ ] Security boundary validation
- [ ] Performance testing
- [ ] Failure scenario testing
- [ ] Compliance verification

---

## 11. Appendices

### 11.1 Reference Architecture Diagrams

#### 11.1.1 Network Architecture Diagram
```
┌─────────────────────────────────────────────────────────────────┐
│                    AWS Account                                   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │              project-{uuid}-vpc (10.0.0.0/16)              │ │
│  │                                                             │ │
│  │  ┌─────────────────────┐    ┌─────────────────────┐        │ │
│  │  │  Private Subnet 1a  │    │  Private Subnet 1b  │        │ │
│  │  │   10.0.10.0/24      │    │   10.0.20.0/24      │        │ │
│  │  │                     │    │                     │        │ │
│  │  │  ┌───────────────┐  │    │  ┌───────────────┐  │        │ │
│  │  │  │Lambda project-│  │    │  │Lambda project-│  │        │ │
│  │  │  │{uuid}-*       │  │    │  │{uuid}-*       │  │        │ │
│  │  │  └───────────────┘  │    │  └───────────────┘  │        │ │
│  │  └─────────────────────┘    └─────────────────────┘        │ │
│  │            │                           │                   │ │
│  │            └─────────────┬─────────────┘                   │ │
│  │                          │                                 │ │
│  │                 ┌─────────────────┐                        │ │
│  │                 │ VPC Gateway     │                        │ │
│  │                 │ Endpoint        │                        │ │
│  │                 │ (DynamoDB)      │                        │ │
│  │                 └─────────────────┘                        │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                  Governance Layer                          │ │
│  │                                                             │ │
│  │  ┌─────────────┐  ┌──────────────┐  ┌─────────────────┐    │ │
│  │  │EventBridge  │→ │ Governance   │→ │ Resource Policy │    │ │
│  │  │Rule         │  │ Lambda       │  │ Attachment      │    │ │
│  │  └─────────────┘  └──────────────┘  └─────────────────┘    │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                  DynamoDB Tables                           │ │
│  │                                                             │ │
│  │     project-{uuid}-user-sessions                           │ │
│  │     project-{uuid}-conversation-history                    │ │
│  │     project-{uuid}-model-metadata                          │ │
│  │                                                             │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ▼
                ┌──────────────────┐
                │  AWS DynamoDB    │
                │  Service         │
                │ (Private Backbone)│
                └──────────────────┘
```

### 11.2 Policy Templates

#### 11.2.1 Lambda Execution Role Policy Template
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DynamoDBVPCOnlyAccess",
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
        "dynamodb:TransactGetItems",
        "dynamodb:TransactWriteItems",
        "dynamodb:DescribeTable"
      ],
      "Resource": [
        "arn:aws:dynamodb:*:*:table/project-${uuid}-*",
        "arn:aws:dynamodb:*:*:table/project-${uuid}-*/index/*",
        "arn:aws:dynamodb:*:*:table/project-${uuid}-*/stream/*"
      ],
      "Condition": {
        "StringEquals": {
          "aws:SourceVpce": "vpce-project-${uuid}-dynamodb",
          "aws:SourceVpc": "vpc-project-${uuid}"
        }
      }
    },
    {
      "Sid": "DenyNonVPCAccess",
      "Effect": "Deny",
      "Action": "dynamodb:*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:SourceVpce": "vpce-project-${uuid}-dynamodb"
        }
      }
    }
  ]
}
```

### 11.3 Frequently Asked Questions

#### 11.3.1 General Questions

**Q: Why can't I access DynamoDB from my local development environment?**
A: DynamoDB access is restricted to project VPC only. For local development, use AWS CLI with assumed ML Scientist role and VPN connection to project VPC.

**Q: How do I know if my table has the correct resource policy?**
A: Use the diagnostic Lambda function to check policy compliance, or review CloudWatch metrics for policy attachment success.

**Q: Can I create tables outside the project naming convention?**
A: No. All tables must follow the `project-{uuid}-` naming pattern. Non-compliant tables will be quarantined automatically.

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-07-26 | Engineering Team | Initial document creation |

---

## Approval

**Engineering Team Lead**: _________________ Date: _________

**Security Team Lead**: _________________ Date: _________

**ML Platform Lead**: _________________ Date: _________

---

**END OF DOCUMENT**