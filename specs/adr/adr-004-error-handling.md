# ADR-004: API Error Handling & User Guidance

**Status:** Proposed  
**Date:** 2026-02-28  
**Decision Maker:** Product & Architecture Team  
**Affected Components:** FRD-006 (System Integration), All Features (FRD-001 through FRD-005)

---

## Context

The Cooking Agent must handle errors gracefully and provide clear user guidance when things go wrong. Three approaches were considered:

### Option A: Silent Failures (Not Acceptable)
- Errors logged but not communicating to user
- Leads to frustration and support issues
- Violates user trust

### Option B: Technical Error Messages (Not Ideal)
- Expose system errors directly to users
- "Database connection failed" or "JSON parsing error"
- Confuses non-technical users
- Not actionable

### Option C: User-Centric Error Handling (Recommended)
- **Clear, plain-English error messages**
- User-actionable recovery steps
- **Graceful degradation** where possible
- Detailed logging for debugging

---

## Decision

**Adopt Option C (User-Centric Error Handling) for MVP Phase 1.**

---

## Rationale

### User Experience Priority
- **Clarity:** Users understand what went wrong
- **Guidance:** Users know what to do next
- **Trust:** System behaves predictably
- **Support:** Fewer support tickets from confused users

### Business Metrics
- Error recovery rate (how many users retry successfully)
- Conversation continuation rate (avoid abandonment on error)
- User satisfaction (4/5 star target)

### Implementation Feasibility
- Map system errors → user-friendly messages
- Include actionable suggestions
- Log technical details for debugging
- Minimal overhead

---

## Error Handling Framework

### Error Classification

#### **Tier 1: Critical Feature Errors** (Block Conversation)
- Feature: Ingredient Input (FRD-001), AI Recommendations (FRD-002)
- Impact: Core functionality broken
- Response: Return error, ask user to retry or try different input

#### **Tier 2: Secondary Feature Errors** (Partial Degradation)
- Feature: Recipe Filtering (FRD-003)
- Impact: Feature unavailable but conversation continues
- Response: Return partial results with explanation

#### **Tier 3: Non-Critical Feature Errors** (Silent Graceful Failure)
- Feature: Storage (FRD-005), Logging
- Impact: No user-visible consequence
- Response: Log error, retry in background, don't inform user

#### **Tier 4: Integration Errors** (System-Level)
- External service failures (Azure AI Foundry timeout, database down)
- Impact: Multiple features affected
- Response: Graceful degradation with explanation

---

## Error Message Framework

### Error Response Format

```json
{
  "success": false,
  "error": {
    "userMessage": "Clear, friendly message for user",
    "suggestions": ["Action 1", "Action 2", "Action 3"],
    "retryable": true | false,
    "code": "ERROR_CODE_FOR_LOGGING"
  },
  "sessionId": "session-abc-123",
  "metadata": {
    "timestamp": "2026-02-28T10:00:05Z",
    "technicalDetails": "Logged but not shown to user"
  }
}
```

### Mapping: System Errors → User Messages

#### **FRD-001 Errors (Ingredient Processing)**

| System Error | User Message | Suggestions |
|--------------|--------------|-------------|
| Empty input | "Please tell me what ingredients you have." | 1. Try listing a few ingredients<br/>2. Example: "chicken, rice, peppers" |
| Ambiguous ingredient | "I'm not sure what 'xyz' is. Did you mean...?" | 1. Select from suggestions<br/>2. Spell it differently |
| Invalid characters | "I didn't understand that. Try using ingredient names." | 1. Check spelling<br/>2. Use common ingredient names |

#### **FRD-002 Errors (AI Recommendations)**

