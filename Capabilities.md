
## az prototype — Key Learnings Summary

### How to Start
- Run `az prototype init` → answer prompts → generates `prototype.yaml`
- Then: `design` → `build` → `deploy` (4-stage re-entrant workflow)
- Design the **whole project at once**, not piece by piece — the AI needs full context to make good architecture decisions

### Adding Features Later
- Add new specs to `prototype.yaml`, then re-run `design` → `build` → `deploy`
- The workflow is **re-entrant** — AI reads accumulated `discovery.yaml` and remembers previous decisions
- No need to start over unless things get unwieldy

### When to Reset
- Use `--reset` when: 5+ iterations, 15+ confirmed items, or the design feels bloated
- **Reset only deletes state files** (`.prototype/state/*.yaml`) — generated code in `concept/` and deployed Azure resources are untouched
- After reset, AI starts fresh with no memory of prior decisions

### Scaling to Large Systems
- Works well, but split large systems into **separate projects by bounded context** (e.g., `agri-ingest/`, `agri-analytics/`, `agri-portal/`)
- Each project gets its own `prototype.yaml`, own design, own deploy
- Connect them at infra boundaries (shared VNet, Event Hub, API gateway)

### What It Generates
- **Primarily infrastructure** (Terraform/Bicep) — that's the heaviest output
- Also generates: app scaffolding, CI/CD, docs, backlog, cost analysis
- Application code is starter-quality (not production-complete)

### Microsoft Fabric Support
- **IaC (Terraform/Bicep):** Solid for Fabric **capacity** provisioning
- **REST API scripts:** Generated for workspace creation, notebook execution (workspaces can't be created via IaC)
- **Code samples:** Spark notebooks, T-SQL, KQL — decent starting points
- **Gaps:** Fabric-internal items (pipelines, semantic models, deployment pipelines) need manual work
- **Data mesh:** Partially supported — domain separation via workspaces is manual

### Synthetic Data
- **No dedicated command** for generating synthetic datasets
- Agents will include small **seed scripts and mock data** in generated app code
- For large-scale synthetic data, use external tools (Faker, ADF, custom scripts)

### Research Capability
- 5 agents can **search Microsoft Learn** automatically when they need info (`[SEARCH: query]` pattern)
- Scoped to **Microsoft Learn only** — won't search Stack Overflow, Wikipedia, or general web
- For broader research: connect an **MCP server** (e.g., Lightpanda headless browser) or do the research yourself and feed results into specs
- The AI model's own training data covers general knowledge (algorithms, data structures) but may not be current

### App Code Generation — What You Actually Get
- Code is **functional and runnable** — not empty stubs or scaffolding
- Generates complete APIs (FastAPI / Express / ASP.NET minimal APIs) with real route handlers
- Working Azure SDK client initialization using `DefaultAzureCredential`
- Proper Dockerfiles: multi-stage build, non-root user, health checks
- `.env.example` with all required environment variables documented
- `requirements.txt` / `package.json` / `.csproj` with correct dependencies
- Health check endpoints (`/health` or `/healthz`)
- Error handling: auth errors (401/403), transient failures (429/5xx) with SDK retry
- **If you deploy infra first, then build and deploy the app, it connects and works**

#### What the app code won't have (by design)
- Full business logic — only the core user flow for the demo
- User authentication flows (MSAL, B2C) — only service-to-service managed identity
- Comprehensive input validation or tests
- Production scaling, circuit breakers, or WAF
- Frontend is basic if included at all

#### Non-negotiable even in POC code
- Managed identity for all service-to-service auth
- No hardcoded secrets anywhere
- Encryption at rest, TLS 1.2+
- RBAC over access policies
- Entra-only auth for databases
- Resource tagging, naming conventions

#### Production gap tracking
- Every shortcut is logged as a **production backlog item** (P1–P4 priority)
- Run `az prototype generate` to get the full backlog of what to harden before going live

### Splitting Large Systems
- No `--split` command — the tool works with **one `prototype.yaml` at a time**
- During design, describe the full system and **ask the AI to recommend boundaries**
- The cloud-architect will propose domain groupings (e.g., "ingestion," "analytics," "portal")
- You manually create separate project folders and run `init` + `design` in each
- Reference shared resources (VNet, Event Hub) as existing infrastructure across projects

### MCP Extension Point
- Drop a handler file into `.prototype/mcp/` to connect agents to any external tool/API
- Lightpanda example: headless browser for web navigation and content extraction
- Configure in `prototype.yaml` under `mcp.servers`, secrets in `prototype.secrets.yaml`