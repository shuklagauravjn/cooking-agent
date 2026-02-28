# ADR-002: Session & Conversation Storage Architecture

**Status:** Proposed  
**Date:** 2026-02-28  
**Decision Maker:** Architecture Team  
**Affected Components:** FRD-005 (Conversation History & Persistence), FRD-006 (System Integration)

---

## Context

The Cooking Agent requires persistent storage for conversation history, user interactions, and session state to enable:
- Multi-turn conversations (sessions lasting up to 30 minutes)
- Conversation resumption (retrieve and continue after session timeout)
- User history retrieval (see past ~50 conversations)
- Context preservation (recommendations, filters applied)

Three storage approaches were evaluated:

1. **Option A:** Relational Database (PostgreSQL, Azure SQL)
   - Structured schema for conversations and messages
   - ACID compliance and data consistency
   - Scalable to millions of records
   - Requires database infrastructure and migrations

2. **Option B:** NoSQL Database (MongoDB, Cosmos DB)
   - Flexible schema for conversation documents
   - Horizontal scaling
   - Good for semi-structured conversation data
   - Operational complexity

3. **Option C:** File-Based Storage (JSON files, Azure Blob Storage)
   - **Recommended for MVP Phase 1**
   - Minimal infrastructure
   - Easy to backup and inspect
   - Suitable for <10,000 concurrent users

---

## Decision

**Adopt Option C (File-Based Storage) for MVP Phase 1.**

---

## Rationale

### MVP Constraints
- **Target Users:** 1,000 concurrent active sessions (Phase 1)
- **Storage Needed:** ~5MB per user × 10,000 users = 50GB initial
- **Cost:** Elastic storage (no reserved capacity)
- **Complexity:** Minimal operational overhead

### Implementation Simplicity
- No database schema design/migrations
- Direct file I/O (no connection pooling)
- Easy local development and testing
- Simple backup strategy (file copies)
- Data human-readable (debugging friendly)

### Performance Requirements Met
- File I/O latency <200ms for typical operations
- Memory-mapped file speeds up repeated access
- Caching strategies reduce file I/O

### Deployment Efficiency
- No database deployment/provisioning
- Lower operational surface area
- Faster application startup
- Single artifact to deploy (app + data files)

---

## Implementation Details

### Storage Layout

```
storage/
├── conversations/
│   ├── session-001/
│   │   ├── metadata.json        # sessions, timing, metadata
│   │   ├── messages.jsonl       # line-delimited JSON for messages
│   │   └── interactions.json    # recipes shown, filters applied
│   ├── session-002/
│   └── ...
├── recommended/                 # index of bookmarked recipes
│   ├── session-001.json
│   └── ...
└── index.json                   # quick lookup: sessionId → folder
```

### Session File Format

**metadata.json:**
```json
{
  "sessionId": "uuid",
  "userId": "user-or-session-id",
  "createdAt": "2026-02-28T10:00:00Z",
  "lastActivityAt": "2026-02-28T10:15:30Z",
  "messageCount": 8,
  "ingredientHistory": ["chicken", "rice", "peppers"],
  "activeFilters": {"difficulty": "easy"},
  "status": "active|archived|expired"
}
```

**messages.jsonl** (one JSON object per line):
```
{"role":"user","content":"I have chicken and rice","timestamp":"2026-02-28T10:00:05Z"}
{"role":"agent","content":"Great! Here are 5 recipes...","timestamp":"2026-02-28T10:00:08Z"}
```

**interactions.json:**
```json
{
  "shows": [
    {"recipeId": "recipe-001", "recipeName": "Chicken Fried Rice", "shownAt": "2026-02-28T10:00:08Z"},
    {"recipeId": "recipe-002", "recipeName": "Chicken Stir Fry", "shownAt": "2026-02-28T10:00:08Z"}
  ],
  "bookmarked": ["recipe-001"],
  "filters": [
    {"type": "difficulty", "value": "easy", "appliedAt": "2026-02-28T10:05:00Z"}
  ]
}
```

### API Operations

```typescript
// Store new conversation
saveConversation(sessionId, conversation) → success/error

// Retrieve conversation
getConversation(sessionId) → conversation object or error

// List user's conversations
listConversations(userId, limit=50) → array of summaries

// Append message to conversation
appendMessage(sessionId, message) → success/error

// Delete conversation
deleteConversation(sessionId) → success/error

// Archive old conversations
archiveConversations(olderThan=90days) → count archived
```

### Storage Location Options

**Development:**
```
/tmp/cooking-agent-storage/conversations/
```

**Staging/Production:**
```
# On Linux/macOS
/var/lib/cooking-agent/storage/conversations/

# On Windows
C:\ProgramData\cooking-agent\storage\conversations\

# Or Azure Blob Storage (future)
https://storageaccount.blob.core.windows.net/conversations/
```

---

## Consequences

### Positive
✅ Minimal infrastructure required
✅ Simple backup strategy (zip directory)
✅ Fast local development cycles
✅ Debugging: files human-readable
✅ No database operational burden
✅ Lower hosting costs

### Negative
❌ Not designed for 100,000+ concurrent users
❌ File system scaling limits
❌ Manual data migration for Phase 2
❌ No built-in replication/failover
❌ Requires file system reliability

### Risks & Mitigations
| Risk | Mitigation |
|------|-----------|
| Disk space exhaution | Implement archival policy (90-day retention); monitor disk usage |
| File corruption | Regular backups; implement file sync checks |
| Concurrent access conflicts | Single-writer pattern; file locking mechanism |
| Scalability limits | Phase 2: migrate to database when reaching 5,000 users |

---

## Transition to Phase 2

**Scale-Up Trigger:** When reaching 5,000+ concurrent active sessions  
**Migration Strategy:**
1. Implement database schema (PostgreSQL recommended)
2. Build data importer from JSON files to database
3. Update storage layer with database driver
4. Gradual migration: new sessions → DB, old sessions stay in files
5. Batch migrate archived sessions to database
6. Complete cutover after 2 weeks of parallel operation

**Phase 2 Storage Decision (Proposed):**
- **PostgreSQL** for conversations (structured, ACID, proven)
- **Redis** for session cache (active conversation state)
- **Azure Blob Storage** for long-term archived conversations

---

## Approved By

- [ ] Product Manager - _____ Date _____
- [ ] Technical Lead - _____ Date _____
- [ ] Architecture Team - _____ Date _____

---

## Related Documents

- [TECHNICAL_REVIEW_SUMMARY.md](../../TECHNICAL_REVIEW_SUMMARY.md) § Session & Context Management
- [FRD-005: Conversation History & Persistence](../features/conversation-history-and-persistence.md) § 8.5-8.8
- [FRD-006: System Integration](../features/system-integration-and-orchestration.md) § 9

---

## Alternatives Considered

**Option A (Relational Database)** - Deferred to Phase 2
- Over-engineered for MVP with <1,000 concurrent users
- Database schema design overhead
- Deployment/maintenance complexity
- Better choice post-MVP when scaling beyond 5,000 users

**Option B (NoSQL)** - Deferred to Phase 2
- Same complexity as relational DB for this use case
- No compelling advantage over file-based for MVP
- More expensive than file storage (managed service)
- Consider for Phase 2 if requiring horizontal scaling

