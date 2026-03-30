# az prototype — Comprehensive Documentation

> **Version:** 0.2.1-beta.5 (Preview)
> **Author:** Joshua Davis (Microsoft)
> **License:** MIT
> **Repository:** [github.com/Azure/az-prototype](https://github.com/Azure/az-prototype)

---

## Table of Contents

1. [What is az prototype?](#1-what-is-az-prototype)
2. [Architecture Overview](#2-architecture-overview)
3. [Prerequisites](#3-prerequisites)
4. [Installation](#4-installation)
5. [Quick Start Guide](#5-quick-start-guide)
6. [The Four-Stage Workflow](#6-the-four-stage-workflow)
   - [Stage 1: Init](#stage-1-init)
   - [Stage 2: Design](#stage-2-design)
   - [Stage 3: Build](#stage-3-build)
   - [Stage 4: Deploy](#stage-4-deploy)
7. [AI Providers & Models](#7-ai-providers--models)
   - [GitHub Copilot (Recommended)](#github-copilot-recommended)
   - [GitHub Models](#github-models)
   - [Azure OpenAI](#azure-openai)
   - [Choosing the Right Model](#choosing-the-right-model)
8. [The Multi-Agent System](#8-the-multi-agent-system)
   - [Built-in Agents](#built-in-agents)
   - [Custom Agents](#custom-agents)
   - [Agent Resolution Order](#agent-resolution-order)
   - [Agent YAML Format](#agent-yaml-format)
9. [Configuration](#9-configuration)
   - [prototype.yaml](#prototypeyaml)
   - [Naming Strategies](#naming-strategies)
   - [Landing Zones (ALZ)](#landing-zones-alz)
   - [Secrets Management](#secrets-management)
10. [Workload Templates](#10-workload-templates)
11. [Governance & Policies](#11-governance--policies)
    - [Built-in Policies](#built-in-policies)
    - [Custom Policies](#custom-policies)
    - [Anti-Pattern Detection](#anti-pattern-detection)
12. [Analysis Commands](#12-analysis-commands)
    - [Error Analysis](#error-analysis)
    - [Cost Analysis](#cost-analysis)
13. [Documentation & Backlog Generation](#13-documentation--backlog-generation)
    - [Generating Docs](#generating-docs)
    - [Spec-Kit Generation](#spec-kit-generation)
    - [Backlog Generation](#backlog-generation)
14. [Knowledge System](#14-knowledge-system)
15. [MCP Server Integration](#15-mcp-server-integration)
16. [Telemetry](#16-telemetry)
17. [Project Structure](#17-project-structure)
18. [Complete Command Reference](#18-complete-command-reference)
19. [Troubleshooting](#19-troubleshooting)
20. [End-to-End Walkthrough](#20-end-to-end-walkthrough)

---

## 1. What is az prototype?

`az prototype` is an Azure CLI extension that lets you go from an idea to a deployed Azure prototype in four commands. It uses a team of specialized AI agents to analyze requirements, design architecture, generate infrastructure-as-code (Terraform or Bicep) and application code, and deploy everything to Azure — all from the command line.

The extension was born from the **Innovation Factory**, a solution engineering program within Microsoft enterprise field sales that delivers rapid prototypes for enterprise customers. It encapsulates that entire process into a repeatable, AI-driven CLI workflow.

**Key capabilities:**

- **Conversational requirements gathering** — describe what you want in natural language, provide documents/screenshots, and an AI business analyst surfaces gaps and assumptions
- **Multi-agent code generation** — 11 specialized agents (architect, Terraform, Bicep, app developer, security reviewer, etc.) collaborate to produce production-grade code
- **Policy-driven governance** — 13 built-in governance policies ensure generated code follows Azure security and reliability best practices
- **Incremental deployment** — staged deployments with preflight checks, rollback, dry-run previews, and QA-first error routing
- **Three AI providers** — GitHub Copilot, GitHub Models, or Azure OpenAI — with support for models from OpenAI, Anthropic, Google, Meta, and DeepSeek

---

## 2. Architecture Overview

The extension is built as a Python package (`azext_prototype`) that plugs into the Azure CLI framework. Internally, it is organized into these subsystems:

```
┌─────────────────────────────────────────────────────────┐
│                    Azure CLI Framework                    │
│              (az prototype <command> ...)                │
├─────────────────────────────────────────────────────────┤
│  Commands Layer   (commands.py, custom.py, _params.py)  │
├──────────┬──────────┬────────────┬──────────────────────┤
│  Agents  │    AI    │  Stages    │  Governance          │
│ Registry │ Providers│ Orchestr.  │ Policies/Standards   │
│ Loader   │ Copilot  │ Init       │ Anti-patterns        │
│ Built-in │ GitHub   │ Design     │ Policy Engine        │
│ Custom   │ Azure OAI│ Build      │                      │
│          │          │ Deploy     │                      │
├──────────┴──────────┴────────────┴──────────────────────┤
│  Config │ Knowledge │ MCP │ Naming │ Templates │ UI/TUI │
└─────────────────────────────────────────────────────────┘
```

**Data flow:**

1. `init` creates the project scaffold and `prototype.yaml`
2. `design` uses the biz-analyst + cloud-architect agents to produce an architecture saved in `.prototype/state/discovery.yaml`
3. `build` uses the architecture to generate staged IaC + app code, saving state in `.prototype/state/build.yaml`
4. `deploy` runs the generated code against Azure, saving state in `.prototype/state/deploy.yaml`

Each stage is **re-entrant** — you can return to refine your design or rebuild specific components without starting over.

---

## 3. Prerequisites

| Requirement | Details |
|---|---|
| **Azure CLI** | Version 2.50 or later |
| **Azure subscription** | With appropriate permissions to create resources |
| **Python** | 3.10, 3.11, or 3.12 |
| **GitHub CLI (`gh`)** | Required for `copilot` and `github-models` providers |
| **GitHub Copilot license** | Business or Enterprise — required only for the `copilot` provider |
| **Terraform or Bicep CLI** | Whichever IaC tool you choose at project init |
| **Azure OpenAI resource** | Only if using the `azure-openai` provider |

---

## 4. Installation

### Install the Extension

```bash
az extension add --name prototype
```

### Upgrade to Latest Stable

```bash
az extension update --name prototype
```

### Upgrade Including Preview Versions

```bash
az extension update --name prototype --allow-preview
```

### Verify Installation

```bash
az prototype --help
```

---

## 5. Quick Start Guide

This is the fastest way to go from zero to a deployed prototype:

```bash
# 1. Initialize a new project
az prototype init --name my-app --location eastus

# 2. Change into the project directory
cd my-app

# 3. Run the interactive design session (conversational requirements gathering)
az prototype design

# 4. Generate all infrastructure and application code
az prototype build

# 5. Deploy to Azure
az prototype deploy
```

That's it. Four commands take you from an idea to a running Azure environment.

### Using Templates for Common Patterns

If your prototype matches a common workload, skip the discovery conversation with a template:

```bash
# Web app with SQL backend
az prototype init --name my-web-app --location eastus --template web-app

# Serverless API
az prototype init --name my-api --location eastus --template serverless-api

# AI-powered application
az prototype init --name my-ai-app --location eastus --template ai-app
```

### Providing Requirements as Files

Feed in documents, diagrams, and screenshots as design inputs:

```bash
az prototype design --artifacts ./requirements/ --context "Build a data pipeline for IoT telemetry"
```

The agents can ingest **PDFs, DOCX, PPTX, XLSX, images, and screenshots** — including extracting embedded images from documents and sending them to the AI via the vision API.

---

## 6. The Four-Stage Workflow

```
 ┌──────┐    ┌────────┐    ┌───────┐    ┌────────┐
 │ Init │ -> │ Design │ -> │ Build │ -> │ Deploy │
 └──────┘    └────────┘    └───────┘    └────────┘
              re-entrant    re-entrant   re-entrant
```

### Stage 1: Init

**Purpose:** Create the project scaffold and configuration file.

```bash
az prototype init --name my-prototype --location eastus
```

This creates:
- A `prototype.yaml` configuration file
- Project directory structure
- Optionally validates GitHub authentication (for Copilot/GitHub Models providers)

**Key options:**

| Flag | Description | Default |
|---|---|---|
| `--name` | Project name (required) | — |
| `--location` | Azure region (required) | — |
| `--iac-tool` | `terraform` or `bicep` | `terraform` |
| `--ai-provider` | `copilot`, `github-models`, or `azure-openai` | `copilot` |
| `--environment` | `dev`, `staging`, or `prod` | `dev` |
| `--model` | AI model to use | `claude-sonnet-4` (copilot) / `gpt-4o` (others) |
| `--template` | Workload template to use | none |

### Stage 2: Design

**Purpose:** Analyze requirements, identify gaps, and generate an architecture design.

```bash
# Interactive mode — the AI asks you questions
az prototype design

# Provide artifacts and context
az prototype design --artifacts ./specs/ --context "Real-time analytics dashboard"

# Resume a previous session
az prototype design --status

# Reset and start over
az prototype design --reset
```

**What happens during design:**

1. The **biz-analyst** agent initiates a conversational discovery session, asking questions about your requirements
2. Gaps, unstated assumptions, and architectural conflicts are surfaced automatically
3. The **cloud-architect** agent produces an architecture document with service selections, topology, and dependencies
4. Scope tracking separates items into **in-scope**, **out-of-scope**, and **deferred**
5. Cost-aware service selection compares pricing models during design
6. The design state is saved to `.prototype/state/discovery.yaml`

**Re-entrant:** Run `az prototype design` again at any time to refine the architecture. Use `--skip-discovery` to regenerate the architecture from existing discovery state without re-answering questions.

### Stage 3: Build

**Purpose:** Generate infrastructure-as-code and application code in dependency-ordered stages.

```bash
# Full interactive build
az prototype build

# Build only infrastructure
az prototype build --scope infra

# Build only application code
az prototype build --scope apps

# Preview what would be generated
az prototype build --dry-run

# Auto-accept all policy recommendations (for CI/CD)
az prototype build --auto-accept
```

**What happens during build:**

1. The architecture design is decomposed into fine-grained **deployment stages** (each infrastructure component, database, and application gets its own stage)
2. **Cross-stage dependencies** are managed via Terraform remote state or Bicep parameter passing — never hardcoded names
3. **Governance policies** are enforced after each stage — violations result in interactive resolution (accept, override with justification, or regenerate)
4. **Anti-pattern detection** scans generated code for security issues, missing identity configs, hardcoded names, etc.
5. Each stage includes a runnable `deploy.sh` with error handling and post-deployment verification
6. A **conversational review loop** lets you provide feedback to regenerate specific stages
7. Type `done` to accept the build

**Build slash commands:**

| Command | Description |
|---|---|
| `/status` | Stage completion summary |
| `/stages` | Full deployment plan |
| `/files` | List all generated files |
| `/policy` | Policy check summary |
| `/help` | Available commands |

### Stage 4: Deploy

**Purpose:** Deploy to Azure with preflight checks, staged execution, rollback, and error routing.

```bash
# Interactive deployment session
az prototype deploy

# Dry-run (what-if / terraform plan) without executing
az prototype deploy --dry-run

# Deploy a single stage non-interactively
az prototype deploy --stage 1

# View deployment status
az prototype deploy --status

# Force full redeployment
az prototype deploy --force
```

**The 7-phase interactive session:**

1. **Load build state** — imports deployment stages from build output
2. **Plan overview** — displays stage status, confirms proceeding
3. **Preflight** — checks subscription, IaC tool, resource group, resource providers
4. **Stage-by-stage deploy** — executes each pending stage with real-time output
5. **Output capture** — captures Terraform/Bicep outputs to JSON
6. **Deploy report** — summarizes results
7. **Interactive loop** — slash commands for further operations

**Preflight checks:**
- Azure subscription is set and accessible
- Azure tenant matches the target
- IaC tool (Terraform or Bicep) is installed
- Target resource group exists (offers fix command if missing)
- Required Azure resource providers are registered

**Deploy slash commands:**

| Command | Description |
|---|---|
| `/status` | Deployment status for all stages |
| `/deploy [N\|all]` | Deploy a specific stage or all pending |
| `/rollback [N\|all]` | Roll back a deployed stage (reverse order enforced) |
| `/redeploy N` | Roll back and redeploy a specific stage |
| `/plan N` | What-if/terraform plan for a stage |
| `/outputs` | Display captured deployment outputs |
| `/preflight` | Re-run preflight checks |
| `/login` | Run `az login` interactively |
| `/help` | Show available commands |

**Rollback:** Enforces reverse deployment order — you cannot roll back stage N while a higher-numbered stage (N+1, N+2, ...) is still deployed. Use `/rollback all` to roll back all stages in the correct order.

**AI is optional for deploy:** The deploy stage is 100% subprocess-based (Terraform/Bicep/az CLI). Users without a configured AI provider can still deploy. QA error diagnosis degrades gracefully without AI.

---

## 7. AI Providers & Models

The extension supports three AI providers. All AI interactions go through a unified interface, so switching providers is a configuration change.

### GitHub Copilot (Recommended)

**Config value:** `copilot`

The recommended provider. Routes requests directly to the GitHub Copilot enterprise API. Provides access to models from OpenAI, Anthropic, and Google under a single authentication flow.

**Why recommended:**
- Broadest model selection, including Anthropic Claude (the best models for code generation)
- Simple authentication via Copilot CLI or GitHub CLI
- No Azure OpenAI resource required

**Prerequisites:** Active GitHub Copilot Business or Enterprise license.

**Setup:**
```bash
# Option A — Copilot CLI (recommended, especially for EMU accounts)
copilot login

# Option B — Environment variable
export COPILOT_GITHUB_TOKEN=gho_your_token

# Option C — GitHub CLI
gh auth login
```

**Credential resolution order:**
1. `COPILOT_GITHUB_TOKEN` env var
2. `GH_TOKEN` env var
3. Copilot CLI keychain (Windows Credential Manager / macOS Keychain)
4. Copilot SDK config files (`~/.config/github-copilot/`)
5. `gh auth token` (GitHub CLI subprocess)
6. `GITHUB_TOKEN` env var

**Key Claude models available on Copilot:**

| Model | Best For |
|---|---|
| `claude-sonnet-4` | **Default.** Best balance of quality, speed, and cost for code generation |
| `claude-sonnet-4.5` | Excellent coding model |
| `claude-opus-4.6` | Most capable model for nuanced design trade-offs |
| `claude-haiku-4.5` | Fastest, good for simpler tasks |

### GitHub Models

**Config value:** `github-models`

Routes requests through the GitHub Models inference API. Access to OpenAI, Meta Llama, DeepSeek, and other open-weight models.

**Important:** Anthropic Claude models are **not available** on GitHub Models.

**Setup:**
```bash
az prototype config set --key ai.provider --value github-models
gh auth login  # Needs models:read scope
```

### Azure OpenAI

**Config value:** `azure-openai`

Routes requests to your own Azure OpenAI Service deployment. You control the model version, region, and networking. Uses Azure AD (Entra ID) via `DefaultAzureCredential` — API keys are not supported.

**Best for:** Enterprise deployments with data residency, private networking, or compliance requirements.

**Setup:**
```bash
az prototype config set --key ai.provider --value azure-openai
az prototype config set --key ai.azure_openai.endpoint --value https://<resource>.openai.azure.com/
az prototype config set --key ai.model --value <deployment-name>
```

**Security:** Only endpoints matching `https://<resource>.openai.azure.com/` are accepted. Public OpenAI endpoints are blocked.

### Choosing the Right Model

| Use Case | Provider | Model |
|---|---|---|
| General prototyping | `copilot` | `claude-sonnet-4` |
| Complex architecture design | `copilot` | `claude-opus-4.6` |
| Fast iteration / cost-sensitive | `copilot` | `claude-haiku-4.5` |
| Very large codebases (1M context) | `copilot` | `gpt-4.1` or `gemini-2.5-pro` |
| Enterprise / compliance | `azure-openai` | `gpt-4o` |
| Open-weight models | `github-models` | `meta/meta-llama-3.1-405b-instruct` |

**Switching models:**
```bash
az prototype config set --key ai.model --value claude-opus-4.6
az prototype config show  # Verify current configuration
```

---

## 8. The Multi-Agent System

The extension uses a team of 11 specialized AI agents, each with a defined role, capabilities, system prompt, and constraints. Agents collaborate through a formal contract system with declared inputs, outputs, and delegation targets.

### Built-in Agents

| Agent | Role | What It Does |
|---|---|---|
| `cloud-architect` | Architecture | Cross-service coordination, Azure service selection, architecture design |
| `terraform-agent` | Terraform | Terraform IaC module generation with remote state dependencies |
| `bicep-agent` | Bicep | Bicep template generation with parameter passing |
| `app-developer` | Development | Application code generation (APIs, Functions, containers) |
| `doc-agent` | Documentation | Project documentation, as-built records, cost reports |
| `qa-engineer` | QA / Analysis | Error diagnosis from logs, strings, or screenshots; fix coordination |
| `biz-analyst` | Business Analysis | Requirements gap analysis, interactive design dialogue |
| `cost-analyst` | Cost Analysis | Azure cost estimation at S/M/L t-shirt sizes |
| `project-manager` | Coordination | Scope management, backlog generation, task assignment, escalation |
| `security-reviewer` | Security | Pre-deployment IaC security scanning (RBAC, public endpoints, secrets) |
| `monitoring-agent` | Monitoring | Azure Monitor, Application Insights, dashboard generation |

### Custom Agents

You can add your own agents or override built-in ones:

```bash
# Interactive creation — walks through description, capabilities, system prompt, etc.
az prototype agent add --name my-data-agent

# Start from a built-in definition as a template
az prototype agent add --name my-architect --definition cloud_architect

# Add from an existing YAML file
az prototype agent add --name security --file ./security-checker.yaml

# Override a built-in agent
az prototype agent override --name cloud-architect --file ./my-architect.yaml

# Test an agent interactively
az prototype agent test --name my-agent --prompt "Design a web app with Redis"

# Export an agent as portable YAML
az prototype agent export --name qa-engineer --output-file ./qa.yaml

# Update an existing agent
az prototype agent update --name my-agent --description "Updated description"

# Remove a custom agent or override
az prototype agent remove --name my-data-agent
```

### Agent Resolution Order

When looking up an agent by name, the system resolves in this order:

1. **Custom agents** (in `.prototype/agents/`)
2. **Overrides** (custom agents with the same name as a built-in)
3. **Built-in agents** (shipped with the extension)

This means a custom agent or override always wins over the built-in definition.

### Agent YAML Format

```yaml
name: my-custom-agent
description: Custom agent for specific use case
role: architect
system_prompt: |
  You are a specialized architect for ...
constraints:
  - Must use managed identity
  - Must follow naming conventions
tools:
  - terraform
  - bicep
```

Custom agent definitions are stored in `.prototype/agents/` within your project.

---

## 9. Configuration

### prototype.yaml

All project settings live in `prototype.yaml` at the project root. It is created by `az prototype init` and can be modified via `az prototype config set` or edited directly.

```yaml
project:
  name: my-prototype
  location: eastus
  environment: dev
  iac_tool: terraform  # or bicep

naming:
  strategy: microsoft-alz  # microsoft-alz | microsoft-caf | simple | enterprise | custom
  org: contoso
  env: dev
  zone_id: zd              # ALZ zone ID

ai:
  provider: copilot  # copilot | github-models | azure-openai
  model: claude-sonnet-4

agents:
  custom_dir: ./.prototype/agents/
  overrides: {}

deploy:
  track_changes: true
```

**Managing configuration:**

```bash
# Interactive configuration wizard
az prototype config init

# Show all settings
az prototype config show

# Get a specific value
az prototype config get --key ai.provider

# Set a value
az prototype config set --key ai.model --value claude-opus-4.6
```

### Naming Strategies

All agents use a shared naming resolver to generate consistent Azure resource names. Four built-in strategies are available, plus a fully custom option.

| Strategy | Pattern | Example (resource group for "api" service) |
|---|---|---|
| `microsoft-alz` **(default)** | `{zoneid}-{type}-{service}-{env}-{region}` | `zd-rg-api-dev-eus` |
| `microsoft-caf` | `{type}-{org}-{service}-{env}-{region}-{instance}` | `rg-contoso-api-dev-eus-001` |
| `simple` | `{org}-{service}-{type}-{env}` | `contoso-api-rg-dev` |
| `enterprise` | `{type}-{bu}-{org}-{service}-{env}-{region}-{instance}` | `rg-it-contoso-api-dev-eus-001` |
| `custom` | User-defined pattern | Depends on your pattern |

**Change the naming strategy:**
```bash
az prototype config set --key naming.strategy --value microsoft-caf
```

### Landing Zones (ALZ)

When using the `microsoft-alz` naming strategy, resources are assigned to a landing zone:

| Zone ID | Description |
|---|---|
| `pc` | Connectivity Platform (networking, DNS, firewall) |
| `pi` | Identity Platform (Entra ID, RBAC) |
| `pm` | Management Platform (Log Analytics, App Insights) |
| `zd` | Development Zone **(default)** |
| `zt` | Testing Zone |
| `zs` | Staging Zone |
| `zp` | Production Zone |

**Change the landing zone:**
```bash
az prototype config set --key naming.zone_id --value zp
```

### Secrets Management

Sensitive values (API keys, subscription IDs, tokens, service principal credentials) are automatically isolated to a separate `prototype.secrets.yaml` file that is git-ignored. When displayed via `config show`, secret values are masked as `***`.

```bash
# Service principal credentials are stored in the secrets file
az prototype config set --key deploy.service_principal.client_id --value abc123
az prototype config set --key deploy.service_principal.client_secret --value mysecret
```

---

## 10. Workload Templates

Templates provide a pre-configured service topology that matches common Azure workload patterns. They serve as optional starting points — the agent team will still customize the architecture based on your specific requirements.

| Template | Description | Key Services |
|---|---|---|
| `web-app` | Containerized web app with SQL backend and APIM gateway | Container Apps, SQL, Key Vault, APIM |
| `data-pipeline` | Event-driven data pipeline with serverless Cosmos DB | Functions, Cosmos DB, Storage, Event Grid |
| `ai-app` | AI-powered app with Azure OpenAI and conversation history | Container Apps, OpenAI, Cosmos DB, APIM |
| `microservices` | Multi-service architecture with async messaging | Container Apps (x3), Service Bus, APIM |
| `serverless-api` | Serverless REST API with auto-pause SQL | Functions, SQL, Key Vault, APIM |

**Usage:**
```bash
az prototype init --name my-app --location eastus --template web-app
```

Templates are defined as YAML files in the extension's `templates/workloads/` directory.

---

## 11. Governance & Policies

The extension is governance-aware by default. Governance policies are declarative rules that guide how AI agents generate infrastructure and application code.

### How Governance Works

1. **Guard rails during generation** — agents receive policy rules as part of their system prompt and follow them when producing code
2. **Automated compliance checks** — rules with template checks are evaluated against workload templates at load time
3. **Interactive policy resolution** — during build, violations are presented conversationally: accept (default), override with justification, or regenerate

**Severity levels:**

| Level | Keyword | Meaning |
|---|---|---|
| `required` | **MUST** | The agent must follow this rule. A violation is a defect. |
| `recommended` | **SHOULD** | Follow unless there's a justified reason not to. |
| `optional` | **MAY** | Best practice. May skip if not relevant. |

### Built-in Policies

| Policy | Category | Services Covered |
|---|---|---|
| Container Apps | Azure | Container Apps, Container Registry |
| Key Vault | Azure | Key Vault |
| SQL Database | Azure | SQL Database |
| Cosmos DB | Azure | Cosmos DB |
| Storage | Azure | Storage |
| App Service | Azure | App Service |
| Azure Functions | Azure | Functions |
| Monitoring | Azure | Monitor, App Insights |
| Managed Identity | Security | Cross-service |
| Authentication | Security | Cross-service |
| Data Protection | Security | Cross-service |
| Network Isolation | Security | Cross-service |
| APIM-to-Container-Apps | Integration | API Management, Container Apps |

**Key rules enforced:**
- Use managed identity for all service-to-service authentication
- Enable encryption at rest for all data services
- Use RBAC authorization (not access policies) for Key Vault
- Enable soft-delete and purge protection on Key Vault
- Use Microsoft Entra authentication for SQL Database
- Deploy Container Apps in VNET-integrated environments
- Never hardcode credentials in source code or config files
- Assign least-privilege RBAC roles

### Custom Policies

Add custom policies via the `.prototype/policies/` directory in your project:

```yaml
# .prototype/policies/my-policy.policy.yaml
name: my-custom-policy
category: security
services:
  - container-apps
rules:
  - id: CUSTOM-001
    severity: required
    description: All container images must come from the private registry
```

### Anti-Pattern Detection

Post-generation scanning across 9 domains:
- **Security** — hardcoded credentials, admin passwords, disabled encryption
- **Authentication** — disabled auth without companion managed identity
- **Networking** — overly permissive firewall rules
- **Storage** — public blob access
- **Containers** — insecure container configs
- **Encryption** — missing encryption at rest/in transit
- **Monitoring** — missing diagnostic settings
- **Cost** — over-provisioned SKUs
- **Completeness** — incomplete deployment scripts, hardcoded cross-stage references

Anti-pattern warnings are surfaced during build for interactive resolution.

---

## 12. Analysis Commands

### Error Analysis

Diagnose errors from inline strings, log files, or screenshots:

```bash
# Analyze an inline error message
az prototype analyze error --input "ResourceNotFound - The Resource was not found"

# Analyze a log file
az prototype analyze error --input ./deploy.log

# Analyze a screenshot (uses vision/multi-modal AI)
az prototype analyze error --input ./error-screenshot.png
```

The QA engineer agent identifies the root cause, proposes a fix, and tells you which commands to run to redeploy.

### Cost Analysis

Estimate Azure costs at Small/Medium/Large t-shirt sizes:

```bash
# Generate cost estimation
az prototype analyze costs

# Get JSON output
az prototype analyze costs --output-format json

# Force fresh analysis (bypass cache)
az prototype analyze costs --refresh
```

Cost queries are made against the **Azure Retail Prices API**. Results are cached in `.prototype/state/cost_analysis.yaml` and reused unless the design changes.

---

## 13. Documentation & Backlog Generation

### Generating Docs

Generate documentation from 6 built-in templates:

```bash
az prototype generate docs
az prototype generate docs --path ./deliverables/docs
```

| Template | Description |
|---|---|
| `ARCHITECTURE.md` | High-level and detailed architecture diagrams |
| `DEPLOYMENT.md` | Step-by-step deployment guide |
| `DEVELOPMENT.md` | Developer setup and local dev guide |
| `CONFIGURATION.md` | Azure service configuration reference |
| `AS_BUILT.md` | As-built record of delivered solution |
| `COST_ESTIMATE.md` | Azure cost estimates at t-shirt sizes |

The doc agent fills templates with real architecture details when design context is available.

### Spec-Kit Generation

Generate a spec-kit documentation bundle:

```bash
az prototype generate speckit
az prototype generate speckit --path ./my-speckit
```

Produces spec-kit files (`constitution.md`, `spec.md`, `plan.md`, `tasks.md`) aligned with the [spec-kit](https://github.com/github/spec-kit) format. Enriched with discovery state, build stages, deploy status, and cost analysis.

### Backlog Generation

Generate a structured backlog and push to GitHub Issues or Azure DevOps:

```bash
# Interactive backlog session with GitHub
az prototype generate backlog --provider github

# Quick mode — generate, confirm, push
az prototype generate backlog --provider github --quick

# Azure DevOps work items
az prototype generate backlog --provider devops --org myorg --project myproject

# Show current backlog status
az prototype generate backlog --status
```

**Features:**
- AI-generated epics, user stories, and tasks from the architecture design
- Scope-aware: in-scope → stories, deferred → separate epic, out-of-scope → excluded
- Interactive review loop with slash commands (`/list`, `/show`, `/add`, `/remove`, `/push`)
- GitHub Issues with checkbox task lists and effort labels
- Azure DevOps Features → User Stories → Tasks hierarchy
- Completed POC stages marked as done automatically

---

## 14. Knowledge System

The extension ships with 25 Azure service knowledge files covering Terraform patterns, Bicep patterns, application code, common pitfalls, RBAC requirements, and private endpoint configuration.

**Token-budgeted context:** Knowledge is loaded within configurable token limits so agents get relevant context without exceeding model capacity.

**POC vs. production annotations:** Knowledge files distinguish what's appropriate now versus what belongs in the backlog (production hardening items).

**Contributing knowledge:**

```bash
# Interactive contribution
az prototype knowledge contribute

# Non-interactive
az prototype knowledge contribute --service cosmos-db --description "RU throughput must be >= 400"

# Preview without submitting
az prototype knowledge contribute --service redis --description "Cache eviction pitfall" --draft
```

Contributions are submitted as structured GitHub Issues to the knowledge repository.

---

## 15. MCP Server Integration

The extension supports the **Model Context Protocol (MCP)** for extending agents with external tools:

- **Handler-based plugin system** — JSON-RPC over HTTP, stdio, or custom transports
- **Per-stage and per-agent scoping** — restrict tools to specific build phases or agent roles
- **AI-driven tool calling** — agents discover and invoke tools through the standard function-calling loop
- **Code-driven tool calling** — stages can invoke MCP tools outside the AI loop
- **Circuit breaker** — automatic disable after consecutive failures
- **Custom handlers** — loaded from `.prototype/mcp/` Python files at runtime

---

## 16. Telemetry

The extension collects limited diagnostic telemetry to improve reliability and performance. Telemetry includes command names, AI provider/model, Azure region, resource types, and success/failure status. Sensitive values are redacted.

**Not collected:** subscription IDs, resource names, user principal names, email addresses, customer content.

**Opt out:**
```bash
az config set core.collect_telemetry=no
```

See [TELEMETRY.md](TELEMETRY.md) for the complete data disclosure.

---

## 17. Project Structure

After running `init → design → build`, your project directory looks like this:

```
my-prototype/
├── prototype.yaml                  # Project configuration
├── prototype.secrets.yaml          # Secrets (git-ignored)
├── .prototype/
│   ├── agents/                     # Custom agent definitions
│   ├── policies/                   # Custom governance policies
│   ├── anti_patterns/              # Custom anti-pattern rules
│   ├── mcp/                        # Custom MCP handlers
│   └── state/
│       ├── discovery.yaml          # Design/architecture state
│       ├── build.yaml              # Build stage state
│       ├── deploy.yaml             # Deployment state
│       ├── cost_analysis.yaml      # Cached cost estimates
│       └── backlog.yaml            # Backlog state
├── concept/
│   ├── infra/
│   │   ├── stage-01-resource-group/
│   │   │   ├── main.tf (or main.bicep)
│   │   │   └── deploy.sh
│   │   ├── stage-02-key-vault/
│   │   ├── stage-03-sql-database/
│   │   └── ...
│   ├── apps/
│   │   ├── api/
│   │   │   ├── app.py
│   │   │   └── deploy.sh
│   │   └── ...
│   ├── db/
│   │   └── ...
│   ├── docs/
│   │   ├── ARCHITECTURE.md
│   │   ├── DEPLOYMENT.md
│   │   └── ...
│   └── .specify/                   # Spec-kit bundle
│       ├── manifest.json
│       ├── constitution.md
│       ├── spec.md
│       ├── plan.md
│       └── tasks.md
└── .gitignore
```

---

## 18. Complete Command Reference

| Command | Description |
|---|---|
| `az prototype init` | Initialize a new prototype project |
| `az prototype design` | Analyze requirements and generate architecture |
| `az prototype build` | Generate infrastructure and application code |
| `az prototype deploy` | Deploy to Azure with staged deployments |
| `az prototype status` | Show project status across all stages |
| `az prototype analyze error` | Diagnose an error from text, logs, or screenshots |
| `az prototype analyze costs` | Estimate Azure costs at S/M/L tiers |
| `az prototype config init` | Interactive configuration wizard |
| `az prototype config show` | Display current configuration |
| `az prototype config get` | Get a single config value |
| `az prototype config set` | Set a config value |
| `az prototype generate docs` | Generate documentation from templates |
| `az prototype generate speckit` | Generate spec-kit documentation bundle |
| `az prototype generate backlog` | Generate and push backlog items |
| `az prototype knowledge contribute` | Submit a knowledge contribution |
| `az prototype agent list` | List all agents |
| `az prototype agent add` | Add a custom agent |
| `az prototype agent override` | Override a built-in agent |
| `az prototype agent show` | Show agent details |
| `az prototype agent remove` | Remove a custom agent or override |
| `az prototype agent update` | Update a custom agent |
| `az prototype agent test` | Test an agent with a prompt |
| `az prototype agent export` | Export an agent as YAML |

For detailed parameter documentation, see [COMMANDS.md](COMMANDS.md).

---

## 19. Troubleshooting

| Problem | Solution |
|---|---|
| `No prototype project found` | Run `az prototype init` first, or `cd` into the project directory |
| `No Copilot credentials found` | Run `copilot login`, or set `COPILOT_GITHUB_TOKEN` env var, or switch to `--ai-provider github-models` |
| `403 Forbidden` with copilot | Your token likely came from an unapproved OAuth app. Run `copilot login`. Common with EMU accounts. |
| `401 Unauthorized` with copilot | Ensure you have an active Copilot Business or Enterprise license |
| `401 Unauthorized` with github-models | Your token needs `models:read` scope. Run `gh auth refresh --scopes models:read` |
| Claude model on `github-models` | Claude is not available on GitHub Models. Switch to `copilot` provider |
| `Invalid Azure OpenAI endpoint` | Must match `https://<resource>.openai.azure.com/`. Public OpenAI endpoints are blocked |
| Terraform not found | Install Terraform CLI and ensure it's on your PATH |
| Bicep not found | Install Bicep CLI: `az bicep install` |
| Resource provider not registered | The deploy preflight will detect this and offer the registration command |
| Design seems wrong | Run `az prototype design --reset` to start fresh, or run `az prototype design` again to refine |
| Build generated wrong code | During the build review loop, provide feedback to regenerate specific stages |
| Deployment failed | Use `az prototype analyze error --input <error>` to diagnose, then `/redeploy N` in the deploy session |

---

## 20. End-to-End Walkthrough

This walkthrough builds a containerized web API with a SQL Database backend, Key Vault for secrets, and API Management for the gateway.

### Step 1: Initialize the Project

```bash
az prototype init \
  --name contoso-api \
  --location eastus \
  --iac-tool terraform \
  --ai-provider copilot \
  --environment dev \
  --template web-app

cd contoso-api
```

### Step 2: Review and Customize Configuration

```bash
# Check the generated config
az prototype config show

# Optionally change naming strategy
az prototype config set --key naming.strategy --value microsoft-caf
az prototype config set --key naming.org --value contoso
```

### Step 3: Design the Architecture

```bash
# Start the interactive design session
az prototype design

# The biz-analyst will ask questions about your requirements:
# - What data does the API manage?
# - What authentication method for end users?
# - Expected traffic volume?
# - Any compliance requirements?
#
# Answer the questions. The agent will detect gaps and ask follow-ups.
# When satisfied, the cloud-architect generates the architecture.
```

Or, provide everything upfront:

```bash
az prototype design \
  --artifacts ./requirements/ \
  --context "REST API for customer orders. 
    Authentication via Entra ID. 
    Expected 1000 requests/minute. 
    Must store PII — need encryption at rest."
```

### Step 4: Generate Code

```bash
# Start the interactive build session
az prototype build

# The system generates dependency-ordered stages:
#   Stage 1: Resource Group
#   Stage 2: Key Vault
#   Stage 3: SQL Database
#   Stage 4: Container Registry
#   Stage 5: Container App (API)
#   Stage 6: API Management
#
# Governance policies are checked after each stage.
# If violations are found, you choose: accept, override, or regenerate.
#
# Review the build report. Provide feedback to regenerate specific stages.
# Type 'done' to accept.
```

### Step 5: Deploy to Azure

```bash
# Start the interactive deployment
az prototype deploy

# Preflight checks run automatically:
#   ✓ Subscription accessible
#   ✓ Terraform installed
#   ✓ Resource group exists (or offers to create it)
#   ✓ Resource providers registered
#
# Each stage deploys sequentially with real-time output.
# If a stage fails, QA diagnoses the error and suggests a fix.
# Use /redeploy N to retry after fixing.
```

### Step 6: Generate Documentation and Backlog

```bash
# Generate project documentation
az prototype generate docs

# Generate and push a backlog to GitHub Issues
az prototype generate backlog --provider github --org contoso --project contoso-api

# Estimate costs
az prototype analyze costs
```

### Step 7: Check Status

```bash
# Full project status across all stages
az prototype status --detailed
```

Output:
```
Project: contoso-api (eastus, dev)
IaC: terraform | AI: copilot (claude-sonnet-4) | Naming: microsoft-caf

  Design   [✓] Complete (8 exchanges, 12 confirmed, 0 open)
  Build    [✓] Complete (6/6 stages accepted, 28 files, 0 policy overrides)
  Deploy   [✓] Complete (6/6 deployed, 0 failed, 0 rolled back)

  0 file(s) changed since last deployment
```

---

*For the full command parameter reference, see [COMMANDS.md](COMMANDS.md).*
*For model selection guidance, see [MODELS.md](MODELS.md).*
*For governance policy details, see [POLICIES.md](POLICIES.md).*
*For feature overview, see [FEATURES.md](FEATURES.md).*
*For telemetry disclosure, see [TELEMETRY.md](TELEMETRY.md).*
