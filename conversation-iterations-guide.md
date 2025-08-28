# Conversation Iterations & Database Flow Guide

## Overview

This document provides a complete understanding of how conversation iterations work in the chatbot system, from user input to database storage. It combines the conversation flow, database design, and exploration techniques into a single reference guide.

## What is a Conversation Iteration?

### Definition
A **conversation iteration** is one complete turn in a chat conversation, consisting of:

1. **User Input** → Stored as `author="human"`
2. **AI Processing** → LangGraph workflow execution
3. **AI Response** → Stored as `author="ai"`
4. **Optional RAG Data** → Stored as `author="retriever"` (when Knowledge Base is used)

### Iteration Lifecycle

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   User Message  │───▶│  AI Processing  │───▶│   AI Response   │
│   messageId: 1  │    │   (LangGraph)   │    │   messageId: 3  │
│  author: human  │    │                 │    │   author: ai    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       ▲
         │              ┌─────────────────┐              │
         │              │ Retriever Data  │──────────────┘
         │              │   messageId: 2  │
         │              │ author:retriever│
         │              └─────────────────┘
         │                       │
         ▼                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                    DynamoDB Storage                             │
│  conversationId: "conv-123"                                     │
│  ├── messageId: 1, author: "human", message: "What's my balance?"│
│  ├── messageId: 2, author: "retriever", message: "Account data..." │
│  └── messageId: 3, author: "ai", message: "Your balance is $1,234" │
└─────────────────────────────────────────────────────────────────┘
```

### Turn Count Management

```python
# In conversation.py - Line 342
session_context.turn_count = int(request.context.system.turnCount) + 1
```

Each iteration increments the turn count, allowing the system to track conversation length and apply policies based on interaction depth.

## Database Storage Pattern

### Message ID Sequencing

Every message gets an auto-incremented `messageId` within a conversation:

```python
# From aws.py - Lines 55-62
response = self.get_conversation_last_interaction(session_context.conversation_id)
if response:
    max_message_id = int(response['messageId']['N'])    # Get current maximum
    topic = response.get('topic', {}).get('S', '')      # Preserve topic
else:
    max_message_id = 0                                  # New conversation
    topic = None

# New message gets: max_message_id + 1
```

### Iteration Security Check

Before each iteration, the system validates conversation ownership:

```python
# From conversation.py - Lines 272-281
last_interaction = service_aws.get_conversation_last_interaction(conversation_id)
if last_interaction and last_interaction.get('clientId', {}).get('S', '') != client_id:
    logger.warning("Client ID mismatch - unauthorized access attempt")
    return Response(status_code=status.HTTP_403_FORBIDDEN)
```

**Database Query for Security Check:**
```python
# From aws.py - Lines 114-125
def get_conversation_last_interaction(self, conversation_id: str):
    response = self.dynamodb.query(
        TableName=INTERACTIONS_TABLE,
        KeyConditionExpression='conversationId = :conversationId',
        ExpressionAttributeValues={':conversationId': {'S': conversation_id}},
        ScanIndexForward=False,    # Descending order (latest first)
        Limit=1                    # Only get the most recent
    )
    return response.get('Items', [None])[0]
```

## Complete Iteration Data Flow

### Phase 1: Iteration Setup
```python
# Controller extracts headers and validates input
session_id = get_session_id()     # From X-LZVA-SESSION-ID header
client_id = get_client_id()       # From x-santander-client-id header
conversation_id = path_parameter   # From URL: /conversations/{id}/messages
```

### Phase 2: Security & Ownership Validation
```python
# Check conversation ownership using last interaction
last_interaction = get_conversation_last_interaction(conversation_id)
validate_client_ownership(last_interaction, client_id)
```

### Phase 3: SessionContext Creation
```python
# Build context for this iteration
session_context = SessionContext(
    conversation_id=conversation_id,
    client_id=client_id,
    session_id=session_id,
    interaction_id=uuid.uuid4(),      # Unique per iteration
    response_id=uuid.uuid4(),         # Unique per AI response
    turn_count=previous_count + 1     # Iteration counter
)
```

### Phase 4: Message Storage Pattern
```python
# Store user message (iteration start)
create_interaction(
    session_context=session_context,
    message=user_question,
    author="human",
    span_id=telemetry_span_id
)

# Store retriever data (if RAG is used)
create_interaction(
    session_context=session_context,
    message=retrieved_documents,
    author="retriever",
    span_id=telemetry_span_id
)

