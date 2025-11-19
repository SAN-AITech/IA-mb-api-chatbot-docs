[← Back to Documentation Home](README.md)

# Recent Codebase Changes

## Overview

This document tracks the significant changes made to the IA MB API Chatbot codebase in the latest version, including new files, moved code sections, and updated references.

## 📁 New Files Added

### 1. **`test_api.py`** - Development Testing Server
- **Purpose**: Simple FastAPI server for testing connectivity and basic API functionality
- **Location**: Root directory
- **Endpoints**: 
  - `GET /` - Basic health check
  - `GET /health` - Service health status  
  - `GET /test` - Test endpoint verification
  - `GET /docs` - Auto-generated API documentation
- **Port**: 8082 (to avoid conflicts with main application)
- **Usage**: Development and WSL connectivity testing

### 2. **`scripts/discover-tables.sh`** - DynamoDB Discovery Tool
- **Purpose**: Automated script to discover and list DynamoDB table names
- **Features**:
  - Environment variable detection
  - AWS CLI integration for table listing
  - Configuration file scanning
  - Developer guidance for database exploration
- **Usage**: `./scripts/discover-tables.sh`

### 3. **`web/proxy.conf.local.json`** - Local Development Proxy
- **Purpose**: Proxy configuration for local development environment
- **Key Endpoints**:
  - `/oauth/token` → External authentication service
  - `/api` → Local backend at `localhost:8083`
  - `/uploadsas` → External ingestion service
  - `/docs-convassistants` → External documentation service
- **Usage**: Angular development server proxy configuration

## 🔄 Code Movement & Architecture Simplification

### **Major Architecture Evolution: Memory System**

#### **🧠 New: Dual-Memory Architecture Implementation**
- **AgentCore Memory System**: New AWS Bedrock AgentCore integration for intelligent memory management
- **Location**: `/src/ia_mb_api_chatbot/services/agent_execution/agent_memory.py`
- **Classes**: 
  - `AgentMemoryAgentCore`: Semantic search and user preference learning
  - `AgentMemoryDynamoDB`: Traditional DynamoDB-based memory (maintained for compatibility)
  - `SaveThreadMemory`: Threaded, dual-storage memory persistence

#### **🏗️ Agent Execution Framework Modularization**
- **New Module**: `/src/ia_mb_api_chatbot/services/agent_execution/`
- **Components**:
  - `AgentExecutor`: Main orchestration and initialization
  - `MessageProcessor`: Streaming and message handling
  - `AgentGuardrails`: Safety and compliance checking
- **Benefits**: Clear separation of concerns, better testability, enhanced maintainability

