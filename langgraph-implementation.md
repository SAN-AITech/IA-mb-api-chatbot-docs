[← Back to Documentation Home](README.md)

# LangGraph & Agent Implementation

## LangGraph & Agent Implementation

### 1. **Modern Agent Execution Framework**

The application uses a **modular agent execution architecture** with LangGraph at its core:

```
/src/ia_mb_api_chatbot/services/agent_execution/
├── agent_executor.py      # Main orchestration and initialization
├── agent_memory.py        # Dual-memory management (AgentCore + DynamoDB)
├── agent_guardrails.py    # Safety and compliance checking
└── message_processor.py   # Message streaming and processing
```

**Core Agent Class**: `/src/ia_mb_api_chatbot/services/agent.py`

### 2. **Updated State Management**

The application uses LangGraph's StateGraph to orchestrate the conversation flow:

```python
class State(MessagesState):
    documents: Annotated[List[Document], operator.add]
    user_question: str
    contact_center_answer: CCAnswer
    deep_links: Optional[list[str]]
    suggestions: Optional[SuggestionsAnswer]
    guardrail_applied: bool
```

**Key Changes from Previous Version:**
- Simplified state structure focused on essential conversation elements
- Enhanced type annotations with `Optional[list[str]]` for flexibility
- Added `guardrail_applied` flag for safety tracking

### 3. **Modern Graph Structure & Agent Initialization**

**Agent Executor Orchestration:**

```python
class AgentExecutor:
    def __init__(self):
        self.settings = ia_mb_api_chatbot.configuration.settings.get_settings()
        self.service_aws = AWS()
        
    async def init_agent(self):
        # Initialize guardrails based on configuration
        if self.settings.guardrails.use_alinia_guardrails:
            self.agent_guardrails = AgentGuardrailsAlinia()
        else:
            self.agent_guardrails = AgentGuardrailsBedrock(self.service_aws)
        
        # Initialize memory system based on configuration
        if self.settings.chat.use_agentcore_memory:
            self.agent_memory = AgentMemoryAgentCore(
                session_context=self.session_context,
                service_aws=self.service_aws
            )
        else:
            self.agent_memory = AgentMemoryDynamoDB(
                session_context=self.session_context,
                service_aws=self.service_aws
            )
        
        # Create and initialize agent
        tracer_provider = TracerProvider()
        self.agent = Agent(self.suggestions, tracer_provider, self.agent_guardrails)
        await self.agent.init_llm_models()
        
        # Initialize message processor for streaming
        self.message_processor = MessageProcessor(
            agent=self.agent,
            agent_memory=self.agent_memory,
            agent_guardrails=self.agent_guardrails,
            session_context=self.session_context,
            service_aws=self.service_aws,
            stream_mode=["messages", "values"]
        )
```

**Current Graph Flow:**

```
START → [Parallel Branches] → END
     ↓
  chatbot ←→ tools (tool calling loop)
     ↓
check_cc_redirect (parallel)
     ↓  
deep_link_invoke (parallel)
     ↓
suggestions (parallel)
```

**Execution Architecture:**

```bach
AgentExecutor
├── Agent (graph orchestration)
│   ├── LLM with bound tools
│   ├── Tool registry (KB + MCP)
│   └── Graph compilation
├── MessageProcessor (stream handling)
│   ├── Dual stream processing
│   ├── Citation formatting
│   └── Response generation
├── AgentMemory (state persistence)
│   ├── AgentCore (semantic search)
│   └── DynamoDB (traditional storage)
└── AgentGuardrails (safety)
    ├── Alinia (external service)
    └── Bedrock (AWS native)
```

### 4. **Enhanced Agent Components**

#### **Main Chatbot Node**

- **Function**: `Agent.chatbot(state: State)`
- **Purpose**: Core conversation processing with guardrail checking
- **Process**: 
  1. Applies guardrails to user input
  2. Invokes LLM with tools if guardrails pass
  3. Returns AI response or guardrail message

#### **Tools Node**

- **Function**: `Agent.call_tool(state: State)`  
- **Purpose**: Handle tool calls from the LLM (Knowledge Base retrieval, MCP servers)
- **Features**:
  - **Knowledge Base Tools**: Amazon Bedrock Knowledge Base retrieval
  - **MCP Tools**: Model Context Protocol server integration
  - **Dynamic Tool Registry**: Runtime tool registration and execution

#### **Contact Center Check Node** (Optional)

- **Function**: `Agent.check_cc_redirect(state: State)`
- **Purpose**: Determine if user needs human agent transfer
- **Classification**: Returns 'AGENT' or 'OTHER' based on intent detection

#### **Deep Links Node** (Optional)

- **Function**: `Agent.deep_link_invoke(state: State)`
- **Purpose**: Extract relevant deep links from conversation context
- **Integration**: Works with both DSPy and LangGraph implementations

#### **Suggestions Node** (Optional)

- **Function**: `Agent.suggestions_invoke(state: State)`
- **Purpose**: Generate follow-up question suggestions
- **Flexibility**: Supports different suggestion algorithms

