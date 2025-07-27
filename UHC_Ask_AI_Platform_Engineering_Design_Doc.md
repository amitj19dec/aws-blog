# UHC Ask AI Platform - Engineering Team Design Document

## Document Information
- **Document Version**: 1.0
- **Last Updated**: July 26, 2025
- **Owner**: Engineering Team
- **Stakeholders**: ML Engineering Team, Platform Architecture

---

## 1. Executive Summary

This document outlines the engineering team's responsibilities for provisioning and managing the infrastructure for the UHC Ask AI Platform. The design follows a shared responsibility model where engineering manages core infrastructure and security boundaries, while ML scientists handle business logic implementation within defined constraints.

### Key Principles
- **Security by Design**: Zero-trust architecture with strict resource isolation
- **Controlled Self-Service**: ML teams get autonomy within engineering-defined boundaries
- **Scalable Governance**: Standardized naming conventions and automated compliance
- **Cost Optimization**: Resource-based access controls and monitoring

---

## 2. Engineering Team Responsibilities

### 2.1 Core Infrastructure
- **Network Architecture**: VPC, subnets, security groups, VPC endpoints
- **Managed Services**: OpenSearch, Neptune, ElastiCache, S3, SageMaker
- **IAM Foundation**: Base roles, policies, and security boundaries
- **Monitoring Infrastructure**: CloudWatch, X-Ray, cost monitoring
- **CI/CD Pipeline**: Infrastructure deployment automation

### 2.2 Security & Compliance
- **Network Security**: Private subnet enforcement, security group management
- **Access Control**: IAM policy design and enforcement
- **Data Protection**: Encryption at rest and in transit
- **Audit & Compliance**: CloudTrail, resource tagging, governance automation

### 2.3 Operational Excellence
- **Infrastructure as Code**: CloudFormation/CDK templates
- **Resource Monitoring**: Performance metrics and alerting
- **Cost Management**: Budget controls and optimization
- **Disaster Recovery**: Backup strategies and recovery procedures

---

## 3. Resource Architecture & Naming Convention

### 3.1 Naming Standard
All resources follow the pattern: `project-{uuid}-{resource-type}`

**Examples:**
- VPC: `project-abc123-vpc`
- Lambda Role: `project-abc123-lambda-role`
- S3 Bucket: `project-abc123-documents`
- OpenSearch: `project-abc123-vectorstore`

### 3.2 Resource Inventory

#### Network Infrastructure
```
VPC: project-{uuid}-vpc
Private Subnets: project-{uuid}-private-subnet-1a, project-{uuid}-private-subnet-1b
Public Subnets: project-{uuid}-public-subnet-1a, project-{uuid}-public-subnet-1b
Security Groups:
  - project-{uuid}-lambda-sg
  - project-{uuid}-opensearch-sg
  - project-{uuid}-neptune-sg
  - project-{uuid}-elasticache-sg
```

#### Data & Storage Services
```
S3 Buckets:
  - project-{uuid}-documents
  - project-{uuid}-models
  - project-{uuid}-code-artifacts
  - project-{uuid}-logs
  
OpenSearch: project-{uuid}-vectorstore
Neptune: project-{uuid}-knowledge-graph
ElastiCache: project-{uuid}-redis
```

#### AI/ML Services
```
SageMaker:
  - Domain: project-{uuid}-sagemaker-domain
  - User Profiles: project-{uuid}-ml-user-profile
  
Bedrock Knowledge Base: project-{uuid}-kb
```

#### IAM Resources
```
Roles:
  - project-{uuid}-lambda-role
  - project-{uuid}-ml-scientist-role
  - project-{uuid}-sagemaker-execution-role
  
Policies:
  - project-{uuid}-lambda-policy
  - project-{uuid}-ml-scientist-policy
```

---

## 4. Infrastructure Provisioning Sequence

### Phase 1: Foundation (Dependencies: None)
1. **Create VPC** with CIDR block and DNS support
2. **Create subnets** across multiple AZs (private/public)
3. **Create Internet Gateway** and NAT Gateway
4. **Configure Route Tables** for public/private routing
5. **Create VPC Endpoints** for S3, DynamoDB, Bedrock

### Phase 2: Security & IAM (Dependencies: VPC)
6. **Create Security Groups** with least-privilege rules
7. **Create Lambda Execution Role** with baseline permissions
8. **Create ML Scientist Role** with controlled self-service permissions
9. **Create KMS Keys** for encryption

### Phase 3: Data Services (Dependencies: VPC, Security Groups)
10. **Provision S3 Buckets** with versioning and encryption
11. **Create ElastiCache Cluster** in private subnets
12. **Provision Neptune Cluster** with backup configuration
13. **Setup Parameter Store** hierarchy structure