#### **⚙️ Configuration Management Evolution**
- **AWS Parameter Store Primary**: Configuration now loads automatically from AWS Parameter Store using `ok-config`
- **Optional .env Override**: `.env` file now only overrides specific parameters, all others load from Parameter Store
- **No Manual Setup Required**: Database tables, models, guardrails automatically configured via Parameter Store
- **Parameter Store Console**: Developers can explore parameters at [AWS Parameter Store Console](https://eu-west-1.console.aws.amazon.com/systems-manager/parameters/?region=eu-west-1&tab=Table)

#### **🌐 Frontend Integration Changes**
- **Integrated Frontend**: UI now available at `/gui` path (NiceGUI-based, not separate Angular server)
- **Single Server**: No need for separate `ng serve` - frontend integrated into main FastAPI application
- **API Documentation**: Available at `/docs` path (Swagger UI integrated)
- **Static Assets**: Served at `/static/*` path

### Controller Layer (`chatbot_api/controllers/conversation.py`)
- **`send_message_to_agent` endpoint**: Moved from **L98** → **L109** (+11 lines)
- **⚠️ Important Discovery**: The controller is now **significantly simpler** than previously documented
- **Current Implementation**: Pure routing layer with no business logic
- **Architecture Change**: All validation and processing moved to service layer

**Actual Current Code**:
```python
async def send_message_to_agent(conversation_id: str, request: SendSaveMessageRequest, 
                               client_id: Optional[str] = Depends(get_client_id),
                               session_id: Optional[str] = Depends(get_session_id)) -> EventSourceResponse:
    return await conversation_service.process_user_input(client_id, session_id, conversation_id, request)
```

### Service Layer (`chatbot_api/services/conversation.py`)  
- **`process_user_input` method**: Moved from **L251** → **L259** (+8 lines)
- **`_execute` method**: Moved from **L322** → **L346** (+24 lines)
- **Business Logic**: All validation, security, and processing logic concentrated here

### Database Interaction Points
- **`get_conversation_last_interaction` call**: Moved from **L272** → **L280** (+8 lines)
- **Client authorization check**: Moved from **L273-L281** → **L281-L289** (+8 lines)

## 📋 Impact Assessment

### ✅ **What Remains Unchanged**

- **Core API functionality**: All endpoints work the same way
- **Request/response models**: No changes to data structures
- **Database schema**: Same DynamoDB table structure (enhanced, not replaced)
- **Business logic flow**: Conversation processing logic intact
- **LangGraph Foundation**: Core orchestration framework maintained

### ⚠️ **What Changed**

- **🆕 Configuration Source**: **Primary change** - AWS Parameter Store now primary configuration source, .env optional override only
- **🆕 Frontend Integration**: **Major change** - UI integrated at `/gui` path, no separate Angular server needed
- **🆕 API Documentation**: **Built-in** - Swagger UI available at `/docs` path
- **🆕 Memory Architecture**: **Enhanced** - dual-tier memory system with AgentCore integration
- **🆕 Agent Execution**: **Modularized** - agent execution framework for better maintainability
- **🆕 Tool Integration**: **Enhanced** - support for MCP servers and dynamic tool registry

### 🛠️ **Developer Impact**

#### **Configuration Changes**
- **No Manual Environment Setup**: Configuration loads automatically from Parameter Store
- **Optional .env Override**: Only use .env for specific parameter overrides during development
- **Parameter Exploration**: Use [AWS Parameter Store Console](https://eu-west-1.console.aws.amazon.com/systems-manager/parameters/?region=eu-west-1&tab=Table)

#### **Development Workflow Changes**
- **Single Server**: Start only one server with `uv run src/ia_mb_api_chatbot/run.py`
- **Integrated Frontend**: Access UI at `http://localhost:8082/gui` (no separate Angular server)
- **Built-in Documentation**: API docs at `http://localhost:8082/docs`

## 🛠️ Developer Actions Required

### 1. **Update IDE Bookmarks**
If you have saved line number bookmarks, update them:
- Controller endpoint: L98 → L109
- Main service method: L251 → L259  
- Core execution: L322 → L346

### 2. **Use New Development Tools**
- **Database Discovery**: Run `./scripts/discover-tables.sh` to explore your DynamoDB tables
- **Connectivity Testing**: Use `python test_api.py` for basic API testing
- **Local Proxy**: Update Angular development to use `proxy.conf.local.json`

### 3. **New Memory System Configuration**

Add these to your environment/config for AgentCore memory:

```yaml
chat:
  use_agentcore_memory: true        # Enable AgentCore memory (false = DynamoDB only)
  agentcore_memory_id: "memory-123" # AgentCore memory identifier
```

### 4. **Updated Documentation References**

Updated documentation reflecting current architecture:
- **[Memory Management](memory-management.md)** - **🆕 Enhanced** with dual-memory system documentation
- **[LangGraph Implementation](langgraph-implementation.md)** - **🆕 Updated** with modular execution framework
- **Cross-references**: All internal documentation links updated for current line numbers

## 🎯 Migration Notes

### For Developers
- **Debugging**: Update any debugging scripts that reference specific line numbers
- **Code Review**: New line numbers for code review references
- **Testing**: Leverage new `test_api.py` for development testing

### For Operations  
- **Monitoring**: No changes to runtime behavior or monitoring requirements
- **Deployment**: No changes to deployment procedures
- **Configuration**: No changes to environment variables or config files

---

**Last Updated**: November 2025  
**Version**: Current main branch  
**Status**: ✅ Documentation fully synchronized with codebase including memory system evolution
