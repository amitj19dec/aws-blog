# AWS Glue Integration - Project Level Design

## Document Information
- **Document Version**: 1.0
- **Last Updated**: July 26, 2025
- **Owner**: Engineering Team
- **Integration With**: UHC Ask AI Platform Architecture

---

## 1. Overview

This document outlines the integration of AWS Glue into the project-level architecture following the established shared responsibility model. Glue enables ML scientists to build scalable ETL pipelines while maintaining strict project isolation and security boundaries.

### Design Principles
- **Project Isolation**: Complete separation using `project-{uuid}-` naming convention
- **Network Security**: All Glue jobs execute within project VPC private subnets
- **Shared Responsibility**: Engineering provisions infrastructure, ML teams implement business logic
- **Resource Governance**: Automated compliance and cost controls

---

## 2. Shared Responsibility Matrix

| **Component** | **Engineering Team** | **ML Scientist Team** |
|---------------|---------------------|----------------------|
| **IAM Roles** | Create execution roles with policies | Use provided roles for jobs/crawlers |
| **Data Catalog** | Create catalog database namespace | Define table schemas and metadata |
| **Network Config** | VPC, subnets, security groups, endpoints | None |
| **S3 Storage** | Provision buckets for scripts/temp/artifacts | Upload job scripts and manage data |
| **Job Execution** | Resource limits, timeout policies | Job logic, scheduling, monitoring |
| **Cost Control** | Budget limits, resource caps | Efficient script development |

---

## 3. Engineering Team Responsibilities

### 3.1 IAM Foundation
```
Roles:
├── project-{uuid}-glue-job-role        → ETL job execution
├── project-{uuid}-glue-crawler-role    → Data discovery
└── project-{uuid}-glue-dev-role        → Development endpoints

Policies:
├── project-{uuid}-glue-data-access     → S3, OpenSearch, Neptune access
├── project-{uuid}-glue-catalog-access  → Data Catalog permissions
└── project-{uuid}-glue-network-access  → VPC execution permissions
```

### 3.2 Infrastructure Resources
```
Data Catalog:
└── project-{uuid}-catalog-db           → Logical namespace for table metadata

S3 Buckets:
├── project-{uuid}-glue-scripts         → ETL job scripts storage
├── project-{uuid}-glue-temp            → Temporary processing files
└── project-{uuid}-glue-artifacts       → Job outputs and logs

Network:
├── project-{uuid}-glue-sg              → Security group for worker communication
├── VPC endpoints for Glue service APIs
└── Private subnet configuration
```

### 3.3 Resource Governance
```
Job Execution Limits:
├── Max Workers: 50 per job
├── Max Capacity: 100 DPU per project
├── Timeout: 2 hours default, 8 hours maximum
└── Auto-termination on idle

Cost Controls:
├── Spot instance preference for non-critical workloads
├── Budget alerts at 80% of monthly allocation
└── Resource utilization monitoring
```

---

## 4. Glue Job Execution Role Design

### 4.1 Core Permissions
```json
{
  "Sid": "GlueJobManagement",
  "Effect": "Allow",
  "Action": [
    "glue:GetJob",
    "glue:GetJobRun",
    "glue:UpdateJob",
    "glue:GetJobBookmark",
    "glue:ResetJobBookmark"
  ],
  "Resource": "arn:aws:glue:*:*:job/project-${uuid}-*"
}
```

### 4.2 Data Catalog Access
```json
{
  "Sid": "DataCatalogAccess",
  "Effect": "Allow",
  "Action": [
    "glue:GetDatabase",
    "glue:GetTable",
    "glue:GetTables",
    "glue:CreateTable",
    "glue:UpdateTable",
    "glue:GetPartitions",
    "glue:BatchCreatePartition"
  ],
  "Resource": [
    "arn:aws:glue:*:*:catalog",
    "arn:aws:glue:*:*:database/project-${uuid}-catalog-db",
    "arn:aws:glue:*:*:table/project-${uuid}-catalog-db/*"
  ]
}
```