| System Error | User Message | Suggestions |
|--------------|--------------|-------------|
| No recipes match | "I couldn't find recipes with those exact ingredients.<br/>Try adding more ingredients or different ones." | 1. Add more ingredients<br/>2. Try common ingredients<br/>3. Start over |
| AI model timeout | "I'm taking too long to search. Please try again." | 1. Click Retry<br/>2. Try with fewer ingredients<br/>3. Try again in a moment |
| AI model unavailable | "I'm temporarily unable to search recipes. Try again soon." | 1. Retry in 1 minute<br/>2. Try fewer ingredients |

#### **FRD-003 Errors (Filtering)**

| System Error | User Message | Suggestions |
|--------------|--------------|-------------|
| No recipes after filter | "No recipes match all your filters.<br/>Here's what I found with easier filters:" | 1. Remove difficulty filter<br/>2. Remove time filter<br/>3. See all recipes |
| Filter service down | "Showing all recipes (filters temporarily unavailable)." | 1. Try filtering again<br/>2. Browse all recipes |

#### **FRD-004 Errors (Conversation)**

| System Error | User Message | Suggestions |
|--------------|--------------|-------------|
| Unclear intent | "I'm not sure what you're asking.<br/>Would you like to:" | 1. Search for new recipes<br/>2. See details about a recipe<br/>3. Apply filters<br/>4. Start over |
| Session expired | "Your session expired. Would you like to start a new search?" | 1. Start new search<br/>2. Continue with different ingredients |

#### **FRD-005 Errors (Storage)**

| System Error | User Message | Suggestions |
|--------------|--------------|-------------|
| Cannot save | (No message - silent retry) | (Background retry) |
| Cannot retrieve history | "I'm having trouble retrieving your history." | 1. Retry<br/>2. Start fresh search |

#### **System-Level Errors**

| Scenario | User Message | Suggestions |
|----------|--------------|-------------|
| Database down | "I'm having technical issues. Try again in a moment." | 1. Retry<br/>2. Try later |
| All services down | "The service is temporarily unavailable.<br/>Please try again soon." | 1. Try again in a few minutes<br/>2. Check back later |

---

## Implementation Guidelines

### Error Message Principles

1. **Plain Language**
   - ❌ "NLP parsing failed for input tokenization"
   - ✅ "I didn't understand that ingredient. Could you spell it differently?"

2. **Never Blame the User**
   - ❌ "You didn't provide valid ingredients"
   - ✅ "I'm having trouble understanding those ingredients."

3. **Provide Path Forward**
   - ❌ "Error: 500"
   - ✅ "Something went wrong. Click Retry to try again."

4. **Keep It Short**
   - ❌ "The Azure AI Foundry recommendation service has encountered a timeout due to excessive load on the vector database lookup operation..."
   - ✅ "I'm taking too long. Please try again."

5. **Be Honest About Issues**
   - ✅ "I'm temporarily unable to search. Try again in a moment."
   - ✅ "That didn't work out. Want to try something else?"

### Error Logging (Technical Details)

```json
{
  "timestamp": "2026-02-28T10:00:05Z",
  "sessionId": "session-abc-123",
  "level": "ERROR",
  "feature": "FRD-002",
  "action": "recipe_search",
  "error": {
    "code": "AI_MODEL_TIMEOUT",
    "message": "Azure AI Foundry recommendation request exceeded 2 second timeout",
    "details": {
      "attemptedIngredients": ["chicken", "rice"],
      "responseTime": "2100ms",
      "retryCount": 0
    }
  },
  "userExperienced": false
}
```

### Error Metrics to Track

```
- Error rate by type (FRD-001, FRD-002, FRD-003, etc.)
- User retry rate after error
- Conversation abandonment after error
- Most common errors (prioritize fixes)
- Error recovery success rate
```

---

## Specific Feature Error Handlers

### FRD-001: Ingredient Processing Error Handler

