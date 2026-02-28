# Architecture Decision Records (ADRs) Index

**Project:** Cooking Agent MVP  
**Last Updated:** February 28, 2026  
**Status:** All ADRs Proposed (Awaiting Approval)

---

## Overview

This directory contains Architecture Decision Records (ADRs) documenting major technical decisions for the Cooking Agent project. Each ADR explains the **context**, **decision**, and **rationale** for critical choices that shape the system's design.

**Format:** Modified Architectural Decision Records (MADR) - see [AGENTS.md](../../AGENTS.md) for documentation standards.

---

## Quick Reference Table

| ID | Title | Status | Complexity | Impact | Phase |
|----|----|--------|----------|--------|-------|
| [ADR-001](adr-001-recipe-data-source.md) | Recipe Data Source Strategy | Proposed | Medium | HIGH | Phase 1 |
| [ADR-002](adr-002-session-storage.md) | Session & Conversation Storage | Proposed | Medium | HIGH | Phase 1 |
| [ADR-003](adr-003-data-flow-architecture.md) | Feature Data Flow Architecture | Proposed | High | CRITICAL | Phase 1 |
| [ADR-004](adr-004-error-handling.md) | API Error Handling & User Guidance | Proposed | Low | MEDIUM | Phase 1 |
| [ADR-005](adr-005-deployment-architecture.md) | MVP Deployment Architecture | Proposed | High | CRITICAL | Phase 1 |

---

## ADR Summaries

### 🔴 [ADR-001: Recipe Data Source Strategy](adr-001-recipe-data-source.md)

**Decision:** Use curated CSV/JSON file with 10,000+ recipes for MVP Phase 1.

**Why:** Simplicity, no external dependencies, cost-effective, meets MVP requirements.

**Alternatives Rejected:** Public APIs (Spoonacular), Database (over-engineered for MVP).

**Transition Plan:** Migrate to API integration in Phase 2 when user base grows.

**Key Stakeholders:** Product Manager, Data Team

---

### 🔴 [ADR-002: Session & Conversation Storage Architecture](adr-002-session-storage.md)

**Decision:** Use file-based storage (JSON files) for conversation history and session state.

**Why:** Minimal infrastructure, easy backup/inspection, supports MVP scale (1,000 users).

**Alternatives Rejected:** Relational DB (over-engineered), NoSQL (unnecessary complexity).

**Storage Format:**
```
/storage/
├── conversations/
│   ├── session-001/
│   │   ├── metadata.json
│   │   ├── messages.jsonl
│   │   └── interactions.json
```

**Transition Plan:** Migrate to PostgreSQL + Redis in Phase 2 at 5,000+ user scale.

**Key Stakeholders:** Architecture Team, DevOps

---

### 🔴 [ADR-003: Feature Data Flow Architecture](adr-003-data-flow-architecture.md)

**Decision:** Synchronous orchestrator pattern where Conversational Interface (FRD-004) orchestrates calls to other features.

**Why:** Simple, synchronous request/response model sufficient for MVP scale; easy to debug and test.

**Pattern:**
```
User Input 
  → Conversational Interface (router)
    → FRD-001 (ingredient parsing)
    → FRD-002 (AI recommendations)
    → FRD-003 (filtering, optional)
    → FRD-005 (storage)
  → Response to user
```

**Error Handling Strategy:**
- **Tier 1 (Critical):** Block conversation, ask retry (FRD-001, FRD-002)
- **Tier 2 (Secondary):** Partial results with note (FRD-003)
- **Tier 3 (Non-critical):** Silent graceful failure (FRD-005)

**Context Preservation:** Session state loaded/saved before/after each request.

**Transition Plan:** Event-driven architecture for Phase 2+ scaling.

**Key Stakeholders:** Architecture Team, Development Team

---

### 🟡 [ADR-004: API Error Handling & User Guidance](adr-004-error-handling.md)

**Decision:** Implement user-centric error handling with plain-language messages and actionable suggestions.

**Why:** Clear user experience, high conversation continuation rate, fewer support issues.

**Example Mapping:**
```
System Error: "AI model timeout"
  → User Message: "I'm taking too long. Please try again."
  → Suggestions: [Retry, Try with fewer ingredients, Try later]
```

**Error Classification:**
- Tier 1 (Critical): Block conversation with retry option
- Tier 2 (Secondary): Return partial results
- Tier 3 (Non-critical): Log silently, continue

**Logging:** Detailed technical logs for debugging; user-friendly messages only.

**Key Stakeholders:** Product Manager, UX Team

---

### 🔴 [ADR-005: MVP Deployment Architecture](adr-005-deployment-architecture.md)

**Decision:** Deploy on Azure App Service (B1 Standard) with CI/CD pipeline from GitHub.

**Why:** Simple, cost-effective, native Azure AI Foundry integration, auto-scaling ready.

**Architecture:**
```
GitHub → GitHub Actions (build, test) → Azure App Service
                                            ↓
                                    (Recipe data, Sessions, AI Model)
```

**Deployment Process:**
1. Push to main branch
2. CI/CD triggers: lint, test, build
3. Deploy to App Service
4. Health checks verify

**Monitoring:** Application Insights for logs, performance, alerts.

