[← Back to Documentation Home](README.md)

# Database Schema Documentation

## DynamoDB Tables Structure

### 1. **INTERACTIONS_TABLE** (Main conversation storage)

**Primary Key Structure:**
- **Partition Key**: `conversationId` (String) - Groups all messages in a conversation
- **Sort Key**: `messageId` (Number) - Auto-incremented sequence within conversation

**Complete Item Structure:**
```python
{
    # Primary Keys
    'conversationId': {'S': 'uuid-string'},        # Partition key
    'messageId': {'N': '1'},                       # Sort key (auto-increment)
    
    # Core Message Data
    'interactionId': {'S': 'uuid-string'},         # Unique interaction identifier
    'author': {'S': 'human|ai|retriever'},         # Message source type
    'message': {'S': 'actual message content'},    # The text content
    'createdAt': {'S': '2025-08-27T10:30:00Z'},   # ISO timestamp
    'responseId': {'S': 'uuid-string'},            # Response tracking ID
    
    # Security & Validation
    'guardrailApplied': {'BOOL': True},            # Content safety flag
    'clientId': {'S': 'client-identifier'},       # User/client ownership
    
    # Session Context
    'sessionId': {'S': 'session-uuid'},           # Session reference
    
    # Analytics Data (enriched from session)
    'browser': {'S': 'Chrome/Firefox/Safari'},     # User browser
    'device': {'S': 'Desktop/Mobile/Tablet'},      # Device type  
    'pageUrl': {'S': 'https://example.com/chat'},  # Source page
    'channel': {'S': 'web/mobile/api'},           # Communication channel
    
    # Conversation Management
    'topic': {'S': 'conversation topic'},          # Optional conversation title
    'spanId': {'S': 'telemetry-span-id'},         # Distributed tracing
    
    # Feedback System
    'score': {'N': '5'},                          # User rating (1-5)
    'scoringType': {'S': 'thumbs_up|thumbs_down'}, # Feedback type
    'scoreMessage': {'S': 'feedback comment'}      # User feedback text
}
```

**Secondary Indexes:**
- **`clientId-conversationId-index`**: For retrieving user's conversations
  - Partition Key: `clientId`
  - Sort Key: `conversationId`
  - Used by: `get_conversations()` method

### 2. **SESSIONS_TABLE** (Session metadata)

**Primary Key:**
- **Partition Key**: `sessionId` (String)

**Item Structure:**
```python
{
    'sessionId': {'S': 'session-uuid'},           # Primary key
    'createdAt': {'S': '2025-08-27T10:30:00Z'},   # Session start time
    'clientId': {'S': 'client-identifier'},       # User identifier
    
    # Analytics Data
    'browser': {'S': 'Chrome 120.0'},             # Browser version
    'device': {'S': 'Desktop'},                   # Device type
    'pageUrl': {'S': 'https://bank.com/chat'},    # Source page
    'channel': {'S': 'web'}                       # Channel type
}
```

## Database Operations Flow

### **Message Storage Pattern:**
```
1. User sends message
   ↓
2. get_conversation_last_interaction() → Get max messageId
   ↓
3. create_interaction() → Store with messageId + 1
   ↓
4. AI processes and responds
   ↓
5. create_interaction() → Store AI response with messageId + 1
```

### **Query Patterns:**

1. **Get Latest Interaction** (Security check):
   ```python
   query(
       KeyConditionExpression='conversationId = :id',
       ScanIndexForward=False,  # Descending order
       Limit=1                  # Only latest
   )
   ```

2. **Get Full Conversation** (History):
   ```python
   query(
       KeyConditionExpression='conversationId = :id',
       ScanIndexForward=True    # Ascending order (chronological)
   )
   ```

3. **Get User's Conversations**:
   ```python
   query(
       IndexName='clientId-conversationId-index',
       KeyConditionExpression='clientId = :client',
       FilterExpression='attribute_exists(topic)'  # Only saved conversations
   )
   ```

## Performance Characteristics

- **Hot Partition**: Recent conversations get high read/write activity
- **Auto-scaling**: DynamoDB scales based on partition usage
- **Query Efficiency**: Single conversation queries are O(1) for latest interaction
- **Pagination**: Large conversations use DynamoDB pagination
