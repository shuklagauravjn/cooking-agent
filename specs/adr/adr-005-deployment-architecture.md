# ADR-005: MVP Deployment Architecture

**Status:** Proposed  
**Date:** 2026-02-28  
**Decision Maker:** DevOps & Architecture Team  
**Affected Components:** All systems, Infrastructure, CI/CD

---

## Context

The Cooking Agent MVP requires a deployment strategy that is:
- Simple to set up and maintain
- Cost-effective for 1,000 concurrent users (Phase 1)
- Supportive of future scaling to 10,000+ users (Phase 2)
- Aligned with Azure ecosystem (per PRD requirement)

Three deployment approaches were evaluated:

### Option A: Single-Instance VM (AWS EC2 / Azure VM)
- Single virtual machine hosts entire application
- Simple deployment (one artifact)
- Cost-effective for MVP (~$50-100/month)
- Single point of failure
- Manual scaling

### Option B: Container Orchestration (Kubernetes / AKS)
- Over-engineered for MVP scale
- High operational complexity
- Overkill for 1,000 concurrent users
- Better for Phase 2+ scaling

### Option C: Serverless with App Service (Recommended for MVP)
- Azure App Service (or Azure Functions)
- Auto-scaling built-in
- Pays only for usage
- Easy integration with Azure services
- Simple CI/CD pipeline

---

## Decision

**Adopt Option C (Azure App Service) for MVP Phase 1 Deployment.**

**Why Azure specifically:**
- PRD specifies "deploy to Azure"
- Azure AI Foundry integration native
- Consistent ecosystem (App Service → AI Services → Blob Storage)
- Cost-effective for MVP scale

---

## Rationale

### MVP Fit
- **Scale:** 1,000 concurrent users easily supported by single App Service instance
- **Cost:** ~$50-150/month for B1 instance; scales with usage
- **Simplicity:** Deploy from Git (auto-deploy on push)
- **Monitoring:** Built-in Application Insights

### Future Readiness
- **Phase 2 Scaling:** Scale out to multiple instances (auto-scaling rules)
- **Database:** Easy integration with Azure SQL or Cosmos DB
- **Caching:** Leverage Azure Redis Cache
- **Storage:** Native Azure Blob Storage integration

### Azure Ecosystem Benefits
- **AI Integration:** Azure AI Foundry seamlessly integrated
- **Authentication:** Azure AD (Microsoft Entra ID) for future user auth
- **Development:** Consistent with Azure Cloud Shell & tools
- **Monitoring:** Application Insights for observability

---

## Architecture: Deployment Topology

### MVP Architecture (Phase 1)

```
┌─────────────────────────────────────────────────────────────┐
│  GitHub Repository                                          │
│  ├─ Main branch                                             │
│  ├─ Automated CI/CD pipeline                               │
│  └─ Triggers on push                                        │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  Azure DevOps / GitHub Actions                             │
│  ├─ Build (npm/pip install, lint, tests)                   │
│  ├─ Package (Docker image)                                 │
│  └─ Deploy (push to App Service)                           │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  Azure App Service (Single Instance)                        │
│  ├─ Tier: B1 Standard                                      │
│  ├─ Runtime: Node.js 20 or Python 3.11                     │
│  └─ App Settings (API keys, DB connection)                 │
└─────────────────────────────────────────────────────────────┘
                     │
         ┌───────────┼───────────┐
         ▼           ▼           ▼
    ┌────────┐  ┌──────────┐  ┌────────────┐
    │ Recipe │  │ Session  │  │ Azure AI   │
    │ Data   │  │ Storage  │  │ Foundry    │
    │ (JSON) │  │ (Files)  │  │ Model      │
    └────────┘  └──────────┘  └────────────┘

```

### Phase 2+ Architecture (Future)

