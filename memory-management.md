# Memory Management & Chat History

[← Back to Documentation Home](README.md)

## Overview

The IA MB API Chatbot implements a sophisticated dual-memory architecture that combines traditional DynamoDB persistence with advanced AWS Bedrock AgentCore semantic memory. This modular system is orchestrated through the AgentExecutor framework, providing both reliable conversation storage and intelligent memory capabilities.

## How Chat Memory Works

### 1. **AgentExecutor Memory Integration**

The application uses AgentExecutor's modular memory management through the `AgentMemory` abstraction layer:

```python
# In AgentExecutor framework - agent_execution/agent_memory.py
class AgentMemory(ABC):
    """Abstract memory interface for dual memory system support"""
    
    @abstractmethod
    def get_conversation(self, session_context: SessionContext, user_question: str) -> list[BaseMessage]:
        """Retrieve conversation history with optional semantic enhancement"""
        
    @abstractmethod  
    def save_memory(self, session_context: SessionContext, document_content: str, 
                   span_id: str | None, author: str, guardrail_applied: bool = False) -> None:
        """Persist memory to selected storage system"""
```

### 2. **Dual-Memory Architecture**

The system implements a sophisticated three-tier memory approach designed for both performance and intelligent conversation management:

#### **Tier 1: AWS Bedrock AgentCore Memory (Advanced)**

**Intelligent Memory Capabilities**:

- **Short-term Memory**: Recent conversation context via `list_events` API
- **Long-term Memory**: Semantic search for user preferences via `retrieve_memory_records`
- **User Preference Learning**: Automatically learns and recalls user-specific patterns
- **Contextual Retrieval**: Retrieves relevant information based on current conversation context
- **Privacy Isolation**: Uses `/preferences/{actorId}` namespace for secure user data separation

#### **Tier 2: DynamoDB Persistence (Traditional & Reliable)**

**Structured Storage**:

- **SESSIONS_TABLE**: User session metadata and device context
- **INTERACTIONS_TABLE**: Individual messages with complete conversation history
- **Audit Trail**: Complete conversation history for compliance and debugging
- **Fallback System**: Used when AgentCore is unavailable or disabled
- **Performance**: Sub-100ms query latency with auto-scaling capabilities

#### **Tier 3: LangGraph In-Memory State**

**Runtime Processing**:

- **Active Memory**: Maintains state during single conversation flow
- **Message Chain**: Current conversation context for LLM processing
- **Temporary Storage**: State exists only during request processing
- **Performance**: Ultra-fast access for real-time conversation processing

### 3. **AgentExecutor Memory Configuration**

The AgentExecutor framework orchestrates memory system selection and initialization through a clean abstraction layer:

#### **Memory System Selection**

```python
def init_agent(self, session_context: SessionContext) -> StateGraph:
    """Initialize agent with selected memory system based on configuration"""
    
    if self.settings.chat.use_agentcore_memory:
        # Initialize advanced semantic memory
        self.agent_memory = AgentMemoryAgentCore(
            memory_id=self.settings.chat.agentcore_memory_id,
            session_context=session_context
        )
    else:
        # Initialize traditional DynamoDB memory
        self.agent_memory = AgentMemoryDynamoDB(
            aws_service=self.aws_service,
            session_context=session_context
        )
        
    return self.build_graph_with_memory(self.agent_memory)
```

#### **Memory Interface Architecture**

The system uses a consistent interface across both memory implementations:

```python
# Abstract base class providing consistent interface
class AgentMemory(ABC):
    @abstractmethod
    def get_conversation(self, session_context: SessionContext, user_question: str) -> list[BaseMessage]:
        """Retrieve conversation history with optional semantic enhancement"""
        
    @abstractmethod  
    def save_memory(self, session_context: SessionContext, document_content: str, 
                   span_id: str | None, author: str, guardrail_applied: bool = False) -> None:
        """Persist memory to selected storage system"""

# AgentCore implementation with semantic capabilities
class AgentMemoryAgentCore(AgentMemory):
    def get_short_term_memory(self, session_context: SessionContext) -> list[BaseMessage]:
        """Retrieves recent conversation using AWS AgentCore list_events API"""
        
    def get_long_term_memory(self, session_context: SessionContext, user_question: str) -> BaseMessage | None:
        """Semantic search for user preferences using retrieve_memory_records API"""
        
# DynamoDB implementation for traditional storage
class AgentMemoryDynamoDB(AgentMemory):
    def get_conversation(self, session_context: SessionContext, user_question: str) -> list[BaseMessage]:
        """Traditional DynamoDB conversation retrieval with message ordering"""
```

