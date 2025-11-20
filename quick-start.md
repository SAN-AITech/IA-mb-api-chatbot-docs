# Quick Start Guide

[← Back to Documentation Home](README.md)

## What is IA MB API Chatbot?

The IA MB API Chatbot is an enterprise-grade conversational AI system built for banking and financial services. It provides secure, intelligent customer support through a modular architecture with integrated web interface, powered by AWS Bedrock and designed for high-scale production environments.

## Architecture at a Glance

```text
Integrated Frontend → FastAPI → AgentExecutor Framework → AWS Bedrock → Dual Memory System
       ↑                ↑            ↑                      ↑              ↑
   NiceGUI UI      API Layer   Modular Agents         Claude AI    AgentCore + DynamoDB
```

## Core Components

### **Integrated Frontend (NiceGUI)**

- **Purpose**: Built-in web interface at `/gui` path
- **Technology**: NiceGUI framework integrated with FastAPI
- **Features**: Real-time chat, configuration management, file upload
- **Access**: Single-server deployment with no separate frontend needed

### **API Layer (FastAPI)**

- **Purpose**: RESTful endpoints for conversation management
- **Security**: Session-based authentication with client isolation  
- **Performance**: Async processing with streaming responses
- **Documentation**: Interactive Swagger UI at `/docs` path

### **AgentExecutor Framework (Modular AI Processing)**

- **Architecture**: Specialized components for different AI tasks
- **Components**: AgentExecutor, MessageProcessor, AgentMemory, AgentGuardrails
- **Orchestration**: LangGraph StateGraph manages conversation workflows
- **AI Models**: Claude (Anthropic) via AWS Bedrock
- **Safety**: Integrated guardrails for content filtering
- **Tools**: MCP servers, Knowledge Base, dynamic tool registry

### **Dual Memory System**

- **AgentCore Memory (Primary)**: Semantic search and user preference learning
- **DynamoDB Memory (Fallback)**: Traditional conversation persistence
- **Design**: Runtime switching between memory systems
- **Security**: Client-based access control and data isolation

## Key Concepts

### **Conversations**

- **Definition**: A sequence of messages between user and AI managed by AgentExecutor
- **Identification**: Unique UUID per conversation
- **Persistence**: Messages stored in dual memory system (AgentCore + DynamoDB)
- **Security**: Client ownership prevents cross-user access
- **Processing**: Modular AgentExecutor framework handles message flow

### **Sessions**

- **Purpose**: Track user analytics and device context
- **Data**: Browser, device, page URL, channel information
- **Usage**: Enriches conversation data for insights
- **Lifecycle**: Created per user visit, linked to interactions

### **Interactions**

- **Definition**: Individual messages within conversations processed by MessageProcessor
- **Types**: Human messages, AI responses, retriever results, tool outputs
- **Storage**: Dual storage in both AgentCore memory and DynamoDB
- **Linking**: Response pairs connected via responseId
- **Streaming**: Real-time delivery through Server-Sent Events

## Common Use Cases

### **1. Customer Support Chat**

```bash
# Start new conversation
POST /conversations/{conversation_id}/messages
{
    "message": "I need help with my account balance",
    "session_id": "session-uuid"
}

# Continue conversation
POST /conversations/{conversation_id}/messages
{
    "message": "Can you show me recent transactions?",
    "session_id": "session-uuid"
}
```

### **2. Knowledge Base Queries**

```bash
# Ask specific banking questions
POST /conversations/{conversation_id}/messages
{
    "message": "What are your foreign transaction fees?",
    "session_id": "session-uuid"
}
```

### **3. Conversation Management**

```bash
# Get conversation history
GET /conversations/{conversation_id}

# Get user's saved conversations  
GET /conversations/user

# Save conversation with topic
PUT /conversations/{conversation_id}/topic
{
    "topic": "Account Balance Inquiry"
}
```

## API Endpoints Overview

### **Core Conversation API**

