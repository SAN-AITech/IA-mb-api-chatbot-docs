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
- **Database schema**: Same DynamoDB table structure
- **Business logic flow**: Conversation processing logic intact
- **Configuration**: Environment variables and settings unchanged

### ⚠️ **What Changed**
- **Documentation references**: Line numbers in docs needed updating
- **Architecture clarity**: Controller layer is simpler than previously documented  
- **Development tooling**: New helper scripts and proxy configurations added (**not part of main application**)
- **Code organization**: Business logic more clearly separated between controller and service layers

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

### 3. **Documentation Updates**
- **Line References**: Main documentation (send-message-flow.md) has been updated
- **Cross-references**: Check any custom documentation for outdated line numbers

## 🔗 Updated Documentation Links

The following documentation has been updated to reflect current line numbers and improved organization:
- **[Send Message Flow](send-message-flow.md)** - Complete flow with updated references and detailed request model usage
- **[Request & Response Models](request-response-models.md)** - **New comprehensive models documentation**
- **Cross-references**: All internal documentation links updated

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

**Last Updated**: September 2025  
**Version**: Current main branch  
**Status**: ✅ Documentation fully synchronized with codebase
