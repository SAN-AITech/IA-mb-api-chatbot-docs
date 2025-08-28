# Database Architecture & Management

## Overview

The IA MB API Chatbot uses Amazon DynamoDB as its primary database for storing conversation data, session information, and user interactions. The design emphasizes scalability, performance, and real-time access patterns.

## Database Design Philosophy

### Single-Table Design Pattern

- **Principle**: Store related data in one table with composite keys
- **Benefits**: Reduced latency, simplified operations, cost efficiency
- **Trade-offs**: Complex access patterns, careful key design required

### Key Design Decisions

- **Auto-incrementing Message IDs**: Maintains chronological order within conversations
- **Client-based Access Control**: Security through data ownership
- **Session Enrichment**: Analytics data linked to interactions
- **Eventual Consistency**: Acceptable for conversation flow

## Table Structures

### INTERACTIONS_TABLE (Primary Storage)

#### Primary Key Design

```text
Partition Key: conversationId (String)    # Groups all messages in one conversation
Sort Key:      messageId (Number)         # Orders messages chronologically (1, 2, 3...)
```

#### Complete Item Schema

```json
{
  // Primary Keys
  "conversationId": {"S": "conversation-uuid"},        // Partition key
  "messageId": {"N": "1"},                             // Sort key (auto-increment)
  
  // Core Message Data
  "interactionId": {"S": "interaction-uuid"},          // Unique interaction ID
  "author": {"S": "human|ai|retriever"},              // Message source type
  "message": {"S": "actual message content"},          // The text content
  "createdAt": {"S": "2025-08-28T10:30:00Z"},         // ISO timestamp
  "responseId": {"S": "response-uuid"},                // Response tracking ID
  
  // Security & Ownership
  "clientId": {"S": "client-identifier"},              // User ownership
  "sessionId": {"S": "session-uuid"},                 // Session reference
  "guardrailApplied": {"BOOL": false},                // Content safety flag
  
  // Analytics Data (enriched from session)
  "browser": {"S": "Chrome 120.0"},                   // User browser
  "device": {"S": "Desktop"},                         // Device type
  "pageUrl": {"S": "https://bank.com/chat"},          // Source page
  "channel": {"S": "web"},                            // Channel type
  
  // Conversation Management
  "topic": {"S": "Account Balance Inquiry"},          // Conversation title
  "spanId": {"S": "telemetry-span-id"},              // Distributed tracing
  
  // Feedback System
  "score": {"N": "5"},                                // User rating (1-5)
  "scoringType": {"S": "thumbs_up|thumbs_down"},     // Feedback type
  "scoreMessage": {"S": "feedback comment"}           // User feedback text
}
```

#### Secondary Indexes

#### clientId-conversationId-index (GSI)

- **Partition Key**: `clientId`
- **Sort Key**: `conversationId`
- **Purpose**: Find all conversations for a specific user
- **Filter**: `attribute_exists(topic)` (only saved conversations)
- **Access Pattern**: User conversation history

### SESSIONS_TABLE (Metadata Storage)

#### Session Key Design

```text
Partition Key: sessionId (String)    # Unique session identifier
```

#### Session Schema

```json
{
  "sessionId": {"S": "session-uuid"},                 // Primary key
  "createdAt": {"S": "2025-08-28T10:00:00Z"},        // Session start time
  "clientId": {"S": "client-identifier"},            // User identifier
  
  // Analytics Data
  "browser": {"S": "Chrome 120.0"},                  // Browser version
  "device": {"S": "Desktop"},                        // Device type
  "pageUrl": {"S": "https://bank.com/chat"},         // Entry point
  "channel": {"S": "web"}                            // Channel type
}
```

## Data Access Patterns

### Pattern 1: Conversation Processing Flow

#### 1.1 Security Validation

```python
# Get latest interaction for ownership check
def get_conversation_last_interaction(conversation_id: str):
    response = dynamodb.query(
        TableName=INTERACTIONS_TABLE,
        KeyConditionExpression='conversationId = :id',
        ExpressionAttributeValues={':id': {'S': conversation_id}},
        ScanIndexForward=False,    # Descending order (latest first)
        Limit=1                    # Only most recent
    )
    return response.get('Items', [None])[0]
```

#### 1.2 Message ID Generation

