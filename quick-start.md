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

**Important**: In development mode (`ng serve` + direct FastAPI), JWT authentication is completely bypassed for easier development and testing.

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

### **Environment Variables**

```bash
# AWS Configuration
AWS_DEFAULT_REGION=eu-west-1
INTERACTIONS_TABLE=chatbot-interactions-dev
SESSIONS_TABLE=chatbot-sessions-dev

# Agent Configuration  
BEDROCK_MODEL_ID=anthropic.claude-3-sonnet-20240229-v1:0
BEDROCK_REGION=eu-west-1

# API Configuration
API_PREFIX=/chatbot/api/v1
CORS_ORIGINS=["https://mybank.com"]
```

### **Development vs Production**

```yaml
# config/dev.yaml
database:
  interactions_table: "chatbot-interactions-dev"
  sessions_table: "chatbot-sessions-dev"
agent:
  model_id: "anthropic.claude-3-haiku-20240307-v1:0"  # Faster/cheaper

# config/prod.yaml  
database:
  interactions_table: "chatbot-interactions-prod"
  sessions_table: "chatbot-sessions-prod"
agent:
  model_id: "anthropic.claude-3-sonnet-20240229-v1:0"  # More capable
```

## Running the System

### **Local Development**

#### **Backend API (Python)**

```bash
# Install dependencies with UV (enterprise JFrog repository)
uv sync

# Set environment variables
export AWS_DEFAULT_REGION=eu-west-1
export INTERACTIONS_TABLE=chatbot-interactions-dev

# Start backend server
uv run python chatbot_api/run.py
```

#### **Frontend Web Interface (Angular)**

```bash
# Navigate to web directory
cd web

# Install Node.js dependencies
npm install

# Start development server
ng serve --verbose
```

#### **Full Local Setup**

```bash
# Terminal 1: Start backend
export AWS_REGION=eu-west-1
export SERVICE_NAME=mb-api-chatbot-mc
export AWS_PROFILE_NAME=default
export SERVER_PORT=8083
export SERVER_RELOAD=true
uv run python chatbot_api/run.py

# Terminal 2: Start frontend
cd web
ng serve --verbose
```

The backend will run on `http://localhost:8083` and the frontend on `http://localhost:4200`.

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

1. **Setup**: Install UV and Node.js, configure AWS credentials
2. **Backend**: `uv run python chatbot_api/run.py`
3. **Frontend**: `cd web && ng serve --verbose`
4. **Read**: [Architecture Overview](architecture-overview.md) - System design
5. **Understand**: [Conversation Flow](conversation-flow.md) - Message processing
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

# Check Angular CLI
ng version
```

This guide provides everything needed to understand and start using the IA MB API Chatbot system.
