[← Back to Documentation Home](README.md)

# System Architecture Overview

## Introduction

The IA MB API Chatbot is an enterprise-grade conversational AI system that combines modern web technologies with AWS AI services to deliver intelligent, context-aware customer interactions. The system is designed for scalability, security, and maintainability.

## High-Level Architecture

```
┌─────────────────┐    ┌─────────────────────────────────────┐    ┌─────────────────┐
│   Integrated    │    │         FastAPI Backend             │    │   AWS Services  │
│   Frontend      │◄──►│                                     │◄──►│   (AI/Storage)  │
│   (NiceGUI)     │    │  ┌─────────────────────────────┐    │    │                 │
│ • Chat UI       │    │  │    Agent Execution Layer    │    │    │ • Bedrock LLMs  │
│ • Real-time SSE │    │  │                             │    │    │ • AgentCore     │
│ • File Upload   │    │  │ • AgentExecutor             │    │    │ • Knowledge Base│
│ • Configuration │    │  │ • MessageProcessor          │    │    │ • DynamoDB      │
│                 │    │  │ • AgentMemory (dual)        │    │    │ • Guardrails    │
│                 │    │  │ • AgentGuardrails           │    │    │ • MCP Servers   │
│                 │    │  └─────────────────────────────┘    │    │                 │
└─────────────────┘    └─────────────────────────────────────┘    └─────────────────┘
         │                              │                                  │
         └──────────────────────────────┼──────────────────────────────────┘
                                        │
                          ┌─────────────────────────────┐
                          │        Data Flow            │
                          │                             │
                          │ User Question               │
                          │      ↓                      │
                          │ AgentExecutor.init_agent()  │
                          │      ↓                      │
                          │ RAG Retrieval + Tools       │
                          │      ↓                      │
                          │ MessageProcessor streams    │
                          │      ↓                      │
                          │ Memory persistence          │
                          └─────────────────────────────┘
```

## System Components

### **Frontend Layer (Integrated NiceGUI)**
- **Technology**: NiceGUI framework integrated within FastAPI
- **Responsibilities**:
  - Real-time chat interface with Server-Sent Events
  - Configuration management UI
  - File upload and attachment handling
  - Authentication and session management
- **Key Features**:
  - Integrated web interface at `/gui` path
  - No separate server deployment required
  - Real-time UI updates and responsiveness

### **Backend Layer (FastAPI)**

- **Technology**: Python 3.13 with FastAPI framework
- **Architecture Pattern**: Modular Agent Execution with clean separation
- **Responsibilities**:
  - RESTful API endpoints for chat operations
  - Real-time streaming with Server-Sent Events
  - Session and conversation management
  - Agent execution orchestration

#### **Modular Agent Execution Architecture**

```text
Agent Service Layer         ──► LangGraph orchestration and tool integration
     │
Agent Execution Framework   ──► Specialized execution components
     ├── AgentExecutor       ──► Core orchestration and initialization
     ├── MessageProcessor    ──► Dual stream handling and response processing
     ├── AgentMemory        ──► Memory system abstraction (AgentCore/DynamoDB)
     └── AgentGuardrails    ──► Safety and content filtering
     │
Controllers Layer          ──► API endpoints and request handling
     │
Domain Layer              ──► Data models and business entities
     │
Infrastructure            ──► AWS integrations and external services
```

### **AI/ML Layer (AWS Bedrock + LangGraph + MCP)**

- **LangGraph**: StateGraph-based conversation orchestration
- **AWS Bedrock**: Foundation models for text generation
- **AgentCore Memory**: Advanced semantic memory with user preference learning
- **Knowledge Base**: RAG (Retrieval-Augmented Generation) for contextual responses
- **Guardrails**: Content safety and compliance filtering
- **MCP Servers**: Model Context Protocol for external tool integration

### **Memory Layer (Dual System)**

- **AgentCore (Primary)**: AWS Bedrock's advanced memory system
  - Semantic search capabilities
  - User preference learning
  - Context-aware memory retrieval
- **DynamoDB (Fallback)**: Traditional persistence layer
  - INTERACTIONS_TABLE: All conversation messages and responses
  - SESSIONS_TABLE: User session metadata and analytics
- **Runtime Switching**: Configurable memory system selection

## Data Flow Architecture

### **Conversation Processing Pipeline**

```text
1. User Input Validation
   ├── Session validation (headers)
   ├── Request structure validation
   └── Content safety check (AgentGuardrails)

2. Agent Initialization (AgentExecutor)
   ├── Memory system selection (AgentCore/DynamoDB)
   ├── Tool registry setup (MCP servers, Knowledge Base)
   ├── LangGraph state initialization
   └── Context assembly from conversation history

3. Agent Execution Framework
   ├── StateGraph orchestration with tool calling
   ├── Knowledge Base retrieval (RAG)
   ├── Foundation model processing (Bedrock)
   ├── Memory persistence (dual system)
   └── Response generation with citations

4. Stream Processing (MessageProcessor)
   ├── Dual stream handling (messages + values)
   ├── Real-time delivery via Server-Sent Events
   ├── Progressive response building
   └── State management (init/delta/end)

5. Post-processing
   ├── Memory storage (conversation context)
   ├── Analytics and session tracking
   └── Cleanup and resource management
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

- **AWS Parameter Store (Primary)**: `ok-config` system for automatic configuration loading
- **Environment Variables (.env)**: Optional local overrides for development
- **YAML Configuration**: Agent settings, prompts, and behavior templates
- **Feature Flags**: Dynamic system behavior control via configuration switches

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
| Frontend | NiceGUI (integrated) | User interface and real-time interaction |
| Backend | Python 3.13, FastAPI, AgentExecutor | API services and modular execution |
| AI/ML | LangGraph, AWS Bedrock, MCP Servers | Agent orchestration and AI processing |
| Memory | AWS AgentCore, DynamoDB (dual) | Advanced semantic and traditional memory |
| Configuration | AWS Parameter Store, .env | Dynamic configuration management |
| Storage | DynamoDB | Conversation and session persistence |
| Infrastructure | AWS Services | Cloud platform and managed services |
| Development | UV, WSL2 | Package management and development environment |

## Next Steps

To dive deeper into specific aspects of the system:

1. **Understand Core Concepts**: Read [Key Concepts](./key-concepts.md) for terminology
2. **Learn the Flow**: Study [Send Message Flow](./send-message-flow.md) for detailed processing
3. **Explore Database**: Review [Database Design](./database-design.md) for storage patterns
4. **API Reference**: Check [API Endpoints](./api-endpoints.md) for integration details
5. **AI Implementation**: Examine [LangGraph Implementation](./langgraph-implementation.md) for AI workflow
6. **Memory Management**: Read [Memory Management](./memory-management.md) for dual memory system details

This modular architecture provides a solid foundation for building scalable, secure, and maintainable conversational AI applications with advanced memory capabilities and flexible tool integration.
