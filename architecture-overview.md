# System Architecture Overview

## Introduction

The IA MB API Chatbot is an enterprise-grade conversational AI system that combines modern web technologies with AWS AI services to deliver intelligent, context-aware customer interactions. The system is designed for scalability, security, and maintainability.

## High-Level Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Angular Web   │    │   FastAPI       │    │   AWS Services  │
│   Frontend      │◄──►│   Backend       │◄──►│   (AI/Storage)  │
│                 │    │                 │    │                 │
│ • Chat UI       │    │ • Conversation  │    │ • Bedrock LLMs  │
│ • Real-time SSE │    │ • LangGraph     │    │ • Knowledge Base│
│ • File Upload   │    │ • Session Mgmt  │    │ • DynamoDB      │
│ • Configuration │    │ • Authentication│    │ • Guardrails    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                        │                        │
         └────────────────────────┼────────────────────────┘
                                  │
                    ┌─────────────────┐
                    │   Data Flow     │
                    │                 │
                    │ User Question   │
                    │      ↓          │
                    │ RAG Retrieval   │
                    │      ↓          │
                    │ LLM Processing  │
                    │      ↓          │
                    │ Response Stream │
                    │      ↓          │
                    │ Database Store  │
                    └─────────────────┘
```

## System Components

### **Frontend Layer (Angular)**
- **Technology**: Angular 20.1.3 with TypeScript
- **Responsibilities**:
  - Real-time chat interface with Server-Sent Events
  - Configuration management UI
  - File upload and attachment handling
  - Authentication and session management
- **Key Features**:
  - Responsive design for multiple devices
  - Progressive Web App capabilities
  - Internationalization support

### **Backend Layer (FastAPI)**
- **Technology**: Python 3.13 with FastAPI framework
- **Architecture Pattern**: Clean Architecture with layered separation
- **Responsibilities**:
  - RESTful API endpoints for chat operations
  - Real-time streaming with Server-Sent Events
  - Session and conversation management
  - Integration orchestration with AWS services

#### **Backend Layer Structure**
```
Controllers Layer    ──► API endpoints and request handling
     │
Services Layer      ──► Business logic and orchestration
     │
Domain Layer        ──► Data models and business entities
     │
Infrastructure      ──► AWS integrations and external services
```

### **AI/ML Layer (AWS Bedrock + LangGraph)**
- **LangGraph**: Conversation workflow orchestration
- **AWS Bedrock**: Foundation models for text generation
- **Knowledge Base**: RAG (Retrieval-Augmented Generation) for contextual responses
- **Guardrails**: Content safety and compliance filtering

### **Storage Layer (DynamoDB)**
- **INTERACTIONS_TABLE**: All conversation messages and responses
- **SESSIONS_TABLE**: User session metadata and analytics
- **Design Pattern**: Single-table design with composite keys

## Data Flow Architecture

### **Conversation Processing Pipeline**

```
1. User Input Validation
   ├── Session validation (headers)
   ├── Request structure validation
   └── Content safety check

2. Conversation Context Assembly  
   ├── Retrieve conversation history
   ├── Build SessionContext
   └── Security validation (ownership)

3. AI Processing (LangGraph)
   ├── Knowledge Base retrieval (RAG)
   ├── Agent classification (human vs AI)
   ├── Foundation model processing
   └── Response generation

4. Real-time Streaming
   ├── Server-Sent Events to frontend
   ├── Progressive response building
   └── State management (init/delta/end)

5. Persistence
   ├── Store user message
   ├── Store retrieved documents
   └── Store AI response
```

### **Security and Access Control**

```
Request ──► Header Validation ──► Session Lookup ──► Client Authorization
  │              │                      │                    │
  │         X-LZVA-SESSION-ID      sessionId exists     clientId matches
  │         x-santander-client-id        │              conversation owner
  │                │                     │                    │
  └─────────────── │ ────────────────────│────────────────────┘
                   ▼                     ▼
              403 Forbidden         Continue Processing