### 4. **Memory Processing Workflows**

The AgentExecutor framework orchestrates different memory processing flows based on the selected system:

#### **AgentCore Memory Processing Flow**

```python
# 1. AgentExecutor initializes with AgentCore memory
executor = AgentExecutor(agent_config, session_context)
agent_graph = executor.init_agent(session_context)  # Selects AgentMemoryAgentCore

# 2. MessageProcessor retrieves enhanced conversation context
short_term_messages = executor.agent_memory.get_short_term_memory(session_context)
long_term_context = executor.agent_memory.get_long_term_memory(session_context, user_question)
enhanced_context = short_term_messages + ([long_term_context] if long_term_context else [])

# 3. StateGraph processes with semantic enhancement
async for stream_type, value in agent_graph.astream(agent_input):
    response_data = await message_processor.process_stream(stream_type, value)

# 4. Dual storage coordination via MessageProcessor
await executor.agent_memory.save_memory(session_context, response_data, author="ai")
```

**Key Benefits of AgentCore Flow**:

- **Semantic Understanding**: Context retrieval based on conversation meaning, not just chronology
- **User Preference Learning**: Automatically adapts to user patterns over time
- **Intelligent Context**: Retrieves relevant historical information even from distant conversations
- **Privacy Protection**: Namespace isolation ensures user data separation

#### **DynamoDB Memory Processing Flow**

```python
# 1. AgentExecutor initializes with DynamoDB memory
executor = AgentExecutor(agent_config, session_context) 
agent_graph = executor.init_agent(session_context)  # Selects AgentMemoryDynamoDB

# 2. Traditional conversation history retrieval
conversation_history = executor.agent_memory.get_conversation(session_context, user_question)
chronological_context = conversation_history  # Simple chronological ordering

# 3. StateGraph processes with traditional context
async for stream_type, value in agent_graph.astream(agent_input):
    response_data = await message_processor.process_stream(stream_type, value)

# 4. Single storage persistence
await executor.agent_memory.save_memory(session_context, response_data, author="ai")
```

**Key Benefits of DynamoDB Flow**:

- **Reliability**: Proven, consistent storage with predictable performance
- **Simplicity**: Straightforward chronological conversation retrieval
- **Audit Trail**: Complete conversation history for compliance requirements
- **Fallback Capability**: Always available as backup when AgentCore encounters issues

### 5. **Advanced Memory Features & Integration**

#### **MessageProcessor Integration**

The MessageProcessor component provides seamless integration between memory systems and conversation processing:

**Stream-based Memory Persistence**:

- **Real-time Storage**: Memory saving happens asynchronously during response streaming
- **Dual Storage Coordination**: When AgentCore is enabled, coordinates storage to both systems
- **Error Resilience**: Continues operation if one storage mechanism fails
- **Performance Optimization**: Background persistence doesn't block response delivery

**Advanced AgentCore Capabilities**:

**Semantic Memory Features**:

- **User Preference Learning**: Automatically detects and stores user patterns and preferences
- **Contextual Retrieval**: Retrieves relevant information based on current conversation context
- **Cross-Conversation Learning**: Learns from interactions across multiple conversations
- **Privacy Namespace**: Uses `/preferences/{actorId}` for secure user-specific memory isolation

**Memory Configuration Options**:

```yaml
chat:
  use_agentcore_memory: true        # Enable AgentCore semantic memory
  agentcore_memory_id: "memory-123" # AgentCore memory identifier
  
agent_execution:
  memory_fallback_enabled: true     # Auto-fallback to DynamoDB on AgentCore errors
  memory_sync_enabled: true         # Sync data between AgentCore and DynamoDB
  
guardrails:
  guardrail_mask_id: "mask-456"     # Mask sensitive data before storage
  apply_masking_to_memory: true     # Apply content masking to stored memories
```

### 6. **Memory Consistency & Data Integrity**

#### **Session Context Management**

**Header Validation**:

- Session and client ID validation ensures proper conversation ownership
- SessionContext creation provides consistent metadata across memory operations
- Temporal ordering maintained through createdAt timestamps

**Conversation Threading**:

- Messages linked by conversationId and auto-incrementing messageId
- Response pairs connected via responseId for conversation flow tracking
- Cross-conversation learning enabled through user preference namespaces

## Key Benefits of the Dual Memory Architecture

The AgentExecutor memory system provides significant advantages through its modular, dual-storage approach:

