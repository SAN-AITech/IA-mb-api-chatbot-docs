# Database Design & Data Management

## DynamoDB Tables Structure

### 1. **Sessions Table**

**Purpose**: Store user session metadata and analytics

**Schema** - *Ver implementación en [aws.py#L28-L36](../chatbot_api/services/aws.py#L28-L36)*:
```json
{
  "sessionId": "string (Partition Key)",
  "createdAt": "ISO timestamp",
  "browser": "string",
  "device": "string", 
  "pageUrl": "string",
  "channel": "string",
  "clientId": "string"
}
```

**Access Patterns** - *Ver queries en [aws.py#L42-L46](../chatbot_api/services/aws.py#L42-L46)*:

- Get session by sessionId
- Analytics tracking by client properties

### 2. **Interactions Table**

**Purpose**: Store individual chat messages and responses

**Schema** - *Ver implementación en [aws.py#L55-L85](../chatbot_api/services/aws.py#L55-L85)*:
```json
{
  "conversationId": "string (Partition Key)",
  "messageId": "number (Sort Key)",
  "interactionId": "string (unique)",
  "author": "human | assistant",
  "message": "string (content)",
  "createdAt": "ISO timestamp",
  "responseId": "string",
  "guardrailApplied": "boolean",
  "sessionId": "string",
  "topic": "string (optional)",
  "clientId": "string (optional)",
  "browser": "string",
  "device": "string",
  "pageUrl": "string", 
  "channel": "string",
  "spanId": "string (optional)"
}
```

**Access Patterns** - *Ver queries en [aws.py#L95-L110](../chatbot_api/services/aws.py#L95-L110)*:

- Get conversation by conversationId (with messageId ordering)
- Get last interaction for message sequencing
- Analytics queries by session/client properties

## Data Flow Patterns

### 1. **Session Creation** - *[Ver código](../chatbot_api/services/aws.py#L27)*

```python
aws_service.create_session(session_id, analytics, client_id)
```

### 2. **Message Storage** - *[Ver código](../chatbot_api/services/aws.py#L50)*

```python
# Auto-increment messageId based on conversation
last_interaction = aws_service.get_conversation_last_interaction(conversation_id)
new_message_id = last_interaction.messageId + 1

aws_service.create_interaction(
    session_context,
    message_content,
    author="human|assistant"
)
```

### 3. **Conversation Retrieval** - *[Ver código](../chatbot_api/services/aws.py#L95)*

```python
# Paginated retrieval for large conversations
conversations = aws_service.get_conversation(conversation_id)
# Returns ordered list by messageId
```

## Data Consistency

### 1. **Message Ordering**
- MessageId ensures chronological order within conversations
- Auto-increment prevents race conditions
- CreatedAt provides temporal backup ordering

### 2. **Session Linking**
- Every interaction linked to session via sessionId
- Session metadata copied to interactions for analytics
- Client context preserved across conversation

### 3. **Conversation Threading**
- ConversationId groups related messages
- InteractionId provides unique message identification
- ResponseId links user questions to AI responses

## Analytics & Monitoring

### 1. **User Tracking**
- Browser, device, pageUrl captured per session
- Channel tracking for multi-platform support
- ClientId for user identification across sessions

### 2. **Performance Monitoring**
- SpanId for distributed tracing
- CreatedAt timestamps for latency analysis
- GuardrailApplied for content safety tracking

### 3. **Topic Classification**
- Topic field for conversation categorization
- Enables content analysis and routing decisions
- Supports business intelligence queries

## Data Lifecycle

1. **Session Start**: Session record created with analytics
2. **Message Exchange**: Interactions stored with full context
3. **Conversation Growth**: Messages accumulate under conversationId
4. **Session End**: No explicit cleanup (retained for analytics)
5. **Data Retention**: Configurable via DynamoDB TTL (if implemented)
