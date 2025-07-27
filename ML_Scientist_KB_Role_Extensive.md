# ML Scientist Role - Bedrock Knowledge Base IAM Details

## Document Information
- **Document Version**: 1.0
- **Last Updated**: July 27, 2025
- **Owner**: Engineering Team
- **Audience**: ML Scientists, Platform Architects
- **Review Cycle**: Quarterly

---

## 1. Executive Summary

This document provides comprehensive IAM policy details for ML Scientists working with Amazon Bedrock Knowledge Base within the project-based isolation model. All permissions are strictly scoped to resources following the `project-{uuid}-` naming convention, ensuring complete project isolation while enabling self-service capabilities for AI application development.

### Key Principles
- **Project-Scoped Access**: All KB operations limited to project-specific resources
- **Query-Only Permissions**: No administrative or configuration modification capabilities
- **Secure by Default**: Strict IAM conditions prevent cross-project access
- **Audit-Ready**: Complete logging and monitoring of all KB interactions

---

## 2. Bedrock Knowledge Base IAM Policy

### 2.1 Core Bedrock Permissions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "BedrockKnowledgeBaseAccess",
      "Effect": "Allow",
      "Action": [
        "bedrock:Retrieve",
        "bedrock:RetrieveAndGenerate",
        "bedrock:GetKnowledgeBase",
        "bedrock:ListKnowledgeBases"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "bedrock:knowledgeBaseId": "project-${uuid}-kb"
        }
      }
    },
    {
      "Sid": "BedrockFoundationModelAccess",
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel"
      ],
      "Resource": [
        "arn:aws:bedrock:*::foundation-model/amazon.titan-embed-text-v1",
        "arn:aws:bedrock:*::foundation-model/anthropic.claude-3-sonnet-20240229-v1:0",
        "arn:aws:bedrock:*::foundation-model/anthropic.claude-3-haiku-20240307-v1:0"
      ],
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": ["us-east-1", "us-west-2"]
        }
      }
    }
  ]
}
```

### 2.2 Knowledge Base Query Permissions Detail

#### Retrieve Operation
```json
{
  "Sid": "BedrockRetrieveOperation",
  "Effect": "Allow",
  "Action": "bedrock:Retrieve",
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "bedrock:knowledgeBaseId": "project-${uuid}-kb"
    },
    "NumericLessThanEquals": {
      "bedrock:numberOfResults": 100
    }
  }
}
```

**Capabilities:**
- Execute semantic search queries against project knowledge base
- Retrieve relevant document chunks with similarity scores
- Control number of results (maximum 100 per query)
- Filter by metadata attributes configured by Engineering

**Limitations:**
- Cannot access knowledge bases from other projects
- Result count limited to prevent resource abuse
- No direct vector manipulation capabilities
- Cannot modify retrieval algorithms or parameters

#### RetrieveAndGenerate Operation
```json
{
  "Sid": "BedrockRAGOperation", 
  "Effect": "Allow",
  "Action": "bedrock:RetrieveAndGenerate",
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "bedrock:knowledgeBaseId": "project-${uuid}-kb"
    },
    "StringLike": {
      "bedrock:modelId": [
        "anthropic.claude-3-sonnet-*",
        "anthropic.claude-3-haiku-*"
      ]
    }
  }
}
```

**Capabilities:**
- Combine retrieval with text generation for Q&A workflows
- Use approved foundation models for response generation
- Configure generation parameters (temperature, max tokens)
- Implement custom prompt engineering strategies

**Limitations:**
- Limited to pre-approved foundation models
- Cannot modify model configurations or fine-tuning
- Generation costs count toward project budget limits
- No access to experimental or custom models

### 2.3 Supporting Service Permissions

#### CloudWatch Logging and Metrics
```json
{
  "Sid": "BedrockMonitoring",
  "Effect": "Allow", 
  "Action": [
    "logs:CreateLogGroup",
    "logs:CreateLogStream", 
    "logs:PutLogEvents",
    "cloudwatch:PutMetricData"
  ],
  "Resource": [
    "arn:aws:logs:*:*:log-group:/aws/lambda/project-${uuid}-*",
    "arn:aws:logs:*:*:log-group:/project-${uuid}/bedrock/*"
  ],
  "Condition": {
    "StringEquals": {
      "cloudwatch:namespace": [
        "project-${uuid}/bedrock",
        "AWS/Bedrock"
      ]
    }
  }
}
```

#### Parameter Store Integration
```json
{
  "Sid": "BedrockConfiguration",
  "Effect": "Allow",
  "Action": [
    "ssm:GetParameter",
    "ssm:GetParameters",
    "ssm:GetParametersByPath"
  ],
  "Resource": [
    "arn:aws:ssm:*:*:parameter/project-${uuid}/bedrock/*",
    "arn:aws:ssm:*:*:parameter/project-${uuid}/app-config/*"
  ]
}
```

---

## 3. Operational Capabilities

### 3.1 Query Operations

#### Basic Semantic Search
```python
# Example Lambda implementation pattern
import boto3
import json
import logging