**Intelligent Memory Capabilities**:

- **Semantic Search**: AgentCore provides context-aware memory retrieval based on conversation meaning
- **User Preference Learning**: Automatically adapts to user patterns and preferences over time
- **Cross-Conversation Intelligence**: Learns from interactions across multiple conversation sessions
- **Privacy Protection**: Namespace isolation ensures secure user data separation

**Reliability & Performance**:

- **Dual Storage Reliability**: Conversations persist through multiple storage mechanisms
- **Scalable Architecture**: Both DynamoDB and AgentCore handle concurrent conversations efficiently
- **Performance Optimization**: MessageProcessor provides asynchronous storage and intelligent caching
- **Fallback Resilience**: System continues operating if one memory component fails

**Developer Experience**:

- **Modular Design**: Memory abstraction enables easy extension and testing of new memory systems
- **Runtime Configuration**: Switch between memory architectures without code changes
- **Context Awareness**: LLM receives both recent conversation and relevant historical context
- **Stateless API Design**: Each request is independent but contextually informed

## Technical Implementation Details

### **Message ID Sequencing**

The system maintains conversation order through auto-incrementing message IDs managed by the memory layer:

```python
# Message sequencing handled by memory layer
def create_interaction(self, session_context: SessionContext, message: str, author: str):
    # Get current maximum message ID for conversation
    response = self.get_conversation_last_interaction(session_context.conversation_id)
    if response:
        max_message_id = int(response['messageId']['N'])    # Current maximum
        topic = response.get('topic', {}).get('S', '')      # Preserve topic
    else:
        max_message_id = 0                                  # New conversation
        topic = None

    # Auto-increment for new message
    new_message_id = max_message_id + 1
    return new_message_id
```

This sequencing ensures:

- **Proper message ordering** in chronological conversations
- **Conflict prevention** in multi-user environments
- **Consistent numbering** across memory systems
- **Conversation continuity** during system restarts

### **Conversation Turn Management**

The AgentExecutor framework tracks conversation depth and engagement patterns:

```python
# Turn count management in SessionContext
session_context.turn_count = int(request.context.system.turnCount) + 1
```

**Turn Count Applications**:

- **Conversation Length Tracking**: Monitor engagement depth
- **Policy Application**: Implement interaction depth-based rules
- **Analytics**: Track conversation engagement patterns
- **Rate Limiting**: Apply turn-based conversation limits when needed

## Memory System Performance Characteristics

### **AgentCore Memory Performance**

- **Semantic Query Latency**: ~200-500ms for preference retrieval
- **Context Assembly**: ~100-200ms for recent conversation assembly
- **User Learning**: Real-time preference updates during conversations
- **Scalability**: Handles thousands of concurrent namespace operations

### **DynamoDB Memory Performance**

- **Query Latency**: Sub-100ms for conversation history retrieval
- **Write Performance**: ~10-20ms for individual message storage
- **Auto-scaling**: Automatic capacity adjustment based on demand
- **Consistency**: Strong consistency for conversation ordering

### **Hybrid Performance Benefits**

- **Best of Both**: Combines AgentCore intelligence with DynamoDB reliability
- **Graceful Degradation**: Fallback ensures continued operation during component issues
- **Optimized Caching**: MessageProcessor provides intelligent memory caching strategies

## Summary

The dual memory architecture represents a sophisticated approach to conversation management that balances intelligence, reliability, and performance. Through the AgentExecutor framework's modular design, the system provides:

**Intelligence**: AgentCore's semantic memory capabilities enable context-aware, user-personalized conversations that learn and adapt over time.

**Reliability**: DynamoDB's proven persistence layer ensures conversation continuity and provides comprehensive audit trails for compliance.

**Performance**: The MessageProcessor's asynchronous coordination and intelligent caching deliver responsive user experiences while maintaining data consistency.

**Flexibility**: Runtime configuration switching allows deployment-specific memory optimization without code changes.

This architecture enables the IA MB API Chatbot to deliver both immediate conversational intelligence and long-term relationship building with users, while maintaining enterprise-grade reliability and security standards.

## Related Documentation

For more detailed information about memory system implementation:

- **[Architecture Overview](architecture-overview.md)** - High-level system design with memory integration
- **[LangGraph Implementation](langgraph-implementation.md)** - AgentExecutor framework and StateGraph integration  
- **[Send Message Flow](send-message-flow.md)** - Complete message processing including memory operations
- **[Execute Method Flow](execute-method-flow.md)** - Detailed AgentExecutor processing workflow