### 5. **Graph Building Logic**

**Current Implementation:**

```python
def build(self) -> CompiledStateGraph:
    """Builds the state graph for the agent."""
    graph_builder = StateGraph(State)

    # Core conversation flow
    graph_builder.add_node("chatbot", self.chatbot)
    graph_builder.add_node("tools", self.call_tool)
    
    graph_builder.add_edge(START, "chatbot")
    graph_builder.add_conditional_edges("chatbot", tools_condition, {"tools": "tools", END: END})
    graph_builder.add_edge("tools", "chatbot")

    # Optional feature nodes (configurable)
    if not self.agent_config.DisableContactCenter:
        graph_builder.add_node("check_cc_redirect", self.check_cc_redirect)
        graph_builder.add_edge(START, "check_cc_redirect")
        graph_builder.add_edge("check_cc_redirect", END)

    if self.agent_config.DeepLinks:
        graph_builder.add_node("deep_link_invoke", self.deep_link_invoke)
        graph_builder.add_edge(START, "deep_link_invoke")
        graph_builder.add_edge("deep_link_invoke", END)

    if self.suggestions:
        graph_builder.add_node("suggestions", self.suggestions_invoke)
        graph_builder.add_edge(START, "suggestions")
        graph_builder.add_edge("suggestions", END)

    return graph_builder.compile()
```

**Key Features:**
- **Conditional Node Addition**: Nodes are added based on configuration
- **Parallel Processing**: Multiple feature nodes can run in parallel from START
- **Tool Integration**: Automatic tool calling with `tools_condition`

### 6. **Enhanced Tool Integration**

**Tool Registry System:**

```python
# Knowledge Base Tools (Dynamic Registration)
for i, kb_id in enumerate(self.settings.kb.knowledge_base_id):
    retriever_tool_def = Tool(
        name=self.settings.kb.knowledge_base_name[i],
        func=partial(self.retriever_tool, kb_id),
        description=self.settings.kb.knowledge_base_description[i],
    )
    self.tool_registry_objects[self.settings.kb.knowledge_base_name[i]] = retriever_tool_def

# MCP (Model Context Protocol) Tools
if self.agent_config.MCPServers:
    mcp_config = {}
    for i, mcp_server in enumerate(self.agent_config.MCPServers):
        mcp_server_name = mcp_server.replace("{ENV}", self.settings.general.environment)
        mcp_config[f"server_{i}"] = {
            "url": f"https://{mcp_server_name}/mcp",
            "transport": "streamable_http",
        }
    client = MultiServerMCPClient(mcp_config)
    mcp_tools = await client.get_tools()
    for tool in mcp_tools:
        self.tool_registry_objects[tool.name] = tool

# Bind all tools to LLM
self.llm_with_tools = self.model_executor.init_chat_model(...).bind_tools(tools)
```

**Tool Execution Workflow:**

1. **Tool Detection**: `tools_condition` checks if LLM response contains tool calls
2. **Tool Routing**: `Agent.call_tool()` routes to appropriate handler:
   - **Knowledge Base Tools**: `_call_kb_tool()` with score filtering and citation formatting
   - **MCP Server Tools**: `_call_mcp_tool()` with JSON parsing and error handling
3. **Result Processing**: Tools return documents and formatted responses
4. **Citation Management**: Automatic reference numbering and URL generation

**Tool Processing Features:**

- **Knowledge Base Tools**: 
  - Amazon Bedrock Knowledge Base retrieval
  - Score threshold filtering (`minimum_score_for_documents`)
  - Automatic citation formatting with numbered references
  - Presigned URL generation for PDF access
- **MCP Server Tools**: 
  - External service integration via Model Context Protocol
  - JSON response parsing with error recovery
  - Citation extraction and formatting
  - Flexible response handling
- **Error Resilience**: Graceful handling of tool failures and malformed responses
- **Deduplication**: Automatic removal of duplicate documents from multiple tool calls

### 7. **Streaming Response & Message Processing**

**Dual Stream Mode Architecture:**

```python
# MessageProcessor handles dual stream processing
async for message_type, value in self.agent.astream(
    {
        "messages": [SystemMessage(prompt)] + previous_conversation + [HumanMessage(user_question)],
        "user_question": user_question,
    },
    stream_mode=["messages", "values"]
):
    if message_type == "updates":
        await self._process_update_message(value)
    elif message_type == "values" and isinstance(value, dict):
        async for message in self._process_values_message(value):
            yield message
    elif isinstance(value, tuple):
        async for message in self._process_message(value):
            yield message
```

**Stream Processing Types:**

- **"messages" Stream**: Real-time AI response chunks for live chat interface
- **"values" Stream**: Complete node outputs for metadata (citations, suggestions, etc.)
- **"updates" Stream**: Node completion notifications and state changes

**Response Generation Pipeline:**