```
┌──────────────────────────────────────────────────────────────┐
│  Load Balancer (Azure Traffic Manager / App Gateway)        │
└────────────────────┬─────────────────────────────────────────┘
                     │
         ┌───────────┼───────────┬──────────┐
         ▼           ▼           ▼          ▼
    ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
    │ App      │ │ App      │ │ App      │ │ App      │
    │ Service  │ │ Service  │ │ Service  │ │ Service  │
    │ East US  │ │ West US  │ │ Europe   │ │ (Auto-  │
    │ (Primary)│ │ (Standby)│ │ (Future) │ │  scale) │
    └──────────┘ └──────────┘ └──────────┘ └──────────┘
         │
    ┌────┴─────────────────┐
    ▼                      ▼
┌──────────────────┐  ┌──────────────────┐
│ PostgreSQL DB    │  │ Redis Cache      │
│ (Azure Database  │  │ (Azure Cache     │
│  for PostgreSQL) │  │  for Redis)      │
└──────────────────┘  └──────────────────┘
         │
         ▼
    ┌──────────────┐
    │ Blob Storage │
    │ (Archival)   │
    └──────────────┘

```

---

## MVP Deployment Specification

### 1. Application Runtime

```yaml
# runtime.txt (for Oryx auto-deploy detection)
node-20      # Or: python-3.11
```

### 2. Configuration Management

**App Service Configuration (Azure Portal or IaC):**

```yaml
App Settings:
  NODE_ENV: production                    # or FLASK_ENV
  AZURE_SUBSCRIPTION_ID: <subscription>
  AZURE_RESOURCE_GROUP: cooking-agent-rg
  AZURE_AI_MODEL_ENDPOINT: <foundry-endpoint>
  AZURE_AI_MODEL_KEY: <foundry-key>
  SESSION_STORAGE_PATH: /home/site/storage/sessions/
  RECIPE_DATABASE_PATH: /home/site/storage/recipes.json
  LOG_LEVEL: info
  APP_INSIGHTS_KEY: <insights-instrumentation-key>

Connection Strings:
  None (for MVP - file-based storage)

Startup Command (for Node.js):
  npm start

Startup Command (for Python):
  gunicorn app:app
```

### 3. Deployment Process

**Stage 1: Development**
```bash
# Local development
npm install
npm test
npm run dev            # Local server on :3000
```

**Stage 2: CI/CD Pipeline** (GitHub Actions)
```yaml
name: Build & Deploy

on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node
        uses: actions/setup-node@v3
        with:
          node-version: '20'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run linting
        run: npm run lint
      
      - name: Run tests
        run: npm test
      
      - name: Build
        run: npm run build    # If needed
      
      - name: Deploy to Azure App Service
        uses: azure/webapps-deploy@v2
        with:
          app-name: cooking-agent-app
          publish-profile: ${{ secrets.AZURE_PUBLISH_PROFILE }}
          package: .
```

**Stage 3: Staging Validation**
```bash
# Integration tests in staging
npm run test:integration -- https://cooking-agent-staging.azurewebsites.net
```

**Stage 4: Production Deployment**
```bash
# Manual approval required
# Slots deployment (blue-green) if using Premium tier
```

### 4. Storage Setup

**Recipe Database:**
```
/home/site/storage/
├── recipes.json          # 10,000+ recipes (20-30MB)
└── ingredients.json      # 5,000 ingredients (2-3MB)
```

**Session Storage:**
```
/home/site/storage/
├── conversations/
│   ├── session-001/
│   │   ├── metadata.json
│   │   ├── messages.jsonl
│   │   └── interactions.json
│   └── ...
└── index.json
```

**Backup Strategy:**
- Daily backup of `/home/site/storage/` to Blob Storage
- Retention: 30-day rolling window
- Restore: Manual restore via Azure Portal if needed

---

## Operations & Monitoring

### 1. Application Health Checks

```
GET /health
{
  "status": "ok",
  "version": "1.0.0",
  "timestamp": "2026-02-28T10:00:00Z",
  "components": {
    "database": "ok",
    "ai_model": "responding",
    "storage": "ok"
  }
}
```

### 2. Monitoring & Alerts (Application Insights)

**Metrics to Track:**
- Request rate (requests/sec)
- Response time (P50, P95, P99)
- Error rate (5xx, 4xx)
- CPU usage
- Memory usage
- Disk usage

