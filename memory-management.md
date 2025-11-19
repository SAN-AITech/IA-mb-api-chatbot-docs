# Memory Management & Chat History

[← Back to Documentation Home](README.md)

## How Chat Memory Works

### 1. **LangGraph State Management**

The application uses LangGraph's `MessagesState` to manage conversation memory:

```python
class State(MessagesState):
    documents: Annotated[List[Document], operator.add]
    user_question: str
    contact_center_answer: CCAnswer
    deep_links: Optional[list[str]]
    suggestions: Optional[SuggestionsAnswer]
    guardrail_applied: bool
```

### 2. **Modern Dual-Memory Architecture**

**Three-Tier Memory System (Updated):**

#### Tier 1: AWS Bedrock AgentCore Memory (New!)

- **Short-term Memory**: Recent conversation context via `list_events` API
- **Long-term Memory**: Semantic search for user preferences via `retrieve_memory_records`
- **User Preferences**: Learns and recalls user-specific information
- **Namespace**: Uses `/preferences/{actorId}` for personalized memory

#### Tier 2: DynamoDB Persistence (Traditional)

- **Sessions Table**: Stores user session metadata
- **Interactions Table**: Stores individual messages with conversation context
- **Audit Trail**: Complete conversation history for compliance
- **Fallback**: Used when AgentCore is disabled

#### Tier 3: LangGraph In-Memory State

- **Runtime memory**: Active during single conversation flow
- **Message chain**: Maintains conversation context for LLM
- **Temporary state**: Lost when request completes

### 3. **Dynamic Memory Configuration**

The system now supports **runtime switching** between memory types:

```python
# In AgentExecutor.init_executor_configuration()
if self.settings.chat.use_agentcore_memory:
    self.agent_memory = AgentMemoryAgentCore()
else:
    self.agent_memory = AgentMemoryDynamoDB()
```

**Memory Class Architecture:**

```python
# Abstract base class
class AgentMemory(ABC):
    def get_conversation(self, session_context: SessionContext, user_question: str) -> list[BaseMessage]:
        ...
    def save_memory(self, session_context: SessionContext, document_content: str, 
                   span_id: str | None, author: str, guardrail_applied: bool = False) -> None:
        ...

# AgentCore implementation
class AgentMemoryAgentCore(AgentMemory):
    def get_short_term_memory(self, session_context: SessionContext) -> list[BaseMessage]:
        # Retrieves recent conversation using AWS AgentCore list_events API
        
    def get_long_term_memory(self, session_context: SessionContext, user_question: str) -> BaseMessage | None:
        # Semantic search for user preferences using retrieve_memory_records API
        
# DynamoDB implementation  
class AgentMemoryDynamoDB(AgentMemory):
    def get_conversation(self, session_context: SessionContext, user_question: str) -> list[BaseMessage]:
        # Traditional DynamoDB conversation retrieval
```

### 4. **Enhanced Memory Lifecycle**

**AgentCore Memory Flow:**

```python
# 1. Initialize memory type based on configuration
if settings.chat.use_agentcore_memory:
    memory = AgentMemoryAgentCore()

# 2. Retrieve conversation with semantic enhancement
short_term = memory.get_short_term_memory(session_context)
long_term = memory.get_long_term_memory(session_context, user_question)
combined_memory = short_term + [long_term] if long_term else short_term

# 3. Process with LLM using enhanced context
response = agent.process_with_langgraph(combined_memory, user_input)

# 4. Dual storage - both AgentCore and DynamoDB
memory.save_memory(session_context, response, author="ai")
```

**DynamoDB Memory Flow (Traditional):**

```python
# 1. Load conversation history from DynamoDB only
conversation_history = aws_service.get_conversation(conversation_id)

# 2. Build message chain for LangGraph
messages = build_message_chain(conversation_history)

# 3. Process with LLM using current state
response = agent.process_with_langgraph(messages, user_input)

# 4. Save new interaction to DynamoDB only
aws_service.create_interaction(session_context, response)
```

### 5. **Advanced Memory Features**

**Long-term Semantic Memory (AgentCore Only):**

- **User Preference Learning**: Automatically learns user patterns and preferences
- **Semantic Search**: Retrieves relevant user information based on current question context
- **Personalized Responses**: Enables context-aware responses based on user history
- **Privacy Namespace**: Uses `/preferences/{actorId}` for user-specific memory isolation

**Threaded Memory Persistence:**

- **Asynchronous Storage**: Memory saving happens in background threads for performance
- **Dual Storage**: When AgentCore is enabled, saves to both AgentCore and DynamoDB
- **Guardrail Integration**: Applies content masking before storage
- **Error Resilience**: Continues operation if one storage mechanism fails

**Memory Configuration Options:**

```yaml
chat:
  use_agentcore_memory: true        # Enable AgentCore memory (false = DynamoDB only)
  agentcore_memory_id: "memory-123" # AgentCore memory identifier
guardrails:
  guardrail_mask_id: "mask-456"     # Mask sensitive data before storage
```

### 6. **Memory Consistency**

- **Session Validation**: Headers ensure proper session context
- **Conversation Threading**: Messages linked by conversationId and messageId
- **Temporal Ordering**: Messages ordered by createdAt timestamp

## Key Benefits

- **Intelligent Memory**: AgentCore provides semantic search and user preference learning
- **Persistent Memory**: Conversations survive application restarts
- **Scalable**: Both DynamoDB and AgentCore handle multiple concurrent conversations
- **Context-Aware**: LLM has access to both recent conversation and relevant historical context
- **Stateless API**: Each request is independent but context-aware
- **Flexible Configuration**: Runtime switching between memory architectures
- **Performance Optimized**: Threaded storage and intelligent caching

## Message ID Sequencing

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

This ensures proper message ordering and prevents conflicts in multi-user environments.

## Turn Count Management

```python
# In conversation.py - Line 342
session_context.turn_count = int(request.context.system.turnCount) + 1
```

Each iteration increments the turn count, allowing the system to:

- Track conversation length
- Apply policies based on interaction depth  
- Monitor conversation engagement patterns
- Implement turn-based limitations if needed
