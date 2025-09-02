[← Back to Documentation Home](README.md)

# Send Message to Agent - Complete Flow & Database Guide

## Overview

The `send_message_to_agent` endpoint is the core functionality of the chatbot API. It processes user messages through a sophisticated AI pipeline using LangGraph, AWS Bedrock, and real-time streaming responses. This comprehensive guide covers the complete flow from API request to database storage, including conversation iteration concepts and database architecture.

**High-Level Flow**: The endpoint receives user input, identifiers, and request parameters. It validates security by looking up the last interaction, builds a comprehensive SessionContext, and then passes both the SendSaveMessageRequest and SessionContext to the core `_execute` method that orchestrates the entire AI conversation workflow.

## Quick Architecture Overview

```text
User Message → Security Validation → Session Context → Agent Processing → Database Storage → Response
     ↓              ↓                    ↓                ↓                 ↓              ↓
 HTTP Request   Client Ownership    Context Building   LangGraph      Message Storage   JSON Response
```

**Flow Summary**: 
1. **HTTP Request** arrives with user message and metadata
2. **Client Ownership** verified through database lookup  
3. **Context Building** creates comprehensive SessionContext
4. **LangGraph** processes message through AI pipeline
5. **Message Storage** persists conversation to DynamoDB
6. **JSON Response** streams back to user in real-time

## Endpoint Details