**Alert Thresholds:**
```
- Response time P95 > 3 seconds          → Alert
- Error rate > 1%                         → Alert
- CPU usage > 80%                         → Alert
- Disk usage > 85%                        → Alert
- App Service down (ping failed)          → Critical Alert
```

### 3. Logging

**Application Logs:**
- Structured JSON logs (Application Insights)
- Retention: 30 days minimum
- Log level: INFO (production), DEBUG (staging)

**Error Logs:**
- All errors logged with full context
- Searchable by sessionId
- Alert on ERROR or CRITICAL level

### 4. Scaling Strategy

**Phase 1 (MVP):**
- Single instance: B1 Standard
- App Service Plan: Standard (auto-scaling available)
- Manual scaling if needed

**Phase 2 Scale Triggers:**
```
When metric triggers:
  - CPU avg > 70% (last 5 min) → Add instance
  - Memory avg > 75% (last 5 min) → Add instance
  - Request queue > 50 (last 5 min) → Add instance

Rules:
  - Max 4 instances for Phase 2
  - Min 2 instances for redundancy
  - Cool-down: 5 minutes between scale events
```

---

## Security & Compliance

### 1. Network Security

```
- HTTPS only (SSL/TLS certificate from Azure)
- No HTTP traffic allowed
- App Service authentication: Not required (MVP)
- API key protection: Stored in Azure Key Vault (Phase 2)
```

### 2. Data Protection

```
- Recipe data: Non-sensitive (public)
- Session data: Minimal PII (user messages)
- Encryption at rest: Blob Storage native
- Encryption in transit: HTTPS only
```

### 3. Access Control

```
- Application configuration: Stored in App Service settings
- API keys: Protected via Azure Key Vault (future)
- Database credentials: Connection strings (future)
- Admin access: Restricted to authorized personnel
```

---

## Disaster Recovery

### 1. Backup & Restore

**Backup Schedule:**
- Daily automated backup of storage (Blob Storage)
- Weekly full application backup
- Retention: 30 days

**Restore Procedure:**
1. Create new App Service app
2. Deploy application code
3. Restore storage from backup
4. Update DNS to point to new app
5. Verify health checks pass

**RTO:** 30 minutes  
**RPO:** 1 hour

### 2. Failover Strategy (Phase 2+)

**For MVP Phase 1:**
- No automatic failover
- Manual deployment to new instance if needed

**For Phase 2+:**
- Geographic redundancy (multiple regions)
- Azure Traffic Manager for failover
- Database replication

---

## Cost Estimation (MVP Phase 1)

```
Azure App Service (B1 Standard):        $50/month
Application Insights (basic logs):      $0-25/month
Azure AI Foundry (if applicable):       $0-100/month (pay-per-call)
Blob Storage (backups):                 $1-5/month

Total Estimated Cost:                   $55-180/month
```

---

## Deployment Checklist (Pre-Release)

- [ ] Recipe database (10,000+ recipes) prepared and tested
- [ ] Ingredient database (5,000 ingredients) prepared
- [ ] App Service infrastructure provisioned (B1 or higher)
- [ ] CI/CD pipeline configured and tested
- [ ] Environment variables and secrets configured
- [ ] Application logs and monitoring configured
- [ ] Backup strategy implemented
- [ ] Health check endpoint working
- [ ] Load testing passed (100 concurrent users)
- [ ] Security baseline checks passed
- [ ] Documentation complete

---

## Approved By

- [ ] Product Manager - _____ Date _____
- [ ] Technical Lead - _____ Date _____
- [ ] DevOps/Infrastructure Team - _____ Date _____

---

## Related Documents

- [PRD § 12: Deployment & Environment Requirements](../prd.md)
- [TECHNICAL_REVIEW_SUMMARY.md](../../TECHNICAL_REVIEW_SUMMARY.md) § Deployment
- [ADR-002: Session Storage](adr-002-session-storage.md)
- [ADR-003: Data Flow Architecture](adr-003-data-flow-architecture.md)

---

## References

- **Azure App Service:** https://learn.microsoft.com/azure/app-service/
- **CI/CD Best Practices:** https://docs.github.com/actions
- **Azure Deployment Guide:** https://learn.microsoft.com/azure/developer/intro/