### Phase 4: AI/ML Infrastructure (Dependencies: Data Services)
14. **Create OpenSearch Cluster** with security configuration
15. **Setup SageMaker Domain** and user profiles
16. **Create Bedrock Knowledge Base** (references S3 + OpenSearch)
17. **Configure API Gateway** base infrastructure

### Phase 5: Monitoring & Governance (Dependencies: All Services)
18. **Setup CloudWatch Dashboards** for infrastructure metrics
19. **Configure Cost Budgets** and alerts
20. **Deploy Compliance Automation** (Config rules, auto-tagging)
21. **Setup Backup Strategies** for stateful services

---

## 5. IAM Security Model

### 5.1 Lambda Execution Role Permissions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "VPCAccess",
      "Effect": "Allow",
      "Action": [
        "ec2:CreateNetworkInterface",
        "ec2:DescribeNetworkInterfaces",
        "ec2:DeleteNetworkInterface"
      ],
      "Resource": "*"
    },
    {
      "Sid": "OpenSearchAccess",
      "Effect": "Allow",
      "Action": ["es:*"],
      "Resource": "arn:aws:es:*:*:domain/project-${uuid}-*"
    },
    {
      "Sid": "S3Access",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::project-${uuid}-*/*"
    },
    {
      "Sid": "BedrockAccess",
      "Effect": "Allow",
      "Action": ["bedrock:*"],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "bedrock:knowledgeBaseId": "project-${uuid}-*"
        }
      }
    },
    {
      "Sid": "ParameterStoreAccess",
      "Effect": "Allow",
      "Action": ["ssm:GetParameter", "ssm:GetParameters", "ssm:PutParameter"],
      "Resource": "arn:aws:ssm:*:*:parameter/project-${uuid}/*"
    },
    {
      "Sid": "SecretsManagerAccess",
      "Effect": "Allow",
      "Action": ["secretsmanager:GetSecretValue"],
      "Resource": "arn:aws:secretsmanager:*:*:secret:project-${uuid}-*"
    }
  ]
}
```

### 5.2 ML Scientist Role Permissions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowLambdaManagement",
      "Effect": "Allow",
      "Action": [
        "lambda:CreateFunction",
        "lambda:UpdateFunctionCode",
        "lambda:UpdateFunctionConfiguration",
        "lambda:DeleteFunction",
        "lambda:GetFunction",
        "lambda:ListFunctions",
        "lambda:InvokeFunction"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "lambda:FunctionName": "project-${uuid}-*"
        },
        "StringEquals": {
          "lambda:VpcIds": "vpc-${project-vpc-id}",
          "lambda:Role": "arn:aws:iam::${account}:role/project-${uuid}-lambda-role"
        }
      }
    },
    {
      "Sid": "AllowStepFunctions",
      "Effect": "Allow",
      "Action": ["states:*"],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "states:StateMachineName": "project-${uuid}-*"
        }
      }
    },
    {
      "Sid": "AllowDynamoDB",
      "Effect": "Allow",
      "Action": ["dynamodb:*"],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "dynamodb:TableName": "project-${uuid}-*"
        }
      }
    },
    {
      "Sid": "DenyNonVPCLambda",
      "Effect": "Deny",
      "Action": "lambda:CreateFunction",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "lambda:VpcIds": "vpc-${project-vpc-id}"
        }
      }
    },
    {
      "Sid": "ReadOnlyInfrastructure",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "es:DescribeDomains",
        "neptune:DescribeDBClusters"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:ResourceTag/Project": "project-${uuid}"
        }
      }
    }
  ]
}
```

---

## 6. Network Security Architecture

### 6.1 VPC Design
- **CIDR Block**: 10.0.0.0/16
- **Multi-AZ**: Minimum 2 availability zones
- **Public Subnets**: 10.0.1.0/24, 10.0.2.0/24 (NAT Gateways only)
- **Private Subnets**: 10.0.10.0/24, 10.0.20.0/24 (All application resources)

### 6.2 Security Group Rules

#### Lambda Security Group
```
Outbound Rules:
- HTTPS (443) to OpenSearch Security Group
- MySQL/Aurora (3306) to Neptune Security Group  
- Redis (6379) to ElastiCache Security Group
- HTTPS (443) to 0.0.0.0/0 (VPC Endpoints)
```

#### OpenSearch Security Group
```
Inbound Rules:
- HTTPS (443) from Lambda Security Group
- HTTPS (443) from SageMaker Security Group
```

#### Neptune Security Group
```
Inbound Rules:
- Port 8182 from Lambda Security Group
- Port 8182 from SageMaker Security Group
```

### 6.3 VPC Endpoints
- **S3 Gateway Endpoint**: For private S3 access
- **DynamoDB Gateway Endpoint**: For private DynamoDB access
- **Bedrock Interface Endpoint**: For private Bedrock API access
- **SageMaker Interface Endpoint**: For private SageMaker access

---

## 7. Monitoring & Governance

### 7.1 CloudWatch Strategy
- **Infrastructure Metrics**: CPU, memory, network for all managed services
- **Custom Namespaces**: `project-{uuid}/infrastructure`
- **Log Groups**: `/aws/lambda/project-{uuid}-*`, `/aws/opensearch/project-{uuid}-*`
- **Retention Policy**: 30 days for application logs, 90 days for audit logs

### 7.2 Cost Management
- **Budget Alerts**: Monthly budget per project with 80%/100% thresholds
- **Cost Allocation Tags**: Project, Environment, CostCenter, Owner
- **Resource Optimization**: Regular review of underutilized resources

### 7.3 Compliance Automation
```json
{
  "ConfigRules": [
    {
      "RuleName": "lambda-vpc-compliance",
      "Source": "AWS_CONFIG_RULE",
      "Scope": {
        "ComplianceResourceTypes": ["AWS::Lambda::Function"]
      }
    },
    {
      "RuleName": "resource-naming-compliance", 
      "Source": "CUSTOM_LAMBDA",
      "Parameters": {
        "NamingPattern": "project-${uuid}-*"
      }
    }
  ]
}
```

### 7.4 Auto-Tagging Strategy
- **EventBridge Rule**: Trigger on resource creation
- **Lambda Function**: Auto-tag with project metadata
- **Required Tags**: Project, Owner, Environment, CostCenter

---

## 8. Disaster Recovery & Business Continuity

### 8.1 Backup Strategy
- **S3**: Cross-region replication for critical data
- **OpenSearch**: Daily snapshots to S3
- **Neptune**: Automated backups with 7-day retention
- **ElastiCache**: Redis persistence enabled

### 8.2 High Availability
- **Multi-AZ Deployment**: All managed services across 2+ AZs
- **Auto Scaling**: Configured for OpenSearch and ElastiCache
- **Health Checks**: CloudWatch alarms for all critical services

### 8.3 Recovery Procedures
- **RTO Target**: 4 hours for non-critical services, 1 hour for critical
- **RPO Target**: 24 hours for data loss tolerance
- **Runbooks**: Documented procedures for common failure scenarios

---

## 9. Shared Responsibility Matrix

| **Category** | **Engineering Team** | **ML Team** |
|--------------|---------------------|--------------|
| **Infrastructure** | VPC, Subnets, Security Groups, Managed Services | None |
| **IAM** | Base roles, policies, security boundaries | Parameter values, secrets content |
| **Application Logic** | None | Lambda functions, Step Functions, business logic |
| **Data Management** | S3 buckets, OpenSearch clusters, backup policies | Data processing, model training |
| **Monitoring** | Infrastructure metrics, cost monitoring | Application metrics, business KPIs |
| **Security** | Network security, encryption, compliance framework | Secure coding practices, secret management |
| **Cost Control** | Resource limits, budget alerts, optimization | Efficient resource usage |

---

## 10. Implementation Timeline

### Week 1-2: Foundation
- VPC and networking infrastructure
- Basic IAM roles and policies
- S3 buckets and KMS keys

### Week 3-4: Core Services  
- OpenSearch and Neptune clusters
- ElastiCache deployment
- SageMaker domain setup

### Week 5-6: Integration & Security
- Bedrock Knowledge Base configuration
- API Gateway base setup
- Security group finalization

### Week 7-8: Monitoring & Governance
- CloudWatch dashboards
- Cost monitoring setup
- Compliance automation deployment

### Week 9-10: Testing & Handover
- End-to-end testing
- ML team onboarding
- Documentation and training

---

## 11. Success Metrics

### Security Metrics
- **100%** resource compliance with naming conventions
- **Zero** cross-project resource access violations
- **100%** network traffic within private subnets

### Operational Metrics
- **< 1 hour** infrastructure provisioning time
- **99.9%** managed service availability
- **< 5 minutes** ML team resource creation time

### Cost Metrics
- **Monthly budget adherence** within 5% variance
- **Resource utilization** > 70% for compute resources
- **Cost per project** tracked and optimized

---

## 12. Appendices

### Appendix A: CloudFormation Templates
- VPC and Networking Stack
- IAM Roles and Policies Stack  
- Data Services Stack
- AI/ML Services Stack
- Monitoring Stack

### Appendix B: Runbooks
- Infrastructure Deployment Procedures
- Troubleshooting Guide
- Incident Response Procedures
- Cost Optimization Checklist

### Appendix C: ML Team Onboarding
- Access Request Process
- Development Environment Setup
- Best Practices Guide
- Sample Lambda Function Templates

---

**Document Classification**: Internal Use  
**Review Cycle**: Quarterly  
**Next Review Date**: October 26, 2025