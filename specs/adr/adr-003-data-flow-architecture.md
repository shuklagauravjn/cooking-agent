# ADR-003: Feature Data Flow Architecture

**Status:** Proposed  
**Date:** 2026-02-28  
**Decision Maker:** Architecture Team  
**Affected Components:** All FRDs (FRD-001 through FRD-005)

---

## Context

The Cooking Agent comprises 5 interconnected features (FRD-001 through FRD-005) that must work together seamlessly. The data flowing through these features must be:

1. **Consistent** - Same data format throughout
2. **Traceable** - Full request context preserved end-to-end
3. **Resilient** - Failures in one feature don't cascade to others
4. **State-Preserving** - Multi-turn conversation context maintained

Two architectural approaches were considered:

### Option A: Orchestrator Pattern (Recommended for MVP)
- Conversational Interface (FRD-004) acts as central orchestrator
- Features called explicitly based on detected user intent
- Simple synchronous request/response
- State managed in session context
- Suitable for <1,000 concurrent users

### Option B: Event-Driven Architecture
- Features communicate via events (message queue)
- Scalable to high concurrency
- Loose coupling between features
- Requires message infrastructure (RabbitMQ, Kafka)
- Over-engineered for MVP

---

## Decision

**Adopt Option A (Orchestrator Pattern) for MVP Phase 1.**

---

## Rationale

### MVP Scope & Constraints
- **Concurrent Users:** 1,000 (orchestrator pattern sufficient)
- **Latency Requirement:** <2 seconds (synchronous acceptable)
- **Operational Complexity:** Minimal
- **Development Timeline:** 6-7 weeks (synchronous simpler to build/test)

### Simplicity & Velocity
- Synchronous request/response easier to reason about
- Debugging: full call stack visible
- Testing: no async/timing issues
- Performance sufficient for MVP scale

### Feature Independence
- Each feature has single responsibility
- Easy to test in isolation
- Failures don't cascade (proper error handling)
- Feature reuse in other contexts

---

## Implementation: Data Flow Specification

### Architecture Diagram

```
┌─────────────────────────────────────────────┐
│  User Input (Text Message)                  │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│  FRD-004: Conversational Interface          │
│  ├─ Parse message                           │
│  ├─ Recognize intent                        │
│  ├─ Retrieve session context                │
│  └─ Route to appropriate feature(s)         │
└──────────────────┬──────────────────────────┘
                   │
      ┌────────────┼────────────┬─────────────┐
      │            │            │             │
      ▼            ▼            ▼             ▼
  FRD-001     FRD-002      FRD-003      (None)
  ├─ Parse    ├─ Search   ├─ Filter    └─ Continue
  │           │           │
  └─ Normalize└─ Rank     └─ Reorder
      │            │            │
      └────────────┼────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│  FRD-005: Store Results                     │
│  ├─ Save conversation turn                  │
│  ├─ Store recommendations                   │
│  └─ Log interactions                        │
└─────────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│  Return Response to User                    │
└─────────────────────────────────────────────┘
```

### Request Flow: Step-by-Step

**1. User Message Received**
```json
{
  "message": "I have chicken, rice, and peppers",
  "sessionId": "session-abc-123"
}
```

**2. Parse & Intent Recognition (FRD-004)**
```json
{
  "detectedIntent": "ingredient_submission",
  "normalizedInput": "I have chicken, rice, and peppers",
  "sessionId": "session-abc-123",
  "timestamp": "2026-02-28T10:00:05Z"
}
```

**3. Route to FRD-001 (Ingredient Processing)**
```json
{
  "sessionId": "session-abc-123",
  "rawInput": "I have chicken, rice, and peppers"
}
│
└─→ Response:
{
  "ingredients": [
    {"name": "chicken", "confidence": 100},
    {"name": "rice", "confidence": 100},
    {"name": "bell_pepper", "confidence": 95}
  ],
  "sessionId": "session-abc-123"
}
```

**4. Route to FRD-002 (AI Recommendations)**
```json
{
  "ingredients": [
    {"name": "chicken", "confidence": 100},
    {"name": "rice", "confidence": 100},
    {"name": "bell_pepper", "confidence": 95}
  ],
  "sessionId": "session-abc-123",
  "resultCount": 5
}
│
└─→ Response:
{
  "recommendations": [
    {
      "recipe": { "id": "recipe-001", "name": "Chicken Stir Fry", ... },
      "matchScore": 95,
      "reasoning": "Uses 3/3 of your ingredients..."
    },
    { /* ... */ }
  ],
  "sessionId": "session-abc-123"
}
```

**5. Store in FRD-005 (Persistence)**
```json
{
  "sessionId": "session-abc-123",
  "turn": {
    "userMessage": "I have chicken, rice, and peppers",
    "agentResponse": "Great! Here are 5 recipes...",
    "recommendations": [ /* recipes */ ],
    "timestamp": "2026-02-28T10:00:08Z"
  }
}
│
└─→ Success or error
```

**6. Return Response**
```json
{
  "agentResponse": "Great! Here are 5 recipes using chicken, rice, and peppers.",
  "recommendations": [ /* recipe list */ ],
  "nextActions": ["Filter by difficulty", "Tell me more about the first recipe"],
  "sessionId": "session-abc-123"
}
```