- `POST /conversations/{id}/messages` - Send message to AI
- `GET /conversations/{id}` - Retrieve conversation history
- `GET /conversations/user` - List user's conversations
- `PUT /conversations/{id}/topic` - Save/update conversation topic
- `DELETE /conversations/{id}` - Delete conversation

### **Session Management**

- `POST /sessions` - Create new session
- `GET /sessions/{id}` - Retrieve session data

### **Feedback System**

- `POST /feedback` - Submit message feedback
- `GET /feedback/stats` - Retrieve feedback analytics

### **Health & Configuration**

- `GET /health` - System health status
- `GET /configuration` - Agent configuration

## Authentication

### **Development vs Production Authentication**

| Mode | Authentication | Client ID Source | Frontend Access |
|------|----------------|------------------|-----------------|
| **Development** | Session-based | Header validation | Integrated at `/gui` |
| **Production** | Full OIDC/JWT | JWT claim `username` | Integrated at `/gui` |

**Important**: The system includes an integrated NiceGUI frontend at `/gui` path in both development and production modes. No separate frontend server is required.

### **Session-Based Authentication (Current)**

```javascript
// Request headers
{
    "X-LZVA-SESSION-ID": "session-uuid",
    "X-SANTANDER-CLIENT-ID": "client-identifier", 
    "Content-Type": "application/json"
}
```

### **Security Model**

- **Client Isolation**: Users can only access their own conversations
- **Session Validation**: Headers validated on every request
- **Conversation Ownership**: Verified through database lookup
- **Content Safety**: AgentGuardrails component filters harmful content
- **Memory Isolation**: Dual memory system provides data separation

## Database Schema (Simplified)

### **Dual Memory Architecture**

#### **AgentCore Memory (Primary)**

```text
Semantic Memory: User preferences and context via AWS Bedrock AgentCore
Long-term Learning: Automated user preference detection
Short-term Memory: Recent conversation context
Namespace: /preferences/{actorId} for user isolation
```

#### **DynamoDB Storage (Fallback + Audit)**

**INTERACTIONS_TABLE**

```text
Primary Key: conversationId + messageId
Contains: messages, metadata, analytics, feedback

Example:
conversationId="conv-123", messageId=1  → "Hello, I need help"
conversationId="conv-123", messageId=2  → "I'd be happy to help you..."
```

**SESSIONS_TABLE**

```text
Primary Key: sessionId
Contains: user device/browser info, analytics data

Example:
sessionId="sess-456" → {browser: "Chrome", device: "Desktop", ...}
```

## Configuration

### **AWS Parameter Store (Primary Configuration)**

The system automatically loads configuration from AWS Parameter Store using `ok-config`:

