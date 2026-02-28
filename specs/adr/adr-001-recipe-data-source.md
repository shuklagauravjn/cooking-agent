# ADR-001: Recipe Data Source Strategy (Phase 1)

**Status:** Proposed  
**Date:** 2026-02-28  
**Decision Maker:** Product & Architecture Team  
**Affected Components:** FRD-002 (AI Recipe Recommendation Engine), FRD-005 (Persistence)

---

## Context

The Cooking Agent requires a base dataset of 10,000+ recipes to power the recommendation engine. Three options were evaluated during technical review:

1. **Option A:** Public Recipe APIs (Spoonacular, Edamam, TheMealDB)
   - Real-time access to recipe libraries
   - Requires API keys and rate limiting
   - Dependency on external service availability
   - Cost increases with usage volume

2. **Option B:** Curated CSV/JSON File (Recommended for MVP)
   - Static dataset of 10,000-15,000 recipes
   - No external dependencies
   - Deployed with application
   - Manual updates required

3. **Option C:** Embedded Database (PostgreSQL, MongoDB)
   - Scalable for future growth
   - Full control over data
   - Adds infrastructure complexity
   - Upfront setup cost

---

## Decision

**Adopt Option B (Curated CSV/JSON File) for MVP Phase 1.**

---

## Rationale

### MVP Principles
- **Simplicity:** CSV/JSON file is simplest to implement and test
- **No External Dependencies:** Reduces operational complexity and deployment risk
- **Reduced Cost:** No API calls per recipe search (no per-request costs)
- **Predictability:** 10,000 recipes provides sufficient variety for MVP launch
- **Offline Capable:** System works without external API dependencies

### Performance Requirements Met
- Caching 10,000 recipes in memory uses ~50-100MB (acceptable for MVP)
- Recipe lookups O(1) with in-memory hash table
- No network latency from API calls
- Supports <2 second recommendation latency target

### Timeline Advantage
- No API key negotiation or rate limiting setup
- Recipes can be sourced from public datasets immediately
- Faster time to MVP delivery

---

## Implementation Details

### Recipe Data Format
```json
{
  "recipes": [
    {
      "id": "recipe-001",
      "name": "Chicken Stir Fry",
      "ingredients": [
        {"name": "chicken", "quantity": "2", "unit": "lbs"},
        {"name": "bell_pepper", "quantity": "2", "unit": "cups"}
      ],
      "instructions": "1. Heat oil... 2. Add chicken...",
      "cookingTimeMinutes": 25,
      "servings": 4,
      "difficulty": "easy",
      "cuisine": "Asian",
      "tags": ["stir-fry", "quick", "protein"]
    }
  ]
}
```

### Sourcing Strategy
1. **Primary Source:** Check GitHub for publicly available recipe datasets
   - Examples: Open Recipe Dataset, RecipeDB exports
   - Verify license compatibility (MIT, CC0, or CC-BY)

2. **Fallback:** Curate recipes from public recipe websites
   - CookingLight.com (check licensing)
   - BBC Food (check licensing)
   - AllRecipes (check licensing)

3. **Quality Assurance:**
   - Ensure 10,000+ recipes with complete metadata
   - Verify ingredients are canonical (normalized)
   - Test recipe diversity across cuisines and difficulty levels

### Deployment
- Store `recipes.json` in repository or configuration location
- Load into memory on app startup (one-time load)
- Support recipe cache reload via admin endpoint (future)

---

## Consequences

### Positive
✅ Simple implementation and testing
✅ No external service dependencies
✅ Predictable performance
✅ Cost-effective for MVP
✅ Fast time to market

### Negative
❌ Recipe updates require application redeploy
❌ Limited to 10,000-15,000 recipes initially
❌ Manual curation/updates
❌ Cannot react to real-time recipe requests

### Risks & Mitigations
| Risk | Mitigation |
|------|-----------|
| Recipe quality/appropriateness | Curate dataset carefully; implement user feedback mechanism |
| Recipe database licensing | Verify all sources have permissive licenses; document provenance |
| Recipe updates needed | Phase 2: implement API integration for dynamic recipes |
| Recipe dataset sourcing effort | Start with existing GitHub datasets to accelerate |

---

## Transition to Phase 2

**Decision Point:** After MVP reaches 5,000+ active users  
**Evaluation:** Will switch to Option A (API integration) if:
- Users exhaust recipe variety
- Demand for recipe updates increases
- User feedback indicates recipe staleness

**Phase 2 Plan:**
- Integrate Spoonacular or Edamam API
- Migrate recipe data to database
- Implement real-time recipe caching layer
- Maintain CSV fallback for resilience

---

## Approved By

- [ ] Product Manager - _____ Date _____
- [ ] Technical Lead - _____ Date _____
- [ ] Architecture Team - _____ Date _____

---

## Related Documents

- [TECHNICAL_REVIEW_SUMMARY.md](../../TECHNICAL_REVIEW_SUMMARY.md) § Recipe Data Source
- [FRD-002: AI Recipe Recommendation Engine](../features/ai-recipe-recommendation-engine.md) § 8.5.2
- [PRD](../prd.md) § 11.1

---

## Alternatives Considered

**Option A (Public APIs)** - Rejected for MVP
- Adds external dependency and deployment risk
- Per-request costs accumulate with scaling
- Rate limiting complexity for multi-tenancy
- Better suited for Phase 2 when user base grows

**Option C (Database)** - Rejected for MVP
- Over-engineered for MVP needs
- Database infrastructure adds operational burden
- Extra deployment/maintenance requirements
- Preferred approach for Phase 2+

