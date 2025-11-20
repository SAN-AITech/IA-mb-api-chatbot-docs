# Send Message to Agent - Complete Flow & Database Guide

[← Back to Documentation Home](README.md)

## Overview

The `send_message_to_agent` endpoint is the core functionality of the chatbot API. It processes user messages through a sophisticated AI pipeline using the modular AgentExecutor framework, AWS Bedrock, and real-time streaming responses. This comprehensive guide covers the complete flow from API request to database storage, including conversation iteration concepts and database architecture.

**High-Level Flow**: The endpoint receives user input, identifiers, and request parameters. It validates security by looking up the last interaction, builds a comprehensive SessionContext, and then passes both the SendSaveMessageRequest and SessionContext to the AgentExecutor framework that orchestrates the entire AI conversation workflow through specialized components.

## Quick Architecture Overview

```text
User Message → Security Validation → Session Context → AgentExecutor Processing → Database Storage → Response
     ↓              ↓                    ↓                    ↓                     ↓              ↓
 HTTP Request   Client Ownership    Context Building   Agent Framework      Message Storage   JSON Response
```

**Flow Summary**:

1. **HTTP Request** arrives with user message and metadata
2. **Client Ownership** verified through database lookup  
3. **Context Building** creates comprehensive SessionContext
4. **Agent Framework** processes message through modular AgentExecutor system
5. **Message Storage** persists conversation to dual memory system
6. **JSON Response** streams back to user in real-time

## Endpoint Details

**Route**: `POST /conversations/{conversation_id}/messages`  
**Controller**: [conversation.py](../src/ia_mb_api_chatbot/controllers/conversation.py)  
**Service Method**: [agent.py (build)](../src/ia_mb_api_chatbot/services/agent.py)  
**Execution Framework**: [agent_execution/ module](../src/ia_mb_api_chatbot/services/agent_execution/)

## 📢 **Recent Changes Notice**

> **⚠️ Important**: This documentation reflects the current modular architecture with AgentExecutor framework:
>
> - **Agent Architecture**: Now uses **AgentExecutor** framework with specialized components
> - **Message Processing**: **MessageProcessor** handles dual stream processing
> - **Memory System**: **Dual memory** system (AgentCore + DynamoDB) with runtime switching
> - **Execution Flow**: **Modular components** replace monolithic `_execute` method
>
> All references in this document reflect the current modular implementation.

## Request Flow Architecture

```text
User Input → Validation → Security Check → Context Building → AgentExecutor Framework → Real-time Streaming → Dual Memory Storage
     ↓              ↓            ↓              ↓                       ↓                        ↓                    ↓
  JSON Request → Headers → Last Interaction → SessionContext → AgentExecutor.init_agent() → MessageProcessor → AgentCore/DynamoDB
                                                     ↓
                                    LangGraph StateGraph + AWS Bedrock + Knowledge Base + MCP Servers
```

> **📋 Data Models Reference**: For detailed API request/response models and data structures, see [Request & Response Models Documentation](request-response-models.md)

## What is a Conversation Iteration?

### **Definition**

A **conversation iteration** is one complete turn in a chat conversation, consisting of:

1. **User Input** → Stored as `author="human"`
2. **AI Processing** → LangGraph workflow execution  
3. **AI Response** → Stored as `author="ai"`
4. **Optional RAG Data** → Stored as `author="retriever"` (when Knowledge Base is used)

### **Iteration Lifecycle Visualization**

```bach
┌─────────────────┐     ┌─────────────────┐    ┌─────────────────┐
│   User Message  │───▶│  AI Processing  │───▶│   AI Response   │
│   messageId: 1  │     │   (LangGraph)   │    │   messageId: 3  │
│  author: human  │     │                 │    │   author: ai    │
└─────────────────┘     └─────────────────┘    └─────────────────┘
         │                       │                       ▲
         │              ┌─────────────────┐              │
         │              │ Retriever Data  │──────────────┘
         │              │   messageId: 2  │
         │              │ author:retriever│
         │              └─────────────────┘
         │                       │
         ▼                       ▼
┌────────────────────────────────────────────────────────────────────┐
│                    DynamoDB Storage                                │
│  conversationId: "conv-123"                                        │
│  ├── messageId: 1, author: "human", message: "What's my balance?"  │
│  ├── messageId: 2, author: "retriever", message: "Account data..." │
│  └── messageId: 3, author: "ai", message: "Your balance is $1,234" │
└────────────────────────────────────────────────────────────────────┘
```

