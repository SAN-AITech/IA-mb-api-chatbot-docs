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
        # Initialize memory type based on configuration
        if self.settings.chat.use_agentcore_memory:
            self.agent_memory = AgentMemoryAgentCore()
        else:
            self.agent_memory = AgentMemoryDynamoDB()
        
        # Initialize guardrails
        if self.settings.guardrails.use_alinia_guardrails:
            self.agent_guardrails = AgentGuardrailsAlinia()
        else:
            self.agent_guardrails = AgentGuardrailsBedrock(self.service_aws)
    
    async def init_agent(self):
        tracer_provider = TracerProvider()
        self.agent = Agent(self.suggestions, tracer_provider, self.agent_guardrails)
        await self.agent.init_llm_models()
```

**Current Graph Flow:**

```
START → [Conditional Branches] → END
     ↓
  chatbot → tools → chatbot
     ↓
check_cc_redirect (if enabled)
     ↓  
deep_link_invoke (if enabled)
     ↓
suggestions (if enabled)
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
    client = MultiServerMCPClient(mcp_config)
    mcp_tools = await client.get_tools()
    for tool in mcp_tools:
        self.tool_registry_objects[tool.name] = tool
```

**Tool Processing Features:**

- **Knowledge Base Tools**: Amazon Bedrock Knowledge Base integration with score filtering
- **MCP Server Tools**: External service integration via Model Context Protocol
- **Citation Support**: Automatic citation generation for knowledge base responses
- **Error Resilience**: Graceful handling of tool failures

### 6. **Streaming Response**

The system supports real-time streaming via Server-Sent Events:

```python
async def stream_response():
    for chunk in agent.stream_process(user_input):
        yield {
            "event": "message_delta",
            "data": chunk.content
        }
    yield {
        "event": "message_complete", 
        "data": final_response
    }
```

### 7. **Tools Integration**

The agent can use various tools:

- **Knowledge Base Retriever**: RAG document search
- **Contact Center Classifier**: Intent classification
- **Suggestion Generator**: Follow-up question generation
- **Guardrail Validator**: Content safety checking

### 8. **Error Handling**

- **Guardrail Violations**: Content blocked and logged
- **Knowledge Base Failures**: Graceful degradation to general responses
- **LLM Timeouts**: Retry logic with backoff
- **State Corruption**: Recovery mechanisms implemented

## Enhanced Benefits of Current LangGraph Approach

- **Modular Execution Framework**: Clear separation between orchestration, memory, guardrails, and processing
- **Dynamic Configuration**: Runtime switching of memory types and optional features  
- **Advanced Tool Integration**: Support for both Knowledge Base and MCP server tools
- **State Persistence**: Enhanced memory with semantic search capabilities
- **Flexible Routing**: Conditional logic based on classifications and configurations
- **Performance Optimized**: Asynchronous tool calling and threaded memory operations
- **Observable**: Each node can be monitored and debugged independently
- **Scalable**: Individual components can be optimized independently
- **Configurable**: Feature nodes can be enabled/disabled via configuration
