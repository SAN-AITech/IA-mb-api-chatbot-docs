[← Back to Documentation Home](README.md)

# Key Concepts & Terminology

This document defines the essential concepts and terminology used throughout the IA MB API Chatbot system. Understanding these concepts is crucial for working with the system effectively.

## Core Identifiers

### **SessionId** 
- **Purpose**: Tracks a web browser session that can span multiple conversations
- **Scope**: Single user session across multiple conversations
- **Lifetime**: Until browser session ends or explicit logout
- **Example**: `session-abc123def456`
- **Used in**: Session tracking, analytics data, user context

### **ConversationId**
- **Purpose**: Unique identifier for a single chat conversation
- **Scope**: One conversation thread with complete message history
- **Lifetime**: Permanent (can be restored later)
- **Example**: `conv-789xyz012abc`
- **Used in**: Message storage, conversation retrieval, history management

### **InteractionId**
- **Purpose**: Identifies a single user interaction within a conversation
- **Scope**: One user question and corresponding AI response
- **Lifetime**: Permanent (linked to conversation)
- **Example**: `interaction-456def789ghi`
- **Used in**: Message tracking, telemetry, debugging

### **ResponseId**
- **Purpose**: Tracks the AI's response to a user interaction (redundant with InteractionId)
- **Scope**: Single AI response
- **Lifetime**: Permanent
- **Example**: `response-123ghi456jkl`
- **Used in**: Response tracking, feedback linking

## Conversation Flow Concepts

### **Iteration**
A complete conversation turn cycle consisting of:
1. **User Input** → Stored with `author="human"`
2. **AI Processing** → LangGraph workflow execution
3. **AI Response** → Stored with `author="ai"`
4. **Optional RAG Data** → Stored with `author="retriever"`

**Turn Count**: Tracks the number of iterations in a conversation for policy enforcement.

### **Message Authors**
- **`"human"`** - User messages and questions
- **`"ai"`** - Chatbot responses and answers
- **`"retriever"`** - RAG documents from Knowledge Base

### **SessionContext**
A data structure containing all contextual information for processing a user request:

```python
SessionContext(
    zone_id=str(timezone.utc),
    session_start_time=datetime.now(timezone.utc).isoformat(),
    conversation_id=conversation_id,
    is_streaming=is_streaming,
    client_id=client_id,
    interaction_id=interaction_id,
    session_id=session_id,
    virtual_agent_id=f'LuzIA_{ENVIRONMENT}',
    response_id=response_id
)
```

## Database Structure Concepts

### **DynamoDB Table Design**

#### **INTERACTIONS_TABLE**
- **Primary Key**: `conversationId` (Partition) + `messageId` (Sort)
- **Purpose**: Store all messages and responses
- **Ordering**: Messages ordered chronologically by `messageId`
- **Access Patterns**: Get conversation history, get latest interaction

#### **SESSIONS_TABLE**
- **Primary Key**: `sessionId` (Partition)
- **Purpose**: Store session metadata and analytics
- **Data**: Browser, device, page URL, channel information

### **Message ID Sequencing**
- **Auto-increment**: Each new message gets `max_message_id + 1`
- **Scope**: Within a single conversation
- **Purpose**: Maintains chronological order
- **Implementation**: Query last interaction, increment by 1

## AI/ML Processing Concepts

### **LangGraph State Management**
The system uses LangGraph to orchestrate conversation processing through a state machine:

```python
class State(MessagesState):
    documents: List[Document]           # RAG documents from Knowledge Base
    user_question: str                  # Original user input
    contact_center_answer: CCAnswer     # Agent classification result
    deep_links: Optional[List[str]]     # Related resource links
    suggestions: Optional[SuggestionsAnswer]  # Follow-up questions
    guardrail_applied: bool            # Content safety flag
```

### **Agent Classification**
- **Purpose**: Determine if user needs human agent transfer
- **Options**: `"AGENT"` (transfer to human) or `"OTHER"` (continue with AI)
- **Implementation**: Specialized LLM prompt for intent detection
- **Impact**: Controls conversation routing

### **RAG (Retrieval-Augmented Generation)**
- **Knowledge Base**: AWS Bedrock Knowledge Base for document retrieval
- **Process**: Query → Retrieve relevant docs → Include in LLM context
- **Storage**: Retrieved documents stored as `author="retriever"`
- **Purpose**: Provide accurate, up-to-date information

