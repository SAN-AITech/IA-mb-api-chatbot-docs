[← Back to Documentation Home](README.md)

# LangGraph & Agent Implementation

## LangGraph Flow Design

### 1. **State Management**

The application uses LangGraph's StateGraph to orchestrate the conversation flow:

```python
class State(MessagesState):
    documents: Annotated[List[Document], operator.add]
    user_question: str
    contact_center_answer: CCAnswer
    deep_links: Optional[List[str]] = None
    suggestions: Optional[SuggestionsAnswer] = None
    guardrail_applied: bool = False
```

### 2. **Graph Structure**

The conversation flow is implemented as a directed graph:

```
START → Document Retrieval → Contact Center Classification → LLM Response → END
                ↓
        Knowledge Base Search
                ↓  
           RAG Processing
```

### 3. **Key Components**

#### Document Retrieval Node
- Queries AWS Knowledge Base using user input
- Retrieves relevant documents for context
- Filters documents by relevance score
- Adds documents to state for LLM processing

#### Contact Center Classification Node  
- Classifies if user needs human agent transfer
- Uses specialized LLM prompt for intent detection
- Returns 'AGENT' or 'OTHER' classification
- Influences routing decisions

#### LLM Response Node
- Processes user question with retrieved documents
- Uses foundation model (Bedrock) for response generation
- Applies guardrails for content safety
- Generates contextual response with citations

### 4. **Agent Configuration**

The agent behavior is controlled via configuration:

```python
class AgentConfig:
    foundation_model: str
    knowledge_base_id: str
    guardrail_id: str
    prompt_template: str
    enable_suggestions: bool
    enable_contact_center: bool
```

### 5. **RAG Implementation**

**Retrieval Augmented Generation Process:**

1. **Query Processing**: User question analyzed and expanded
2. **Knowledge Retrieval**: Semantic search in Knowledge Base
3. **Document Ranking**: Results scored and filtered
4. **Context Building**: Top documents combined with conversation history
5. **Response Generation**: LLM generates answer using context
6. **Citation Generation**: Response includes source references

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

## Benefits of LangGraph Approach

- **Modular Design**: Each processing step is isolated and testable
- **State Persistence**: Conversation context maintained throughout flow
- **Flexible Routing**: Conditional logic based on classifications
- **Observable**: Each node can be monitored and debugged
- **Scalable**: Individual nodes can be optimized independently