> **💾 Memory & State Management**: For details on message ID sequencing and turn count management, see [Memory Management Documentation](memory-management.md)

## Complete Processing Flow

### **Phase 1: Request Validation & Setup**

The controller layer is intentionally simple, following clean architecture principles by acting as a pure routing layer through the conversation controller.

#### **FastAPI Automatic Validation & SendSaveMessageRequest Structure**

When a request arrives at the `/conversations/{conversation_id}/messages` endpoint, FastAPI automatically validates the incoming JSON against the `SendSaveMessageRequest` model using Pydantic. This ensures data integrity before any business logic executes.

**Complete Request Model Structure** - *See [Request & Response Models](request-response-models.md) for full documentation*:

```python
class SendSaveMessageRequest(BaseModel):
    input: Input = Field(default=None)      # Message content and processing options
    context: Context = Field(default=None)  # Session and system context

class Input(BaseModel):
    messageType: str = Field(default=None)  # Optional message category
    text: str                               # Required: User's message content
    options: Options = Field(default=None)  # Processing configuration

class Options(BaseModel):
    stream: bool        # Enable real-time streaming response
    suggestions: bool   # Generate follow-up question suggestions

class Context(BaseModel):
    execution: Execution = Field(default=None)  # Session identifiers
    system: System = Field(default=None)        # User environment info
```

**Key Validation Points**:

- **`input.text`**: Only required field - the actual user message
- **`input.options.stream`**: Controls response delivery method (SSE vs HTTP)
- **`input.options.suggestions`**: Whether to generate follow-up questions
- **`context.system.turnCount`**: Used for conversation state management
- **All other fields**: Optional with sensible defaults

#### **Header-Based Dependency Injection**

```python
client_id: Optional[str] = Depends(get_client_id)      # X-SANTANDER-CLIENT-ID header
session_id: Optional[str] = Depends(get_session_id)    # X-LZVA-SESSION-ID header
```

#### **Direct Service Delegation**

```python
# Controller acts as pure routing layer - no business logic
return await conversation_service.process_user_input(
    client_id, session_id, conversation_id, request
)
```

**Key Design Decision**: All validation, security checks, and business logic are handled in the service layer, keeping the controller focused solely on HTTP routing.

### **Phase 2: Security Validation & Context Building**

This critical security phase ensures conversation ownership and prevents unauthorized access before building the execution context.

1. **Retrieve Last Interaction**:

   ```python
   last_interaction = service_aws.get_conversation_last_interaction(conversation_id)
   ```

2. **Database Query Implementation**:

   ```python
   def get_conversation_last_interaction(self, conversation_id: str):
       """Retrieve the last interaction of a conversation."""
       response = self.dynamodb.query(
           TableName=INTERACTIONS_TABLE,
           KeyConditionExpression='conversationId = :conversationId',
           ExpressionAttributeValues={
               ':conversationId': {'S': conversation_id}
           },
           ScanIndexForward=False,    # Descending order (latest first)
           Limit=1                    # Only get the most recent interaction
       )
       if response.get('Items'):
           return response['Items'][0]  # Return the latest interaction item
       return None                      # No interactions found (new conversation)
   ```

3. **Client Authorization Check**:

   ```python
   if last_interaction and last_interaction.get('clientId', {}).get('S', '') != client_id:
       logger.warning(
           "Client ID %s does not match the last interaction client ID %s for conversation %s",
           client_id,
           last_interaction.get('clientId', {}).get('S', ''),
           conversation_id,
       )
       return Response(status_code=status.HTTP_403_FORBIDDEN)  # Block unauthorized access
   ```

   **Key Security Features**:
   - **Conversation Ownership**: Ensures only the original client can continue a conversation
   - **DynamoDB Structure**: Uses DynamoDB's native data types (`{'S': 'value'}` for strings)
   - **Error Handling**: Returns 403 Forbidden for mismatched client IDs
   - **New Conversations**: `None` result allows new conversations to proceed

