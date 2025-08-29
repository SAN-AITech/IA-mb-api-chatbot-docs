[← Back to Documentation Home](README.md)

# AWS DynamoDB Exploration Guide

## Accessing DynamoDB in AWS Console

### **Step 1: Access AWS Console**
1. Go to AWS Console: https://console.aws.amazon.com/
2. Login with your AWS credentials
3. Navigate to **DynamoDB** service
4. Select region: **eu-west-1** (based on your code: `AWS_REGION = 'eu-west-1'`)

### **Step 2: Find Your Tables**
Look for tables with these environment variable names:
- **`COMMONS.INTERACTIONS-TABLE`** - Main conversation storage
- **`COMMONS.SESSIONS-TABLE`** - Session metadata

### **Step 3: Explore Table Structure**

#### **For INTERACTIONS_TABLE:**

1. **Overview Tab:**
   - Primary key: `conversationId` (Partition), `messageId` (Sort)  
   - Item count and table size
   - Read/Write capacity settings

2. **Items Tab:**
   - Browse actual conversation data
   - See real message exchanges
   - Filter by `conversationId` to see full conversations
   - Sort by `messageId` to see chronological order

3. **Indexes Tab:**
   - **Global Secondary Index**: `clientId-conversationId-index`
   - Used for: Getting all conversations for a specific user
   - Partition Key: `clientId`

4. **Metrics Tab:**
   - Read/Write operations per second
   - Storage utilization
   - Hot partition detection

#### **For SESSIONS_TABLE:**

1. **Overview Tab:**
   - Primary key: `sessionId`
   - Session count and analytics data

2. **Items Tab:**
   - Browse session metadata
   - See browser, device, pageUrl data
   - Match `sessionId` with interactions

### **Step 4: Query Examples in AWS Console**

#### **Find a specific conversation:**
```
Partition key: conversationId = "your-conversation-id"
```

#### **Find user's latest interaction:**
```
Partition key: conversationId = "conversation-id"
Sort: Descending by messageId
Limit: 1
```

#### **Find all conversations for a user:**
```
Index: clientId-conversationId-index
Partition key: clientId = "your-client-id"
Filter: attribute_exists(topic)
```

### **Step 5: Analyze Data Patterns**

#### **Conversation Structure:**
- Each conversation = multiple items with same `conversationId`
- Messages ordered by `messageId` (1, 2, 3, ...)
- Pattern: human → ai → human → ai

#### **Author Types:**
- `"human"` - User messages
- `"ai"` - Chatbot responses  
- `"retriever"` - RAG document content

#### **Message Flow Example:**
```
conversationId: "conv-123"
├── messageId: 1, author: "human", message: "Hello"
├── messageId: 2, author: "ai", message: "Hi! How can I help?"
├── messageId: 3, author: "human", message: "What is my balance?"
├── messageId: 4, author: "retriever", message: "Retrieved bank account data..."
└── messageId: 5, author: "ai", message: "Your balance is $1,234.56"
```

### **Step 6: CloudWatch Metrics**

Navigate to **CloudWatch** → **DynamoDB** metrics:
- **ConsumedReadCapacityUnits** - Query frequency
- **ConsumedWriteCapacityUnits** - Message creation rate
- **ThrottledRequests** - Performance bottlenecks
- **ItemCount** - Database growth over time

### **Environment Variables to Check:**

In your deployment/environment configuration:
```bash
echo $COMMONS.INTERACTIONS-TABLE    # Your main table name
echo $COMMONS.SESSIONS-TABLE        # Your sessions table name  
echo $AWS_REGION                    # Should be 'eu-west-1'
```

## CLI Alternative

You can also explore via AWS CLI:

```bash
# List tables
aws dynamodb list-tables --region eu-west-1

# Describe table structure
aws dynamodb describe-table --table-name YOUR_INTERACTIONS_TABLE --region eu-west-1

# Query specific conversation
aws dynamodb query \
  --table-name YOUR_INTERACTIONS_TABLE \
  --key-condition-expression "conversationId = :id" \
  --expression-attribute-values '{":id":{"S":"your-conversation-id"}}' \
  --region eu-west-1

# Get latest interaction
aws dynamodb query \
  --table-name YOUR_INTERACTIONS_TABLE \
  --key-condition-expression "conversationId = :id" \
  --expression-attribute-values '{":id":{"S":"your-conversation-id"}}' \
  --scan-index-forward false \
  --limit 1 \
  --region eu-west-1
```