```python
# Auto-increment message ID for new interactions
def get_next_message_id(conversation_id: str):
    last_interaction = get_conversation_last_interaction(conversation_id)
    if last_interaction:
        return int(last_interaction['messageId']['N']) + 1
    return 1  # First message in conversation
```

#### 1.3 Interaction Storage

```python
# Store new message with complete context
def create_interaction(session_context, message, author, span_id=None):
    message_id = get_next_message_id(session_context.conversation_id)
    
    item = {
        'conversationId': {'S': session_context.conversation_id},
        'messageId': {'N': str(message_id)},
        'interactionId': {'S': session_context.interaction_id},
        'author': {'S': author},
        'message': {'S': message},
        'createdAt': {'S': datetime.now(timezone.utc).isoformat()},
        'responseId': {'S': session_context.response_id},
        'sessionId': {'S': session_context.session_id},
        'guardrailApplied': {'BOOL': guardrail_applied}
    }
    
    # Enrich with session data
    if session_context.client_id:
        item['clientId'] = {'S': session_context.client_id}
    
    if session_context.session_id:
        session_data = get_session(session_context.session_id)
        if session_data:
            item.update({
                'browser': session_data['browser'],
                'device': session_data['device'],
                'pageUrl': session_data['pageUrl'],
                'channel': session_data['channel']
            })
    
    dynamodb.put_item(TableName=INTERACTIONS_TABLE, Item=item)
```

### Pattern 2: Conversation Retrieval

#### 2.1 Full Conversation History

```python
# Get complete conversation in chronological order
def get_conversation(conversation_id: str):
    paginator = dynamodb.get_paginator('query')
    response_iterator = paginator.paginate(
        TableName=INTERACTIONS_TABLE,
        KeyConditionExpression='conversationId = :id',
        ExpressionAttributeValues={':id': {'S': conversation_id}},
        ScanIndexForward=True    # Ascending order (chronological)
    )
    
    items = []
    for response in response_iterator:
        items.extend(response.get('Items', []))
    return items
```

#### 2.2 User Conversation List

```python
# Get all conversations for a user (saved only)
def get_user_conversations(client_id: str, limit=0, page=0):
    response = dynamodb.query(
        TableName=INTERACTIONS_TABLE,
        IndexName='clientId-conversationId-index',
        KeyConditionExpression='clientId = :client',
        FilterExpression='attribute_exists(topic)',
        ExpressionAttributeValues={':client': {'S': client_id}}
    )
    return response.get('Items', [])
```

### Pattern 3: Conversation Management

#### 3.1 Topic Updates

```python
# Update conversation topic across all interactions
def update_interaction_topic(conversation_id: str, message_id: str, topic: str):
    if topic:
        dynamodb.update_item(
            TableName=INTERACTIONS_TABLE,
            Key={
                'conversationId': {'S': conversation_id},
                'messageId': {'N': message_id}
            },
            UpdateExpression='SET topic = :topic',
            ExpressionAttributeValues={':topic': {'S': topic}}
        )
    else:
        # Remove topic (soft delete)
        dynamodb.update_item(
            TableName=INTERACTIONS_TABLE,
            Key={
                'conversationId': {'S': conversation_id},
                'messageId': {'N': message_id}
            },
            UpdateExpression='REMOVE topic'
        )
```

#### 3.2 Feedback Storage

```python
# Store user feedback for specific interaction
def store_feedback(interaction, score, scoring_type, message):
    dynamodb.update_item(
        TableName=INTERACTIONS_TABLE,
        Key={
            'conversationId': interaction['conversationId'],
            'messageId': interaction['messageId']
        },
        UpdateExpression='SET score = :score, scoringType = :type, scoreMessage = :msg',
        ExpressionAttributeValues={
            ':score': {'N': str(score)},
            ':type': {'S': scoring_type},
            ':msg': {'S': message}
        }
    )
```

## Performance Characteristics

### Read Patterns

- **Hot Path**: Latest interaction queries (security checks)
- **Warm Path**: Full conversation retrieval (history loading)
- **Cold Path**: User conversation lists (less frequent)

### Write Patterns

- **High Frequency**: New message creation during active conversations
- **Medium Frequency**: Topic updates and feedback submission
- **Low Frequency**: Session creation and cleanup

