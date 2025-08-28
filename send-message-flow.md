# Send Message to Agent - Complete Flow Documentation

## Overview

The `send_message_to_agent` endpoint is the core functionality of the chatbot API. It processes user messages through a sophisticated AI pipeline using LangGraph, AWS Bedrock, and real-time streaming responses.

## Endpoint Details

**Route**: `POST /conversations/{conversation_id}/messages`  
**Controller**: [conversation.py#L98](../chatbot_api/controllers/conversation.py#L98)  
**Service Method**: [conversation.py#L300](../chatbot_api/services/conversation.py#L300)

## Request Flow Architecture

```
User Input → Validation → DB Status Check → LangGraph Processing → Real-time Streaming → Database Storage
     ↓              ↓            ↓                ↓                     ↓                ↓
  JSON Request → Headers → Last Interaction → Agent Workflow → Server-Sent Events → DynamoDB
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

### **Phase 3: Conversation History Loading** - *[conversation.py#L360-L375](../chatbot_api/services/conversation.py#L360-L375)*

1. **Retrieve Previous Messages**:
   ```python
   previous_conversation_items = service_aws.get_conversation(conversation_id)
   ```

2. **Build Message Chain**:
   ```python
   previsous_conversation = []
   for item in previous_conversation_items:
       if item.get('guardrailApplied', {}).get('BOOL', False):
           continue  # Skip blocked content
       
       if item.get('author', {}).get('S', '') in ['human', 'retriever']:
           message = HumanMessage(item.get('message', {}).get('S', ''))
       else:
           message = AIMessage(item.get('message', {}).get('S', ''))
       
       previsous_conversation.append(message)
   ```

### **Phase 4: Agent Processing & LangGraph Execution** - *[conversation.py#L330-L390](../chatbot_api/services/conversation.py#L330-L390)*

1. **Agent State Construction**:
   ```python
   agent_input = {
       'messages': [SystemMessage(PROMPT)] + previsous_conversation + [HumanMessage(user_question)],
       'user_question': user_question,
   }
   ```

2. **Streaming Agent Execution**:
   ```python
   async for type, value in agent.astream(agent_input, stream_mode=["messages", "values"]):
   ```

3. **Telemetry Metadata with SessionContext** - *[conversation.py#L376-L385](../chatbot_api/services/conversation.py#L376-L385)*:
   ```python
   metadata = {
       "san": {
           "conversation_id": session_context.conversation_id,
           "response_id": session_context.response_id,
           "interaction_id": session_context.interaction_id,
           "client_id": session_context.client_id,
       }
   }
   with using_metadata(metadata):
       # Agent processing with full context tracking
   ```

4. **Multi-Modal Processing**:
   - **Contact Center Classification**: Determines if human agent needed
   - **Document Retrieval**: RAG from Knowledge Base
   - **Response Generation**: LLM content creation
   - **Suggestions Generation**: Follow-up questions
   - **Deep Links**: Related resources

### **Phase 5: Real-Time Response Streaming** - *[conversation.py#L390-L470](../chatbot_api/services/conversation.py#L390-L470)*

#### **5.1 Contact Center Classification**
```python
if 'contact_center_answer' in value:
    cc_answer = value['contact_center_answer'].answer
    if cc_answer == 'AGENT':
        # Transfer to human agent
        yield SendMessageToAgentResponse.build_transfer_to_agent_response()
        return
```

#### **5.2 Content Streaming**
```python
if isinstance(chunk.content, str):
    chunk = chunk.content
else:
    chunk = ''.join([c.get('text', '') for c in chunk.content])

total_message += chunk

if chunk and is_streaming:
    yield SendMessageToAgentResponse.build_send_conversation_response(
        chunk, session_context, DELTA_STATE
    )
```

#### **5.3 Deep Links Processing**
```python
if 'deep_links' in value and value['deep_links']:
    yield SendMessageToAgentResponse.build_deep_links_response(
        session_context, deep_links, state
    )
```

#### **5.4 Suggestions Processing**
```python
if 'suggestions' in value and value['suggestions'].followup_questions:
    yield SendMessageToAgentResponse.build_suggestions_response(
        session_context, suggestions, state
    )
```

### **Phase 6: Database Persistence** - *[conversation.py#L465-L505](../chatbot_api/services/conversation.py#L465-L505)*

1. **Store User Message with SessionContext**:
   ```python
   service_aws.create_interaction(
       session_context=session_context,  # Full context passed to AWS service
       message=user_question,
       author="human",
       span_id=span_id,
       guardrail_applied=final_state.get('guardrail_applied', False)
   )
   ```

2. **Database Status Update Mechanism** - *[aws.py#L49-L95](../chatbot_api/services/aws.py#L49-L95)*:

   **2.1 Get Next Message ID**:
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

   **2.2 Build DynamoDB Item**:
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

   **2.3 Conditional Data Enrichment**:
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

   **2.4 Execute Database Write**:
   ```python
   self.dynamodb.put_item(
       TableName=INTERACTIONS_TABLE,     # COMMONS.INTERACTIONS-TABLE env var
       Item=item                         # Complete interaction record
   )
   ```

3. **SessionContext Usage in AWS Service** - *[aws.py#L55-L85](../chatbot_api/services/aws.py#L55-L85)*:
   ```python
   def create_interaction(self, session_context: SessionContext, message: str, ...):
       item = {
           'conversationId': {'S': session_context.conversation_id},
           'interactionId': {'S': session_context.interaction_id},
           'responseId': {'S': session_context.response_id},
           'sessionId': {'S': session_context.session_id}
       }
       
       if session_context.client_id:
           item['clientId'] = {'S': session_context.client_id}
           
       # Enrich with session data using session_id from context
       session_data = self.get_session(session_context.session_id)
   ```

4. **Store AI Response**:
   ```python
   service_aws.create_interaction(
       session_context=session_context,  # Same context for response tracking
       message=service_aws.apply_mask_guardrail(total_message),
       author="ai",
       span_id=span_id,
       guardrail_applied=final_state.get('guardrail_applied', False)
   )
   ```

### **Phase 7: Response Completion** - *[conversation.py#L505-L515](../chatbot_api/services/conversation.py#L505-L515)*

1. **Final Event**:
   ```python
   yield SendMessageToAgentResponse.build_send_conversation_response(
       "", session_context, END_STATE
   )
   ```

2. **Server-Sent Events Return**:
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
