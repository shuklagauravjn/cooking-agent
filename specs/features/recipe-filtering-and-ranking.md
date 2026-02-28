# 📋 Feature Requirement Document (FRD)
## Feature: Recipe Filtering & Ranking

**Feature ID:** FRD-003  
**Status:** Active  
**Last Updated:** February 28, 2026  
**Priority:** P1 (High)

---

## 1. Overview
This feature allows users to refine recipe recommendations by applying filters (cuisine type, cooking time, difficulty level) and enables secondary ranking to reorder recommendations based on user preferences.

---

## 2. Business Context
**Traceability to PRD:**
- [REQ-4] The system must support filtering recommendations by cuisine type, cooking time, and difficulty level
- [REQ-8] The system must rank recipes by match score

**User Story:**
```gherkin
As a user with limited time
I want to filter recipes by cooking time and difficulty
So that I can find recipes that fit my schedule and skill level

As a user exploring cuisines
I want to filter recipes by cuisine type
So that I can discover specific cooking traditions
```

---

## 3. Feature Description

### 3.1 What the Feature Does
After receiving recipe recommendations from the AI engine, users can:
1. Apply filters to narrow down results
2. Re-rank results based on applied filters
3. See updated recipe recommendations matching filter criteria
4. Modify filters and see results updated in real-time

### 3.2 Scope
**In Scope:**
- Filter by cuisine type (Italian, Asian, Mexican, Mediterranean, Indian, etc.)
- Filter by cooking time in predefined ranges (≤15m, 15-30m, 30-60m, >60m)
- Filter by difficulty level (easy, medium, hard)
- Apply multiple filters simultaneously (AND logic)
- Remove individual filters without resetting all
- Display filter options available for current recipe set
- Show count of recipes matching each filter option

**Out of Scope:**
- Custom time ranges (fixed ranges only for Phase 1)
- Dietary restrictions filtering (future phase)
- Ingredient-specific filtering (e.g., "no shellfish")
- Nutritional value filtering (future phase)
- Multi-select within single filter category (e.g., Italian AND Chinese simultaneously)
- Saved filter preferences across sessions

---

## 4. Inputs & Outputs

### 4.1 Inputs
- **Recipe List:** Ranked recommendations from AI Recipe Recommendation Engine
- **Filter Selection:** User-selected filters in format:
  - Cuisine type (string or enum)
  - Cooking time range (predefined ranges)
  - Difficulty level (easy/medium/hard)
- **Filter Operation:** Add filter, remove filter, clear all filters

### 4.2 Outputs
- **Filtered Recipe List:** Recipes matching all applied filters, ordered by:
  1. Ingredient match score (primary)
  2. Cooking time (secondary, ascending)
  3. Difficulty level (tertiary, ascending)
  
- **Filter Metadata:**
  - Count of recipes for each available filter option
  - Current active filters
  - Number of recipes matching current filter(s)
  - "No results" message if filters eliminate all recipes

### 4.3 Error Cases
- **No matching recipes:** Suggest removing most restrictive filter
- **Invalid filter:** Return error message with valid filter options
- **Empty filter list:** Return full unfiltered recommendations

---

## 5. Acceptance Criteria

- [ ] **AC-1:** Users can filter by cuisine type (minimum 8 types: Italian, Asian, Mexican, Mediterranean, Indian, American, French, Spanish)
- [ ] **AC-2:** Users can filter by cooking time in 4 ranges: ≤15m, 15-30m, 30-60m, >60m
- [ ] **AC-3:** Users can filter by difficulty level: easy, medium, hard
- [ ] **AC-4:** Multiple filters apply with AND logic (all conditions must be met)
- [ ] **AC-5:** Filtered results are re-ranked by ingredient match score first
- [ ] **AC-6:** Users can remove individual filters without resetting others
- [ ] **AC-7:** Users see count of recipes for each filter option
- [ ] **AC-8:** Filtering operation completes within 500ms
- [ ] **AC-9:** System displays helpful message when filters eliminate all recipes
- [ ] **AC-10:** Filter state persists during conversation turn (can be modified)

---

## 6. Dependencies

### 6.1 Internal Dependencies
- AI Recipe Recommendation Engine (FRD-002) - provides initial recipe list
- Ingredient Input Processing (FRD-001) - for understanding user context

### 6.2 External Dependencies
- Recipe metadata must include cuisine type, cooking time, difficulty level
- Filter definitions and valid values configuration

### 6.3 Blocked By
- Recipe database must populate cuisine, time, and difficulty fields

### 6.4 Blocks
- None (secondary feature, can be implemented after core recommendations)

---

## 7. Technical Constraints

- **Performance:** Filtering and re-ranking must complete within 500ms
- **Filter Options:** Predefined, static filter values (not dynamically generated)
- **Cuisine Types:** Minimum 8 supported cuisines, expandable to 15+
- **Time Ranges:** Fixed ranges only (≤15m, 15-30m, 30-60m, >60m)
- **Sorting Logic:** Recipe metadata must support sorting by match score, time, difficulty

---

## 8. Non-Functional Requirements

- **Usability:** Filter options displayed clearly with current selections highlighted
- **Responsiveness:** Real-time filter updates without full page reloads
- **Scalability:** Handle filtering across 100+ recipe recommendations
- **Accessibility:** Filter UI accessible to screen readers and keyboard navigation

---

## 8.5 API Requirements

### 8.5.1 Recipe Filtering API
**Endpoint:** `POST /api/recipes/filter` (internal)
**Request Format:**
```json
{
  "recipes": [],
  "filters": {
    "cuisine": "string (single value)",
    "cookingTime": "string (enum: <=15|15-30|30-60|>60)",
    "difficulty": "string (enum: easy|medium|hard)"
  },
  "sessionId": "string"
}
```

**Response Format:**
```json
{
  "filteredRecipes": [],
  "matchCounts": {
    "cuisine": {},
    "cookingTime": {},
    "difficulty": {}
  },
  "activeFilters": {},
  "resultCount": "number",
  "metadata": { 
    "processingTimeMs": "number" 
  }
}
```

---

## 8.6 Filter Value Definitions (MVP - Static List)

### Supported Cuisines (Minimum 8)
- Italian, Asian, Mexican, Mediterranean, Indian, American, French, Spanish

### Cooking Time Ranges (Fixed)
- ≤15 minutes, 15-30 minutes, 30-60 minutes, >60 minutes

### Difficulty Levels
- Easy, Medium, Hard

---

## 8.7 Edge Case Handling

- **No Results After Filtering:** Return message: "No recipes match all your criteria. Try removing the most restrictive filter."
- **Single Filter Removal:** Support removing individual filters without page refresh
- **All Filters Removed:** Return to original ranked results
- **Filter State Persistence:** Maintain filter state during conversation turn

---

## 9. Feature Integration Points

**Consumes Data From:**
- AI Recipe Recommendation Engine (FRD-002) - initial recipe recommendations
- Recipe data source - cuisine, time, difficulty fields

**Produces Data For:**
- Conversational Interface (FRD-004) - filtered results to display
- Conversation History (FRD-005) - filter interactions stored

---

## 9. Future Considerations

- Custom time range selection
- Dietary restriction filters (vegetarian, vegan, gluten-free, keto, etc.)
- Allergen-based filtering
- Ingredient exclusion filtering
- Nutritional value filtering (calories, protein, fat, carbs)
- Saved filter preferences per user
- Smart filter suggestions based on user patterns