def query_knowledge_base(query_text, max_results=10):
    """
    Execute semantic search against project knowledge base
    """
    bedrock = boto3.client('bedrock-agent-runtime')
    
    try:
        response = bedrock.retrieve(
            knowledgeBaseId=os.environ['KB_ID'],  # project-{uuid}-kb
            retrievalQuery={
                'text': query_text
            },
            retrievalConfiguration={
                'vectorSearchConfiguration': {
                    'numberOfResults': min(max_results, 100)
                }
            }
        )
        
        # Log usage for monitoring
        log_kb_usage('retrieve', query_text, len(response['retrievalResults']))
        
        return response['retrievalResults']
        
    except Exception as e:
        logger.error(f"KB query failed: {str(e)}")
        raise
```

#### RAG-Based Q&A
```python
def generate_answer(question, context_limit=5):
    """
    Use RAG pattern for question answering
    """
    bedrock = boto3.client('bedrock-agent-runtime')
    
    try:
        response = bedrock.retrieve_and_generate(
            input={
                'text': question
            },
            retrieveAndGenerateConfiguration={
                'type': 'KNOWLEDGE_BASE',
                'knowledgeBaseConfiguration': {
                    'knowledgeBaseId': os.environ['KB_ID'],
                    'modelArn': 'arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-sonnet-20240229-v1:0',
                    'retrievalConfiguration': {
                        'vectorSearchConfiguration': {
                            'numberOfResults': context_limit
                        }
                    }
                }
            }
        )
        
        # Extract answer and citations
        answer = response['output']['text']
        citations = response['citations']
        
        return {
            'answer': answer,
            'sources': citations,
            'timestamp': datetime.utcnow().isoformat()
        }
        
    except Exception as e:
        logger.error(f"RAG generation failed: {str(e)}")
        raise
```

### 3.2 Advanced Integration Patterns

#### Multi-Source Knowledge Aggregation
```python
def hybrid_knowledge_search(query, include_graph=True):
    """
    Combine KB results with Neptune graph data
    """
    # Query knowledge base
    kb_results = query_knowledge_base(query)
    
    # Optionally query Neptune for structured data
    if include_graph:
        graph_results = query_neptune_graph(query)
        return combine_results(kb_results, graph_results)
    
    return process_kb_results(kb_results)
```

#### Conversation Context Management
```python
def conversational_query(user_input, conversation_id):
    """
    Maintain conversation context using DynamoDB and KB
    """
    # Retrieve conversation history
    context = get_conversation_context(conversation_id)
    
    # Formulate contextualized query
    enhanced_query = build_contextual_query(user_input, context)
    
    # Query knowledge base with context
    kb_response = query_knowledge_base(enhanced_query)
    
    # Update conversation state
    update_conversation_context(conversation_id, user_input, kb_response)
    
    return kb_response