### 4.3 Network and Monitoring
```json
{
  "Sid": "VPCAndMonitoring",
  "Effect": "Allow",
  "Action": [
    "ec2:CreateNetworkInterface",
    "ec2:DeleteNetworkInterface",
    "ec2:DescribeNetworkInterfaces",
    "logs:CreateLogGroup",
    "logs:CreateLogStream",
    "logs:PutLogEvents",
    "cloudwatch:PutMetricData"
  ],
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "ec2:subnet": ["subnet-project-${uuid}-private-*"]
    }
  }
}
```

---

## 5. Network Security Architecture

### 5.1 VPC Configuration
```
Execution Environment:
├── VPC: project-{uuid}-vpc
├── Subnets: project-{uuid}-private-subnet-* (Multi-AZ)
├── Security Group: project-{uuid}-glue-sg
└── No internet gateway access (private only)

VPC Endpoints Required:
├── Glue Service Endpoint     → Job management APIs
├── S3 Gateway Endpoint       → Data access
└── DynamoDB Gateway Endpoint → Glue bookmarking
```

### 5.2 Security Group Rules
```
project-{uuid}-glue-sg:
├── Inbound: None (jobs don't accept connections)
├── Outbound HTTPS (443): To S3, Glue service endpoints
├── Outbound Custom: OpenSearch (443), Neptune (8182)
└── Self-reference: Glue worker-to-worker communication
```

---

## 6. ML Scientist Capabilities

### 6.1 Self-Service Resources
```
Jobs & Workflows:
├── Create/update ETL jobs (naming: project-{uuid}-*)
├── Configure job parameters and scheduling
├── Build Glue workflows for complex pipelines
└── Manage job triggers and dependencies

Crawlers:
├── Create crawlers for data discovery
├── Configure crawler schedules and targets
└── Manage crawler exclusion patterns

Development:
├── Spin up dev endpoints for script development
├── Upload Python/Scala scripts to designated S3 bucket
└── Test jobs in development environment
```

### 6.2 Enforced Constraints
```
Mandatory Settings:
├── Execution Role: project-{uuid}-glue-job-role (locked)
├── VPC: project-{uuid}-vpc (locked)
├── Security Groups: project-{uuid}-glue-sg (locked)
└── Catalog Database: project-{uuid}-catalog-db (locked)

Resource Naming:
├── Jobs: project-{uuid}-job-*
├── Crawlers: project-{uuid}-crawler-*
├── Workflows: project-{uuid}-workflow-*
└── Triggers: project-{uuid}-trigger-*
```

---

## 7. Integration Patterns

### 7.1 Data Pipeline Flow
```
S3 Raw Data → Glue Crawler → Data Catalog → Glue ETL Job → Target Storage
                    ↓                               ↓
            project-{uuid}-catalog-db    OpenSearch/Neptune/S3
```

### 7.2 Service Integration Points
```
Trigger Sources:
├── S3 Event → Lambda → StartGlueJob
├── EventBridge Schedule → Direct job trigger
├── Step Functions → Include Glue job in ML workflow
└── Manual execution via Glue console/API

Output Destinations:
├── OpenSearch: Processed documents and vectors
├── Neptune: Graph relationships and entities
├── S3: Parquet/Delta Lake format for analytics
└── DynamoDB: Job metadata and processing state
```

---

## 8. Monitoring & Governance

### 8.1 Automated Monitoring
```
CloudWatch Metrics:
├── Job execution success/failure rates
├── Worker utilization and performance
├── Data processing throughput
└── Cost per job execution

Alerting:
├── Job failure notifications to ML team
├── Cost threshold breaches to engineering
├── Resource utilization anomalies
└── Security violations (Config rules)
```

### 8.2 Compliance Automation
```
AWS Config Rules:
├── Glue job VPC compliance validation
├── IAM role assignment verification
├── Resource naming convention checks
└── Security group configuration audits

Auto-Tagging:
├── Project: project-{uuid}
├── Service: glue
├── Owner: ml-team
└── CostCenter: auto-populated from project metadata
```

---

## 9. Cost Optimization Strategy

### 9.1 Resource Controls
```
Job Execution:
├── Spot instances for batch workloads (70% cost savings)
├── Auto-scaling based on input data size
├── Job bookmarking for incremental processing
└── Automatic cleanup of temporary files

Monitoring:
├── Per-job cost allocation tracking
├── Weekly cost optimization reports
├── Resource rightsizing recommendations
└── Idle resource identification and cleanup
```