### **Guardrails**
- **Input Guardrails**: Validate user input for safety/appropriateness
- **Output Guardrails**: Mask or filter AI responses
- **Implementation**: AWS Bedrock Guardrails service
- **Storage**: `guardrailApplied` flag tracks when guardrails triggered

## Streaming and Real-Time Concepts

### **Server-Sent Events (SSE)**
- **Purpose**: Real-time streaming of AI responses to frontend
- **Format**: JSON messages with state indicators
- **States**: `"init"` (first chunk), `"delta"` (content chunks), `"end"` (completion)

### **Response States**
- **INITIAL_STATE**: First response chunk sent to client
- **DELTA_STATE**: Incremental content updates during streaming
- **END_STATE**: Signal that response is complete

### **Event Types**
- **Conversation Response**: Main AI response content
- **Deep Links**: Related resources and links
- **Suggestions**: Follow-up question recommendations
- **Citations**: Source references for RAG content
- **Transfer to Agent**: Human handoff signal

## Security and Access Control

### **Client Ownership Validation**
- **Purpose**: Ensure only conversation owner can access/modify
- **Method**: Compare `clientId` from request with last interaction's `clientId`
- **Implementation**: Query latest interaction before processing
- **Response**: 403 Forbidden for mismatched clients

### **Session Validation**
- **Headers Required**: `X-LZVA-SESSION-ID`, `x-santander-client-id`
- **Validation**: Every request validates session and client identity
- **Security**: Prevents unauthorized conversation access

## Performance and Scaling Concepts

### **Hot Partitions**
- **Cause**: Popular conversations get frequent read/write activity
- **Impact**: DynamoDB performance bottlenecks
- **Mitigation**: Auto-scaling, partition key distribution

### **Query Patterns**
- **Latest First**: Security checks and conversation continuation
- **Full History**: Loading complete conversation for context
- **User Lists**: Finding all conversations for a specific user

### **Caching Strategy**
- **Session Data**: Cached for conversation duration
- **Knowledge Base**: AWS manages retrieval caching
- **LLM Responses**: No caching (always fresh responses)

## Technology Stack

### **Development Environment**
- **UV Package Manager**: Fast Python package manager with enterprise JFrog repository integration
- **Python 3.13**: Modern Python runtime with latest language features

### **Backend Technologies**
- **FastAPI**: High-performance async web framework for REST APIs
- **LangGraph**: Conversation workflow orchestration and state management
- **Pydantic**: Data validation and serialization with type safety

### **Frontend Technologies**
- **NiceGUI**: Integrated web framework for Python-based user interfaces
- **Server-Sent Events (SSE)**: Real-time communication for chat interface
- **Integrated Interface**: Frontend available at `/gui` path within main application

### **AI & ML Technologies**
- **AWS Bedrock**: Managed foundation model service (Claude, etc.)
- **LangChain**: LLM application framework and tool integration
- **RAG (Retrieval Augmented Generation)**: Knowledge base integration for enhanced responses
- **AWS Knowledge Base**: Vector database for document retrieval

### **Data Storage**
- **Amazon DynamoDB**: NoSQL database for conversation persistence
- **Single-Table Design**: Optimized access patterns for conversation data
- **Auto-scaling**: Automatic capacity management based on load

## Environment and Configuration

### **Environment Variables**
- **`COMMONS.INTERACTIONS-TABLE`**: DynamoDB table for messages
- **`COMMONS.SESSIONS-TABLE`**: DynamoDB table for sessions
- **`AWS_REGION`**: AWS region for all services (typically `eu-west-1`)
- **`ENVIRONMENT`**: Deployment environment (`dev`, `qa`, `prod`)

### **AWS Services Integration**
- **Bedrock**: Foundation models and guardrails
- **Knowledge Base**: Document retrieval for RAG
- **DynamoDB**: Conversation and session storage
- **CloudWatch**: Monitoring and logging

## Operational Concepts

### **Telemetry and Observability**
- **Span IDs**: Distributed tracing across service calls
- **Metadata**: Request context for monitoring and debugging
- **OpenInference**: Instrumentation for AI/ML observability

### **Error Handling**
- **Graceful Degradation**: System continues with reduced functionality
- **Error Types**: Validation errors, processing errors, AWS service errors
- **Recovery**: Automatic retry for transient failures

### **Feedback System**
- **Types**: Thumbs up/down, star ratings, text comments
- **Storage**: Feedback linked to specific interactions
- **Purpose**: Quality improvement and response evaluation

---

This terminology guide provides the foundation for understanding all other documentation in the system. Refer back to these concepts when exploring specific implementation details or operational procedures.