```

### 3.3 Performance Optimization

#### Intelligent Caching
```python
def cached_kb_query(query_text, cache_ttl=3600):
    """
    Implement ElastiCache integration for KB queries
    """
    cache_key = f"kb_query:{hashlib.md5(query_text.encode()).hexdigest()}"
    
    # Try cache first
    cached_result = get_from_cache(cache_key)
    if cached_result:
        log_cache_hit(query_text)
        return cached_result
    
    # Query KB and cache result
    result = query_knowledge_base(query_text)
    set_cache(cache_key, result, ttl=cache_ttl)
    
    return result
```

#### Batch Query Processing
```python
def batch_process_queries(query_list, batch_size=10):
    """
    Process multiple queries efficiently
    """
    results = []
    for i in range(0, len(query_list), batch_size):
        batch = query_list[i:i + batch_size]
        
        # Process batch with rate limiting
        batch_results = []
        for query in batch:
            result = query_knowledge_base(query)
            batch_results.append(result)
            
            # Rate limiting to prevent throttling
            time.sleep(0.1)
        
        results.extend(batch_results)
    
    return results
```

---

## 4. Security Constraints and Enforcement

### 4.1 Resource Isolation Mechanisms

#### Knowledge Base ID Validation
```json
{
  "Condition": {
    "StringEquals": {
      "bedrock:knowledgeBaseId": "project-${uuid}-kb"
    },
    "StringNotEquals": {
      "bedrock:knowledgeBaseId": [
        "project-*-kb",
        "*"
      ]
    }
  }
}
```

**Enforcement Points:**
- All Bedrock API calls validated against exact KB ID
- Wildcard patterns explicitly denied to prevent enumeration
- Cross-project KB access impossible through IAM constraints
- Runtime validation in Lambda functions for additional security

#### Model Access Restrictions
```json
{
  "Condition": {
    "StringLike": {
      "bedrock:modelId": [
        "anthropic.claude-3-sonnet-*",
        "anthropic.claude-3-haiku-*",
        "amazon.titan-embed-text-v1"
      ]
    },
    "StringNotLike": {
      "bedrock:modelId": [
        "*gpt*",
        "*experimental*", 
        "*custom*"
      ]
    }
  }
}
```

**Security Benefits:**
- Prevents access to unauthorized or expensive models
- Blocks experimental models that may not be production-ready
- Ensures consistent model behavior across project applications
- Maintains cost predictability through approved model list

### 4.2 Network and Data Security

#### VPC Endpoint Requirements
```python
# All Bedrock API calls must go through VPC endpoints
BEDROCK_ENDPOINT_URL = 'https://bedrock-agent-runtime.us-east-1.amazonaws.com'

def create_bedrock_client():
    """
    Create Bedrock client with VPC endpoint configuration
    """
    return boto3.client(
        'bedrock-agent-runtime',
        endpoint_url=BEDROCK_ENDPOINT_URL,
        config=Config(
            region_name=os.environ['AWS_REGION'],
            retries={'max_attempts': 3},
            read_timeout=30
        )
    )
```

#### Input Sanitization and Validation
```python
def sanitize_query_input(user_input):
    """
    Validate and sanitize user queries before KB submission
    """
    # Input length validation
    if len(user_input) > 8000:  # Bedrock limit
        raise ValueError("Query too long")
    
    # Content filtering
    filtered_input = remove_pii(user_input)
    filtered_input = sanitize_html(filtered_input)
    
    # Injection protection
    if contains_injection_patterns(filtered_input):
        raise SecurityException("Potentially malicious input detected")
    
    return filtered_input
```

### 4.3 Audit and Compliance

#### Query Logging Requirements
```python
def log_kb_usage(operation, query, result_count, user_context=None):
    """
    Comprehensive logging for all KB operations
    """
    log_entry = {
        'timestamp': datetime.utcnow().isoformat(),
        'operation': operation,
        'knowledge_base_id': os.environ['KB_ID'],
        'query_hash': hashlib.sha256(query.encode()).hexdigest(),
        'result_count': result_count,
        'user_context': user_context,
        'lambda_function': context.function_name,
        'request_id': context.aws_request_id
    }
    
    logger.info(json.dumps(log_entry))
    
    # Send to monitoring
    put_custom_metric('KBQueryCount', 1)
    put_custom_metric('KBResultCount', result_count)