### **Phase 3: SessionContext Creation**

The system builds a comprehensive SessionContext object that provides all necessary metadata for the AgentExecutor framework.

1. **SessionContext Builder Call**:

   ```python
   session_context = self._create_session_context_builder(
       conversation_id,                           # From URL path parameter
       client_id,                                # From header
       send_message_request.input.options.stream, # From request options  
       session_id,                               # From header
       send_message_request.context.execution.interactionId  # From request or generated
   )
   ```

2. **SessionContext Construction**:

   ```python
   return SessionContext(
       zone_id=str(timezone.utc),
       session_start_time=datetime.now(timezone.utc).isoformat(),
       conversation_id=conversation_id,        # From URL path
       is_streaming=is_streaming,              # From request options
       client_id=client_id,                    # From header
       interaction_id=interaction_id or str(uuid.uuid4()),  # Generated or from request
       session_id=session_id,                  # From header
       virtual_agent_id=f'LuzIA_{ENVIRONMENT}',
       response_id=str(uuid.uuid4()),          # Newly generated UUID
   )
   ```

### **Phase 4: AgentExecutor Framework Processing**

The AgentExecutor framework is the **heart of the entire conversation system**. It orchestrates the AI processing pipeline through specialized components, handles real-time streaming via MessageProcessor, and manages dual memory persistence. This modular system receives the prepared `SessionContext` and `SendSaveMessageRequest` and executes the complete conversation workflow.

#### **4.1 AgentExecutor Initialization**

```python
from src.ia_mb_api_chatbot.services.agent_execution.agent_executor import AgentExecutor
from src.ia_mb_api_chatbot.services.agent_execution.message_processor import MessageProcessor

# Initialize the modular execution framework
executor = AgentExecutor(agent_config, session_context)
message_processor = MessageProcessor(session_context, suggestions_enabled)
```

**Key Initialization Components**:

- **AgentExecutor**: Core orchestration component that manages:
  - Memory system selection (AgentCore vs DynamoDB)
  - Tool registry setup (MCP servers, Knowledge Base)
  - LangGraph StateGraph configuration
  - Context assembly and validation

- **MessageProcessor**: Stream handling component that manages:
  - Dual stream processing (messages + values)
  - Real-time delivery via Server-Sent Events
  - Response formatting and citation processing
  - Memory persistence coordination

#### **4.2 Agent Graph Construction and Tool Integration**

**Dynamic Tool Registry Setup**:

```python
# Agent builds StateGraph with integrated tools
agent_graph = executor.build_graph(
    tools=[
        knowledge_base_tool,     # AWS Knowledge Base for RAG
        mcp_servers_tools,       # Model Context Protocol integrations
        deep_link_tools,         # Navigation and routing tools
        suggestion_tools         # Follow-up question generation
    ],
    memory_system=selected_memory_type  # AgentCore or DynamoDB
)
```

**StateGraph Architecture**:

1. **Input Processing**: User message validation and formatting
2. **Memory Retrieval**: Context loading from selected memory system
3. **Tool Orchestration**: Dynamic tool calling based on user intent
4. **Model Invocation**: AWS Bedrock foundation model processing
5. **Response Generation**: Structured output with citations and metadata

#### **4.3 Dual Stream Processing Architecture**

**MessageProcessor Stream Handling**:

```python
# Process dual streams from agent graph execution
async for stream_type, stream_value in agent_graph.astream(
    agent_input,
    stream_mode=["messages", "values"]  # Dual stream processing
):
    if stream_type == "messages":
        # Real-time text chunks for streaming UI
        await message_processor.process_message_stream(stream_value)
    elif stream_type == "values":
        # Complete node outputs (tools, memory, suggestions)
        await message_processor.process_values_stream(stream_value)
```

**Stream Processing Benefits**:

- **Real-time Experience**: Immediate text streaming to frontend
- **Rich Metadata**: Tool outputs, citations, and system information
- **Memory Integration**: Automatic persistence to selected memory system
- **Error Recovery**: Graceful handling of processing failures

### **Phase 5: Response Processing and Persistence**