1. **Agent Stream**: LangGraph generates dual streams simultaneously
2. **Message Processing**: MessageProcessor handles stream routing and formatting
3. **Citation Processing**: Automatic document citation with presigned URLs
4. **Memory Storage**: Response chunks stored in memory system
5. **Client Delivery**: Server-sent events stream to frontend

**Real-time Features:**

- **Live Typing**: Character-by-character response streaming
- **Progressive Enhancement**: Citations and suggestions appear as generated
- **Early Termination**: Contact center detection can stop processing
- **Error Recovery**: Graceful handling of stream interruptions

### 8. **Memory System Architecture**

**Dual Memory Implementation:**

```python
# AgentCore Memory (AWS Bedrock)
class AgentMemoryAgentCore:
    async def save_memory(self, session_context, document_content, span_id, author):
        # Semantic search enabled memory with user preference learning
        # Automatic context understanding and retrieval optimization
        
# DynamoDB Memory (Traditional)  
class AgentMemoryDynamoDB:
    async def save_memory(self, session_context, document_content, span_id, author):
        # Traditional conversation storage in DynamoDB
        # Direct message history without semantic understanding
```

**Memory Features:**

- **AgentCore**: Semantic search, user preference learning, context understanding
- **DynamoDB**: Traditional message storage, conversation history, session tracking
- **Runtime Switching**: Configuration-based memory type selection
- **Automatic Fallback**: DynamoDB backup when AgentCore unavailable

### 9. **Error Handling & Safety**

**Multi-layer Safety System:**

- **Guardrail Integration**: 
  - **Alinia Guardrails**: External enterprise safety service
  - **Bedrock Guardrails**: AWS native content filtering
  - **Pre-response Validation**: Input filtering before LLM processing
  
- **Tool Execution Safety**:
  - **Timeout Protection**: Tool calls with configurable timeouts
  - **Error Recovery**: Graceful degradation when tools fail
  - **JSON Parsing Resilience**: Multiple parsing strategies for MCP responses
  
- **Stream Interruption Handling**:
  - **Contact Center Detection**: Immediate transfer capability
  - **Error State Management**: Clean stream termination on failures
  - **Memory Consistency**: Proper state saving even during interruptions

**Error Recovery Patterns:**

```python
# Tool execution with error handling
try:
    result = await tool_fn.ainvoke(tool_args)
except Exception as e:
    logger.warning("Tool execution failed: %s", e)
    return {"messages": [AIMessage(content="I apologize, but I couldn't retrieve that information right now.")]}

# JSON parsing with fallback strategies
try:
    dict_result = json.loads(result)
except json.JSONDecodeError:
    try:
        dict_result = json.loads(result.replace("'", '"'))
    except:
        dict_result = {"result": result}  # Fallback to raw response
```

## Enhanced Benefits of Current LangGraph Architecture

### **Architectural Advantages**

- **🏗️ Modular Execution Framework**: Clear separation between orchestration (`AgentExecutor`), processing (`MessageProcessor`), memory (`AgentMemory`), and safety (`AgentGuardrails`)
- **🔄 Dynamic Configuration**: Runtime switching between memory types (AgentCore/DynamoDB) and guardrail systems (Alinia/Bedrock)
- **🛠️ Advanced Tool Integration**: Unified registry supporting both Knowledge Base and MCP server tools with automatic routing
- **🧠 Intelligent Memory**: Dual memory system with semantic search capabilities and traditional conversation storage
- **⚡ Performance Optimized**: Asynchronous tool calling, parallel node execution, and efficient stream processing
- **🔍 Observable**: Each component can be monitored, traced, and debugged independently
- **📈 Scalable**: Individual components can be optimized and scaled independently
- **⚙️ Configurable**: Feature nodes, memory types, and safety systems can be enabled/disabled via configuration

### **LangGraph-Specific Benefits**

- **🌊 Dual Stream Processing**: Simultaneous message and metadata streams for rich real-time experiences
- **🔀 Conditional Routing**: Smart tool calling, contact center detection, and feature enablement
- **🔗 State Persistence**: Enhanced conversation state management with multiple storage backends
- **🛡️ Safety Integration**: Multi-layer guardrail system with graceful degradation
- **🚀 Real-time Streaming**: Live response generation with progressive enhancement
- **🔧 Tool Ecosystem**: Extensible tool framework supporting enterprise integrations
- **📊 Analytics Ready**: Built-in tracing, span management, and performance monitoring
- **🎯 Enterprise Features**: MCP server integration, citation management, and document processing

### **Production-Ready Capabilities**

- **High Availability**: Automatic fallback mechanisms for memory and guardrail systems
- **Error Resilience**: Comprehensive error handling with graceful degradation strategies  
- **Security First**: Multi-layer content filtering and enterprise-grade safety controls
- **Monitoring**: Full observability with tracing, logging, and performance metrics
- **Scalability**: Modular design enables horizontal scaling of individual components
- **Maintainability**: Clear separation of concerns and well-defined interfaces
- **Integration Ready**: MCP protocol support for enterprise tool ecosystem integration
