[← Back to Documentation Home](README.md)

# Architecture Decision Records (ADRs)

This document captures key architectural decisions made in the IA-mb-api-chatbot system, including context, trade-offs, and consequences.

## ADR-001: Parallel Processing vs Sequential Processing in LangGraph Workflow

### Status
**ACCEPTED** - Currently implemented

### Context
The system needs to process user messages through multiple AI components:
- Contact center intent classification (routing decisions)
- Main AI response generation (conversation handling)
- Enhancement services (deep links, suggestions)

Two architectural approaches were considered:
1. **Sequential Processing**: Check contact center first, then conditionally start other nodes
2. **Parallel Processing**: Start all nodes simultaneously

### Decision
**Parallel Processing Architecture** - All LangGraph nodes start processing simultaneously when a user message arrives.

### Implementation
```python
# From agent.py - LangGraph workflow construction
empty_initial 
    ├── chatbot → tools → chatbot → empty_final    ← Main conversation
    ├── check_cc_redirect → empty_final            ← ALWAYS RUNS (safety net)
    ├── deep_link_invoke → empty_final             ← IF ENABLED (navigation help)  
    └── suggestions → empty_final                  ← IF ENABLED (conversation guidance)
```

**Stream Processing**:
```python
# What happens on every message:
User sends message → LangGraph starts ALL parallel nodes:
├── check_cc_redirect → DSPy LLM call (contact center analysis)
├── deep_link_invoke → DSPy LLM call (link extraction)  
├── suggestions → DSPy LLM call (question generation)
└── chatbot → Main LLM call (response generation)
```

### Consequences

#### ✅ Positive Consequences

**Speed & Responsiveness**
- Sub-second response times across all scenarios
- Real-time streaming capabilities with immediate interruption
- Immediate transfer capability when human intervention needed

**User Experience**
- No waiting periods for sequential analysis
- Real-time chat experience similar to ChatGPT
- Instant routing decisions without blocking AI responses

**Development Simplicity**
- Parallel architecture easier to maintain than complex conditionals
- Clear separation of concerns between different AI services
- Simplified debugging with independent node execution

**System Reliability**
- Contact center analysis always runs as safety net
- Fault tolerance through independent node execution
- Graceful degradation when individual nodes fail

#### ❌ Negative Consequences

**Resource Consumption Inefficiency**

**The Core Problem**: When contact center analysis determines a user needs human intervention (`cc_answer == "AGENT"`), the system immediately transfers the user but **cannot cancel already-running LLM calls**.

**Resource Waste Scenarios**:

**Scenario 1: Transfer to Human Agent (30% of cases)**
```python
# User: "I want to cancel my account immediately!"
# ALL nodes start processing simultaneously:
# ✅ contact_center: "AGENT" (USED - triggers transfer)
# ❌ chatbot: Starts response generation (WASTED - stream interrupted)
# ❌ deep_links: Analyzes for links (WASTED - not needed for transfer)
# ❌ suggestions: Generates questions (WASTED - conversation ending)

if cc_answer == "AGENT":
    yield transfer_response  
    return  # ← Interrupts stream, but LLM resources already consumed
```

**Scenario 2: Early Contact Center Response**
```python
# If contact center node completes first (fast analysis):
# ✅ contact_center: "AGENT" → immediate transfer
# ❌ chatbot: 80% complete response generation (WASTED)
# ❌ enhancements: Partial processing (WASTED)
```

**Cost Impact Analysis**:
```
Daily Usage: 10,000 users
Transfer Rate: 30% (3,000 transfers daily)
Wasted Resources per Transfer: 3 LLM calls (chatbot + deep_links + suggestions)
Daily Waste: 9,000 LLM calls providing no user value
Monthly Waste: ~270,000 unnecessary LLM calls
```

**Technical Limitation**: The system processes **outputs** (in `_execute` method) but cannot control **agent flow** (in LangGraph execution). Resource consumption happens during agent execution, but interruption logic happens during output processing.

### Alternative Approaches Considered

#### Option 1: Sequential Processing (Not Implemented)
```python
# More efficient but slower approach:
1. Check contact center classification first
2. If "AGENT" → transfer immediately (no waste)  
3. If "OTHER" → start enhancement nodes conditionally
```

**Trade-offs**:
- ✅ Resource efficient (no wasted LLM calls)
- ❌ Slower response times (sequential delays)
- ❌ Complex conditional logic
- ❌ Reduced user experience responsiveness

#### Option 2: Conditional Node Activation (Not Implemented)
```python
# Conditional enhancement activation:
if cc_answer == "OTHER":
    start_enhancement_nodes()
else:
    skip_enhancements()
```

**Trade-offs**:
- ✅ Efficient resource usage
- ❌ Complex workflow management
- ❌ Requires significant LangGraph modifications
- ❌ Loss of parallel processing benefits

#### Option 3: Early Termination Signals (Advanced, Not Implemented)
```python
# Advanced cancellation system:
if cc_answer == "AGENT":
    cancel_running_nodes()
    cleanup_resources()
```

**Trade-offs**:
- ✅ Best of both worlds (speed + efficiency)
- ❌ Requires custom LangGraph modifications
- ❌ Complex implementation and testing
- ❌ Potential reliability risks with cancellation

### Key Insights

**Design Philosophy**: This is a **conscious architectural trade-off** where **real-time responsiveness and user experience** are prioritized over **resource optimization**.

**Business Justification**: 
- Customer satisfaction from instant responses outweighs LLM cost inefficiencies
- Transfer scenarios (30%) are critical customer service moments requiring immediate handling
- System simplicity reduces development and maintenance costs

**Technical Reality**: 
- LangGraph parallel execution model doesn't support dynamic cancellation
- Output processing happens after resource consumption
- Trade-off between system complexity and resource efficiency

### Monitoring & Future Considerations

**Current Monitoring**:
- Track transfer rates and resource usage patterns
- Monitor cost impact of parallel processing waste
- Measure user satisfaction and response times

**Future Optimization Opportunities**:
1. **LangGraph Enhancement**: If framework adds cancellation support
2. **Predictive Routing**: Early contact center signals to reduce waste
3. **Hybrid Approach**: Critical vs non-critical enhancement processing
4. **Cost Threshold**: Dynamic switching based on usage patterns

### Related Documentation
- [Execute Method Flow](execute-method-flow.md) - Implementation details
- [LangGraph Implementation](langgraph-implementation.md) - Workflow configuration
- [Performance Monitoring](performance-monitoring.md) - Resource usage tracking

---

## Future ADRs

Future architectural decisions will be documented here following the same format:
- ADR-002: [Topic]
- ADR-003: [Topic]
- etc.

### ADR Template

For future decisions, use this template:

```markdown
## ADR-XXX: [Decision Title]

### Status
[PROPOSED | ACCEPTED | REJECTED | SUPERSEDED]

### Context
[What is the issue that we're seeing that is motivating this decision or change?]

### Decision
[What is the change that we're proposing or have agreed to implement?]

### Consequences
[What becomes easier or more difficult to do and any risks introduced by the change?]

### Alternatives Considered
[What other options were considered and why were they not chosen?]
```