Once the AgentExecutor framework completes processing, the MessageProcessor handles response formatting and persistence to the dual memory system.

#### **5.1 Multi-Modal Response Processing**

The MessageProcessor handles different types of responses from the agent graph execution:

**Contact Center Classification**: When the system determines a human agent is needed, it immediately initiates a transfer:

```python
if cc_answer == 'AGENT':
    # Immediate transfer to human agent
    yield transfer_to_agent_response(session_context)
    return  # Stop processing, transfer initiated
```

**Deep Links Processing**: Navigation and routing information for UI enhancement:

```python
if deep_links:
    yield build_deep_links_response(session_context, deep_links)
```

**Suggestions Processing**: Follow-up questions to guide conversation:

```python
if suggestions_answer and suggestions_answer.followup_questions:
    yield build_suggestions_response(session_context, suggestions_answer)
```

#### **5.2 Real-Time Text Streaming**

The MessageProcessor extracts and streams text content in real-time:

```python
# Extract text content from various chunk formats
if isinstance(chunk.content, str):
    chunk_text = chunk.content
else:
    # Handle multi-part content (text + metadata)
    chunk_text = ''.join([
        c.get('text', '') for c in chunk.content 
        if c.get('type', '') == 'text'
    ])

total_message += chunk_text  # Accumulate complete response

# Stream to frontend if enabled
if chunk_text and is_streaming:
    yield build_conversation_response(chunk_text, session_context, "DELTA")
```

#### **5.3 Memory Persistence Pipeline**

The dual memory system ensures conversation persistence across both traditional and advanced memory systems:

**User Message Storage**:

```python
agent_memory.save_memory(
    session_context=session_context,
    message=user_question,
    author="human",
    span_id=span_id,
    guardrail_applied=False
)
```

**Document Citations Storage**: When RAG retrieval occurs:

```python
if documents:
    citations = extract_citations_from_documents(documents)
    document_content = compile_document_content(documents)
    
    # Store retrieved documents as separate interaction
    agent_memory.save_memory(
        session_context=session_context,
        message=apply_guardrail_masking(document_content),
        author="retriever",
        span_id=span_id
    )
    
    # Stream citations to client
    yield build_citations_response(session_context, citations)
```

**AI Response Storage**:

```python
agent_memory.save_memory(
    session_context=session_context,
    message=apply_guardrail_masking(total_message),
    author="ai",
    span_id=span_id,
    guardrail_applied=final_state.get('guardrail_applied', False)
)
```

#### **5.4 Response Finalization**

The MessageProcessor completes the conversation flow:

```python
# Send final response if not streaming
if not is_streaming:
    yield build_conversation_response(total_message, session_context, "DELTA")

# Signal conversation completion
yield build_conversation_response("", session_context, "END")
```

#### **5.5 Error Handling & Recovery**

The AgentExecutor framework provides comprehensive exception management:

```python
try:
    # Agent execution and stream processing
    async for result in executor.process_conversation():
        yield result
except Exception as e:
    logger.error(
        "Error in AgentExecutor processing for conversation: %s",
        session_context.conversation_id,
        exc_info=e
    )
    # Graceful degradation - continue with error response
    yield build_error_response(session_context, str(e))
```

**Key Design Principles**:

- **Fail-Safe Processing**: Errors don't crash the entire conversation
- **Complete Telemetry**: Full error context captured via OpenTelemetry
- **Graceful Degradation**: Partial responses delivered when possible
- **Memory Consistency**: Dual storage ensures data persistence even with failures

### **Phase 6: Database Architecture and Storage Patterns**

The dual memory system (AgentCore + DynamoDB) maintains conversation integrity through sophisticated storage operations that support both traditional persistence and advanced semantic memory.

#### **6.1 Message ID Sequencing**

The system maintains proper message ordering through auto-incrementing message IDs:

```python
def create_interaction(self, session_context: SessionContext, message: str, author: str):
    # Get current maximum message ID for conversation
    response = self.get_conversation_last_interaction(session_context.conversation_id)
    if response:
        max_message_id = int(response['messageId']['N'])    # Get current max ID
        topic = response.get('topic', {}).get('S', '')      # Preserve existing topic
    else:
        max_message_id = 0                                  # First interaction in conversation
        topic = None
    
    # Create new interaction with incremented ID
    new_message_id = max_message_id + 1
```