**Cost Estimate:** $55-180/month (MVP Phase 1).

**Scaling Strategy:**
- Phase 1: Single instance (B1)
- Phase 2: Auto-scaling to 4 instances (multi-region)

**Transition Plan:** Add load balancer, database, Redis cache in Phase 2.

**Key Stakeholders:** DevOps Team, Product Manager

---

## Decision Context & Timeline

### Phase 1 (MVP) - 6-7 Weeks
- All 5 ADRs guide Phase 1 implementation
- Focus on **simplicity** and **minimal infrastructure**
- Goal: Deliver working MVP with 1,000 concurrent user capacity

### Phase 2 (Scale-Up) - 4-6 Weeks Post-MVP
- ADR-001: Transition from CSV to API (recipe data)
- ADR-002: Transition from files to database (sessions)
- ADR-003: Transition from orchestrator to event-driven
- ADR-005: Add redundancy, multi-region deployment

### Phase 3+ (Long-Term)
- Advanced observability, caching, security hardening
- User authentication and personalization
- Multi-language support

---

## How to Use These ADRs

### For Developers
1. **Read ADR-003** first (data flow architecture)
2. **Consult relevant ADR** when implementing a feature
3. **Follow decision rationale** for design guidance
4. **Reference "Consequences"** section for trade-offs

### For Architects
1. **Review all ADRs** for consistency
2. **Track "Transition Plan"** sections for Phase 2 readiness
3. **Monitor "Stakeholders"** for alignment

### For Product Managers
1. **Understand cost implications** (ADR-005)
2. **Know scaling limits** (ADR-001, ADR-002)
3. **Plan Phase 2 iterations** based on transition plans

### For DevOps
1. **Implement ADR-005** deployment architecture
2. **Set up monitoring** per ADR-005
3. **Plan Phase 2 infrastructure** based on transition strategy

---

## Approval Workflow

Each ADR requires sign-off from:
- ✅ **Product Manager** - Business impact approval
- ✅ **Technical Lead** - Implementation feasibility approval
- ✅ **Architecture Team** - Technical soundness approval

### Status Legend
- 🟡 **Proposed:** Awaiting approval
- ✅ **Approved:** Decision locked in
- 🔄 **Superseded:** Replaced by newer ADR
- ⚠️ **Deprecated:** No longer valid

---

## Traceability Matrix

### ADRs → FRDs (Feature Requirements Documents)

| ADR | Impacts FRD | Reason |
|-----|------|--------|
| ADR-001 | FRD-002, FRD-005 | Recipe data structure, storage location |
| ADR-002 | FRD-004, FRD-005, FRD-006 | Session state, context management, API contracts |
| ADR-003 | ALL (FRD-001 through FRD-006) | Defines how all features integrate |
| ADR-004 | All (FRD-001 through FRD-006) | Error handling for all features |
| ADR-005 | All systems | Deployment, scaling, monitoring |

### ADRs → PRD (Product Requirements)

- **ADR-001** supports PRD § 11.1 (Recipe Data Source)
- **ADR-002** supports PRD § 8.1, 9.1 (Architecture, Testing)
- **ADR-003** supports PRD § 8.2 (Data Flow)
- **ADR-004** supports PRD § N/A (Implicit in error handling)
- **ADR-005** supports PRD § 12 (Deployment)

---

## Change Request Process

To propose changes to an existing ADR:

1. **Create issue** on GitHub with `ADR-[#]` prefix
2. **Announce change** in team meeting
3. **Get stakeholder feedback** (Product, Tech Lead, Architecture)
4. **Create new ADR** if changing decision (don't modify existing)
5. **Mark old ADR** as "Superseded" with reference to new ADR
6. **Update this INDEX** with new ADR info

---

## Related Documentation

- **[AGENTS.md](../../AGENTS.md)** - Engineering standards and guardrails
- **[PRD.md](../prd.md)** - Product Requirements Document
- **[TECHNICAL_REVIEW_SUMMARY.md](../../TECHNICAL_REVIEW_SUMMARY.md)** - Technical review findings
- **[FRD Index](../features/)** - Feature Requirements Documents
- **[Features Directory](../features/)** - All feature specifications

---

## Glossary

- **MVP:** Minimum Viable Product (Phase 1 implementation)
- **Phase 1:** MVP launch with 1,000 concurrent user capacity
- **Phase 2:** Scale-up with 5,000+ concurrent users
- **MADR:** Modified Architectural Decision Records format
- **FRD:** Feature Requirements Document
- **PRD:** Product Requirements Document
- **Orchestrator Pattern:** Central component coordinating other components
- **Event-Driven:** Asynchronous component communication via events

---

## Questions & Support

For questions about architectural decisions:

1. **Consult the relevant ADR** - each has detailed rationale
2. **Ask the Tech Lead** - they have context on trade-offs
3. **Review AGENTS.md** - for general engineering standards
4. **Check FRDs** - for how decisions impact feature implementation

---

## Version History

| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 1.0 | Feb 28, 2026 | Initial ADR set (5 ADRs) | Developer Lead Agent |

---

*Last Generated: February 28, 2026*  
*Next Review: After ADR approvals & implementation starts*