---

## Session Context Preservation

### Session State Structure

```json
{
  "sessionId": "session-abc-123",
  "ingredients": [
    {"name": "chicken", "confidence": 100},
    {"name": "rice", "confidence": 100},
    {"name": "bell_pepper", "confidence": 95}
  ],
  "lastRecommendations": [
    {
      "id": "recipe-001",
      "name": "Chicken Stir Fry",
      "matchScore": 95
    }
  ],
  "activeFilters": {
    "difficulty": "easy"
  },
  "conversationHistory": [
    {
      "role": "user",
      "content": "I have chicken, rice, and peppers",
      "timestamp": "2026-02-28T10:00:05Z"
    },
    {
      "role": "agent",
      "content": "Great! Here are 5 recipes...",
      "timestamp": "2026-02-28T10:00:08Z"
    }
  ],
  "metadata": {
    "createdAt": "2026-02-28T10:00:00Z",
    "lastActivityAt": "2026-02-28T10:00:08Z",
    "turnCount": 1
  }
}
```

### Context Flow Rules
1. **Load Context:** Session loaded from FRD-005 at request start
2. **Pass to Features:** Each feature receives sessionId and prior context
3. **Update Context:** Feature returns updated data (new ingredients, recommendations, etc.)
4. **Store Context:** Updated state persisted in FRD-005 after response
5. **Return to User:** Response + context ready for next turn

---

## Error Handling Strategy

### Error Propagation

```
Feature Error
    │
    ├─→ Is Feature Optional? 
    │       ├─ YES: Silently skip, continue (e.g., FRD-005 storage)
    │       └─ NO: Return error to user
    │
    └─→ Critical Feature?
            ├─ YES (FRD-001, FRD-002): Return error message + prompt retry
            └─ NO (FRD-003): Return partial results with note
```

### Specific Error Scenarios

**FRD-001 Fails (Ingredient Parsing):**
```json
{
  "success": false,
  "error": "Unable to understand ingredients. Please try again.",
  "suggestions": ["Try listing ingredients separated by commas"],
  "sessionId": "session-abc-123"
}
```
→ User can retry with different phrasing

**FRD-002 Fails (AI Recommendations):**
```json
{
  "success": false,
  "error": "Unable to find recipes right now. Please try again.",
  "sessionId": "session-abc-123"
}
```
→ Retry suggested; don't persist partial results

**FRD-003 Fails (Filtering):**
```json
{
  "success": true,
  "data": {
    "recommendations": [ /* unfiltered */ ],
    "warning": "Filter not applied due to system issue. Showing all recipes."
  },
  "sessionId": "session-abc-123"
}
```
→ Return unfiltered results; user can retry filter

**FRD-005 Fails (Storage):**
```
// Silent failure - log error, continue conversation
// User experience: conversation works, just not persisted (retry in background)
```

---

## Concurrency & State Management

### Single-Session Concurrency
- One active message per session at a time
- If new request arrives while previous is processing: 
  - Option 1: Queue and process sequentially
  - Option 2: Reject with "Processing previous message"
  - Recommendation: Option 1 for MVP

### Multi-Session Concurrency
- 1,000 concurrent sessions allowed
- Each session independent (no cross-session state sharing)
- Minimal lock contention (file-based storage)

### State Consistency
- Session state NOT loaded from cache
- Always load latest from FRD-005 storage
- After each feature call, state updated
- Before returning to user, full context persisted

---

## Consequences

### Positive
✅ Simple, synchronous request/response model
✅ Full request tracing via sessionId
✅ Easy to reason about and debug
✅ Testing straightforward (no async complexity)
✅ Sufficient for MVP scale (1,000 users)

### Negative
❌ Synchronous: blocking per-request
❌ Not horizontally scalable beyond ~5,000 users
❌ Tight coupling in orchestrator
❌ Sequential feature calls (cannot parallelize)

### Risks & Mitigations
| Risk | Mitigation |
|------|-----------|
| Cascading failures | Implement circuit breaker per feature |
| Orchestrator bottleneck | Phase 2: event-driven architecture |
| Request timeout | Set aggressive timeout (500ms per feature) |
| Context bloat | Limit conversation history to 25 turns |

---

## Transition to Phase 2

**Scale-Up Trigger:** Reaching 5,000 concurrent users  
**Phase 2 Pattern:** Event-Driven Architecture with message queue (RabbitMQ/Kafka)
- Decoupled features (publish events)
- Parallelization of non-dependent features
- Easier horizontal scaling
- Complexity trade-off acceptable at scale

---

## Approved By

- [ ] Product Manager - _____ Date _____
- [ ] Technical Lead - _____ Date _____
- [ ] Architecture Team - _____ Date _____

---

## Related Documents

- [FRD-006: System Integration & Orchestration](../features/system-integration-and-orchestration.md) § 3-7
- [TECHNICAL_REVIEW_SUMMARY.md](../../TECHNICAL_REVIEW_SUMMARY.md) § Architecture Alignment
- All FRDs (FRD-001 through FRD-005)

---

## References

- **Orchestrator Pattern:** https://www.enterpriseintegrationpatterns.com/patterns/messaging/
- **CQRS Pattern:** For future Phase 2 scaling consideration

