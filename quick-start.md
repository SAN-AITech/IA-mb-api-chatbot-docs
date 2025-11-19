[← Back to Documentation Home](README.md)

# Quick Start Guide

## What is IA MB API Chatbot?

The IA MB API Chatbot is an enterprise-grade conversational AI system built for banking and financial services. It provides secure, intelligent customer support through a REST API, powered by AWS Bedrock and designed for high-scale production environments.

## Architecture at a Glance

```text
Web Client → FastAPI → LangGraph Agent → AWS Bedrock → DynamoDB
    ↑           ↑          ↑              ↑            ↑
   UI/UX    API Layer  AI Workflow   Claude AI    Conversation
                                                   Storage
```

## Core Components

### **API Layer (FastAPI)**

- **Purpose**: RESTful endpoints for conversation management
- **Security**: JWT-based authentication with client isolation
- **Performance**: Async processing with proper error handling

### **AI Engine (LangGraph + AWS Bedrock)**

- **Orchestration**: LangGraph manages conversation workflows
- **AI Models**: Claude (Anthropic) via AWS Bedrock
- **Safety**: AWS Guardrails for content filtering
- **Knowledge**: RAG integration with vector databases

### **Data Storage (DynamoDB)**

- **Design**: Single-table pattern for optimal performance
- **Scale**: Auto-scaling with sub-100ms query latency
- **Security**: Client-based row-level access control

## Key Concepts

### **Conversations**

- **Definition**: A sequence of messages between user and AI
- **Identification**: Unique UUID per conversation
- **Persistence**: Messages stored with auto-increment IDs
- **Security**: Client ownership prevents cross-user access

### **Sessions**

- **Purpose**: Track user analytics and device context
- **Data**: Browser, device, page URL, channel information
- **Usage**: Enriches conversation data for insights
- **Lifecycle**: Created per user visit, linked to interactions

### **Interactions**

- **Definition**: Individual messages within conversations
- **Types**: Human messages, AI responses, retriever results
- **Storage**: DynamoDB with full conversation context
- **Linking**: Response pairs connected via responseId

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

| Mode | Authentication | Client ID Source | JWT Validation |
|------|----------------|------------------|----------------|
| **Development** | Bypassed | Hardcoded `'clientid'` | None |
| **Production** | Full OIDC/JWT | JWT claim `username` | NGINX validation |

**Important**: In development mode with integrated frontend, JWT authentication is completely bypassed for easier development and testing.

### **JWT Token Requirements (Production Only)**

```javascript
// Request headers
{
    "Authorization": "Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImtpZCI6...",
    "Content-Type": "application/json"
}

// Token must contain
{
    "sub": "client-identifier",  // Used for conversation ownership
    "iat": 1640995200,          // Token issued time
    "exp": 1640998800           // Token expiration
}
```

### **Security Model**

- **Client Isolation**: Users can only access their own conversations
- **Conversation Ownership**: Verified on every message
- **Content Safety**: AWS Guardrails filter harmful content

## Database Schema (Simplified)

### **INTERACTIONS_TABLE**

```text
Primary Key: conversationId + messageId
Contains: messages, metadata, analytics, feedback

Example:
conversationId="conv-123", messageId=1  → "Hello, I need help"
conversationId="conv-123", messageId=2  → "I'd be happy to help you..."
```

### **SESSIONS_TABLE**

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

### **Access the Application**

The application provides an integrated web interface:

- **Main UI**: `http://localhost:8082/gui` (NiceGUI-based frontend)
- **API Docs**: `http://localhost:8082/docs` (Interactive Swagger documentation)
- **Health**: `http://localhost:8082/health`

**Note**: No separate development server needed - the frontend is integrated at `/gui`.

### **Configuration Notes**

- **Parameter Store**: Configuration loads automatically from AWS Parameter Store
- **No Manual Setup**: Database tables, models, guardrails configured in Parameter Store
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

- **Response Time**: API endpoint latency
- **Database Performance**: DynamoDB read/write capacity
- **AI Processing**: Bedrock invocation success rates  
- **Error Rates**: Failed requests by endpoint

## Next Steps

### **For Developers**

1. **Setup**: Install UV, configure AWS credentials
2. **Backend**: `uv run src/ia_mb_api_chatbot/run.py`
3. **Access**: Open `http://localhost:8082/gui` for the integrated frontend
4. **Read**: [Architecture Overview](architecture-overview.md) - System design
5. **Understand**: [Send Message Flow](send-message-flow.md) - Complete message processing and database guide
6. **Explore**: [Database Architecture](database-architecture.md) - Data design

### **For Operations**

1. **Setup**: Configure AWS credentials and environment variables
2. **Deploy**: Use Docker or direct Python deployment
3. **Monitor**: Set up CloudWatch alerts and health checks

### **For Integration**

1. **API Reference**: Test endpoints with provided examples
2. **Authentication**: Implement JWT token generation
3. **Error Handling**: Plan for network and service failures

## Support

### **Common Issues**

- **AWS Credentials**: Ensure proper IAM permissions for DynamoDB and Bedrock
- **CORS Errors**: Configure allowed origins in environment
- **Database Errors**: Verify table names and regions match
- **Model Access**: Ensure Bedrock model access is enabled in AWS account

### **Debugging**

```bash
# Enable debug logging
export LOG_LEVEL=DEBUG

# Check AWS connectivity
aws dynamodb describe-table --table-name chatbot-interactions-dev

# Test Bedrock access
aws bedrock list-foundation-models --region eu-west-1

# Verify UV installation and dependencies
uv --version
uv tree  # Show dependency tree
```

This guide provides everything needed to understand and start using the IA MB API Chatbot system.
