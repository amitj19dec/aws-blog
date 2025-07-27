# Amazon Lex Integration - Project Level Design

## Document Information
- **Document Version**: 1.0
- **Last Updated**: July 27, 2025
- **Owner**: Engineering Team
- **Integration With**: UHC Ask AI Platform Architecture
- **Classification**: Internal Use
- **Review Cycle**: Quarterly

---

## Table of Contents
1. [Executive Summary](#1-executive-summary)
2. [Architecture Overview](#2-architecture-overview)
3. [Shared Responsibility Matrix](#3-shared-responsibility-matrix)
4. [Engineering Team Infrastructure](#4-engineering-team-infrastructure)
5. [Security and IAM Model](#5-security-and-iam-model)
6. [ML Scientist Capabilities](#6-ml-scientist-capabilities)
7. [Integration Patterns](#7-integration-patterns)
8. [Monitoring and Governance](#8-monitoring-and-governance)
9. [ML Scientist Provisioning Workflow](#9-ml-scientist-provisioning-workflow)
10. [Operational Procedures](#10-operational-procedures)
11. [Troubleshooting Guide](#11-troubleshooting-guide)
12. [Implementation Timeline](#12-implementation-timeline)
13. [Appendices](#13-appendices)

---

## 1. Executive Summary

### 1.1 Purpose
This document defines the integration of Amazon Lex V2 into the UHC Ask AI Platform's project-based isolation architecture. Lex serves as the conversational interface orchestrator, managing structured dialog flows while seamlessly integrating with the platform's RAG-enabled LLM pipeline.

### 1.2 Key Design Principles
- **Conversational Orchestration**: Lex manages dialog flow, slot collection, and intent routing
- **RAG Integration**: Seamless handoff to project RAG pipeline for complex reasoning
- **Universal Lambda Role**: All fulfillment functions use `project-{uuid}-lambda-role`
- **VPC-First Security**: Complete network isolation within project boundaries
- **Project Isolation**: Zero cross-project conversation access or data leakage

### 1.3 Strategic Positioning
Lex functions as the **"Smart Router"** in the Ask AI platform:
- **Structured Conversations**: Forms, transactions, data collection workflows
- **Intent Classification**: Route to appropriate backend services (RAG, APIs, human agents)
- **Session Management**: Maintain context across multi-turn conversations
- **Channel Orchestration**: Unified experience across web, voice, mobile platforms

### 1.4 Success Metrics
- **100%** bot resources following project naming conventions
- **< 2 seconds** intent recognition and routing time
- **90%+** conversation completion rate for structured workflows
- **Zero** cross-project conversation data access incidents

---

## 2. Architecture Overview

### 2.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Project-{uuid} Boundary                      │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │              Conversation Layer                             │ │
│  │                                                             │ │
│  │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    │ │
│  │  │   Lex Bot   │    │   Lex Bot   │    │   Lex Bot   │    │ │
│  │  │ (Customer   │    │ (Claims)    │    │ (Benefits)  │    │ │
│  │  │  Service)   │    │             │    │             │    │ │
│  │  └─────────────┘    └─────────────┘    └─────────────┘    │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                   │                               │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │              Orchestration Layer                           │ │
│  │                                                             │ │
│  │         ┌─────────────────────────────────────┐             │ │
│  │         │      Lambda Fulfillment            │             │ │
│  │         │   (Universal Execution Role)       │             │ │
│  │         └─────────────────────────────────────┘             │ │
│  │                           │                                 │ │
│  │    ┌──────────────┬───────┼───────┬──────────────┐         │ │
│  │    │              │       │       │              │         │ │
│  │    ▼              ▼       ▼       ▼              ▼         │ │
│  │ ┌─────────┐ ┌──────────┐ │ ┌──────────┐ ┌─────────────┐   │ │
│  │ │   RAG   │ │DynamoDB  │ │ │OpenSearch│ │   Direct    │   │ │
│  │ │Pipeline │ │ Tables   │ │ │  Index   │ │   APIs      │   │ │
│  │ └─────────┘ └──────────┘ │ └──────────┘ └─────────────┘   │ │
│  │                         │                                 │ │
│  │                         ▼                                 │ │
│  │                 ┌──────────────┐                          │ │
│  │                 │   Bedrock    │                          │ │
│  │                 │Knowledge Base│                          │ │
│  │                 └──────────────┘                          │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Component Relationships

#### 2.2.1 Lex as Conversation Controller
- **Intent Recognition**: Classify user requests into actionable intents
- **Slot Management**: Collect required information through natural dialog
- **Fulfillment Routing**: Direct intents to appropriate processing logic
- **Session Persistence**: Maintain conversation context and user state

#### 2.2.2 Lambda as Universal Orchestrator
- **Single Execution Role**: `project-{uuid}-lambda-role` accesses all project resources
- **Business Logic Hub**: Implements intent-specific processing workflows
- **Service Integration**: Connects Lex to RAG pipeline, databases, and external APIs
- **Response Formatting**: Converts backend responses to Lex-compatible formats

#### 2.2.3 RAG Pipeline Integration
- **Complex Query Routing**: Send unstructured questions to RAG system
- **Knowledge Base Access**: Query project Bedrock Knowledge Base
- **Vector Search**: Leverage project OpenSearch for semantic retrieval
- **LLM Processing**: Generate contextual responses using project AI models

---

## 3. Shared Responsibility Matrix

| **Component** | **Engineering Team** | **ML Scientist Team** |
|---------------|---------------------|----------------------|
| **Bot Infrastructure** | Provision IAM roles, VPC config, service limits | Create bots, configure intents, manage aliases |
| **Fulfillment Functions** | Provide universal Lambda execution role | Implement business logic using provided role |
| **Network Security** | VPC, subnets, security groups, endpoints | None |
| **RAG Integration** | Provision OpenSearch, Bedrock KB, access policies | Design conversation flows, implement query logic |
| **Monitoring** | Infrastructure metrics, cost tracking, compliance | Application metrics, conversation analytics |
| **Intent Design** | Provide templates and best practices | Create domain-specific intents and utterances |
| **Channel Integration** | Platform connectors, API configurations | Deploy bots to specific channels |
| **Data Management** | Encryption, backup, retention policies | Conversation data modeling, user session management |

---

## 4. Engineering Team Infrastructure

### 4.1 IAM Foundation

#### 4.1.1 Bot Management Role
```json
{
  "RoleName": "project-{uuid}-ml-scientist-role",
  "AssumeRolePolicyDocument": {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {"AWS": "arn:aws:iam::${account}:user/ml-scientist"},
        "Action": "sts:AssumeRole"
      }
    ]
  },
  "ManagedPolicies": ["AmazonLexBotManagement"],
  "InlinePolicies": {
    "LexProjectScope": {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Sid": "LexBotManagement",
          "Effect": "Allow",
          "Action": [
            "lex:CreateBot",
            "lex:UpdateBot",
            "lex:DeleteBot",
            "lex:DescribeBot",
            "lex:ListBots",
            "lex:CreateBotVersion",
            "lex:CreateBotAlias",
            "lex:UpdateBotAlias",
            "lex:DeleteBotAlias",
            "lex:TagResource",
            "lex:UntagResource"
          ],
          "Resource": "*",
          "Condition": {
            "StringLike": {
              "lex:bot-name": "project-${uuid}-*"
            }
          }
        },
        {
          "Sid": "IntentAndSlotManagement",
          "Effect": "Allow",
          "Action": [
            "lex:CreateIntent",
            "lex:UpdateIntent",
            "lex:DeleteIntent",
            "lex:CreateSlotType",
            "lex:UpdateSlotType",
            "lex:DeleteSlotType",
            "lex:BuildBotLocale",
            "lex:DescribeBotLocale"
          ],
          "Resource": "arn:aws:lex:*:*:bot/project-${uuid}-*/*"
        },
        {
          "Sid": "DenyNonProjectBots",
          "Effect": "Deny",
          "Action": "lex:*",
          "Resource": "*",
          "Condition": {
            "StringNotLike": {
              "lex:bot-name": "project-${uuid}-*"
            }
          }
        }
      ]
    }
  }
}
```

#### 4.1.2 Universal Lambda Execution Role (Extended for Lex)
```json
{
  "RoleName": "project-{uuid}-lambda-role",
  "ExistingPermissions": "All project resource access",
  "AdditionalLexPermissions": {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "LexRuntimeAPI",
        "Effect": "Allow",
        "Action": [
          "lex:PostContent",
          "lex:PostText", 
          "lex:GetSession",
          "lex:PutSession",
          "lex:DeleteSession"
        ],
        "Resource": "arn:aws:lex:*:*:bot-alias/project-${uuid}-*/*"
      },
      {
        "Sid": "LexFulfillmentLogging", 
        "Effect": "Allow",
        "Action": [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ],
        "Resource": "arn:aws:logs:*:*:log-group:/aws/lex/project-${uuid}-*"
      }
    ]
  }
}
```

### 4.2 Network Infrastructure

#### 4.2.1 VPC Configuration
```yaml
# Engineering provisions these network components
Network Setup:
├── VPC: project-{uuid}-vpc (existing)
├── Private Subnets: project-{uuid}-private-subnet-* (existing)  
├── Security Groups: project-{uuid}-lambda-sg (existing)
└── VPC Endpoints: Lex service endpoints (if available in region)

Lex Integration:
├── All Lambda fulfillment functions execute in private subnets
├── No direct internet access for Lex fulfillment
├── Communication to Lex service via VPC endpoints or NAT Gateway
└── Logs routed through VPC Log endpoints
```

#### 4.2.2 Security Group Rules
```yaml
project-{uuid}-lambda-sg (existing, no changes needed):
├── Outbound HTTPS (443): To OpenSearch, S3, DynamoDB endpoints
├── Outbound Custom: Neptune (8182), ElastiCache (6379)
├── Outbound HTTPS (443): To Lex service endpoints  
└── Inbound: None (Lambda functions don't accept connections)
```

### 4.3 Resource Provisioning

#### 4.3.1 Lex Service Limits and Quotas
```yaml
Service Quotas (Engineering sets):
├── Maximum bots per project: 10
├── Maximum intents per bot: 100
├── Maximum slot types per bot: 50
├── Session timeout: 5-30 minutes (configurable)
└── Build timeout: 10 minutes maximum

Cost Controls:
├── Budget alerts at 80% of monthly Lex allocation
├── Request rate monitoring for cost optimization
├── Channel-specific cost tracking
└── Automated cleanup of unused bot versions
```

#### 4.3.2 S3 Integration Buckets (Existing)
```yaml
Existing S3 Buckets (used by Lex):
├── project-{uuid}-documents     → Intent training data, bot exports
├── project-{uuid}-code-artifacts → Lambda fulfillment function code
├── project-{uuid}-logs          → Conversation logs and analytics
└── project-{uuid}-models        → Custom NLU model artifacts
```

---

## 5. Security and IAM Model

### 5.1 Multi-Layer Security Architecture

#### 5.1.1 Layer 1: Identity-Based Access Control
```json
{
  "Principle": "Least Privilege with Project Scope",
  "Implementation": {
    "BotManagement": "ML Scientists can only create/manage project-{uuid}-* bots",
    "Fulfillment": "Lambda functions use universal project role only",
    "CrossProjectDenial": "Explicit deny for non-project resources"
  }
}
```

#### 5.1.2 Layer 2: Network-Based Isolation
```yaml
Network Controls:
├── VPC Boundary: All Lex fulfillment within project VPC
├── Private Subnets: No direct internet access
├── Security Groups: Controlled service-to-service communication
└── VPC Endpoints: Private AWS service connectivity
```

#### 5.1.3 Layer 3: Resource-Based Policies
```json
{
  "BotResourcePolicy": {
    "Version": "2012-10-17", 
    "Statement": [
      {
        "Sid": "ProjectOnlyAccess",
        "Effect": "Allow",
        "Principal": {
          "AWS": [
            "arn:aws:iam::${account}:role/project-${uuid}-lambda-role",
            "arn:aws:iam::${account}:role/project-${uuid}-ml-scientist-role"
          ]
        },
        "Action": "lex:*",
        "Resource": "*",
        "Condition": {
          "StringEquals": {
            "aws:SourceAccount": "${account}"
          }
        }
      }
    ]
  }
}
```

### 5.2 Data Protection Strategy

#### 5.2.1 Conversation Data Security
```yaml
Encryption:
├── In Transit: TLS 1.2+ for all Lex API communications
├── At Rest: CloudWatch Logs encrypted with project KMS key
├── Session Data: Stored in project DynamoDB with encryption
└── Audio Data: Encrypted storage if voice channels used

Access Controls:
├── Conversation Logs: Accessible only to project Lambda functions
├── Session Data: Scoped to project DynamoDB tables
├── User PII: Managed through project-specific data handling
└── Audit Trail: All Lex operations logged via CloudTrail
```

---

## 6. ML Scientist Capabilities

### 6.1 Self-Service Bot Management

#### 6.1.1 Bot Lifecycle Operations
```yaml
Allowed Operations:
├── Create bots with project-{uuid}-* naming
├── Configure intents, slots, and utterances
├── Build and test bots in development environment
├── Create versions and aliases for deployment
├── Configure fulfillment with project Lambda functions
├── Export/import bot definitions for backup
└── Delete unused bots and versions

Enforced Constraints:
├── Bot names must follow project-{uuid}-* pattern
├── Fulfillment must use project-{uuid}-lambda-role only
├── Cannot access or modify other project bots
├── Cannot create custom IAM roles or policies
└── Must deploy within project VPC boundaries
```

#### 6.1.2 Intent and Conversation Design
```python
# ML Scientist creates intents following these patterns

class IntentPatterns:
    TRANSACTIONAL = {
        "name": "CheckClaimStatus",
        "sample_utterances": [
            "What's the status of my claim",
            "Check my claim from last month", 
            "Has my claim been processed"
        ],
        "required_slots": ["member_id"],
        "optional_slots": ["claim_number", "date_range"],
        "fulfillment_type": "direct_api"
    }
    
    INFORMATIONAL = {
        "name": "ExplainBenefits",
        "sample_utterances": [
            "What are my dental benefits",
            "Explain my coverage for surgery",
            "What's covered under my plan"
        ],
        "required_slots": [],
        "optional_slots": ["benefit_type", "procedure"],
        "fulfillment_type": "rag_pipeline"
    }
    
    CONVERSATIONAL = {
        "name": "GeneralInquiry", 
        "sample_utterances": [
            "I have a question about my policy",
            "Can you help me understand something",
            "I need information about healthcare"
        ],
        "required_slots": [],
        "optional_slots": ["topic"],
        "fulfillment_type": "llm_handoff"
    }
```

### 6.2 Fulfillment Function Development

#### 6.2.1 Universal Lambda Role Usage
```python
import os
import boto3
import json
import logging
from datetime import datetime

class LexFulfillmentHandler:
    def __init__(self):
        # Access project resources using universal Lambda role
        self.project_id = os.environ['PROJECT_ID']  # project-{uuid}
        
        # All project resources accessible via universal role
        self.dynamodb = boto3.resource('dynamodb')
        self.opensearch_endpoint = os.environ['OPENSEARCH_ENDPOINT']
        self.bedrock_kb_id = os.environ['BEDROCK_KB_ID']
        self.neptune_endpoint = os.environ['NEPTUNE_ENDPOINT']
        
        # Initialize project-scoped clients
        self.session_table = self.dynamodb.Table(f"{self.project_id}-user-sessions")
        self.conversation_table = self.dynamodb.Table(f"{self.project_id}-conversations")
        
    def lambda_handler(self, event, context):
        """Universal fulfillment handler for all project Lex bots"""
        try:
            intent_name = event['sessionState']['intent']['name']
            session_id = event['sessionId']
            
            # Store conversation event
            self.log_conversation_event(event)
            
            # Route based on intent type
            if intent_name in ['CheckClaimStatus', 'UpdateAddress', 'PayBill']:
                return self.handle_transactional_intent(event)
            elif intent_name in ['ExplainBenefits', 'FindProvider', 'PolicyInfo']:
                return self.handle_informational_intent(event)
            else:
                return self.handle_conversational_intent(event)
                
        except Exception as e:
            logger.error(f"Fulfillment error: {str(e)}")
            return self.create_error_response(str(e))
```

### 6.3 Integration with Existing RAG Pipeline

#### 6.3.1 Seamless RAG Handoff
```python
def handle_informational_intent(self, event):
    """Route complex queries to existing RAG pipeline"""
    user_query = event['inputTranscript']
    intent_name = event['sessionState']['intent']['name']
    
    # Extract any collected slots for context
    slots = event['sessionState']['intent']['slots']
    context = self.build_query_context(slots)
    
    # Query existing RAG infrastructure
    rag_response = self.query_rag_pipeline(user_query, context)
    
    # Format response for Lex
    return self.create_close_response(rag_response)

def query_rag_pipeline(self, query, context=None):
    """Integration with existing project RAG system"""
    try:
        # Option 1: Use Bedrock Knowledge Base (managed RAG)
        if self.bedrock_kb_id:
            return self.query_bedrock_kb(query, context)
        
        # Option 2: Use custom OpenSearch + LLM pipeline
        else:
            return self.query_custom_rag(query, context)
            
    except Exception as e:
        logger.error(f"RAG pipeline error: {str(e)}")
        return "I'm having trouble accessing that information right now. " \
               "Let me connect you with a representative who can help."

def query_bedrock_kb(self, query, context):
    """Query project Bedrock Knowledge Base"""
    bedrock_client = boto3.client('bedrock-agent-runtime')
    
    # Add context to query if available
    enhanced_query = f"{query}"
    if context:
        enhanced_query = f"Given the context: {context}, {query}"
    
    response = bedrock_client.retrieve_and_generate(
        input={'text': enhanced_query},
        retrieveAndGenerateConfiguration={
            'type': 'KNOWLEDGE_BASE',
            'knowledgeBaseConfiguration': {
                'knowledgeBaseId': self.bedrock_kb_id,
                'modelArn': 'arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-v2'
            }
        }
    )
    
    return response['output']['text']
```

---

## 7. Integration Patterns

### 7.1 Conversation Flow Patterns

#### 7.1.1 Structured Transaction Flow
```
User: "I want to check my claim status"
   ↓
Lex: Intent = CheckClaimStatus
   ↓
Lex: "I'll help you check your claim. What's your member ID?"
   ↓ 
User: "12345678"
   ↓
Lambda: Query project DynamoDB claims table
   ↓
Lambda: Return formatted claim information
   ↓
Lex: "Your claim #ABC123 was submitted on 1/15 and is currently being processed..."
```

#### 7.1.2 Information Retrieval Flow
```
User: "What are my dental benefits for orthodontics?"
   ↓
Lex: Intent = ExplainBenefits, Slots = {benefit_type: "dental", procedure: "orthodontics"}
   ↓
Lambda: Query RAG pipeline with context
   ↓
RAG: Search project knowledge base + generate response
   ↓
Lambda: Format response for conversational delivery
   ↓
Lex: "Your dental plan covers orthodontics at 50% after deductible..."
```

#### 7.1.3 Hybrid Conversation Flow
```
User: "I have a complex question about my policy coverage"
   ↓
Lex: Intent = ComplexInquiry
   ↓
Lambda: Assess complexity and route appropriately
   ↓
Decision Tree:
├── Simple → Direct database lookup
├── Complex → RAG pipeline with full context
├── Ambiguous → Clarifying questions via Lex
└── Escalation needed → Human agent handoff
```

### 7.2 Channel Integration Patterns

#### 7.2.1 Multi-Channel Deployment
```yaml
Channel Configurations:
├── Web Chat Widget:
│   └── Embedded in project customer portal
├── Voice Interface:
│   └── Integration with Amazon Connect for phone support
├── Mobile App:
│   └── Native iOS/Android SDK integration
└── SMS/Text:
    └── Two-way messaging through project messaging service
```

#### 7.2.2 Channel-Specific Adaptations
```python
def format_response_for_channel(self, response_text, channel):
    """Adapt responses for different channels"""
    if channel == 'voice':
        # Shorter, more conversational for voice
        return self.create_voice_response(response_text)
    elif channel == 'web':
        # Rich formatting with buttons and links
        return self.create_web_response(response_text)
    elif channel == 'sms':
        # Concise text within character limits
        return self.create_sms_response(response_text)
    else:
        return self.create_default_response(response_text)
```

---

## 8. Monitoring and Governance

### 8.1 Automated Monitoring Framework

#### 8.1.1 CloudWatch Metrics and Dashboards
```yaml
Lex Monitoring Dashboard: project-{uuid}-lex-monitoring
├── Intent Recognition Metrics:
│   ├── Intent recognition accuracy by intent
│   ├── Confidence score distributions 
│   ├── Slot filling success rates
│   └── Conversation completion rates
├── Performance Metrics:
│   ├── Response latency (Lex + Lambda)
│   ├── Fulfillment function duration
│   ├── Error rates by intent type
│   └── Session duration and turnover
├── Business Metrics:
│   ├── Conversations per day/hour
│   ├── Intent distribution analysis
│   ├── Escalation rates to human agents
│   └── User satisfaction scores (if collected)
└── Cost Metrics:
    ├── Requests per dollar spent
    ├── Channel-specific cost analysis
    ├── Function execution costs
    └── Monthly budget utilization
```

#### 8.1.2 Custom Metrics Implementation
```python
def put_custom_metrics(self, event, response_type, processing_time):
    """Track custom business metrics"""
    cloudwatch = boto3.client('cloudwatch')
    
    intent_name = event['sessionState']['intent']['name']
    
    metrics = [
        {
            'MetricName': 'ConversationProcessed',
            'Value': 1,
            'Dimensions': [
                {'Name': 'Project', 'Value': self.project_id},
                {'Name': 'Intent', 'Value': intent_name},
                {'Name': 'ResponseType', 'Value': response_type}
            ]
        },
        {
            'MetricName': 'ProcessingLatency',
            'Value': processing_time,
            'Dimensions': [
                {'Name': 'Project', 'Value': self.project_id},
                {'Name': 'Intent', 'Value': intent_name}
            ]
        }
    ]
    
    cloudwatch.put_metric_data(
        Namespace=f"{self.project_id}/lex",
        MetricData=metrics
    )
```

### 8.2 Governance and Compliance

#### 8.2.1 Automated Compliance Checking
```yaml
AWS Config Rules:
├── lex-bot-naming-compliance:
│   └── Validates all bots follow project-{uuid}-* pattern
├── lex-fulfillment-role-compliance: 
│   └── Ensures all fulfillment uses project-{uuid}-lambda-role
├── lex-vpc-compliance:
│   └── Verifies Lambda fulfillment executes in project VPC
└── lex-resource-tagging-compliance:
    └── Validates required tags on all Lex resources

EventBridge Rules:
├── Lex bot creation/modification events
├── Trigger compliance validation Lambda
├── Alert on non-compliant configurations
└── Automated remediation where possible
```

#### 8.2.2 Cost Governance
```python
# Automated cost monitoring and alerting
class LexCostGovernance:
    def __init__(self, project_id):
        self.project_id = project_id
        self.cost_client = boto3.client('ce')  # Cost Explorer
        self.budgets_client = boto3.client('budgets')
        
    def monitor_lex_costs(self):
        """Monitor Lex costs and trigger alerts"""
        # Get current month Lex costs for project
        current_costs = self.get_lex_costs_current_month()
        
        # Check against budget thresholds
        if current_costs > self.get_budget_threshold(0.8):
            self.send_cost_alert('WARNING', current_costs)
        
        if current_costs > self.get_budget_threshold(1.0):
            self.send_cost_alert('CRITICAL', current_costs)
            # Optionally disable non-essential bots
            
    def optimize_conversation_costs(self):
        """Analyze conversation patterns for cost optimization"""
        # Identify high-cost, low-value conversation patterns
        # Suggest intent consolidation opportunities
        # Recommend direct API vs Lex routing
        pass
```

---

## 9. ML Scientist Provisioning Workflow

### 9.1 Step-by-Step Bot Creation Process

#### 9.1.1 Prerequisites Verification
```bash
# ML Scientist verification checklist
echo "Verifying project access..."

# 1. Confirm IAM role access
aws sts get-caller-identity --profile ml-scientist

# 2. Verify project resources exist
aws s3 ls s3://project-${PROJECT_UUID}-documents --profile ml-scientist

# 3. Test Lambda execution role
aws lambda list-functions --query "Functions[?starts_with(FunctionName, 'project-${PROJECT_UUID}')]" --profile ml-scientist

# 4. Confirm VPC connectivity 
aws ec2 describe-vpcs --filters "Name=tag:Name,Values=project-${PROJECT_UUID}-vpc" --profile ml-scientist
```

#### 9.1.2 Bot Creation Workflow
```python
# Step 1: Create Bot with Project Naming
bot_config = {
    "botName": f"project-{PROJECT_UUID}-customer-service-bot",
    "description": "Customer service bot for project {PROJECT_UUID}",
    "roleArn": f"arn:aws:iam::{ACCOUNT_ID}:role/project-{PROJECT_UUID}-ml-scientist-role",
    "dataPrivacy": {"childDirected": False},
    "idleSessionTTLInSeconds": 300,
    "botTags": {
        "Project": f"project-{PROJECT_UUID}",
        "Owner": "ml-team", 
        "Environment": "dev",
        "CostCenter": "ml-platform"
    }
}

# Step 2: Add Language Configuration
language_config = {
    "localeId": "en_US",
    "description": "English language configuration",
    "nluIntentConfidenceThreshold": 0.40
}

# Step 3: Create Core Intents
intents = [
    {
        "intentName": "CheckClaimStatus",
        "sampleUtterances": [
            "What's the status of my claim",
            "Check my claim from last month",
            "Has my claim been processed",
            "I want to know about my claim",
            "Show me my claim status"
        ],
        "slots": [
            {
                "slotName": "memberId",
                "slotTypeId": "AMAZON.AlphaNumeric", 
                "slotConstraint": "Required",
                "promptSpecification": {
                    "messageGroups": [{
                        "message": {
                            "plainTextMessage": {
                                "value": "I'll help you check your claim status. What's your member ID?"
                            }
                        }
                    }]
                }
            }
        ],
        "fulfillmentCodeHook": {
            "enabled": True
        }
    }
]
```

#### 9.1.3 Lambda Fulfillment Setup
```python
# Step 4: Create fulfillment function using universal role
lambda_config = {
    "FunctionName": f"project-{PROJECT_UUID}-lex-fulfillment",
    "Runtime": "python3.11",
    "Role": f"arn:aws:iam::{ACCOUNT_ID}:role/project-{PROJECT_UUID}-lambda-role",  # Universal role
    "Handler": "lambda_function.lambda_handler",
    "Code": {"ZipFile": fulfillment_code},
    "Environment": {
        "Variables": {
            "PROJECT_ID": f"project-{PROJECT_UUID}",
            "OPENSEARCH_ENDPOINT": f"project-{PROJECT_UUID}-vectorstore.{REGION}.es.amazonaws.com",
            "BEDROCK_KB_ID": f"project-{PROJECT_UUID}-kb",
            "DYNAMODB_SESSION_TABLE": f"project-{PROJECT_UUID}-user-sessions"
        }
    },
    "VpcConfig": {
        "SubnetIds": [f"subnet-project-{PROJECT_UUID}-private-1a", f"subnet-project-{PROJECT_UUID}-private-1b"],
        "SecurityGroupIds": [f"sg-project-{PROJECT_UUID}-lambda"]
    },
    "TracingConfig": {"Mode": "Active"},
    "Tags": {
        "Project": f"project-{PROJECT_UUID}",
        "Service": "lex-fulfillment",
        "Owner": "ml-team"
    }
}
```

### 9.2 Testing and Validation Procedures

#### 9.2.1 Local Development Testing
```python
# Test fulfillment function locally before deployment
def test_fulfillment_locally():
    """Test Lex fulfillment logic before deployment"""
    
    # Mock Lex event structure
    test_event = {
        "sessionId": "test-session-123",
        "inputTranscript": "What's the status of my claim",
        "sessionState": {
            "intent": {
                "name": "CheckClaimStatus",
                "slots": {
                    "memberId": {
                        "value": {
                            "interpretedValue": "12345678"
                        }
                    }
                }
            }
        }
    }
    
    # Test local logic
    handler = LexFulfillmentHandler()
    response = handler.lambda_handler(test_event, None)
    
    assert response['sessionState']['dialogAction']['type'] in ['Close', 'ElicitSlot']
    assert 'messages' in response
    
    print("✓ Local fulfillment testing passed")
```

#### 9.2.2 End-to-End Integration Testing
```python
def test_bot_integration():
    """Test complete bot workflow"""
    
    # 1. Test intent recognition
    test_utterances = [
        "What's the status of my claim",
        "Check my claim",
        "I want to know about my claim status"
    ]
    
    for utterance in test_utterances:
        response = lex_client.recognize_text(
            botId=bot_id,
            botAliasId='TSTALIASID',
            localeId='en_US',
            sessionId='test-session',
            text=utterance
        )
        
        assert response['sessionState']['intent']['name'] == 'CheckClaimStatus'
    
    # 2. Test slot collection
    response = lex_client.recognize_text(
        botId=bot_id,
        botAliasId='TSTALIASID', 
        localeId='en_US',
        sessionId='test-session',
        text='12345678'  # Member ID
    )
    
    # 3. Test fulfillment execution
    assert response['sessionState']['dialogAction']['type'] == 'Close'
    
    print("✓ End-to-end integration testing passed")
```

---

## 10. Operational Procedures

### 10.1 Bot Deployment Pipeline

#### 10.1.1 Development to Production Workflow
```yaml
Development Process:
├── Step 1: Local Development and Testing
│   ├── Create intents and test utterances
│   ├── Develop fulfillment logic locally
│   └── Unit test conversation flows
├── Step 2: Development Bot Deployment
│   ├── Create bot in development alias
│   ├── Deploy Lambda with test configuration
│   └── Integration testing with sample data
├── Step 3: Staging Validation
│   ├── Create staging bot version
│   ├── Load test with production-like data
│   └── User acceptance testing
└── Step 4: Production Deployment
    ├── Create production bot version
    ├── Deploy Lambda with production config
    └── Monitor and validate performance
```

#### 10.1.2 Version Management Strategy
```python
# Bot versioning and alias management
class BotVersionManager:
    def __init__(self, project_id, bot_name):
        self.project_id = project_id
        self.bot_name = f"project-{project_id}-{bot_name}"
        self.lex_client = boto3.client('lexv2-models')
        
    def create_production_version(self, description):
        """Create stable production version"""
        response = self.lex_client.create_bot_version(
            botId=self.bot_id,
            description=description,
            botVersionLocaleSpecification={
                'en_US': {
                    'sourceBotVersion': 'DRAFT'
                }
            }
        )
        
        version = response['botVersion']
        
        # Update production alias
        self.update_alias('PROD', version)
        
        return version
    
    def create_staging_deployment(self):
        """Deploy to staging for testing"""
        # Create version for staging
        version = self.create_production_version("Staging deployment")
        
        # Update staging alias
        self.update_alias('STAGING', version)
        
        return version
```

### 10.2 Monitoring and Maintenance

#### 10.2.1 Performance Monitoring
```python
# Automated performance monitoring
class LexPerformanceMonitor:
    def __init__(self, project_id):
        self.project_id = project_id
        self.cloudwatch = boto3.client('cloudwatch')
        
    def check_bot_health(self):
        """Monitor bot performance metrics"""
        metrics_to_check = [
            'IntentRecognitionAccuracy',
            'ConversationCompletionRate', 
            'FulfillmentLatency',
            'ErrorRate'
        ]
        
        for metric in metrics_to_check:
            current_value = self.get_metric_value(metric)
            threshold = self.get_threshold(metric)
            
            if current_value < threshold:
                self.send_performance_alert(metric, current_value, threshold)
    
    def analyze_conversation_patterns(self):
        """Analyze conversation logs for optimization opportunities"""
        # Query CloudWatch Logs for conversation patterns
        # Identify frequently unrecognized utterances
        # Suggest new training phrases
        # Detect conversation drop-off points
        pass
    
    def generate_weekly_report(self):
        """Generate weekly performance and usage report"""
        report = {
            'total_conversations': self.get_conversation_count(),
            'intent_accuracy': self.get_intent_accuracy_stats(),
            'top_intents': self.get_top_intents(),
            'error_summary': self.get_error_summary(),
            'cost_analysis': self.get_cost_breakdown(),
            'optimization_recommendations': self.get_optimization_suggestions()
        }
        
        return report
```

#### 10.2.2 Maintenance Procedures
```yaml
Weekly Maintenance Tasks:
├── Review conversation logs for optimization opportunities
├── Update intent training data based on user utterances
├── Monitor and optimize fulfillment function performance
├── Review cost reports and optimize conversation flows
└── Update bot documentation and training materials

Monthly Maintenance Tasks:
├── Analyze conversation analytics and user feedback
├── Review and update intent confidence thresholds
├── Optimize slot collection and validation logic
├── Performance testing with realistic load
└── Security review of fulfillment functions

Quarterly Maintenance Tasks:
├── Comprehensive bot performance review
├── Update integration patterns with new platform features
├── Review and optimize cost allocation and budgets
├── Update disaster recovery procedures
└── Stakeholder feedback and roadmap planning
```

---

## 11. Troubleshooting Guide

### 11.1 Common Issues and Solutions

#### 11.1.1 Bot Creation and Configuration Issues

**Issue: Bot creation fails with IAM permission errors**
```yaml
Symptoms:
  - "AccessDenied" when creating bot
  - Cannot select IAM role during bot setup
  - Bot creation times out

Diagnosis:
  1. Verify ML scientist role has correct Lex permissions
  2. Check role trust relationships
  3. Confirm project-scoped resource access

Solution:
  1. Ensure using project-{uuid}-ml-scientist-role
  2. Verify bot name follows project-{uuid}-* pattern
  3. Contact engineering team for IAM policy review
```

**Issue: Lambda fulfillment function cannot be selected**
```yaml
Symptoms:
  - Lambda function not visible in Lex console dropdown
  - "ResourceNotFoundException" when testing fulfillment
  - Fulfillment configuration saves but doesn't work

Diagnosis:
  1. Check Lambda function uses project-{uuid}-lambda-role
  2. Verify function is in same region as Lex bot
  3. Confirm VPC configuration allows Lex communication

Solution:
  1. Recreate Lambda with correct universal execution role
  2. Verify VPC subnets and security groups
  3. Add explicit resource-based policy for Lex invocation
```

#### 11.1.2 Intent Recognition and NLU Issues

**Issue: Poor intent recognition accuracy**
```yaml
Symptoms:
  - Users frequently trigger wrong intents
  - High confidence scores for incorrect intents
  - Fallback intent triggered too often

Diagnosis:
  1. Review sample utterances for intent overlap
  2. Check confidence threshold settings
  3. Analyze conversation logs for patterns

Solution:
  1. Add more diverse sample utterances (10-15 per intent)
  2. Remove ambiguous utterances that overlap between intents
  3. Adjust intent confidence threshold (default 0.40)
  4. Implement better intent disambiguation logic
```

**Issue: Slot collection fails or loops**
```yaml
Symptoms:
  - Bot repeatedly asks for same slot value
  - Slot validation fails for valid inputs
  - Users cannot complete slot collection

Diagnosis:
  1. Check slot type configuration and validation
  2. Review slot prompt design and clarity
  3. Verify fulfillment slot processing logic

Solution:
  1. Improve slot validation regex patterns
  2. Add slot synonyms and alternative formats
  3. Implement better error handling in slot prompts
  4. Add clarification prompts for invalid inputs
```

#### 11.1.3 Integration and Fulfillment Issues

**Issue: Fulfillment function timeout or errors**
```yaml
Symptoms:
  - Lambda function exceeds timeout limit
  - "Internal Server Error" responses
  - Intermittent fulfillment failures

Diagnosis:
  1. Check CloudWatch Logs for Lambda errors
  2. Monitor Lambda execution duration
  3. Verify VPC connectivity to project resources

Solution:
  1. Optimize database queries and API calls
  2. Implement proper error handling and retry logic
  3. Increase Lambda timeout if necessary (max 15 minutes)
  4. Check VPC endpoints and security group rules
```

**Issue: RAG pipeline integration failures**
```yaml
Symptoms:
  - Generic error responses for knowledge queries
  - Slow response times for complex questions
  - Inconsistent RAG response quality

Diagnosis:
  1. Test RAG pipeline independently
  2. Check OpenSearch and Bedrock connectivity
  3. Review query formatting and context passing

Solution:
  1. Verify OpenSearch endpoint and authentication
  2. Implement fallback responses for RAG failures
  3. Optimize query context and prompt engineering
  4. Add caching for frequently asked questions
```

### 11.2 Monitoring and Debugging Tools

#### 11.2.1 Diagnostic Functions
```python
# Diagnostic utilities for troubleshooting
class LexDiagnostics:
    def __init__(self, project_id, bot_id):
        self.project_id = project_id
        self.bot_id = bot_id
        self.lex_client = boto3.client('lexv2-runtime')
        self.logs_client = boto3.client('logs')
        
    def test_intent_recognition(self, test_utterances):
        """Test intent recognition for given utterances"""
        results = []
        
        for utterance in test_utterances:
            response = self.lex_client.recognize_text(
                botId=self.bot_id,
                botAliasId='TSTALIASID',
                localeId='en_US',
                sessionId=f'test-{uuid.uuid4()}',
                text=utterance
            )
            
            results.append({
                'utterance': utterance,
                'intent': response['sessionState']['intent']['name'],
                'confidence': response.get('interpretations', [{}])[0].get('nluConfidence', {}).get('score', 0),
                'slots': response['sessionState']['intent']['slots']
            })
            
        return results
    
    def analyze_conversation_logs(self, start_time, end_time):
        """Analyze conversation logs for patterns and issues"""
        query = f"""
        fields @timestamp, @message
        | filter @message like /ERROR/
        | filter @timestamp >= {start_time} and @timestamp <= {end_time}
        | stats count() by bin(5m)
        """
        
        response = self.logs_client.start_query(
            logGroupName=f'/aws/lambda/project-{self.project_id}-lex-fulfillment',
            startTime=start_time,
            endTime=end_time,
            queryString=query
        )
        
        return response
    
    def test_fulfillment_connectivity(self):
        """Test connectivity to project resources"""
        connectivity_status = {}
        
        # Test DynamoDB connectivity
        try:
            dynamodb = boto3.resource('dynamodb')
            table = dynamodb.Table(f"{self.project_id}-user-sessions")
            table.meta.client.describe_table(TableName=table.name)
            connectivity_status['dynamodb'] = 'Connected'
        except Exception as e:
            connectivity_status['dynamodb'] = f'Error: {str(e)}'
        
        # Test OpenSearch connectivity
        try:
            # Test OpenSearch endpoint
            connectivity_status['opensearch'] = 'Connected'
        except Exception as e:
            connectivity_status['opensearch'] = f'Error: {str(e)}'
        
        # Test Bedrock connectivity
        try:
            bedrock_client = boto3.client('bedrock-agent-runtime')
            bedrock_client.list_knowledge_bases()
            connectivity_status['bedrock'] = 'Connected'
        except Exception as e:
            connectivity_status['bedrock'] = f'Error: {str(e)}'
        
        return connectivity_status
```

---

## 12. Implementation Timeline

### 12.1 Phase 1: Infrastructure Foundation (Weeks 1-2)

#### Week 1: IAM and Security Setup
- [ ] Extend `project-{uuid}-ml-scientist-role` with Lex permissions
- [ ] Add Lex permissions to universal `project-{uuid}-lambda-role`
- [ ] Configure VPC endpoints for Lex service (if available)
- [ ] Setup security groups and network ACLs
- [ ] Create KMS keys for Lex conversation data encryption

#### Week 2: Governance Framework
- [ ] Configure AWS Config rules for Lex compliance
- [ ] Setup EventBridge rules for bot lifecycle monitoring
- [ ] Create CloudWatch dashboards for Lex metrics
- [ ] Implement cost budgets and alerts
- [ ] Setup automated compliance validation

### 12.2 Phase 2: Service Integration (Weeks 3-4)

#### Week 3: Platform Integration
- [ ] Integrate Lex into service catalog as Tier 2 service
- [ ] Create bot templates for common conversation patterns
- [ ] Develop Lambda fulfillment templates with RAG integration
- [ ] Setup monitoring and alerting automation
- [ ] Create documentation and best practices guide

#### Week 4: Testing and Validation
- [ ] End-to-end testing of bot creation workflow
- [ ] Validate cross-service connectivity (OpenSearch, DynamoDB, Bedrock)
- [ ] Test conversation flows with sample data
- [ ] Verify monitoring and cost tracking systems
- [ ] Security boundary validation and penetration testing

### 12.3 Phase 3: ML Team Enablement (Weeks 5-6)

#### Week 5: Training and Onboarding
- [ ] Conduct ML team training on Lex capabilities and constraints
- [ ] Provide hands-on workshops for bot development
- [ ] Setup development environments and access
- [ ] Create sample conversation flows and use cases
- [ ] Establish support and escalation procedures

#### Week 6: Production Readiness
- [ ] Deploy first production bot with full monitoring
- [ ] Validate performance under realistic load
- [ ] Confirm cost tracking and optimization
- [ ] Document operational procedures and troubleshooting
- [ ] Establish maintenance and update schedules

### 12.4 Success Criteria

#### Technical Metrics
- **100%** bot resources follow project naming conventions
- **< 30 seconds** bot creation time through self-service
- **99.9%** fulfillment function success rate
- **Zero** cross-project resource access violations

#### Business Metrics
- **90%+** intent recognition accuracy for business conversations
- **< 2 seconds** average response time for transactional intents
- **30%** reduction in human agent escalations for routine inquiries
- **ML team self-sufficiency** in bot development and maintenance

---

## 13. Appendices

### 13.1 Bot Configuration Templates

#### 13.1.1 Customer Service Bot Template
```json
{
  "botName": "project-{uuid}-customer-service-bot",
  "description": "Main customer service bot for project {uuid}",
  "intents": [
    {
      "intentName": "CheckClaimStatus",
      "sampleUtterances": [
        "What's the status of my claim",
        "Check my claim from last month",
        "Has my claim been processed",
        "I want to know about my claim",
        "Show me my claim status",
        "Where is my claim",
        "Claim status inquiry"
      ],
      "slots": [
        {
          "slotName": "memberId",
          "slotTypeId": "AMAZON.AlphaNumeric",
          "slotConstraint": "Required"
        },
        {
          "slotName": "claimNumber", 
          "slotTypeId": "AMAZON.AlphaNumeric",
          "slotConstraint": "Optional"
        }
      ],
      "fulfillmentType": "transactional"
    },
    {
      "intentName": "ExplainBenefits",
      "sampleUtterances": [
        "What are my dental benefits",
        "Explain my coverage for surgery", 
        "What's covered under my plan",
        "Tell me about my benefits",
        "What does my insurance cover"
      ],
      "slots": [
        {
          "slotName": "benefitType",
          "slotTypeId": "BenefitTypes",
          "slotConstraint": "Optional"
        }
      ],
      "fulfillmentType": "rag_pipeline"
    }
  ],
  "slotTypes": [
    {
      "slotTypeName": "BenefitTypes",
      "values": [
        {"value": "medical", "synonyms": ["health", "healthcare"]},
        {"value": "dental", "synonyms": ["teeth", "tooth"]},
        {"value": "vision", "synonyms": ["eye", "eyes", "optical"]}
      ]
    }
  ]
}
```

#### 13.1.2 Lambda Fulfillment Template
```python
# Production-ready fulfillment template
import json
import boto3
import os
import logging
import uuid
from datetime import datetime
from botocore.exceptions import ClientError

# Configure logging
logger = logging.getLogger()
logger.setLevel(logging.INFO)

class ProjectLexFulfillment:
    def __init__(self):
        self.project_id = os.environ['PROJECT_ID']
        self.region = os.environ['AWS_REGION']
        
        # Initialize AWS clients using universal Lambda role
        self.dynamodb = boto3.resource('dynamodb')
        self.bedrock_client = boto3.client('bedrock-agent-runtime')
        self.cloudwatch = boto3.client('cloudwatch')
        
        # Project resource references
        self.session_table = self.dynamodb.Table(f"{self.project_id}-user-sessions")
        self.conversation_table = self.dynamodb.Table(f"{self.project_id}-conversations")
        self.opensearch_endpoint = os.environ['OPENSEARCH_ENDPOINT']
        self.bedrock_kb_id = os.environ['BEDROCK_KB_ID']
        
    def lambda_handler(self, event, context):
        """Main Lambda handler for Lex fulfillment"""
        start_time = datetime.utcnow()
        
        try:
            logger.info(f"Processing Lex event: {json.dumps(event, default=str)}")
            
            # Extract event details
            intent_name = event['sessionState']['intent']['name']
            session_id = event['sessionId']
            user_input = event['inputTranscript']
            
            # Log conversation event
            self.log_conversation(session_id, intent_name, user_input, event)
            
            # Route based on intent pattern
            if intent_name in ['CheckClaimStatus', 'UpdateAddress', 'PayBill']:
                response = self.handle_transactional_intent(event)
            elif intent_name in ['ExplainBenefits', 'FindProvider', 'PolicyInfo']:
                response = self.handle_informational_intent(event)
            elif intent_name == 'FallbackIntent':
                response = self.handle_fallback_intent(event)
            else:
                response = self.handle_general_intent(event)
            
            # Calculate processing time and log metrics
            processing_time = (datetime.utcnow() - start_time).total_seconds()
            self.put_custom_metrics(intent_name, 'SUCCESS', processing_time)
            
            logger.info(f"Successfully processed intent {intent_name} in {processing_time}s")
            return response
            
        except Exception as e:
            processing_time = (datetime.utcnow() - start_time).total_seconds()
            logger.error(f"Fulfillment error for intent {intent_name}: {str(e)}")
            
            self.put_custom_metrics(intent_name or 'UNKNOWN', 'ERROR', processing_time)
            return self.create_error_response(str(e))
    
    def handle_transactional_intent(self, event):
        """Handle structured business transactions"""
        intent_name = event['sessionState']['intent']['name']
        slots = event['sessionState']['intent']['slots']
        
        if intent_name == 'CheckClaimStatus':
            return self.process_claim_status_check(slots)
        elif intent_name == 'UpdateAddress':
            return self.process_address_update(slots)
        elif intent_name == 'PayBill':
            return self.process_bill_payment(slots)
        else:
            return self.create_close_response("I'm not sure how to help with that request.")
    
    def handle_informational_intent(self, event):
        """Route complex queries to RAG pipeline"""
        user_query = event['inputTranscript']
        intent_name = event['sessionState']['intent']['name']
        slots = event['sessionState']['intent']['slots']
        
        # Build context from collected slots
        context = self.build_query_context(slots)
        
        # Query RAG pipeline
        rag_response = self.query_rag_pipeline(user_query, context)
        
        return self.create_close_response(rag_response)
    
    def query_rag_pipeline(self, query, context=None):
        """Integration with project RAG system"""
        try:
            # Enhance query with context
            enhanced_query = query
            if context:
                enhanced_query = f"Context: {context}\n\nQuestion: {query}"
            
            # Query Bedrock Knowledge Base
            response = self.bedrock_client.retrieve_and_generate(
                input={'text': enhanced_query},
                retrieveAndGenerateConfiguration={
                    'type': 'KNOWLEDGE_BASE',
                    'knowledgeBaseConfiguration': {
                        'knowledgeBaseId': self.bedrock_kb_id,
                        'modelArn': 'arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-v2'
                    }
                }
            )
            
            return response['output']['text']
            
        except Exception as e:
            logger.error(f"RAG pipeline error: {str(e)}")
            return "I'm having trouble accessing that information right now. " \
                   "Let me connect you with a representative who can help."
    
    def log_conversation(self, session_id, intent_name, user_input, event):
        """Log conversation for analytics and debugging"""
        try:
            conversation_item = {
                'conversation_id': str(uuid.uuid4()),
                'session_id': session_id,
                'timestamp': int(datetime.utcnow().timestamp()),
                'project_id': self.project_id,
                'intent_name': intent_name,
                'user_input': user_input,
                'event_data': json.dumps(event, default=str),
                'created_at': datetime.utcnow().isoformat()
            }
            
            self.conversation_table.put_item(Item=conversation_item)
            
        except Exception as e:
            logger.error(f"Failed to log conversation: {str(e)}")
    
    def put_custom_metrics(self, intent_name, status, processing_time):
        """Track custom metrics for monitoring"""
        try:
            self.cloudwatch.put_metric_data(
                Namespace=f"{self.project_id}/lex",
                MetricData=[
                    {
                        'MetricName': 'ConversationProcessed',
                        'Value': 1,
                        'Dimensions': [
                            {'Name': 'Project', 'Value': self.project_id},
                            {'Name': 'Intent', 'Value': intent_name},
                            {'Name': 'Status', 'Value': status}
                        ],
                        'Timestamp': datetime.utcnow()
                    },
                    {
                        'MetricName': 'ProcessingLatency',
                        'Value': processing_time,
                        'Unit': 'Seconds',
                        'Dimensions': [
                            {'Name': 'Project', 'Value': self.project_id},
                            {'Name': 'Intent', 'Value': intent_name}
                        ],
                        'Timestamp': datetime.utcnow()
                    }
                ]
            )
        except Exception as e:
            logger.error(f"Failed to put custom metrics: {str(e)}")

# Lambda entry point
def lambda_handler(event, context):
    handler = ProjectLexFulfillment()
    return handler.lambda_handler(event, context)
```

### 13.2 Frequently Asked Questions

#### 13.2.1 General Questions

**Q: Why can't I create a bot with a custom name outside the project convention?**
A: All resources must follow the `project-{uuid}-*` naming pattern for security isolation and cost allocation. This prevents cross-project access and ensures proper governance.

**Q: Can I use my own IAM role for Lambda fulfillment functions?**
A: No. All Lambda functions must use the universal `project-{uuid}-lambda-role` that Engineering provisions. This role has access to all project resources and maintains security boundaries.

**Q: How do I access other AWS services from my Lex fulfillment function?**
A: The universal Lambda execution role provides access to all project-scoped resources including OpenSearch, DynamoDB, Bedrock, S3, and others. Use the provided environment variables for service endpoints.

**Q: Can I deploy my bot to external channels like Facebook Messenger?**
A: Channel integrations must be configured through the platform's approved methods. Contact the Engineering team for external channel setup and security review.

#### 13.2.2 Technical Questions

**Q: My bot doesn't recognize user intents accurately. How can I improve this?**
A: Add more diverse sample utterances (10-15 per intent), remove overlapping phrases between intents, and adjust the confidence threshold. Analyze conversation logs to identify common unrecognized patterns.

**Q: How do I integrate my bot with the existing RAG pipeline?**
A: Use the provided template functions that query your project's Bedrock Knowledge Base and OpenSearch index. The universal Lambda role has all necessary permissions configured.

**Q: Why is my fulfillment function timing out?**
A: Check VPC connectivity, optimize database queries, and ensure your function can reach project resources through VPC endpoints. Contact Engineering if network issues persist.

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-07-27 | Engineering Team | Initial document creation with comprehensive Lex integration design |

---

## Approval

**Engineering Team Lead**: _________________ Date: _________

**ML Platform Lead**: _________________ Date: _________

**Security Team Lead**: _________________ Date: _________

---

**Classification**: Internal Use  
**Next Review Date**: October 27, 2025  
**Dependencies**: Lambda Execution Policy Design, DynamoDB Access Control Design, IAM Permissions Summary

---

**END OF DOCUMENT**