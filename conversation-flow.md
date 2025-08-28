# Complete Conversation Flow

## Overview

This document provides a comprehensive understanding of how conversations flow through the IA MB API Chatbot system, from initial user message to AI response delivery. It covers both the technical implementation and the database interaction patterns.

## Flow Architecture

```text
User Message → Security Validation → Session Context → Agent Processing → Database Storage → Response
     ↓              ↓                    ↓                ↓                 ↓              ↓
 HTTP Request   Client Ownership    Context Building   LangGraph      Message Storage   JSON Response
```

## Phase 1: Request Reception & Security

### 1.1 Message Arrival

```python
# FastAPI endpoint receives user message
@router.post("/conversations/{conversation_id}/messages")
async def send_message_to_agent(
    conversation_id: str,
    request: SendSaveMessageRequest,
    client_id: str = Depends(extract_client_id)
):
```

**Input Validation:**

- **conversation_id**: UUID format validation
- **message**: Non-empty text content
- **client_id**: Extracted from JWT token
- **session_id**: Optional session context

### 1.2 Ownership Verification

```python
# Check if user owns this conversation
last_interaction = aws_service.get_conversation_last_interaction(conversation_id)

if last_interaction and last_interaction.get('clientId', {}).get('S') != client_id:
    raise HTTPException(status_code=403, detail="Access denied")
```

**Security Logic:**

- **New Conversation**: No ownership check required
- **Existing Conversation**: Must match clientId from last interaction
- **Access Denied**: HTTP 403 if ownership fails

## Phase 2: Session Context Creation

### 2.1 Context Builder Initialization

```python
# Create enriched session context
session_context_builder = conversation_service._create_session_context_builder(
    conversation_id=conversation_id,
    client_id=client_id,
    session_id=request.session_id
)
```

### 2.2 Session Data Enrichment

**Session Lookup Process:**

```python
def _create_session_context_builder(self, conversation_id, client_id, session_id=None):
    builder = SessionContextBuilder()
    builder.conversation_id = conversation_id
    builder.client_id = client_id
    builder.session_id = session_id
    
    # Enrich with session analytics data
    if session_id:
        session = self.session_service.get_session(session_id)
        if session:
            builder.browser = session.get('browser', {}).get('S')
            builder.device = session.get('device', {}).get('S')
            builder.page_url = session.get('pageUrl', {}).get('S')
            builder.channel = session.get('channel', {}).get('S')
    
    return builder
```

**Generated Context:**

- **Unique IDs**: interaction_id, response_id (UUID4)
- **Core Data**: conversation_id, client_id, session_id
- **Analytics**: browser, device, pageUrl, channel
- **Timestamps**: Created automatically during processing

## Phase 3: Database Status Management

### 3.1 Conversation State Detection

```python
# Check conversation existence and get auto-increment ID
def get_conversation_last_interaction(conversation_id: str):
    response = dynamodb.query(
        TableName=INTERACTIONS_TABLE,
        KeyConditionExpression='conversationId = :id',
        ExpressionAttributeValues={':id': {'S': conversation_id}},
        ScanIndexForward=False,    # Latest message first
        Limit=1
    )
    return response.get('Items', [None])[0]
```

### 3.2 Message ID Generation

```python
# Auto-increment logic for chronological ordering
def get_next_message_id(conversation_id: str):
    last_interaction = get_conversation_last_interaction(conversation_id)
    if last_interaction:
        return int(last_interaction['messageId']['N']) + 1
    return 1  # First message in new conversation
```

**Database State Patterns:**

- **New Conversation**: messageId starts at 1
- **Continuing Conversation**: messageId = last_messageId + 1
- **Chronological Order**: Messages always increment (1, 2, 3, 4...)

## Phase 4: User Message Storage

### 4.1 Immediate Storage

```python
# Store user message before AI processing
session_context = session_context_builder.build()

aws_service.create_interaction(
    session_context=session_context,
    message=request.message,
    author="human",
    span_id=span_id
)
```

### 4.2 Database Item Structure

```json
{
  "conversationId": {"S": "conv-uuid"},
  "messageId": {"N": "1"},
  "interactionId": {"S": "interaction-uuid"},
  "author": {"S": "human"},
  "message": {"S": "User's actual message"},
  "createdAt": {"S": "2025-08-28T10:30:00Z"},
  "responseId": {"S": "response-uuid"},
  "clientId": {"S": "client-identifier"},
  "sessionId": {"S": "session-uuid"},
  "guardrailApplied": {"BOOL": false},
  
  // Enriched from session
  "browser": {"S": "Chrome 120.0"},
  "device": {"S": "Desktop"},
  "pageUrl": {"S": "https://bank.com/chat"},
  "channel": {"S": "web"}
}
```