```

#### Cost Monitoring Integration
```python
def track_kb_costs(operation, tokens_used=0):
    """
    Track and monitor KB usage costs
    """
    cost_metrics = {
        'retrieve': 0.0001,  # per query
        'retrieve_and_generate': 0.001 + (tokens_used * 0.00001)  # per query + per token
    }
    
    estimated_cost = cost_metrics.get(operation, 0)
    
    cloudwatch.put_metric_data(
        Namespace=f'project-{os.environ["PROJECT_ID"]}/costs',
        MetricData=[
            {
                'MetricName': 'BedrockKBCost',
                'Value': estimated_cost,
                'Unit': 'None',
                'Dimensions': [
                    {'Name': 'Operation', 'Value': operation},
                    {'Name': 'Project', 'Value': os.environ['PROJECT_ID']}
                ]
            }
        ]
    )
```

---

## 5. Denied Operations and Limitations

### 5.1 Administrative Operations (Explicitly Denied)

```json
{
  "Sid": "DenyKnowledgeBaseAdministration",
  "Effect": "Deny",
  "Action": [
    "bedrock:CreateKnowledgeBase",
    "bedrock:DeleteKnowledgeBase", 
    "bedrock:UpdateKnowledgeBase",
    "bedrock:CreateDataSource",
    "bedrock:DeleteDataSource",
    "bedrock:UpdateDataSource",
    "bedrock:StartIngestionJob",
    "bedrock:StopIngestionJob",
    "bedrock:GetIngestionJob",
    "bedrock:ListIngestionJobs"
  ],
  "Resource": "*"
}
```

**Impact:**
- Cannot create or delete knowledge bases
- Cannot modify KB configuration or data sources
- Cannot trigger manual sync operations
- Cannot access ingestion job status or logs

### 5.2 Cross-Project Access (Blocked)

```json
{
  "Sid": "DenyCrossProjectAccess",
  "Effect": "Deny", 
  "Action": "bedrock:*",
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "bedrock:knowledgeBaseId": "project-${uuid}-kb"
    }
  }
}
```

**Enforcement:**
- No access to knowledge bases from other projects
- Cannot enumerate or discover other project resources
- Prevents accidental cross-project data exposure
- Maintains strict project isolation boundaries

### 5.3 Direct Vector Store Access (Prohibited)

```json
{
  "Sid": "DenyDirectOpenSearchAccess",
  "Effect": "Deny",
  "Action": [
    "es:ESHttpGet",
    "es:ESHttpPost", 
    "es:ESHttpPut",
    "es:ESHttpDelete"
  ],
  "Resource": "arn:aws:es:*:*:domain/project-${uuid}-vectorstore/*",
  "Condition": {
    "StringNotEquals": {
      "aws:userid": "${bedrock-service-role-id}"
    }
  }
}
```

**Rationale:**
- Prevents direct manipulation of vector embeddings
- Ensures data consistency through Bedrock APIs only
- Protects against vector poisoning attacks
- Maintains KB integrity and performance

---

## 6. Error Handling and Troubleshooting

### 6.1 Common Error Scenarios

#### Access Denied Errors
```python
def handle_bedrock_errors(func):
    """
    Decorator for Bedrock error handling
    """
    def wrapper(*args, **kwargs):
        try:
            return func(*args, **kwargs)
        except ClientError as e:
            error_code = e.response['Error']['Code']
            
            if error_code == 'AccessDeniedException':
                logger.error(f"KB access denied: {e}")
                raise PermissionError("Knowledge base access not authorized")
            
            elif error_code == 'ResourceNotFoundException':
                logger.error(f"KB not found: {e}")
                raise ValueError("Knowledge base not available")
            
            elif error_code == 'ThrottlingException':
                logger.warning(f"KB throttled: {e}")
                time.sleep(1)
                return func(*args, **kwargs)  # Retry once
            
            else:
                logger.error(f"Unexpected KB error: {e}")
                raise
    
    return wrapper
