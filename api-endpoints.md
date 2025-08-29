[← Back to Documentation Home](README.md)

# API Endpoints Documentation

## Overview

The chatbot API provides RESTful endpoints for managing conversations, sessions, feedback, and configuration.

## Base URL

```
http://localhost:8083
```

## Authentication

All endpoints require session validation via headers:
- `Session-Id`: Unique session identifier
- `Client-Id`: Optional client identifier

## Endpoints

### 1. Health & Status

#### GET `/health`
Health check endpoint

**Response:**
```json
{
  "status": 200
}
```

#### GET `/`
Root endpoint with API information

**Response:**
```json
{
  "message": "Welcome to ods Chatbot API",
  "health": "/health"
}
```

### 2. Session Management

#### POST `/sessions`
Create a new user session

**Request Body:**
```json
{
  "analytics": {
    "browser": "Chrome/120.0",
    "device": "Desktop",
    "pageUrl": "https://example.com/chat",
    "channel": "web"
  }
}
```

**Response:**
```json
{
  "sessionId": "uuid-string",
  "createdAt": "2025-01-01T12:00:00Z"
}
```

### 3. Conversation Management

#### POST `/conversations`
Create a new conversation

**Request Body:**
```json
{
  "title": "New Conversation"
}
```

**Response:**
```json
{
  "conversationId": "uuid-string",
  "title": "New Conversation", 
  "createdAt": "2025-01-01T12:00:00Z"
}
```

#### GET `/conversations`
Get user's conversation list

**Query Parameters:**
- `limit`: Number of conversations to return (default: 10)
- `offset`: Pagination offset (default: 0)

**Response:**
```json
{
  "conversations": [
    {
      "conversationId": "uuid-string",
      "title": "Previous Chat",
      "lastMessage": "How can I help?",
      "updatedAt": "2025-01-01T11:30:00Z"
    }
  ],
  "total": 5,
  "hasMore": false
}
```

#### GET `/conversations/{conversationId}`
Get specific conversation history

**Response:**
```json
{
  "conversationId": "uuid-string",
  "messages": [
    {
      "messageId": 1,
      "author": "human",
      "message": "Hello",
      "createdAt": "2025-01-01T12:00:00Z"
    },
    {
      "messageId": 2,
      "author": "assistant", 
      "message": "Hi! How can I help you?",
      "createdAt": "2025-01-01T12:00:05Z"
    }
  ]
}
```

### 4. Message Processing

#### POST `/conversations/{conversationId}/messages`
Send message and get AI response

**Request Body:**
```json
{
  "message": "What are your services?",
  "enableSuggestions": true
}
```

**Response (Server-Sent Events):**
Streaming response with multiple events:

```
event: message_start
data: {"responseId": "uuid"}

event: message_delta  
data: {"content": "I can help you with"}

event: message_delta
data: {"content": " various banking services..."}

event: message_complete
data: {
  "responseId": "uuid",
  "content": "Complete response text",
  "documents": [
    {
      "title": "Banking Services Guide",
      "url": "https://kb.example.com/services",
      "excerpt": "Our banking services include..."
    }
  ],
  "suggestions": [
    "Tell me about savings accounts",
    "How do I open an account?"
  ],
  "contactCenterClassification": "OTHER"
}
```

### 5. Feedback Management

#### POST `/conversations/{conversationId}/messages/{messageId}/feedback`
Submit feedback for AI response

**Request Body:**
```json
{
  "rating": "positive",
  "comment": "Very helpful response",
  "category": "accuracy"
}
```

**Response:**
```json
{
  "feedbackId": "uuid-string",
  "status": "submitted"
}
```

### 6. Configuration

#### GET `/configuration`
Get chatbot configuration

**Response:**
```json
{
  "features": {
    "suggestionsEnabled": true,
    "contactCenterEnabled": true,
    "feedbackEnabled": true
  },
  "limits": {
    "maxMessageLength": 1000,
    "maxConversationHistory": 50
  },
  "ui": {
    "theme": "default",
    "language": "en"
  }
}
```

## Error Responses

All endpoints return consistent error format:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid session ID format",
    "details": {
      "field": "sessionId",
      "expectedFormat": "uuid"
    }
  }
}
```

### Common Error Codes

- `VALIDATION_ERROR`: Request validation failed
- `SESSION_NOT_FOUND`: Invalid or expired session
- `CONVERSATION_NOT_FOUND`: Conversation doesn't exist
- `RATE_LIMIT_EXCEEDED`: Too many requests
- `INTERNAL_ERROR`: Server-side processing error
- `GUARDRAIL_VIOLATION`: Content blocked by safety filters

## Rate Limiting

- **Message Endpoint**: 10 requests per minute per session
- **Other Endpoints**: 100 requests per minute per session
- **Burst Allowance**: 20% above limit for short periods
