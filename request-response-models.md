[← Back to Documentation Home](README.md)

# Request & Response Models Documentation

## Overview

This document provides comprehensive documentation for all data models used in the IA MB API Chatbot system, focusing on request and response structures, validation rules, and data flow patterns.

## 📥 Request Models

### **SendSaveMessageRequest** - *[send_save_message_request.py](../chatbot_api/domain/requests/send_save_message_request.py)*

The primary request model for sending messages to the AI agent. This model encapsulates all necessary information for processing a user message through the conversation flow.

**Full File Reference**: *[chatbot_api/domain/requests/send_save_message_request.py](../chatbot_api/domain/requests/send_save_message_request.py)*

#### **Complete Model Structure**

```python
class SendSaveMessageRequest(BaseModel):
    """
    Represents a request object for sending and saving a message
    """
    input: Input = Field(default=None)
    context: Context = Field(default=None)
```

**Code Reference**: *[send_save_message_request.py#L60-L68](../chatbot_api/domain/requests/send_save_message_request.py#L60-L68)*

#### **Input Component**

```python
class Input(BaseModel):
    """
    Represents the input message to be sent
    """
    messageType: str = Field(default=None)    # Type of message being sent
    text: str                                 # User's actual message content
    options: Options = Field(default=None)    # Processing options
```

**Code Reference**: *[send_save_message_request.py#L18-L26](../chatbot_api/domain/requests/send_save_message_request.py#L18-L26)*

**Field Details:**
- **`messageType`**: Optional string indicating message category (e.g., "question", "command")
- **`text`**: **Required** - The actual user message content
- **`options`**: Configuration for response processing (streaming, suggestions)

#### **Options Subcomponent**

```python
class Options(BaseModel):
    """
    Represents the options for the input message
    """
    stream: bool        # Enable real-time streaming response
    suggestions: bool   # Generate follow-up question suggestions
```

**Code Reference**: *[send_save_message_request.py#L11-L17](../chatbot_api/domain/requests/send_save_message_request.py#L11-L17)*

**Field Details:**
- **`stream`**: Controls whether response is streamed in real-time via Server-Sent Events
- **`suggestions`**: Whether to generate suggested follow-up questions after response

#### **Context Component**

```python
class Context(BaseModel):
    """
    Represents the context for the conversation
    """
    execution: Execution = Field(default=None)
    system: System = Field(default=None)
```

**Code Reference**: *[send_save_message_request.py#L53-L59](../chatbot_api/domain/requests/send_save_message_request.py#L53-L59)*

#### **Execution Subcomponent**

```python
class Execution(BaseModel):
    """
    Represents the execution context for the conversation
    """
    interactionId: Optional[str] = Field(default=None)     # Unique interaction identifier
    sessionId: Optional[str] = Field(default=None)         # Session identifier
    conversationId: Optional[str] = Field(default=None)    # Conversation identifier
    virtualAgentId: Optional[str] = Field(default=None)    # Virtual agent identifier
```

**Code Reference**: *[send_save_message_request.py#L43-L51](../chatbot_api/domain/requests/send_save_message_request.py#L43-L51)*

**Field Details:**
- **`interactionId`**: Optional UUID for tracking specific interaction
- **`sessionId`**: Optional session identifier (can also come from headers)
- **`conversationId`**: Optional conversation ID (typically comes from URL path)
- **`virtualAgentId`**: Optional identifier for the virtual agent instance

#### **System Subcomponent**

```python
class System(BaseModel):
    """
    Represents the system information for the conversation
    """
    channel: Optional[str] = Field(default=None)           # Communication channel
    timezone: Optional[str] = Field(default=None)          # User's timezone
    turnCount: Optional[int] = Field(default=None)         # Conversation turn number
    sessionStartTime: Optional[str] = Field(default=None)  # ISO timestamp of session start
    locale: Optional[str] = Field(default=None)            # User's locale (e.g., "en-US")
```

**Code Reference**: *[send_save_message_request.py#L29-L40](../chatbot_api/domain/requests/send_save_message_request.py#L29-L40)*

**Field Details:**
- **`channel`**: Source channel (e.g., "web", "mobile", "api")
- **`timezone`**: User's timezone in IANA format (e.g., "America/New_York")
- **`turnCount`**: Number of conversation turns (used for context management)
- **`sessionStartTime`**: ISO 8601 timestamp when session began
- **`locale`**: User's language/region preference

### **Example Complete Request**

```json
{
  "input": {
    "messageType": "question",
    "text": "What's my account balance?",
    "options": {
      "stream": true,
      "suggestions": true
    }
  },
  "context": {
    "execution": {
      "interactionId": "uuid-12345",
      "sessionId": "session-67890",
      "conversationId": "conv-abc123",
      "virtualAgentId": "LuzIA_prod"
    },
    "system": {
      "channel": "web",
      "timezone": "Europe/Madrid",
      "turnCount": 3,
      "sessionStartTime": "2025-09-01T10:00:00Z",
      "locale": "es-ES"
    }
  }
}
```

## 📤 Response Models

### **SendMessageToAgentResponse** - *[send_message_to_agent_response.py](../chatbot_api/domain/responses/send_message_to_agent_response.py)*

The response model for streaming AI agent responses via Server-Sent Events.

**Full File Reference**: *[chatbot_api/domain/responses/send_message_to_agent_response.py](../chatbot_api/domain/responses/send_message_to_agent_response.py)*

#### **Response States**

```python
INITIAL_STATE = "init"      # First chunk of response
DELTA_STATE = "delta"       # Incremental content chunks  
END_STATE = "end"           # Response completion signal
```

**Code Reference**: *[send_message_to_agent_response.py#L14-L16](../chatbot_api/domain/responses/send_message_to_agent_response.py#L14-L16)*

#### **Response Structure**

```python
class SendMessageToAgentResponse(BaseModel):
    """
    Represents a response object for sending messages to an agent.
    """
    responseId: str           # Unique response identifier
    context: Context          # Response context information
    stream: Stream            # Streaming configuration
    output: Output            # Response content and metadata
```

**Code Reference**: *[send_message_to_agent_response.py#L158-L167](../chatbot_api/domain/responses/send_message_to_agent_response.py#L158-L167)*

## 🔄 Other Request Models

### **CreateGetConversationRequest** - *[create_get_conversation_request.py](../chatbot_api/domain/requests/create_get_conversation_request.py)*

Used for creating new conversations and pagination.

```python
class CreateGetConversationRequest(BaseModel):
    """Represents a request object for creating a conversation"""
    analytics: Analytics    # Analytics tracking information

class GetConversationPaginationRequest(BaseModel):
    """Represents a request object for paginating conversation results"""
    limit: int = Field(0, alias="_limit")           # Results per page
    page: int = Field(0, alias="_page")             # Page number
    sort: str = Field("-modifiedAt", alias="_sort") # Sort order

class SaveConversationRequest(BaseModel):
    """Represents a request object for saving a conversation"""
    topic: str                                      # Conversation topic/title
    conversationId: Optional[str] = None            # Optional conversation ID
```

**Code References**: 
- *CreateGetConversationRequest*: [create_get_conversation_request.py#L11-L16](../chatbot_api/domain/requests/create_get_conversation_request.py#L11-L16)
- *GetConversationPaginationRequest*: [create_get_conversation_request.py#L27-L38](../chatbot_api/domain/requests/create_get_conversation_request.py#L27-L38)
- *SaveConversationRequest*: [create_get_conversation_request.py#L18-L25](../chatbot_api/domain/requests/create_get_conversation_request.py#L18-L25)

### **FeedbackRequest** - *[feedback_request.py](../chatbot_api/domain/requests/feedback_request.py)*

Used for submitting user feedback on AI responses.

```python
class FeedbackRequest(BaseModel):
    """Represents a request object for feedback"""
    responseId: str                          # Target response for feedback
    scoring: Scoring                         # Feedback scoring object
    message: str                             # Feedback message content
    context: Optional[Context] = None        # Optional context information
```

**Code Reference**: *[feedback_request.py#L36-L45](../chatbot_api/domain/requests/feedback_request.py#L36-L45)*

## 🎯 Validation Rules

### **Automatic Validation (FastAPI + Pydantic)**

- **Required Fields**: `input.text` is the only strictly required field
- **Type Validation**: All fields automatically validated against declared types
- **Optional Fields**: Most fields have sensible defaults or can be None
- **Format Validation**: UUIDs, timestamps, and other formats validated automatically

### **Business Logic Validation**

- **Message Length**: Text messages validated for reasonable length
- **Session Context**: Missing session information handled gracefully
- **Conversation Ownership**: Security validation in service layer
- **Turn Count**: Automatically managed and incremented

## 🔗 Usage Patterns

### **Minimal Request**

```json
{
  "input": {
    "text": "Hello",
    "options": {
      "stream": true,
      "suggestions": false
    }
  }
}
```

### **Full Context Request**

Complete request with all optional fields populated for maximum context and tracking capability.

### **Stream vs Non-Stream**

- **`stream: true`**: Real-time response via Server-Sent Events
- **`stream: false`**: Complete response in single HTTP response

---

**Related Documentation:**
- **[Send Message Flow](send-message-flow.md)** - How these models are used in the conversation flow
- **[API Endpoints](api-endpoints.md)** - REST API usage examples
- **[Database Schema](database-schema.md)** - How request data is stored