```

#### Resource Limit Errors
```python
def handle_resource_limits(query_text, max_results):
    """
    Validate against resource limits before API call
    """
    if len(query_text) > 8000:
        raise ValueError("Query text exceeds 8000 character limit")
    
    if max_results > 100:
        logger.warning("Result limit capped at 100")
        max_results = 100
    
    return max_results
```

### 6.2 Monitoring and Alerting

#### Performance Monitoring
```python
def monitor_kb_performance():
    """
    Track KB query performance metrics
    """
    metrics = {
        'query_latency': [],
        'result_quality': [],
        'error_rate': 0,
        'cost_per_query': []
    }
    
    # CloudWatch custom metrics
    for metric_name, value in metrics.items():
        if isinstance(value, list) and value:
            cloudwatch.put_metric_data(
                Namespace=f'project-{os.environ["PROJECT_ID"]}/bedrock',
                MetricData=[{
                    'MetricName': metric_name,
                    'Value': sum(value) / len(value),
                    'Unit': 'None'
                }]
            )
```

#### Alert Configuration
```python
def setup_kb_alerts():
    """
    Configure CloudWatch alarms for KB operations
    """
    alarms = [
        {
            'name': 'KB-High-Error-Rate',
            'metric': 'ErrorRate',
            'threshold': 5.0,  # 5% error rate
            'comparison': 'GreaterThanThreshold'
        },
        {
            'name': 'KB-High-Latency',
            'metric': 'QueryLatency', 
            'threshold': 5000,  # 5 seconds
            'comparison': 'GreaterThanThreshold'
        },
        {
            'name': 'KB-High-Cost',
            'metric': 'DailyCost',
            'threshold': 100.0,  # $100/day
            'comparison': 'GreaterThanThreshold'
        }
    ]
    
    for alarm in alarms:
        cloudwatch.put_metric_alarm(**alarm)
```

---

## 7. Best Practices and Optimization

### 7.1 Query Optimization Strategies

#### Semantic Query Enhancement
```python
def optimize_query(user_input):
    """
    Enhance user queries for better KB retrieval
    """
    # Query expansion
    expanded_query = add_domain_context(user_input)
    
    # Synonym handling
    enhanced_query = expand_synonyms(expanded_query)
    
    # Question reformulation
    if is_question(enhanced_query):
        enhanced_query = reformulate_question(enhanced_query)
    
    return enhanced_query
```

#### Result Quality Assessment
```python
def assess_result_quality(query, results):
    """
    Evaluate and rank KB results for relevance
    """
    scored_results = []
    
    for result in results:
        score = calculate_relevance_score(query, result)
        
        if score > 0.7:  # High confidence threshold
            scored_results.append({
                'content': result,
                'relevance_score': score,
                'confidence': 'high'
            })
        elif score > 0.4:  # Medium confidence
            scored_results.append({
                'content': result,
                'relevance_score': score,
                'confidence': 'medium'
            })
    
    return sorted(scored_results, key=lambda x: x['relevance_score'], reverse=True)
```

### 7.2 Cost Optimization

#### Query Frequency Management
```python
def implement_rate_limiting():
    """
    Prevent excessive KB queries that increase costs
    """
    user_id = get_user_id()
    current_time = time.time()
    
    # Check rate limit (100 queries per hour per user)
    query_count = get_user_query_count(user_id, current_time - 3600)
    
    if query_count >= 100:
        raise RateLimitException("Query rate limit exceeded")
    
    # Track query
    increment_user_query_count(user_id, current_time)
```

#### Intelligent Caching
```python
def smart_cache_strategy(query):
    """
    Implement intelligent caching based on query patterns
    """
    # Cache common queries longer
    if is_common_query(query):
        cache_ttl = 7200  # 2 hours
    # Cache specific queries shorter
    elif is_specific_query(query):
        cache_ttl = 1800  # 30 minutes
    # Don't cache personalized queries
    elif is_personalized_query(query):
        cache_ttl = 0
    else:
        cache_ttl = 3600  # 1 hour default
    
    return cache_ttl