```

## Scalability Design

### **Horizontal Scaling**
- **Frontend**: CDN distribution, static asset optimization
- **Backend**: Container-based deployment with auto-scaling
- **Database**: DynamoDB auto-scaling based on demand
- **AI Services**: AWS Bedrock native scaling

### **Performance Optimization**
- **Caching**: Session data cached during conversation
- **Streaming**: Real-time response delivery
- **Connection Pooling**: Efficient AWS service connections
- **Hot Partition Management**: DynamoDB key distribution

## Integration Points

### **External Systems**
- **Authentication**: Session-based with client ID validation
- **File Storage**: Temporary file handling for document upload
- **Monitoring**: CloudWatch integration for observability
- **Analytics**: Session tracking and conversation metrics

### **Configuration Management**
- **Environment-based**: Separate configs for dev/qa/prod
- **YAML Configuration**: Agent settings and prompts
- **Feature Flags**: Dynamic system behavior control

## Security Architecture

### **Authentication & Authorization**
- **Session Management**: Browser session tracking
- **Client Validation**: Per-request client ID verification
- **Conversation Ownership**: Client-scoped conversation access

### **Content Safety**
- **Input Guardrails**: User input validation and filtering
- **Output Guardrails**: AI response content masking
- **Audit Trail**: Complete conversation logging

### **Data Protection**
- **Encryption**: TLS in transit, DynamoDB encryption at rest
- **Access Control**: IAM-based AWS service permissions
- **Data Retention**: Configurable conversation lifecycle

## Operational Characteristics

### **Monitoring & Observability**
- **Distributed Tracing**: Span tracking across services
- **Metrics Collection**: Performance and usage analytics
- **Error Handling**: Graceful degradation and recovery
- **Health Checks**: System availability monitoring

### **Deployment Architecture**
- **Containerization**: Docker-based application packaging
- **Infrastructure as Code**: CloudFormation/CDK templates
- **Environment Promotion**: Automated dev → qa → prod pipeline
- **Blue-Green Deployment**: Zero-downtime updates

## Technology Stack Summary

| Layer | Technology | Purpose |
|-------|------------|---------|
| Frontend | Angular 20.1.3, TypeScript | User interface and interaction |
| Backend | Python 3.13, FastAPI | API services and business logic |
| AI/ML | LangGraph, AWS Bedrock | Conversation orchestration and AI |
| Storage | DynamoDB | Conversation and session persistence |
| Infrastructure | AWS Services | Cloud platform and managed services |
| Development | UV, Node.js, WSL2 | Development environment and tooling |

## Next Steps

To dive deeper into specific aspects of the system:

1. **Understand Core Concepts**: Read [Key Concepts](./key-concepts.md) for terminology
2. **Learn the Flow**: Study [Send Message Flow](./send-message-flow.md) for detailed processing
3. **Explore Database**: Review [Database Design](./database-design.md) for storage patterns
4. **API Reference**: Check [API Endpoints](./api-endpoints.md) for integration details
5. **AI Implementation**: Examine [LangGraph Implementation](./langgraph-implementation.md) for AI workflow

This architecture provides a solid foundation for building scalable, secure, and maintainable conversational AI applications.
- `agent.py` - LLM agent implementation
- `aws.py` - AWS DynamoDB and Bedrock integration
- `session.py` - Session lifecycle management
- `prompts.py` - Prompt templates management

### 4. **Domain Layer**
- Request/Response models
- Session context management
- Agent configuration

## Key Technologies

- **FastAPI**: REST API framework
- **LangGraph**: Conversation state management
- **LangChain**: LLM integration
- **AWS Bedrock**: Foundation models
- **AWS DynamoDB**: Persistent storage
- **AWS Knowledge Base**: RAG (Retrieval Augmented Generation)
- **Server-Sent Events (SSE)**: Real-time streaming responses

## Data Flow

1. User sends message through Angular frontend
2. FastAPI receives request and validates headers
3. Session context is created/retrieved
4. LangGraph processes the conversation state
5. Agent queries Knowledge Base for relevant documents
6. LLM generates response using context + retrieved docs
7. Response is streamed back via SSE
8. Conversation state is persisted to DynamoDB