- **Parameter Store Console**: [https://eu-west-1.console.aws.amazon.com/systems-manager/parameters/](https://eu-west-1.console.aws.amazon.com/systems-manager/parameters/?region=eu-west-1&tab=Table)
- **Automatic Loading**: No manual setup required - parameters load on startup
- **Parameter Format**: `commons.parameter-name` becomes `PARAMETER_NAME` environment variable
- **All Configuration**: Database tables, AI models, guardrails, etc.

### **Optional .env File Override**

For development, create a `.env` file to override specific parameters:

```bash
# .env file (optional) - only listed parameters override Parameter Store
AWS_REGION=eu-west-1
SERVER_PORT=8082
SERVICE_NAME=mb-api-chatbot-mc
# All other parameters still load from Parameter Store automatically
```

**Important**: Only parameters explicitly listed in `.env` will override Parameter Store values. All other configuration continues to load from Parameter Store.

## Running the System

### **Local Development**

```bash
# Install dependencies from enterprise JFrog repository
export UV_INDEX_PRIVATE_REGISTRY_USERNAME=x756900@opendigitalservices.com
export UV_INDEX_PRIVATE_REGISTRY_PASSWORD=<your-jfrog-token>
uv sync

# Start server - configuration automatically loads from AWS Parameter Store
uv run src/ia_mb_api_chatbot/run.py
```

### **Access the Integrated Application**

The application provides a complete integrated experience:

- **Main UI**: `http://localhost:8082/gui` (NiceGUI-based integrated frontend)
- **API Docs**: `http://localhost:8082/docs` (Interactive Swagger documentation)
- **Health**: `http://localhost:8082/health`

**Key Benefits**:

- **Single Server**: No separate frontend server needed
- **Integrated Experience**: UI, API, and documentation in one application  
- **Real-time Features**: Server-Sent Events for streaming responses
- **Development Simplicity**: One command starts the complete system

### **Configuration Notes**

- **Parameter Store**: Configuration loads automatically from AWS Parameter Store using `ok-config`
- **No Manual Setup**: Database tables, models, guardrails configured automatically
- **Memory System**: AgentCore and DynamoDB dual memory configured via Parameter Store
- **Tool Integration**: MCP servers and dynamic tool registry configured centrally
- **JFrog Token**: Get `UV_INDEX_PRIVATE_REGISTRY_PASSWORD` from [JFrog Portal](https://gluoneurope.jfrog.io/ui/login)
- **Token Expiration**: JFrog tokens expire every 3 months

### **Production Deployment**

```bash
# Using provided scripts
./start-server.sh  # Includes SSL certificates and production config
```

## Monitoring

### **Health Checks**

```bash
# Basic health
GET /health
→ {"status": "healthy", "timestamp": "2025-01-28T10:30:00Z"}

# Database connectivity
GET /health/database
→ {"dynamodb": "connected", "tables": ["interactions", "sessions"]}
```

### **Metrics to Monitor**

- **Response Time**: API endpoint latency and AgentExecutor processing time
- **Memory Performance**: AgentCore memory retrieval and DynamoDB read/write capacity
- **AI Processing**: Bedrock invocation success rates and AgentExecutor component health
- **Stream Processing**: MessageProcessor performance and real-time delivery metrics
- **Error Rates**: Failed requests by endpoint and component-level failures

## Next Steps

### **For Developers**

1. **Setup**: Install UV, configure AWS credentials
2. **Backend**: `uv run src/ia_mb_api_chatbot/run.py`
3. **Access**: Open `http://localhost:8082/gui` for the integrated frontend
4. **Read**: [Architecture Overview](architecture-overview.md) - Modular agent execution system design
5. **Understand**: [Send Message Flow](send-message-flow.md) - Complete AgentExecutor processing flow
6. **Explore**: [Memory Management](memory-management.md) - Dual memory system architecture
7. **Learn**: [LangGraph Implementation](langgraph-implementation.md) - Agent framework details

### **For Operations**

1. **Setup**: Configure AWS credentials and Parameter Store access
2. **Deploy**: Use Docker or direct Python deployment with integrated frontend
3. **Monitor**: Set up CloudWatch alerts for AgentExecutor components and dual memory system

### **For Integration**

1. **API Reference**: Test endpoints with integrated Swagger UI at `/docs`
2. **Authentication**: Implement session-based authentication with proper headers
3. **Error Handling**: Plan for network failures and AgentExecutor component resilience
4. **Memory System**: Understand dual storage pattern (AgentCore + DynamoDB)

## Support

### **Common Issues**

- **AWS Credentials**: Ensure proper IAM permissions for DynamoDB, Bedrock, and Parameter Store
- **Memory System**: Verify AgentCore memory access and DynamoDB fallback configuration
- **Parameter Store**: Confirm automatic parameter loading and ok-config setup
- **Frontend Access**: Integrated UI available at `/gui` - no separate server needed
- **Model Access**: Ensure Bedrock model access is enabled in AWS account

### **Debugging**

```bash
# Enable debug logging
export LOG_LEVEL=DEBUG

# Check AWS connectivity
aws dynamodb describe-table --table-name chatbot-interactions-dev

# Test Bedrock access
aws bedrock list-foundation-models --region eu-west-1

# Verify Parameter Store access
aws ssm get-parameters-by-path --path "/commons" --region eu-west-1

# Verify UV installation and dependencies
uv --version
uv tree  # Show dependency tree
```

This guide provides everything needed to understand and start using the IA MB API Chatbot system with its modern AgentExecutor framework and integrated frontend.