```typescript
function handleIngredientProcessingError(error, userInput) {
  if (error.type === 'EMPTY_INPUT') {
    return {
      userMessage: "Please tell me what ingredients you have.",
      suggestions: ["chicken, rice, peppers", "pasta, tomato sauce", "eggs, cheese"],
      retryable: true
    };
  }
  if (error.type === 'UNRECOGNIZED_INGREDIENT') {
    return {
      userMessage: `I'm not sure what "${error.ingredient}" is. Did you mean one of these?`,
      suggestions: error.suggestions,
      retryable: true
    };
  }
  // Default error
  return {
    userMessage: "I had trouble understanding your ingredients. Please try again.",
    suggestions: ["Try spelling it differently", "Use common ingredient names"],
    retryable: true
  };
}
```

### FRD-002: AI Recommendations Error Handler

```typescript
function handleRecommendationError(error) {
  if (error.code === 'AI_MODEL_TIMEOUT') {
    return {
      userMessage: "I'm taking too long to search. Please try again.",
      suggestions: ["Retry", "Try with fewer ingredients", "Try different ingredients"],
      retryable: true
    };
  }
  if (error.code === 'NO_RECIPES_FOUND') {
    return {
      userMessage: "I couldn't find recipes with those ingredients. Try adding more.",
      suggestions: ["Add more ingredients", "Try different ingredients", "Get inspiration"],
      retryable: true
    };
  }
  if (error.code === 'AI_MODEL_UNAVAILABLE') {
    return {
      userMessage: "I'm temporarily unable to search recipes. Try again soon.",
      suggestions: ["Retry", "Try again in 1 minute"],
      retryable: true
    };
  }
}
```

### FRD-003: Filtering Error Handler

```typescript
function handleFilteringError(error, lastRecommendations) {
  if (error.code === 'NO_RESULTS_AFTER_FILTER') {
    return {
      userMessage: "No recipes match all your filters. Here's what I found with easier filters:",
      data: suggestAlternativeFilters(lastRecommendations),
      suggestions: ["Remove difficulty filter", "Remove time filter", "See all recipes"],
      retryable: false // Retry filtering won't help
    };
  }
}
```

---

## Error Recovery Strategies

### Automatic Retries (Backend)
- **FRD-005 (Storage):** 3 retries with exponential backoff (silent)
- **FRD-002 (AI Model):** 1 automatic retry on timeout
- **Other features:** No automatic retries

### User-Initiated Retries
- Show "Retry" button in error message
- Clear session to retry fresh (if needed)
- Offer alternative approaches if retry fails

### Graceful Degradation
- **No recipes → No filters:** Show unfiltered results
- **Storage fails → Continue conversation:** Don't lose user context
- **Feature timeout → Use cached results:** Serve previous recommendations while retrying

---

## Consequences

### Positive
✅ Clear user experience
✅ High conversation continuation rate
✅ Fewer support issues
✅ Easier debugging with detailed logging
✅ User trust in system reliability

### Negative
❌ More error scenarios to handle
❌ Requires careful message crafting
❌ Error messages maintenance overhead

### Risks & Mitigations

| Risk | Mitigation |
|------|-----------|
| Unhelpful suggestions | User test error messages; iterate based on feedback |
| Over-logging | Implement log rotation and archival policy |
| Cascading errors | Limit retry attempts; timeout after 3 retries |

---

## Approved By

- [ ] Product Manager - _____ Date _____
- [ ] Technical Lead - _____ Date _____
- [ ] Architecture Team - _____ Date _____

---

## Related Documents

- [FRD-006: System Integration § Error Handling](../features/system-integration-and-orchestration.md) § 7.2
- [TECHNICAL_REVIEW_SUMMARY.md](../../TECHNICAL_REVIEW_SUMMARY.md) § Error Handling Requirements
- All Feature RDs (FRD-001 through FRD-005) § Error Handling sections

---

## References

- **Error Handling Best Practices:** Microsoft Error Handling Guide
- **User Experience:** Nielsen Norman Group - Error Messages
- **Logging Standards:** ELK Stack / Structured Logging Guide

