[← Back to Documentation Home](README.md)

# Send Message to Agent - Complete Flow Documentation

## Overview

The `send_message_to_agent` endpoint is the core functionality of the chatbot API. It processes user messages through a sophisticated AI pipeline using LangGraph, AWS Bedrock, and real-time streaming responses.

**High-Level Flow**: The endpoint receives user input, identifiers, and request parameters. It validates security by looking up the last interaction, builds a comprehensive SessionContext, and then passes both the SendSaveMessageRequest and SessionContext to the core `_execute` method that orchestrates the entire AI conversation workflow.

## Endpoint Details

**Route**: `POST /conversations/{conversation_id}/messages`  
**Controller**: [conversation.py#L98](../chatbot_api/controllers/conversation.py#L98)  
**Service Method**: [conversation.py#L251](../chatbot_api/services/conversation.py#L251)  
**Core Engine**: [conversation.py#L322 (_execute)](../chatbot_api/services/conversation.py#L322)

## Request Flow Architecture

```
User Input → Validation → Security Check → Context Building → Core Execution Engine → Real-time Streaming → Database Storage
     ↓              ↓            ↓              ↓                    ↓                     ↓                ↓
  JSON Request → Headers → Last Interaction → SessionContext → _execute() Method → Server-Sent Events → DynamoDB
                                                     ↓
                                        LangGraph Agent + AWS Bedrock + Knowledge Base
```

## Data Types & Models

### 1. **Request Model** - *[send_save_message_request.py](../chatbot_api/domain/requests/send_save_message_request.py)*

```python
class SendSaveMessageRequest:
    input: Input                    # Message content and options
    system: System                  # Channel, timezone, locale info
    execution: Execution            # Session context identifiers

class Input:
    messageType: str               # Type of message
    text: str                      # User's actual message
    options: Options               # Stream and suggestions flags

class Options:
    stream: bool                   # Enable real-time streaming
    suggestions: bool              # Generate follow-up questions
```

### 2. **Response Model** - *[send_message_to_agent_response.py](../chatbot_api/domain/responses/send_message_to_agent_response.py)*

```python
# Response states for streaming
INITIAL_STATE = "init"      # First chunk of response
DELTA_STATE = "delta"       # Incremental content chunks  
END_STATE = "end"           # Response completion signal

# Response types
TYPE_TO_AGENT = "toAgent"   # Transfer to human agent
```

## Complete Processing Flow

### **Phase 1: Request Validation & Setup** - *[conversation.py#L98-L120](../chatbot_api/controllers/conversation.py#L98-L120)*

1. **Input Validation**:
   ```python
   if not request.input.text:
       raise HTTPException(status_code=400, detail="Message text cannot be empty")
   ```

2. **Header Extraction** - *[header_validator.py](../chatbot_api/controllers/header_validator.py)*:
   ```python
   # FastAPI Depends() automatically calls these functions
   session_id = get_session_id()    # Extracts X-LZVA-SESSION-ID header
   client_id = get_client_id()      # Extracts x-santander-client-id header
   ```

3. **Service Layer Call** - *[conversation.py#L115-L125](../chatbot_api/controllers/conversation.py#L115-L125)*:
   ```python
   # Controller passes individual parameters to service
   stream = await conversation_service.process_user_input(
       client_id, session_id, conversation_id, request
   )
   ```

### **Phase 2: Database Status Validation** - *[conversation.py#L272-L281](../chatbot_api/services/conversation.py#L272-L281)*

This critical security check ensures conversation ownership and prevents unauthorized access.

1. **Retrieve Last Interaction** - *[conversation.py#L272](../chatbot_api/services/conversation.py#L272)*:
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

3. **Client Authorization Check** - *[conversation.py#L273-L281](../chatbot_api/services/conversation.py#L273-L281)*:
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

### **Phase 3: SessionContext Creation** - *[conversation.py#L283-L320](../chatbot_api/services/conversation.py#L283-L320)*

1. **SessionContext Builder Call** - *[conversation.py#L283](../chatbot_api/services/conversation.py#L283)*:
   ```python
   session_context = self._create_session_context_builder(
       conversation_id,                           # From URL path parameter
       client_id,                                # From header
       send_message_request.input.options.stream, # From request options  
       session_id,                               # From header
       send_message_request.context.execution.interactionId  # From request or generated
   )
   ```

2. **SessionContext Construction** - *[conversation.py#L311-L320](../chatbot_api/services/conversation.py#L311-L320)*:
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

### **Phase 3: Core Execution Engine (`_execute` Method)** - *[conversation.py#L322-L522](../chatbot_api/services/conversation.py#L322-L522)*

The `_execute` method is the **heart of the entire conversation system**. It orchestrates the AI processing pipeline, handles real-time streaming, and manages database persistence. This method receives the prepared `SessionContext` and `SendSaveMessageRequest` from the `process_user_input` method and executes the complete conversation workflow.

#### **3.1 Method Signature & Initial Setup** - *[conversation.py#L322-L340]*

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

### **Phase 5: Response Completion** - *[conversation.py#L505-L515](../chatbot_api/services/conversation.py#L505-L515)*

#### **5.1 Final Event**:
```python
yield SendMessageToAgentResponse.build_send_conversation_response(
    "", session_context, END_STATE
)
```

#### **5.2 Server-Sent Events Return**:
```python
return EventSourceResponse(
    event_generator(),
    media_type="text/event-stream"
)
```

## SessionContext Complete Journey

### **Creation Flow:**
```
1. Controller Layer (conversation.py#L98-L125):
   ├── FastAPI extracts headers: session_id, client_id
   ├── Gets conversation_id from URL path  
   └── Passes individual parameters to service

2. Service Layer (conversation.py#L283):
   ├── Calls _create_session_context_builder()
   ├── Combines all parameters into SessionContext
   └── Generates new UUIDs for interaction_id, response_id

3. SessionContext Usage Throughout Flow:
   ├── Telemetry metadata (conversation.py#L376-L385)
   ├── SSE response events (all streaming chunks)
   ├── DynamoDB persistence (aws.py#L55-L85)
   └── Error tracking and logging
```

### **SessionContext Structure:**
```python
SessionContext:
├── session_id          # From X-LZVA-SESSION-ID header
├── conversation_id     # From URL path parameter  
├── client_id           # From x-santander-client-id header
├── interaction_id      # Generated UUID for this message exchange
├── response_id         # Generated UUID for AI response
├── zone_id             # UTC timezone
├── session_start_time  # Current timestamp
├── is_streaming        # From request options
└── virtual_agent_id    # LuzIA_{ENVIRONMENT}
```

1. **INITIAL_STATE**: First response chunk
2. **DELTA_STATE**: Incremental content updates
3. **END_STATE**: Response completion signal
4. **Deep Links**: Related resource references
5. **Suggestions**: Follow-up question options
6. **Citations**: Source document references
7. **Transfer to Agent**: Human handoff signal

### **Event Format**:
```json
{
  "type": "delta",
  "responseId": "abc123",
  "conversationId": "conv456", 
  "interactionId": "int789",
  "value": {
    "input": {
      "text": "Response chunk content"
    }
  }
}
```

## Error Handling

### **Validation Errors**:
- Empty message text → 400 Bad Request
- Invalid session → Session validation error

### **Processing Errors**:
- LangGraph failures → 409 Conflict
- AWS service errors → Logged and handled gracefully
- Guardrail violations → Content blocked and logged

## Database Flow Summary

### **Key Database Operations in send_message_to_agent:**

1. **Initial Status Check**:
   - **Method**: `get_conversation_last_interaction(conversation_id)`
   - **Purpose**: Validates conversation ownership and prevents unauthorized access
   - **Query**: DynamoDB query with `ScanIndexForward=False, Limit=1` to get latest interaction
   - **Security**: Compares `clientId` from last interaction with current request client

2. **Message ID Management**:
   - **Auto-increment**: Each new interaction gets `max_message_id + 1`
   - **Ordering**: Maintains chronological message sequence within conversations
   - **Consistency**: Uses last interaction lookup to ensure no gaps in numbering

3. **Status Updates During Processing**:
   - **User Message**: Stored immediately after validation with `author="human"`
   - **AI Response**: Stored after complete LangGraph processing with `author="ai"`
   - **Retriever Data**: Stored for RAG documents with `author="retriever"`
   - **Topic Persistence**: Maintains conversation topic across all interactions

4. **Data Enrichment Process**:
   - **Session Context**: All interactions linked to SessionContext for tracking
   - **Client Metadata**: Browser, device, page URL, channel from session data
   - **Telemetry Integration**: Span IDs for distributed tracing
   - **Guardrail Status**: Content safety validation results

### **Update Mechanisms Available:**

- **`create_interaction()`**: Adds new messages/responses to conversation
- **`update_interaction()`**: Modifies existing interaction properties (e.g., topic)
- **Topic Management**: Can add/remove conversation topics dynamically
- **Conversation Deletion**: Removes topic from all interactions (soft delete)

## Performance Considerations

1. **Streaming**: Real-time user experience with Server-Sent Events
2. **Memory Management**: Conversation history loading and LangGraph state
3. **Database Efficiency**: Batched writes and optimized queries
4. **Caching**: Session context and configuration caching
5. **Monitoring**: Distributed tracing with span IDs

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