### 9.2 Development Efficiency
```
Templates & Examples:
├── Standard ETL job templates for common patterns
├── Pre-configured dev endpoint images
├── Sample scripts for service integrations
└── Best practices documentation

Testing Framework:
├── Local Glue development libraries
├── Sample datasets for testing
├── CI/CD integration for job deployment
└── Automated testing of ETL logic
```

---

## 10. Operational Procedures

### 10.1 ML Scientist Workflow
```
Development:
1. Request dev endpoint from engineering team
2. Develop and test ETL scripts locally
3. Upload scripts to project-{uuid}-glue-scripts bucket
4. Create job definition with project-specific role
5. Test job execution with sample data
6. Deploy to production with appropriate scheduling

Monitoring:
1. Monitor job execution via CloudWatch dashboards
2. Review data quality metrics post-processing
3. Optimize job performance based on metrics
4. Report issues to engineering for infrastructure support
```

### 10.2 Engineering Support
```
Infrastructure:
├── Provision new project Glue infrastructure
├── Scale resource limits based on usage patterns
├── Troubleshoot VPC connectivity issues
└── Optimize managed service configurations

Governance:
├── Regular compliance audits and reporting
├── Cost optimization analysis and recommendations
├── Security policy updates and enforcement
└── Infrastructure updates and maintenance windows
```

---

## 11. Security Considerations

### 11.1 Data Protection
```
Encryption:
├── S3 server-side encryption with project KMS keys
├── Glue Data Catalog encryption at rest
├── TLS 1.2+ for all data in transit
└── Encrypted temporary storage during job execution

Access Control:
├── Project-level resource isolation via IAM
├── Network-level isolation via VPC boundaries
├── Service-level isolation via security groups
└── Data-level isolation via catalog database separation
```

### 11.2 Audit & Compliance
```
Logging:
├── CloudTrail API call logging for all Glue operations
├── VPC Flow Logs for network traffic analysis
├── Glue job execution logs for data processing audit
└── Data lineage tracking through catalog metadata

Compliance:
├── Regular penetration testing of Glue infrastructure
├── Data retention policy enforcement
├── PII/PHI handling compliance validation
└── Cross-project access prevention verification
```

---

## 12. Implementation Checklist

### Phase 1: Infrastructure Setup (Week 1)
- [ ] Create Glue IAM roles with project-scoped policies
- [ ] Provision S3 buckets for scripts, temp, and artifacts
- [ ] Configure VPC endpoints for Glue service access
- [ ] Setup security groups and network ACLs

### Phase 2: Data Catalog & Governance (Week 2)
- [ ] Create project-specific Glue Data Catalog database
- [ ] Configure AWS Config rules for Glue compliance
- [ ] Setup CloudWatch dashboards and alarms
- [ ] Implement auto-tagging automation

### Phase 3: Testing & Validation (Week 3)
- [ ] Create sample ETL job for testing
- [ ] Validate cross-service connectivity (S3, OpenSearch, Neptune)
- [ ] Test development endpoint functionality
- [ ] Verify monitoring and alerting systems

### Phase 4: ML Team Onboarding (Week 4)
- [ ] Conduct ML team training on Glue capabilities
- [ ] Provide documentation and code templates
- [ ] Setup development environments
- [ ] Begin production workload migration

---

## 13. Success Metrics

### Security Metrics
- **100%** Glue jobs executing within project VPC
- **Zero** cross-project data access violations
- **100%** resource naming convention compliance

### Performance Metrics
- **< 5 minutes** job startup time for small datasets
- **90%+** job success rate for production workloads
- **< 10%** resource waste through optimized configurations

### Cost Metrics
- **30%+** cost savings through spot instance usage
- **Monthly budget** adherence within 5% variance
- **Resource utilization** > 75% for allocated capacity

---

**Document Classification**: Internal Use  
**Review Cycle**: Quarterly  
**Next Review Date**: October 26, 2025  
**Dependencies**: VPC Design, IAM Architecture, S3 Storage Strategy