# Store AI response (iteration completion)
create_interaction(
    session_context=session_context,
    message=ai_response,
    author="ai",
    span_id=telemetry_span_id
)
```

## Database Table Structure

### INTERACTIONS_TABLE (Main Storage)

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

### SESSIONS_TABLE (Metadata Storage)

```python
{
    'sessionId': {'S': 'session-uuid'},               # Primary key
    'createdAt': {'S': '2025-08-28T10:00:00Z'},      # Session start
    'clientId': {'S': 'client-identifier'},           # User reference
    'browser': {'S': 'Chrome 120.0'},                 # Browser info
    'device': {'S': 'Desktop'},                       # Device info
    'pageUrl': {'S': 'https://bank.com/chat'},        # Entry point
    'channel': {'S': 'web'}                           # Channel
}
```

### Secondary Indexes

**clientId-conversationId-index:**
- **Purpose**: Find all conversations for a specific user
- **Partition Key**: `clientId`
- **Sort Key**: `conversationId`
- **Filter**: `attribute_exists(topic)` (only saved conversations)

## Query Patterns for Iterations

### 1. Get Conversation History (All Iterations)
```python
def get_conversation(conversation_id: str):
    response = dynamodb.query(
        TableName=INTERACTIONS_TABLE,
        KeyConditionExpression='conversationId = :id',
        ExpressionAttributeValues={':id': {'S': conversation_id}},
        ScanIndexForward=True    # Chronological order
    )
    return response.get('Items', [])
```

### 2. Get Latest Iteration (Security Check)
```python
def get_conversation_last_interaction(conversation_id: str):
    response = dynamodb.query(
        TableName=INTERACTIONS_TABLE,
        KeyConditionExpression='conversationId = :id',
        ExpressionAttributeValues={':id': {'S': conversation_id}},
        ScanIndexForward=False,  # Reverse chronological
        Limit=1                  # Only latest
    )
    return response.get('Items', [None])[0]
```

### 3. Get User's Conversation List
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

## Iteration Performance Characteristics

### Write Patterns
- **Hot Partitions**: Active conversations get frequent writes
- **Sequential Writes**: messageId auto-increment creates predictable access
- **Atomic Operations**: Each message write is independent

### Read Patterns
- **Latest First**: Security checks read most recent interaction
- **Full History**: Conversation loading reads entire history
- **User Lists**: Index queries for conversation discovery

### Scaling Considerations
```python
# Auto-scaling based on:
# - Read Capacity: Conversation history requests
# - Write Capacity: New message frequency
# - Storage: Message content and metadata growth
# - Hot Partitions: Popular conversation activity
```

## Exploring Your Database

### Environment Setup
```bash
# Required environment variables
export COMMONS.INTERACTIONS-TABLE="your-interactions-table-name"
export COMMONS.SESSIONS-TABLE="your-sessions-table-name"  
export AWS_REGION="eu-west-1"
export AWS_PROFILE_NAME="your-aws-profile"
```

### Discovery Script
```bash
# Use the provided discovery script
./scripts/discover-tables.sh
```

### AWS Console Navigation
1. **Go to**: https://console.aws.amazon.com/dynamodb/
2. **Region**: eu-west-1
3. **Find Tables**: Look for interaction and session tables
4. **Explore**:
   - **Items Tab**: Browse actual conversation data
   - **Indexes Tab**: See secondary index structure
   - **Metrics Tab**: Monitor performance

### Sample Queries in AWS Console

**Find a conversation:**
```
Partition key: conversationId = "your-conversation-id"
```

**Find user's latest activity:**
```
Index: clientId-conversationId-index
Partition key: clientId = "your-client-id"
```

**Find conversation timeline:**
```
Partition key: conversationId = "conversation-id"
Sort: messageId (ascending)
```

## Troubleshooting Iterations

### Common Issues

1. **Message ID Gaps**: 
   - **Cause**: Failed writes or concurrent access
   - **Solution**: System continues with next available ID

2. **Orphaned Messages**:
   - **Cause**: SessionContext not properly linked
   - **Detection**: Missing `sessionId` or `clientId`

3. **Security Violations**:
   - **Cause**: Client ID mismatch
   - **Response**: 403 Forbidden, iteration blocked

4. **Topic Inconsistency**:
   - **Cause**: Topic updates not propagated
   - **Fix**: `update_interaction()` method

### Monitoring Queries
```python
# Check iteration completeness
def validate_iteration(conversation_id: str, expected_turn_count: int):
    interactions = get_conversation(conversation_id)
    human_messages = [i for i in interactions if i['author']['S'] == 'human']
    ai_messages = [i for i in interactions if i['author']['S'] == 'ai']
    
    return len(human_messages) == len(ai_messages) == expected_turn_count
```

## Best Practices

### For Developers
1. **Always validate** conversation ownership before iterations
2. **Use SessionContext** consistently across all database operations
3. **Handle failures** gracefully - partial iterations should be recoverable
4. **Monitor turn counts** for conversation length policies

### For Operations
1. **Monitor hot partitions** for performance issues
2. **Track iteration completion rates** for system health
3. **Archive old conversations** based on business requirements
4. **Monitor guardrail application** for content safety

### For Analytics
1. **Turn count analysis** for conversation engagement metrics
2. **Author pattern analysis** for RAG usage statistics
3. **Session linkage** for user journey analysis
4. **Feedback correlation** with iteration quality

---

## Related Documentation

- **[Send Message Flow](./send-message-flow.md)**: Complete endpoint flow documentation
- **[Database Design](./database-design.md)**: Core database architecture
- **[AWS Integration](./aws-integration.md)**: AWS service configuration
- **[Database Schema](./database-schema.md)**: Complete table structures
- **[AWS Database Exploration](./aws-database-exploration.md)**: Console navigation guide

This guide provides the complete picture of how conversation iterations work in your chatbot system, from the initial user input through database storage and retrieval.
