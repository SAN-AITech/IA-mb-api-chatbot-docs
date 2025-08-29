[← Back to Documentation Home](README.md)

# Memory Management & Chat History

## How Chat Memory Works

### 1. **LangGraph State Management**

The application uses LangGraph's `MessagesState` to manage conversation memory:

```python
class State(MessagesState):
    documents: Annotated[List[Document], operator.add]
    user_question: str
    contact_center_answer: CCAnswer
    deep_links: Optional[List[str]] = None
    suggestions: Optional[SuggestionsAnswer] = None
    guardrail_applied: bool = False
```

### 2. **Memory Storage Strategy**

**Two-Layer Memory System:**

#### Layer 1: DynamoDB Persistence
- **Sessions Table**: Stores user session metadata
- **Interactions Table**: Stores individual messages with conversation context
- **Long-term memory**: Survives application restarts

#### Layer 2: LangGraph In-Memory State
- **Runtime memory**: Active during single conversation flow
- **Message chain**: Maintains conversation context for LLM
- **Temporary state**: Lost when request completes

### 3. **Memory Retrieval Process**

1. **Session Creation**: New session stored in DynamoDB
2. **Message Processing**: Each user message creates an interaction record
3. **Context Building**: Previous messages retrieved from DynamoDB to build context
4. **LangGraph State**: Messages loaded into MessagesState for LLM processing
5. **Response Storage**: AI response saved back to DynamoDB

### 4. **Memory Lifecycle**

```python
# 1. Load conversation history from DynamoDB
conversation_history = aws_service.get_conversation(conversation_id)

# 2. Build message chain for LangGraph
messages = build_message_chain(conversation_history)

# 3. Process with LLM using current state
response = agent.process_with_langgraph(messages, user_input)

# 4. Save new interaction to DynamoDB
aws_service.create_interaction(session_context, response)
```

### 5. **Memory Optimization**

- **Pagination**: Large conversations retrieved in chunks
- **Message Limits**: Conversation context truncated to prevent token limits
- **State Cleanup**: Temporary state cleared after request completion

### 6. **Memory Consistency**

- **Session Validation**: Headers ensure proper session context
- **Conversation Threading**: Messages linked by conversationId and messageId
- **Temporal Ordering**: Messages ordered by createdAt timestamp

## Key Benefits

- **Persistent Memory**: Conversations survive application restarts
- **Scalable**: DynamoDB handles multiple concurrent conversations
- **Context-Aware**: LLM has access to full conversation history
- **Stateless API**: Each request is independent but context-aware