```

---

## 8. Support and Escalation

### 8.1 Self-Service Troubleshooting

#### Diagnostic Tools
```python
def diagnose_kb_issues():
    """
    Self-diagnostic tool for ML scientists
    """
    diagnostics = {
        'kb_accessible': test_kb_connectivity(),
        'permissions_valid': validate_permissions(),
        'query_syntax': validate_query_syntax(),
        'results_quality': assess_recent_results(),
        'cost_status': check_budget_status()
    }
    
    return diagnostics
```

#### Performance Analysis
```python
def analyze_kb_performance(time_range='24h'):
    """
    Analyze KB performance over specified time range
    """
    metrics = get_cloudwatch_metrics(
        namespace=f'project-{os.environ["PROJECT_ID"]}/bedrock',
        time_range=time_range
    )
    
    analysis = {
        'avg_latency': calculate_average(metrics['latency']),
        'error_rate': calculate_percentage(metrics['errors'], metrics['total']),
        'cost_trend': analyze_cost_trend(metrics['cost']),
        'query_patterns': analyze_query_patterns(metrics['queries'])
    }
    
    return analysis
```

### 8.2 Engineering Team Escalation

#### Issue Categories

**Infrastructure Issues:**
- Knowledge base unavailability or performance degradation
- VPC connectivity problems affecting KB access
- OpenSearch cluster issues impacting vector search
- Sync job failures or data source problems

**Permission Issues:**
- IAM policy modifications needed for new use cases
- Cross-service integration permission problems
- Security group modifications for network access
- Resource access errors despite correct implementation

**Configuration Requests:**
- Chunking strategy modifications for better retrieval
- Foundation model additions or changes
- Data source additions or modifications
- Performance tuning for specific workloads

#### Escalation Process

1. **Self-Diagnosis**: Use provided diagnostic tools
2. **Documentation Review**: Check troubleshooting guides
3. **Ticket Creation**: Submit detailed issue description with:
   - Project ID and affected resources
   - Error messages and CloudWatch logs
   - Steps to reproduce the issue
   - Business impact assessment
4. **Engineering Review**: Engineering team evaluates and prioritizes
5. **Resolution**: Implementation through Infrastructure as Code
6. **Validation**: ML scientist validates fix in development environment

---

## 9. Compliance and Governance

### 9.1 Audit Requirements

#### Query Audit Trail
- All KB queries logged with user context and timestamp
- Query content hashed for privacy while maintaining auditability
- Result metadata captured for relevance analysis
- Cost attribution tracked per user and project

#### Data Access Patterns
- Regular review of query patterns for anomaly detection
- Cross-project access attempt monitoring
- Unusual query volume or frequency alerts
- Data exfiltration attempt detection

### 9.2 Cost Governance

#### Budget Controls
- Monthly budget limits per project with automated alerts
- Query volume limits to prevent cost overruns
- Real-time cost tracking and reporting
- Automatic throttling when budget thresholds exceeded

#### Usage Optimization
- Regular analysis of query efficiency and optimization opportunities
- Identification of duplicate or unnecessary queries
- Caching effectiveness monitoring and improvement
- Model selection optimization for cost vs. quality

---

## 10. Document Maintenance

### Version History
| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-07-27 | Engineering Team | Initial document creation |

### Review Schedule
- **Monthly**: Usage patterns and cost optimization review
- **Quarterly**: IAM policy review and security assessment
- **Annually**: Complete document revision and updates

### Contact Information
- **Engineering Team**: engineering@company.com
- **ML Platform Team**: ml-platform@company.com  
- **Security Team**: security@company.com
- **Cost Management**: finops@company.com

---

**Classification**: Internal Use Only  
**Distribution**: ML Scientists, Engineering Team, Security Team  
**Retention**: 7 years from creation date