## Phase 5: Agent Processing

### 5.1 LangGraph Orchestration

```python
# Send to AI agent for processing
agent_response = agent_service.process_user_input(
    session_context=session_context,
    user_input=request.message
)
```

**Agent Processing Steps:**

1. **Context Loading**: Retrieve conversation history from database
2. **Prompt Construction**: Build system prompts with user context
3. **RAG Integration**: Query knowledge base if needed
4. **LLM Invocation**: Send to AWS Bedrock (Claude models)
5. **Guardrail Validation**: Check content safety policies
6. **Response Generation**: Format final response

### 5.2 Context-Aware Processing

```python
# Agent loads full conversation for context
conversation_history = aws_service.get_conversation(conversation_id)

# Build chronological message sequence
messages = []
for interaction in conversation_history:
    messages.append({
        'author': interaction['author']['S'],
        'content': interaction['message']['S'],
        'timestamp': interaction['createdAt']['S']
    })
```

## Phase 6: AI Response Storage

### 6.1 Response Persistence

```python
# Store AI response with matching responseId
aws_service.create_interaction(
    session_context=session_context,
    message=agent_response.content,
    author="ai",
    span_id=span_id
)
```

### 6.2 Linked Message Pairs

**Database Pattern:**

```text
MessageId=1: Human message (responseId=uuid-1)
MessageId=2: AI response   (responseId=uuid-1)  # Same responseId links them
MessageId=3: Human message (responseId=uuid-2)
MessageId=4: AI response   (responseId=uuid-2)  # Next pair
```

## Phase 7: Response Delivery

### 7.1 Response Construction

```python
# Build final API response
return SendMessageToAgentResponse(
    response_id=session_context.response_id,
    content=agent_response.content,
    conversation_id=conversation_id,
    interaction_id=session_context.interaction_id
)
```

### 7.2 Client Response Format

```json
{
  "response_id": "uuid-identifying-this-qa-pair",
  "content": "AI generated response text",
  "conversation_id": "conversation-uuid",
  "interaction_id": "ai-message-interaction-uuid"
}
```

## Error Handling & Edge Cases

### Security Failures

```python
# Ownership validation failure
if client_mismatch:
    raise HTTPException(status_code=403, detail="Access denied")
```

### Database Failures

```python
# Handle DynamoDB errors gracefully
try:
    aws_service.create_interaction(...)
except Exception as e:
    logger.error(f"Database error: {e}")
    raise HTTPException(status_code=500, detail="Storage error")
```

### Agent Processing Failures

```python
# LangGraph or Bedrock failures
try:
    response = agent_service.process_user_input(...)
except Exception as e:
    logger.error(f"Agent error: {e}")
    return generic_error_response()
```

## Performance Characteristics

### Database Access Patterns

- **Hot Path**: get_conversation_last_interaction (every message)
- **Warm Path**: get_conversation (context loading)
- **Write Path**: create_interaction (twice per exchange)

### Latency Profile

1. **Security Check**: ~10-20ms (single DynamoDB query)
2. **Context Building**: ~5-10ms (session lookup if needed)
3. **User Storage**: ~20-30ms (DynamoDB write)
4. **Agent Processing**: ~2-5 seconds (LLM inference)
5. **Response Storage**: ~20-30ms (DynamoDB write)
6. **Total**: ~2.1-5.1 seconds typical

### Scaling Bottlenecks

- **LLM Inference**: Primary latency source
- **Database Writes**: Auto-scaling handles load
- **Memory Usage**: SessionContext objects are lightweight

## Monitoring & Observability

### Key Metrics to Track

- **Message Processing Time**: End-to-end latency
- **Database Operation Latency**: DynamoDB response times
- **Agent Processing Time**: LangGraph execution duration
- **Error Rates**: Failed messages by error type

### Distributed Tracing

```python
# Correlation across all operations
span_id = generate_span_id()
# Used in: database writes, agent calls, logging
```

## Integration Points

### Frontend Integration

```javascript
// Client-side conversation management
POST /conversations/{conversationId}/messages
{
    "message": "User input text",
    "session_id": "session-uuid"  // Optional
}
```

### Analytics Integration

```python
# Session data flows to all interactions
session_analytics = {
    "browser": "Chrome 120.0",
    "device": "Desktop", 
    "pageUrl": "https://bank.com/chat",
    "channel": "web"
}
```

### Knowledge Base Integration

```python
# RAG queries during agent processing
if knowledge_needed:
    context = knowledge_base.query(user_message)
    enhanced_prompt = build_prompt_with_context(user_message, context)
```

This flow ensures reliable, secure, and performant conversation management while maintaining complete audit trails and enabling rich analytics.
