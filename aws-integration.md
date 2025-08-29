[← Back to Documentation Home](README.md)

# AWS Services Integration

## AWS Services Overview

This chatbot integrates with multiple AWS services for AI capabilities and data persistence.

### 1. **AWS Bedrock (Foundation Models)**

**Purpose**: Large Language Model hosting and inference

**Configuration**:
```python
FOUNDATION_MODEL = os.getenv("COMMONS.FUNDATION-MODEL")  # e.g., "anthropic.claude-3-sonnet-20240229-v1:0"
```

**Usage**:
- Text generation for chat responses
- Intent classification for contact center routing
- Follow-up question suggestions
- Content analysis and processing

**Integration**:
```python
llm = init_chat_model(
    model=FOUNDATION_MODEL,
    temperature=0.1,
    region_name=AWS_REGION
)
```

### 2. **AWS Knowledge Base (RAG)**

**Purpose**: Retrieval Augmented Generation for contextual responses

**Configuration**:
```python
KNOWLEDGE_BASE_ID = os.getenv("COMMONS.KNOWLEDGE-BASE-ID")
KNOWLEDGE_BASE_NAME = os.getenv("COMMONS.KNOWLEDGE-BASE-NAME")
```

**Usage**:
- Semantic search for relevant documents
- Context enrichment for LLM responses
- Source citation for transparency
- Domain-specific knowledge retrieval

**Integration**:
```python
retriever = AmazonKnowledgeBasesRetriever(
    knowledge_base_id=KNOWLEDGE_BASE_ID,
    retrieval_config={"vectorSearchConfiguration": {"numberOfResults": 10}}
)
```

### 3. **AWS DynamoDB (Data Persistence)**

**Purpose**: Conversation and session storage

**Tables**:
- **Sessions Table**: User session metadata and analytics
- **Interactions Table**: Individual chat messages and responses

**Configuration**:
```python
SESSIONS_TABLE = os.getenv("COMMONS.SESSIONS-TABLE")
INTERACTIONS_TABLE = os.getenv("COMMONS.INTERACTIONS-TABLE")
```

**Usage**:
- Conversation history persistence
- User session tracking
- Analytics and monitoring data
- Multi-turn conversation context

### 4. **AWS Guardrails (Content Safety)**

**Purpose**: Content filtering and safety validation

**Configuration**:
```python
GUARDRAIL_ID = os.getenv("COMMONS.GUARDRAIL-ID")
GUARDRAIL_VERSION = os.getenv("COMMONS.GUARDRAIL-VERSION")
```

**Usage**:
- Input content validation
- Response content filtering
- Harmful content detection
- Compliance enforcement

### 5. **AWS Systems Manager Parameter Store**

**Purpose**: Configuration management

**Usage**:
- Centralized parameter storage
- Environment-specific configurations
- Secure credential management
- Runtime configuration updates

**Integration via `ok_config`**:
```python
if "AWS_PROFILE_NAME" in os.environ:
    ok_config.load_config(os.environ["AWS_PROFILE_NAME"])
else:
    ok_config.load_config()
```

## Security & Authentication

### 1. **IAM Roles & Policies**

The application assumes an IAM role with permissions for:
- Bedrock model access
- Knowledge Base queries
- DynamoDB read/write operations
- Systems Manager parameter access
- Guardrails validation

### 2. **SSL/TLS Configuration**

SSL certificates configured for secure AWS connections:
```python
# SSL configuration for AWS services
os.environ["SSL_CERT_FILE"] = cert_file
os.environ["REQUESTS_CA_BUNDLE"] = cert_file
os.environ["CURL_CA_BUNDLE"] = cert_file
```

### 3. **Environment Isolation**

Different AWS resources per environment:
- Development: `dev.yaml` configuration
- QA: `qa.yaml` configuration
- Production: Environment-specific parameters

## Performance Considerations

### 1. **Connection Pooling**
- Boto3 session reuse across requests
- DynamoDB connection optimization
- Bedrock client connection management

### 2. **Caching Strategy**
- Parameter Store values cached
- Knowledge Base results cached where appropriate
- Session data cached for request duration

### 3. **Rate Limiting**
- Bedrock token limits managed
- DynamoDB throughput considerations
- Knowledge Base query optimization

## Monitoring & Observability

### 1. **AWS CloudWatch Integration**
- Application logs sent to CloudWatch
- Performance metrics collection
- Error tracking and alerting

### 2. **Phoenix Telemetry**
```python
ENABLE_TELEMETRY = (os.getenv("COMMONS.ENABLE-TELEMETRY") or "").lower() == "true"
```

### 3. **Distributed Tracing**
- Span IDs tracked across AWS services
- Request correlation for debugging
- Performance bottleneck identification