#### **6.2 DynamoDB Item Structure**

The system builds comprehensive DynamoDB items with rich metadata:

```python
item = {
    'conversationId': {'S': session_context.conversation_id},  # Partition key
    'messageId': {'N': str(new_message_id)},                  # Sort key (auto-increment)
    'interactionId': {'S': session_context.interaction_id},   # Unique interaction ID
    'author': {'S': author},                                  # human, ai, retriever
    'message': {'S': message},                                # Actual message content
    'createdAt': {'S': datetime.now(timezone.utc).isoformat()}, # Timestamp
    'responseId': {'S': session_context.response_id},         # Response tracking
    'guardrailApplied': {'BOOL': guardrail_applied},          # Content safety flag
    'sessionId': {'S': session_context.session_id}           # Session reference
}

# Enrich with additional metadata
if topic:
    item['topic'] = {'S': topic}                   # Preserve conversation topic
if session_context.client_id:
    item['clientId'] = {'S': session_context.client_id}  # Client ownership
```

#### **6.3 Dual Storage Coordination**

When AgentCore memory is enabled, the system coordinates between both storage systems:

```python
# Primary storage to selected memory system
await agent_memory.save_memory(session_context, message, author, span_id)

# Fallback/audit storage to DynamoDB
if isinstance(agent_memory, AgentMemoryAgentCore):
    # Also persist to DynamoDB for audit trail
    await dynamo_service.create_interaction(session_context, message, author)
```

### **INTERACTIONS_TABLE (Main Storage)**

**Primary Key Design:**

- **Partition Key**: `conversationId` - Groups all messages in one conversation
- **Sort Key**: `messageId` - Orders messages chronologically within conversation

**Complete Item Structure:**

```python
{
    # Primary Keys
    'conversationId': {'S': 'conversation-uuid'},     # Partition key
    'messageId': {'N': '1'},                          # Sort key (auto-increment)
    
    # Iteration Tracking
    'interactionId': {'S': 'interaction-uuid'},       # Unique per iteration
    'responseId': {'S': 'response-uuid'},             # Links user→AI pairs
    'author': {'S': 'human|ai|retriever'},           # Message type
    'message': {'S': 'actual message content'},       # The text
    'createdAt': {'S': '2025-08-28T10:30:00Z'},      # Timestamp
    
    # Security & Ownership
    'clientId': {'S': 'client-identifier'},           # User ownership
    'sessionId': {'S': 'session-uuid'},              # Session reference
    'guardrailApplied': {'BOOL': False},              # Content safety
    
    # Analytics (from session data)
    'browser': {'S': 'Chrome 120.0'},                 # User browser
    'device': {'S': 'Desktop'},                       # Device type
    'pageUrl': {'S': 'https://bank.com/chat'},        # Source page
    'channel': {'S': 'web'},                          # Channel type
    
    # Conversation Management
    'topic': {'S': 'Account Balance Inquiry'},        # Conversation title
    'spanId': {'S': 'telemetry-span-id'},            # Tracing
    
    # Feedback System
    'score': {'N': '5'},                              # User rating (1-5)
    'scoringType': {'S': 'thumbs_up'},               # Feedback type
    'scoreMessage': {'S': 'Very helpful!'}            # User comment
}
```

### **SESSIONS_TABLE (Metadata Storage)**

**Primary Key**: `sessionId` (Partition Key)

**Complete Item Structure:**

```python
{
    'sessionId': {'S': 'session-uuid'},               # Partition key
    'clientId': {'S': 'client-identifier'},           # User reference
    'createdAt': {'S': '2025-08-28T10:00:00Z'},      # Session start
    'browser': {'S': 'Chrome 120.0'},                 # Browser info
    'device': {'S': 'Desktop'},                       # Device type
    'pageUrl': {'S': 'https://bank.com/chat'},        # Source page
    'channel': {'S': 'web'},                          # Channel type
    'spanId': {'S': 'session-telemetry-span'}         # Tracing
}
```

### **Secondary Indexes**

**clientId-conversationId-index**: For retrieving user's conversations