### Scaling Behavior

#### Hot Partitions

- **Cause**: Popular conversations with high message volume
- **Detection**: CloudWatch metrics for consumed capacity
- **Mitigation**:
  - Auto-scaling enabled on table and indexes
  - Conversation archiving for very long conversations
  - Load distribution across multiple conversations

#### Performance Optimization

- **Query Efficiency**: Single-item queries for latest interactions
- **Batch Operations**: Not used (conversations are independent)
- **Connection Pooling**: Boto3 connection reuse
- **Eventual Consistency**: Acceptable for conversation flow

## Data Lifecycle Management

### Conversation States

1. **Active**: Ongoing conversation with recent messages
2. **Saved**: User explicitly saved with topic
3. **Archived**: Old conversations without recent activity
4. **Deleted**: Soft delete by removing topic

### Retention Policies

- **Active Conversations**: Indefinite retention during activity
- **Saved Conversations**: User-controlled retention
- **Unsaved Conversations**: Configurable cleanup (e.g., 30 days)
- **Session Data**: Cleanup after session expiry

### Data Archival Strategy

```python
# Archive old conversations to cold storage
def archive_old_conversations():
    # Identify conversations older than retention period
    # Export to S3 for long-term storage
    # Remove from active DynamoDB table
    # Maintain metadata for potential restoration
```

## Monitoring and Observability

### Key Metrics

- **Read/Write Capacity Utilization**: Table and index performance
- **Throttling Events**: Capacity exceeded indicators
- **Item Count Growth**: Storage utilization trends
- **Query Latency**: Response time monitoring

### Alerting Thresholds

- **Capacity Utilization** > 80%: Scale up warning
- **Throttling Events** > 0: Immediate attention
- **Query Latency** > 100ms: Performance degradation
- **Error Rate** > 1%: System health issues

### Operational Queries

#### Health Checks

```python
# Verify table accessibility and basic operations
def health_check():
    try:
        response = dynamodb.describe_table(TableName=INTERACTIONS_TABLE)
        return response['Table']['TableStatus'] == 'ACTIVE'
    except Exception:
        return False
```

#### Usage Analytics

```python
# Get conversation activity metrics
def get_activity_metrics(date_range):
    # Query conversations created in date range
    # Aggregate by client, conversation count, message volume
    # Calculate engagement metrics
```

## Security and Access Control

### Data Protection

- **Encryption at Rest**: DynamoDB native encryption
- **Encryption in Transit**: TLS for all API calls
- **Access Control**: IAM policies for service accounts
- **Audit Logging**: CloudTrail for all data access

### Client Isolation

- **Row-Level Security**: clientId-based access control
- **Query Filtering**: Automatic client scope application
- **Cross-Client Prevention**: Validation in application layer

### Compliance Considerations

- **Data Residency**: EU-West-1 region compliance
- **Retention Controls**: User data deletion capabilities
- **Audit Trail**: Complete interaction logging
- **Privacy Controls**: Content masking via guardrails

## Troubleshooting Guide

### Common Issues

#### Message ID Gaps

- **Symptom**: Missing message IDs in sequence
- **Cause**: Failed writes or concurrent access
- **Resolution**: System continues with next available ID
- **Prevention**: Proper error handling and retry logic

#### Orphaned Sessions

- **Symptom**: Session data without corresponding interactions
- **Cause**: Session creation without conversation start
- **Resolution**: Cleanup job for expired sessions
- **Prevention**: Session lifecycle management

#### Hot Partition Issues

- **Symptom**: High latency on specific conversations
- **Cause**: Very active conversation overwhelming partition
- **Resolution**: Archive or split conversation
- **Prevention**: Conversation length limits

### Diagnostic Queries

#### Find Problematic Conversations

```python
# Identify conversations with unusual patterns
def find_unusual_conversations():
    # Query conversations with high message counts
    # Identify rapid-fire message patterns
    # Check for repeated failed operations
```

#### Validate Data Integrity

```python
# Check for data consistency issues
def validate_data_integrity(conversation_id):
    messages = get_conversation(conversation_id)
    # Verify message ID sequence
    # Check author alternation patterns
    # Validate session context consistency
```

This database architecture provides a robust foundation for scalable conversation management while maintaining security, performance, and operational simplicity.