**Route**: `POST /conversations/{conversation_id}/messages`  
**Controller**: [conversation.py#L109](../chatbot_api/controllers/conversation.py#L109)  
**Service Method**: [conversation.py#L259](../chatbot_api/services/conversation.py#L259)  
**Core Engine**: [conversation.py#L346 (_execute)](../chatbot_api/services/conversation.py#L346)

## 📢 **Recent Changes Notice**

> **⚠️ Important**: This documentation reflects the current codebase version with updated line numbers. Key method locations have changed:
> - **Controller endpoint**: `send_message_to_agent` now at **L109** (previously L98)
> - **Service method**: `process_user_input` now at **L259** (previously L251)  
> - **Core engine**: `_execute` method now at **L346** (previously L322)
> 
> All line references in this document have been updated to match the current implementation.

## Request Flow Architecture

```
User Input → Validation → Security Check → Context Building → Core Execution Engine → Real-time Streaming → Database Storage
     ↓              ↓            ↓              ↓                    ↓                     ↓                ↓
  JSON Request → Headers → Last Interaction → SessionContext → _execute() Method → Server-Sent Events → DynamoDB
                                                     ↓
                                        LangGraph Agent + AWS Bedrock + Knowledge Base
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

```
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

### **Phase 1: Request Validation & Setup** - *[conversation.py#L109-L128](../chatbot_api/controllers/conversation.py#L109-L128)*

The controller layer is intentionally simple, following clean architecture principles by acting as a pure routing layer.

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

### **Phase 2: Database Status Validation** - *[conversation.py#L280-L289](../chatbot_api/services/conversation.py#L280-L289)*

This critical security check ensures conversation ownership and prevents unauthorized access.

1. **Retrieve Last Interaction** - *[conversation.py#L280](../chatbot_api/services/conversation.py#L280)*:
   ```python
   last_interaction = service_aws.get_conversation_last_interaction(conversation_id)
   ```

2. **Database Query Implementation** - *[aws.py#L114-L125](../chatbot_api/services/aws.py#L114-L125)*:
   ```python
   def get_conversation_last_interaction(self, conversation_id: str):
       """Retrieve the last interaction of a conversation."""
       response = self.dynamodb.query(
           TableName=INTERACTIONS_TABLE,                    # Environment variable: COMMONS.INTERACTIONS-TABLE
           KeyConditionExpression='conversationId = :conversationId',
           ExpressionAttributeValues={
               ':conversationId': {'S': conversation_id}   # DynamoDB String type
           },
           ScanIndexForward=False,                         # Descending order (latest first)
           Limit=1                                         # Only get the most recent interaction
       )
       if response.get('Items'):
           return response['Items'][0]                     # Return the latest interaction item
       return None                                         # No interactions found (new conversation)
   ```

3. **Client Authorization Check** - *[conversation.py#L281-L289](../chatbot_api/services/conversation.py#L281-L289)*:
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

### **Phase 3: SessionContext Creation** - *[conversation.py#L296-L331](../chatbot_api/services/conversation.py#L296-L331)*

1. **SessionContext Builder Call** - *[conversation.py#L296](../chatbot_api/services/conversation.py#L296)*:
   ```python
   session_context = self._create_session_context_builder(
       conversation_id,                           # From URL path parameter
       client_id,                                # From header
       send_message_request.input.options.stream, # From request options  
       session_id,                               # From header
       send_message_request.context.execution.interactionId  # From request or generated
   )
   ```

2. **SessionContext Construction** - *[conversation.py#L327-L345](../chatbot_api/services/conversation.py#L327-L345)*:
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

### **Phase 3: Core Execution Engine (`_execute` Method)** - *[conversation.py#L346-L616](../chatbot_api/services/conversation.py#L346-L616)*

The `_execute` method is the **heart of the entire conversation system**. It orchestrates the AI processing pipeline, handles real-time streaming, and manages database persistence. This method receives the prepared `SessionContext` and `SendSaveMessageRequest` from the `process_user_input` method and executes the complete conversation workflow.

#### **3.1 Method Signature & Initial Setup** - *[conversation.py#L346-L364]*

```python
async def _execute(
    self,
    session_context: SessionContext,    # Built context with IDs and metadata
    request: SendSaveMessageRequest,    # User input and options
) -> EventSourceResponse:              # Server-Sent Events stream
```

**Key Initialization Steps**:
```python
user_question = request.input.text                    # Extract user message
is_streaming = request.input.options.stream          # Real-time streaming flag
suggestions = request.input.options.suggestions      # Follow-up questions flag
session_context.turn_count = int(request.context.system.turnCount) + 1  # Conversation turn tracking
session_context.response_id = uuid.uuid4().hex       # Unique response identifier

agent = Agent(self.agent_config, suggestions)        # Initialize LangGraph agent
```

#### **3.2 Conversation History Reconstruction** - *[conversation.py#L350-L370]*

**Critical for Context Continuity**:
```python
# Load previous conversation from DynamoDB
previous_conversation_items = service_aws.get_conversation(conversation_id=session_context.conversation_id)
previsous_conversation = []

for item in previous_conversation_items:
    # Skip content blocked by AWS Guardrails
    if item.get('guardrailApplied', {}).get('BOOL', False):
        continue
    
    # Convert DynamoDB items to LangChain message format
    if item.get('author', {}).get('S', '') in ['human', 'retriever']:
        message = HumanMessage(item.get('message', {}).get('S', ''))
    else:
        message = AIMessage(item.get('message', {}).get('S', ''))
    
    previsous_conversation.append(message)
```

**Why This Matters**:
- **Context Preservation**: AI maintains conversation memory across turns
- **Message Chain**: Proper human/AI alternation for optimal LLM performance
- **Security Filtering**: Excludes content blocked by guardrails from context
- **Format Translation**: DynamoDB → LangChain message objects

#### **3.3 Agent State Construction & Telemetry** - *[conversation.py#L375-L395]*

**Build Complete Agent Input**:
```python
agent_input = {
    'messages': [SystemMessage(PROMPT)] + previsous_conversation + [HumanMessage(user_question)],
    'user_question': user_question,
}

# OpenTelemetry metadata for distributed tracing
metadata = {
    "san": {
        "conversation_id": session_context.conversation_id,
        "response_id": session_context.response_id,
        "interaction_id": session_context.interaction_id,
        "client_id": session_context.client_id,
    }
}
```

**Message Chain Structure**:
1. **System Prompt**: AI behavior instructions
2. **Conversation History**: Previous human/AI exchanges  
3. **Current Question**: User's new input
4. **Metadata**: Full tracing context for observability

#### **3.4 Core Agent Streaming Loop** - *[conversation.py#L396-L450]*

**The Central Processing Engine**:
```python
with capture_span_context() as capture:
    with using_metadata(metadata):
        async for type, value in agent.astream(
            agent_input,
            stream_mode=["messages", "values"]  # Stream both message chunks and node outputs
        ):
            # Process two types of streaming data:
            # 1. 'messages' - Real-time text chunks for streaming
            # 2. 'values' - Complete node outputs (contact center, deep links, etc.)
```

**Dual Stream Processing**:
- **`type == 'values'`**: Complete outputs from LangGraph nodes (classification, retrieval, suggestions)
- **`type == 'messages'`**: Incremental text chunks for real-time streaming

#### **3.5 Multi-Modal Response Processing**

**3.5.1 Contact Center Classification** - *[conversation.py#L398-L415]*:
```python
if not cc_answer and 'contact_center_answer' in value:
    cc_answer = value['contact_center_answer'].answer
    if cc_answer == 'AGENT':
        # Immediate transfer to human agent
        yield json.dumps(
            SendMessageToAgentResponse.build_transfer_to_agent_response(
                session_context=session_context
            ).model_dump(),
            ensure_ascii=False,
        )
        return  # Stop processing, transfer initiated
```

**3.5.2 Deep Links Processing** - *[conversation.py#L426-L436]*:
```python
if not deep_links and 'deep_links' in value and value['deep_links']:
    deep_links = value['deep_links']
    yield json.dumps(
        SendMessageToAgentResponse.build_deep_links_response(
            session_context=session_context, 
            deeplinks=deep_links, 
            state=state
        ).model_dump(),
        ensure_ascii=False,
    )
```

**3.5.3 Suggestions Processing** - *[conversation.py#L437-L447]*:
```python
if not suggestions_answer and 'suggestions' in value and value['suggestions'].followup_questions:
    suggestions_answer = value['suggestions']
    yield json.dumps(
        SendMessageToAgentResponse.build_suggestions_response(
            session_context=session_context, 
            suggestions=suggestions_answer.followup_questions, 
            state=state
        ).model_dump(),
        ensure_ascii=False,
    )
```

#### **3.6 Real-Time Text Streaming** - *[conversation.py#L451-L470]*

**Content Chunking & Streaming**:
```python
# Extract text content from various chunk formats
if isinstance(chunk.content, str):
    chunk = chunk.content
else:
    # Handle multi-part content (e.g., text + metadata)
    chunk = ''.join([c.get('text', '') if c.get('type','') == 'text' else '' for c in chunk.content])

total_message += chunk  # Accumulate complete response

# Stream only if Contact Center allows and streaming is enabled
if self.agent_config.DisableContactCenter or cc_answer:
    if chunk and is_streaming:
        yield json.dumps(
            SendMessageToAgentResponse.build_send_conversation_response(
                chunk, session_context, DELTA_STATE
            ).model_dump(),
            ensure_ascii=False,
        )
```

#### **3.7 Database Persistence Pipeline** - *[conversation.py#L473-L520]*

**3.7.1 User Message Storage**:
```python
service_aws.create_interaction(
    session_context=session_context,
    message=user_question,
    author="human",
    span_id=span_id,  # OpenTelemetry trace ID
    guardrail_applied=final_state.get('guardrail_applied', False)
)
```

**3.7.2 Document Citations Storage**:
```python
documents = final_state.get('documents', [])
if len(documents) > 0:
    citations = []
    document_content = ''
    for doc in documents:
        document_content += f"{doc.page_content}\n\n"
        citations.append((
            doc.metadata.get('source_metadata', {}).get('x-amz-bedrock-kb-source-uri', ''),
            doc.metadata.get('source_metadata', {}).get('x-amz-bedrock-kb-document-page-number', 0.0)
        ))
    
    # Store retrieved documents as separate interaction
    service_aws.create_interaction(
        session_context=session_context,
        message=service_aws.apply_mask_guardrail(GUARDRAIL_MASK_ID, GUARDRAIL_MASK_VERSION, document_content),
        author="retriever",
        span_id=span_id
    )
    
    # Stream citations to client
    yield json.dumps(
        SendMessageToAgentResponse.build_citations_response(
            session_context=session_context, citations=citations
        ).model_dump(),
        ensure_ascii=False,
    )
```

**3.7.3 AI Response Storage**:
```python
service_aws.create_interaction(
    session_context=session_context,
    message=service_aws.apply_mask_guardrail(GUARDRAIL_MASK_ID, GUARDRAIL_MASK_VERSION, total_message),
    author="ai",
    span_id=span_id,
    guardrail_applied=final_state.get('guardrail_applied', False)
)
```

#### **3.8 Response Finalization** - *[conversation.py#L510-L522]*

**Complete Stream Closure**:
```python
# Send final response if not streaming
if not is_streaming:
    yield json.dumps(
        SendMessageToAgentResponse.build_send_conversation_response(
            total_message, session_context, DELTA_STATE
        ).model_dump(),
        ensure_ascii=False,
    )

# Signal conversation completion
yield json.dumps(
    SendMessageToAgentResponse.build_send_conversation_response(
        "", session_context, END_STATE
    ).model_dump(),
    ensure_ascii=False,
)
```

#### **3.9 Error Handling & Recovery**

**Comprehensive Exception Management**:
```python
except Exception as e:
    logger.error(
        "Error en la generación de eventos para conversationId: %s",
        session_context.conversation_id,
        exc_info=e,
    )
    # Graceful degradation - stream continues with error logged
```

**Key Design Principles**:
- **Fail-Safe Streaming**: Errors don't crash the entire conversation
- **Complete Telemetry**: Full error context captured
- **Graceful Degradation**: Partial responses still delivered when possible

### **Phase 4: Database Status Update Mechanism** - *[aws.py#L49-L95](../chatbot_api/services/aws.py#L49-L95)*

The database persistence that occurs within the `_execute` method relies on sophisticated DynamoDB operations that maintain conversation integrity and sequence.

#### **4.1 Get Next Message ID**:
```python
def create_interaction(self, session_context: SessionContext, message: str, ...):
    # Reuse get_conversation_last_interaction to maintain sequence
    response = self.get_conversation_last_interaction(session_context.conversation_id)
    if response:
        max_message_id = int(response['messageId']['N'])    # Get current max ID
        topic = response.get('topic', {}).get('S', '')      # Preserve existing topic
    else:
        max_message_id = 0                                  # First interaction in conversation
        topic = None
```

#### **4.2 Build DynamoDB Item**:
```python
item = {
    'conversationId': {'S': session_context.conversation_id},  # Partition key
    'messageId': {'N': str(max_message_id + 1)},             # Sort key (auto-increment)
    'interactionId': {'S': session_context.interaction_id},   # Unique interaction ID
    'author': {'S': author},                                 # human, ai, retriever
    'message': {'S': message},                               # Actual message content
    'createdAt': {'S': datetime.now(timezone.utc).isoformat()},  # Timestamp
    'responseId': {'S': session_context.response_id},        # Response tracking
    'guardrailApplied': {'BOOL': guardrail_applied},         # Content safety flag
    'sessionId': {'S': session_context.session_id}          # Session reference
}
```

#### **4.3 Conditional Data Enrichment**:
```python
if topic:
    item['topic'] = {'S': topic}                   # Preserve conversation topic
if session_context.client_id:
    item['clientId'] = {'S': session_context.client_id}  # Client ownership

# Enrich with session analytics data
if session_context.session_id:
    session_data = self.get_session(session_context.session_id)
    if session_data:
        item['browser'] = session_data['browser']    # User browser info
        item['device'] = session_data['device']      # Device type
        item['pageUrl'] = session_data['pageUrl']    # Source page
        item['channel'] = session_data['channel']    # Communication channel
```

#### **4.4 Execute Database Write**:
```python
self.dynamodb.put_item(
    TableName=INTERACTIONS_TABLE,     # COMMONS.INTERACTIONS-TABLE env var
    Item=item                         # Complete interaction record
)
```

## Phase 6: Complete Database Table Structure

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

- **AWS Bedrock**: Foundation model inference
- **AWS Knowledge Base**: Document retrieval for RAG
- **AWS DynamoDB**: Conversation persistence
- **AWS Guardrails**: Content safety validation
- **Phoenix Telemetry**: Observability and monitoring

## Security Features

- **Header Validation**: Session and client ID verification
- **Content Filtering**: Guardrail-based safety checks
- **Input Sanitization**: Message content validation
- **Span Tracking**: Request correlation for security auditing