- **Partition Key**: `clientId`
- **Sort Key**: `conversationId`

## Query Patterns for Conversation Iterations

### **1. Get Conversation History (All Iterations)**

```python
def get_conversation(conversation_id: str):
    response = dynamodb.query(
        TableName=INTERACTIONS_TABLE,
        KeyConditionExpression='conversationId = :id',
        ExpressionAttributeValues={':id': {'S': conversation_id}},
        ScanIndexForward=True  # Chronological order
    )
    return response.get('Items', [])
```

### **2. Get Latest Iteration (Security Check)**

```python
def get_conversation_last_interaction(conversation_id: str):
    response = dynamodb.query(
        TableName=INTERACTIONS_TABLE,
        KeyConditionExpression='conversationId = :id',
        ExpressionAttributeValues={':id': {'S': conversation_id}},
        ScanIndexForward=False,  # Latest first
        Limit=1
    )
    return response.get('Items', [None])[0]
```

### **3. Get User's Conversation List**

```python
def get_conversations(client_id: str):
    response = dynamodb.query(
        TableName=INTERACTIONS_TABLE,
        IndexName='clientId-conversationId-index',
        KeyConditionExpression='clientId = :client',
        FilterExpression='attribute_exists(topic)',
        ExpressionAttributeValues={':client': {'S': client_id}}
    )
    return response.get('Items', [])
```

## Performance Characteristics

### **Write Patterns**

- **Hot Partitions**: Active conversations get frequent writes
- **Sequential Writes**: messageId auto-increment creates predictable access
- **Atomic Operations**: Each message write is independent

### **Read Patterns**

- **Latest First**: Security checks read most recent interaction
- **Full History**: Conversation loading reads entire history  
- **User Lists**: Index queries for conversation discovery

### **Scaling Considerations**

```python
# Auto-scaling based on:
# - Read Capacity: Conversation history requests
# - Write Capacity: New message frequency
# - Storage: Message content and metadata growth
# - Hot Partitions: Popular conversation activity
```

## Integration Points

The AgentExecutor framework integrates with multiple AWS and external services:

- **AWS Bedrock**: Foundation model inference via AgentCore and direct API calls
- **AWS AgentCore**: Advanced memory system with semantic search capabilities  
- **AWS Knowledge Base**: Document retrieval for RAG through integrated tools
- **AWS DynamoDB**: Traditional conversation persistence and audit trail
- **AWS Guardrails**: Content safety validation through AgentGuardrails component
- **MCP Servers**: Model Context Protocol for external tool integration
- **Phoenix Telemetry**: Observability and monitoring via OpenTelemetry spans

## Security Features

The system implements comprehensive security through multiple layers:

- **Header Validation**: Session and client ID verification at the controller level
- **Content Filtering**: AgentGuardrails component provides safety checks
- **Input Sanitization**: Message content validation before processing
- **Conversation Ownership**: Client authorization checks prevent unauthorized access
- **Span Tracking**: Request correlation for security auditing via OpenTelemetry
- **Memory Isolation**: Dual memory system provides data separation and fallback

## Summary

The `send_message_to_agent` endpoint represents a sophisticated, modular architecture that processes user messages through multiple coordinated phases:

1. **Request Processing**: FastAPI validation and header extraction
2. **Security Validation**: Client authorization and conversation ownership checks
3. **Context Building**: Comprehensive SessionContext creation with metadata
4. **Agent Execution**: Modular AgentExecutor framework with specialized components
5. **Response Processing**: MessageProcessor handles streaming and multi-modal responses
6. **Memory Persistence**: Dual storage system ensuring both advanced and traditional memory

This modular design provides scalability, maintainability, and flexibility while ensuring robust conversation management and user experience. The AgentExecutor framework enables easy extension and testing of new capabilities while maintaining backward compatibility with existing storage systems.

For detailed information about specific components:

- **Agent Architecture**: See [LangGraph Implementation](./langgraph-implementation.md)
- **Memory Management**: See [Memory Management Documentation](./memory-management.md)  
- **Execution Flow**: See [Execute Method Flow](./execute-method-flow.md)
- **Data Models**: See [Request & Response Models](./request-response-models.md)
