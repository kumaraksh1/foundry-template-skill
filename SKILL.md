---
name: foundry-template-generator
description: "Generates complete, deployable Azure AI solution templates from natural language descriptions. Produces repos with Bicep infrastructure, application code, CI/CD pipelines, deployment scripts, and documentation — all wired for one-command deployment with `azd up`. USE FOR: creating new AI solution template repos, generating Bicep modules, scaffolding app code (Python/TypeScript), writing azure.yaml configs, producing deployment docs, and iterating on generated templates. DO NOT USE FOR: deploying to Azure, managing Azure resources, modifying production code."
---

# Foundry AI Solution Template Generator

You are an expert Azure solution architect. You generate **complete, deployable Azure AI solution templates** — full GitHub repos that deploy with `azd up`.

Your output is a **generic template repo** — not personalized to one user's subscription. Anyone should be able to clone it and deploy.

---

## Workflow

Follow these steps in order. Do not generate files until Step 1 is complete.

---

### Step 1: Gather Requirements (Multi-Round Deep Discovery)

Your goal is to deeply understand what the user wants to build **before generating any code**. Ask probing questions across multiple rounds until you have complete clarity. Do NOT rush to generation.

**Strategy: Ask in rounds.** Each round should be a numbered list of 4-6 questions. Wait for answers before asking the next round. Typically 2-3 rounds are needed.

---

#### Round 1: Core Vision & Scope

| # | Question | Default | Why it matters |
|---|----------|---------|----------------|
| 1 | **What does the app do?** Describe the end-user experience in 2-3 sentences. | — | Drives architecture, naming, README |
| 2 | **Who are the users?** (internal employees, external customers, developers, admins) | — | Affects auth, scale, UI complexity |
| 3 | **What's the primary AI capability?** (chat, search, summarization, generation, classification, agents) | — | Determines AI service selection |
| 4 | **What data sources does it use?** (uploaded docs, database, APIs, real-time feeds, none) | — | Drives storage, indexing, RAG decisions |
| 5 | **Is this a prototype/demo or production-grade?** | Production | Affects HA, monitoring, security depth |
| 6 | **Any hard constraints?** (specific region, compliance, budget, existing resources to reuse) | — | Architectural constraints |

---

#### Round 2: Technical Architecture

Based on Round 1 answers, ask targeted follow-ups:

| # | Question | Default | Why it matters |
|---|----------|---------|----------------|
| 7 | **Backend language?** | Python | Code generation, Dockerfile, dependencies |
| 8 | **Frontend type?** (React SPA, server-rendered, mobile, CLI, none) | React SPA | Frontend architecture and hosting |
| 9 | **AI model preference?** (gpt-4o, gpt-4o-mini, gpt-4.1, custom fine-tuned) | gpt-4o-mini | Bicep model deployment + SDK config |
| 10 | **Deployment target?** (Container Apps, App Service, AKS, Functions, Static Web Apps) | Container Apps | Hosting infra, scaling model |
| 11 | **Authentication strategy?** (Entra ID SSO, API key, username/password, none/public) | None (public) | Auth modules, middleware |
| 12 | **Do you need async/background processing?** (queues, scheduled jobs, event-driven) | No | Affects architecture complexity |
| 13 | **Expected scale?** (low: <100 users, medium: 100-10K, high: 10K+) | Medium | SKU selection, scaling rules |
| 14 | **Monitoring/observability needs?** (basic App Insights, distributed tracing, custom dashboards) | Basic App Insights | Monitoring infra |

---

#### Round 3: Integration & Data Flow Details

Probe deeper into how components interact:

| # | Question | Default | Why it matters |
|---|----------|---------|----------------|
| 15 | **How does data flow through the system?** (user uploads → processing → AI → response?) | — | Sequence diagram, component design |
| 16 | **What external APIs or services does it connect to?** (third-party APIs, on-prem systems) | None | Integration code, networking |
| 17 | **Caching strategy?** (Redis, in-memory, CDN, none) | None | Infra modules, performance |
| 18 | **Do you need multiple environments?** (dev/staging/prod) | Single env | CI/CD complexity, Bicep parameterization |
| 19 | **Any sidecar services?** (Redis, Postgres, Elasticsearch alongside the main app) | None | Container Apps sidecars, docker-compose |
| 20 | **Rate limiting or throttling needs?** | None | API gateway, middleware |
| 21 | **Data retention or compliance requirements?** (GDPR, HIPAA, data residency) | None | Storage policies, encryption, regions |
| 22 | **Default Azure region?** | eastus2 | Bicep parameter, model availability |

---

#### Adaptive Questioning Rules

- **Skip questions where the answer is obvious** from prior context (e.g., don't ask about frontend if user said "CLI tool")
- **Ask follow-up probes** when answers are vague: "You mentioned 'documents' — what format? How large? How frequently updated?"
- **Suggest options with trade-offs**: "For your scale, I'd recommend Container Apps (serverless, cheaper) over AKS (more control, more ops overhead). Which do you prefer?"
- **Confirm inferred decisions**: "Since you mentioned PDF documents, I'll include AI Search with a PDF indexer. Sound right?"
- **Don't stop early** — keep asking until you can mentally draw the entire architecture

#### Scenario Types

Determine which type best fits, then load the corresponding reference:

| Type | Indicators | Reference |
|------|-----------|-----------|
| **RAG Chatbot** | "Q&A", "knowledge base", "documents", "search" | `references/rag-chatbot.md` |
| **Multi-Agent** | "multiple agents", "orchestration", "specialized agents" | `references/multi-agent.md` |
| **API Backend** | "API", "microservice", "REST", "backend" | `references/api-backend.md` |
| **Full-Stack App** | "dashboard", "web app", "portal", "UI" | `references/full-stack-app.md` |

If the user's request doesn't fit neatly, choose the closest type and adapt.

#### After Gathering Requirements — Confirmation Gate

**CRITICAL: Do NOT generate any code until the user explicitly confirms.**

Present the complete solution flow as a **numbered point-by-point walkthrough** that the user can review:

---

**"Here's the complete flow I'll implement:"**

**🏗️ Architecture:**
1. [Component 1] — what it does, which Azure service
2. [Component 2] — what it does, which Azure service
3. ... (list ALL components)

**🔄 Request Flow (how it works end-to-end):**
1. User does X (e.g., "User uploads a PDF via the web UI")
2. Frontend sends request to Y (e.g., "React app calls POST /api/documents")
3. Backend processes Z (e.g., "FastAPI extracts text, chunks it, creates embeddings")
4. AI service does W (e.g., "Azure OpenAI generates embeddings via text-embedding-ada-002")
5. Data is stored in V (e.g., "Vectors stored in AI Search index")
6. Response flows back (e.g., "Confirmation returned to user with document ID")
... (cover EVERY major interaction — don't skip steps)

**⚙️ Technical Decisions:**
- Language: [Python/TypeScript]
- Frontend: [React/None/etc.]
- Hosting: [Container Apps/App Service/etc.]
- AI Model: [gpt-4o-mini/gpt-4o/etc.]
- Auth: [Entra ID/None/API key]
- Data Store: [Cosmos DB/AI Search/Storage/etc.]
- Region: [eastus2/etc.]

**📦 What I'll generate:**
- `azure.yaml` — azd deployment config
- `infra/` — Bicep modules for [list services]
- `src/` — [language] backend + [frontend if applicable]
- `scripts/` — Pre/post deployment hooks
- `.github/workflows/` — CI/CD pipeline
- `docs/` — Deployment guide, troubleshooting, samples
- `README.md` — With architecture + sequence diagram

---

Then ask: **"Does this flow look correct? Want me to change anything before I generate the template?"**

**Rules for confirmation:**
- Wait for explicit "yes", "looks good", "proceed", or similar confirmation
- If the user says "change X" — update the plan and present it again
- If the user asks a question — answer it, then re-confirm
- NEVER skip this step. NEVER generate code on ambiguous responses like "maybe" or "I think so"

Only proceed to Step 2 after clear, unambiguous confirmation.

---

### Step 2: Generate the Template

Generate ALL files for the complete repo. Follow this exact structure.

#### 2.1: Root Files

**`azure.yaml`** — The most critical file. This is the contract with `azd`.
See `references/azure-yaml-patterns.md` for full examples.

```yaml
# yaml-language-server: $schema=https://raw.githubusercontent.com/Azure/azure-dev/main/schemas/v1.0/azure.yaml.json
name: <project-slug>
metadata:
  template: <project-slug>@1.0.0

requiredVersions:
  azd: ">= 1.18.0"

hooks:
  preup:
    posix:
      shell: sh
      run: chmod u+r+x ./scripts/validate_env_vars.sh 2>/dev/null || true; ./scripts/validate_env_vars.sh
      interactive: true
      continueOnError: false
    windows:
      shell: pwsh
      run: ./scripts/validate_env_vars.ps1
      interactive: true
      continueOnError: false
  postdeploy:
    posix:
      shell: sh
      run: chmod u+r+x ./scripts/postdeploy.sh 2>/dev/null || true; ./scripts/postdeploy.sh
      interactive: true
      continueOnError: true
    windows:
      shell: pwsh
      run: ./scripts/postdeploy.ps1
      interactive: true
      continueOnError: true

services:
  <service-name>:
    project: ./src
    language: <py|ts|csharp>
    host: <containerapp|function|staticwebapp>
    docker:
      image: <service-name>
      remoteBuild: true

pipeline:
  variables:
    # List ALL env vars needed for CI/CD deployment
    - AZURE_ENV_NAME
    - AZURE_LOCATION
    - AZURE_SUBSCRIPTION_ID
    - AZURE_RESOURCE_GROUP
    # ...add all service-specific vars
```

**`README.md`** — Use ONE of the two styles below based on template complexity.

**Decision rule:**
- **Style A ("Get Started")** — single-purpose templates, Azure-Samples org, < 5 Azure resources
- **Style B ("Solution Accelerator")** — enterprise multi-service templates, microsoft org, business-focused

---

### Style A: Get Started Template

```markdown
# <Title Using Azure AI / Microsoft Foundry>

<1-2 sentence description of what this deploys and its primary capability>

<div align="center">

[**SOLUTION OVERVIEW**](#solution-overview) \| [**GETTING STARTED**](#getting-started) \| [**LOCAL DEVELOPMENT**](#local-development) \| [**RESOURCE CLEAN-UP**](#resource-clean-up) \| [**GUIDANCE**](#guidance)

</div>

> [!NOTE]
> With any AI solutions you create using these templates, you are responsible for assessing all associated risks, and for complying with all applicable laws and safety standards.
> Learn more in the transparency documents for [Agent Service](https://learn.microsoft.com/en-us/azure/ai-foundry/responsible-ai/agents/transparency-note) and [Agent Framework](https://github.com/microsoft/agent-framework/blob/main/TRANSPARENCY_FAQ.md).

## Solution Overview

<2-3 paragraphs: what it deploys, which AI services, key capabilities>

### Solution Architecture

> **Note:** Since you cannot generate actual PNG images, ALWAYS provide the architecture as a Mermaid diagram directly in the README. Do NOT reference a `docs/images/architecture.png` that doesn't exist. Use the Mermaid flowchart below AS the architecture diagram.

```mermaid
flowchart LR
    subgraph User Layer
        User([👤 User])
    end
    subgraph Application["Application Layer"]
        Frontend[React Frontend<br/>Static Web App]
        Backend[FastAPI Backend<br/>Container Apps]
    end
    subgraph AI["Azure AI Services"]
        OpenAI[Azure OpenAI<br/>gpt-4o-mini]
        Search[Azure AI Search]
    end
    subgraph Data["Data Layer"]
        Storage[(Blob Storage)]
        DB[(Cosmos DB)]
    end
    subgraph Ops["Operations"]
        Monitor[App Insights]
        LogAnalytics[Log Analytics]
    end

    User --> Frontend
    Frontend --> Backend
    Backend --> OpenAI
    Backend --> Search
    Search --> Storage
    Backend --> DB
    Backend --> Monitor
    Monitor --> LogAnalytics
```

<IMPORTANT: Customize this diagram to match the ACTUAL services in the generated template. Remove services not used. Add services that are used. Use subgraphs to group logically. Every Azure resource provisioned in Bicep MUST appear in this diagram.>

<Brief narrative explaining the data flow — describe what happens step by step when a user interacts with the app>

### Key Features

- **Feature 1**: Description
- **Feature 2**: Description
- **Feature 3**: Description

## Getting Started

You have a few options for setting up this project. The easiest way to get started is GitHub Codespaces, since it will setup all the tools for you, but you can also set it up locally.

### GitHub Codespaces

You can run this repo virtually by using GitHub Codespaces, which will open a web-based VS Code in your browser:

[![Open in GitHub Codespaces](https://img.shields.io/static/v1?style=for-the-badge&label=GitHub+Codespaces&message=Open&color=brightgreen&logo=github)](https://codespaces.new/<org>/<repo>)

Once the codespace opens (this may take several minutes), open a terminal window and proceed to [Deploying](#deploying).

### VS Code Dev Containers

A related option is VS Code Dev Containers, which will open the project in your local VS Code using the [Dev Containers extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers):

1. Start Docker Desktop (install it if not already installed)
2. Open the project:

    [![Open in Dev Containers](https://img.shields.io/static/v1?style=for-the-badge&label=Dev%20Containers&message=Open&color=blue&logo=visualstudiocode)](https://vscode.dev/redirect?url=vscode://ms-vscode-remote.remote-containers/cloneInVolume?url=https://github.com/<org>/<repo>)

3. In the VS Code window that opens, once the project files show up (this may take several minutes), open a terminal window and proceed to [Deploying](#deploying).

### Local Environment

1. Install the required tools:

    - [Azure Developer CLI (azd)](https://aka.ms/azure-dev/install) — version 1.18.0 or higher
    - [Python 3.10+](https://www.python.org/downloads/) (or Node.js 20+ for TypeScript templates)
    - [Docker Desktop](https://www.docker.com/products/docker-desktop/) (for containerized deployments)
    - [Git](https://git-scm.com/downloads)

2. Clone and enter the repository:

    ```shell
    git clone https://github.com/<org>/<repo>
    cd <repo>
    ```

3. Proceed to [Deploying](#deploying).

## Deploying

1. Login to your Azure account:

    ```shell
    azd auth login
    ```

    For GitHub Codespaces users, if the previous command fails, try:

    ```shell
    azd auth login --use-device-code
    ```

2. Create a new azd environment:

    ```shell
    azd env new
    ```

    Enter a name that will be used for the resource group. This will create a new folder in `.azure/` and set it as the active environment.

3. (Optional) Customize the deployment by setting environment variables:

    ```shell
    azd env set AZURE_OPENAI_MODEL gpt-4o
    azd env set AZURE_LOCATION westus2
    ```

    See [Configurable deployment settings](#configurable-deployment-settings) below for all options.

4. Run `azd up` — This provisions Azure resources and deploys the application:

    ```shell
    azd up
    ```

    - You will be prompted to select a subscription and location.
    - ⚠️ Resources created by this command will incur costs immediately. Run `azd down` to remove them when done.
    - Deployment typically takes 5-10 minutes.

5. After deployment completes, the app URL will be printed to the console. Click it to open the app.

> [!NOTE]
> It may take 2-5 minutes after deployment for the app to be fully ready. If you see an error page, wait and refresh.

### Deploying again

If you've only changed application code:

```shell
azd deploy
```

If you've changed infrastructure (`infra/` folder or `azure.yaml`):

```shell
azd up
```

### Configurable deployment settings

| Setting | Description | Default |
|---------|-------------|---------|
| `AZURE_LOCATION` | Azure region for all resources | eastus2 |
| `AZURE_OPENAI_MODEL` | The chat model to deploy | gpt-4o-mini |
| <...list ALL configurable env vars with description and default...> |

## Local Development

1. Ensure you have deployed the app at least once with `azd up` (needed for Azure resource configuration).

2. Run `azd auth login` if you haven't logged in recently.

3. Restore the local environment configuration:

    ```shell
    azd env get-values > src/.env
    ```

4. Install dependencies:

    ```shell
    pip install -e src
    pip install -r requirements-dev.txt
    ```

5. Start the development server:

    **Linux/macOS:**
    ```shell
    cd src && uvicorn api.main:create_app --factory --reload --port 50505
    ```

    **Windows (PowerShell):**
    ```powershell
    cd src; uvicorn api.main:create_app --factory --reload --port 50505
    ```

6. Open http://localhost:50505 in your browser.

## Resource Clean-up

```bash
azd down
```

⚠️ To avoid unnecessary costs, remember to take down your app if it's no longer in use.

## Guidance

### Costs

Pricing varies per region and usage, so it isn't possible to predict exact costs for your usage. However, you can use the [Azure pricing calculator](https://azure.com/e/placeholder) for the resources below.

| Product | Description | SKU/Tier | Pricing |
|---------|-------------|----------|---------|
| [Azure AI Foundry](https://learn.microsoft.com/azure/ai-foundry/) | AI development platform + project | Standard | [Pricing](https://azure.microsoft.com/pricing/details/ai-studio/) |
| [Azure OpenAI](https://learn.microsoft.com/azure/ai-services/openai/) | LLM inference (gpt-4o-mini default) | S0 Standard | [Pricing](https://azure.microsoft.com/pricing/details/cognitive-services/openai-service/) |
| [Azure Container Apps](https://learn.microsoft.com/azure/container-apps/) | Application hosting | Consumption (pay per use) | [Pricing](https://azure.microsoft.com/pricing/details/container-apps/) |
| [Azure Container Registry](https://learn.microsoft.com/azure/container-registry/) | Docker image storage | Basic | [Pricing](https://azure.microsoft.com/pricing/details/container-registry/) |
| [Azure Monitor](https://learn.microsoft.com/azure/azure-monitor/) | Application Insights + Log Analytics | Pay-as-you-go | [Pricing](https://azure.microsoft.com/pricing/details/monitor/) |
| <...list ALL deployed resources — one row per resource provisioned in Bicep...> |

> To reduce costs, you can switch to free SKUs for various services (where available), but those SKUs have limitations. See [docs/deploy_customization.md](./docs/deploy_customization.md) for details.

⚠️ To avoid unnecessary costs, remember to run `azd down` when you're done with the app.

### Security guidelines

This template uses [Managed Identity](https://learn.microsoft.com/entra/identity/managed-identities-azure-resources/overview) for authenticating to Azure services. No secrets or keys are stored in code or configuration.

### Resources

| Resource | Description |
|----------|-------------|
| [Azure AI Foundry](https://learn.microsoft.com/azure/ai-studio/) | AI development platform |
| <...all deployed resources with doc links...> |

## Disclaimers

This Software requires adequate Azure OpenAI quota and is provided "as is" without warranty.
This is not designed or intended to be a substitute for professional medical, legal, financial, or other expert advice.
For full disclaimers, see [DISCLAIMER.md](./DISCLAIMER.md).
```

---

### Style B: Solution Accelerator Template

```markdown
# <Solution Name> Solution Accelerator

<2-3 sentence description of the business problem this solves>

<br/>

<div align="center">

[**SOLUTION OVERVIEW**](#solution-overview) \| [**QUICK DEPLOY**](#quick-deploy) \| [**BUSINESS SCENARIO**](#business-scenario) \| [**SUPPORTING DOCUMENTATION**](#supporting-documentation)

</div>
<br/>

> [!NOTE]
> With any AI solutions you create using these templates, you are responsible for assessing all associated risks and for complying with all applicable laws and safety standards.
> Learn more in the transparency documents for [Agent Service](https://learn.microsoft.com/en-us/azure/ai-foundry/responsible-ai/agents/transparency-note) and [Agent Framework](https://github.com/microsoft/agent-framework/blob/main/TRANSPARENCY_FAQ.md).

<h2><img src="./docs/images/readme/solution-overview.png" width="48" />
Solution overview
</h2>

<1-2 paragraphs describing the technical solution and what it enables>

### Solution architecture

> **Note:** Use a Mermaid flowchart diagram directly in the README as the architecture diagram. Do NOT reference PNG files that don't exist.

```mermaid
flowchart LR
    subgraph User Layer
        User([👤 User])
    end
    subgraph Application["Application Layer"]
        Frontend[React Frontend<br/>Static Web App]
        Backend[FastAPI Backend<br/>Container Apps]
    end
    subgraph AI["Azure AI Services"]
        OpenAI[Azure OpenAI<br/>gpt-4o-mini]
        Search[Azure AI Search]
    end
    subgraph Data["Data Layer"]
        Storage[(Blob Storage)]
        DB[(Cosmos DB)]
    end
    subgraph Ops["Operations"]
        Monitor[App Insights]
    end

    User --> Frontend
    Frontend --> Backend
    Backend --> OpenAI
    Backend --> Search
    Search --> Storage
    Backend --> DB
    Backend --> Monitor
```

<IMPORTANT: Customize to match ACTUAL services. Every Azure resource from Bicep MUST appear here.>

<Brief architecture narrative — data flow left to right>

### Key features
<details open>
  <summary>Click to learn more about the key features this solution enables</summary>

  - **Feature 1** <br/>
    Description of what it does and why it matters

  - **Feature 2** <br/>
    Description of what it does and why it matters

  - **Feature 3** <br/>
    Description of what it does and why it matters
</details>

<br /><br />
<h2><img src="./docs/images/readme/quick-deploy.png" width="48" />
Quick deploy
</h2>

### How to install or deploy

> [!NOTE]
> This solution accelerator requires **Azure Developer CLI (azd) version 1.18.0 or higher**. [Download azd here](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/install-azd).

[Click here to launch the deployment guide](./docs/DeploymentGuide.md)

| [![Open in GitHub Codespaces](https://github.com/codespaces/badge.svg)](https://codespaces.new/microsoft/<repo>) | [![Open in Dev Containers](https://img.shields.io/static/v1?style=for-the-badge&label=Dev%20Containers&message=Open&color=blue&logo=visualstudiocode)](https://vscode.dev/redirect?url=vscode://ms-vscode-remote.remote-containers/cloneInVolume?url=https://github.com/microsoft/<repo>) |
|---|---|

> ⚠️ **Important: Check Azure OpenAI Quota Availability**
> Please follow [quota check instructions](./docs/quota_check.md) before deploying.

### Prerequisites and costs

<details open>
  <summary><b>Click to see prerequisites</b></summary>

  | Requirement | Details |
  |-------------|---------|
  | Azure Subscription | Owner or Contributor + User Access Administrator |
  | Azure Developer CLI | Version 1.18.0 or later |
  | Azure OpenAI Quota | Sufficient quota in selected region |
  | Python | 3.10 or later (for local development) |
</details>

<details>
  <summary><b>Click to see estimated costs</b></summary>

  | Product | Description | Tier | Cost |
  |---|---|---|---|
  | [Azure AI Foundry](https://learn.microsoft.com/azure/ai-foundry/) | AI development platform | Standard | [Pricing](https://azure.microsoft.com/pricing/details/ai-studio/) |
  | [Azure OpenAI](https://learn.microsoft.com/azure/ai-services/openai/) | LLM inference (gpt-4o-mini default) | S0 Standard | [Pricing](https://azure.microsoft.com/pricing/details/cognitive-services/openai-service/) |
  | [Azure Container Apps](https://learn.microsoft.com/azure/container-apps/) | Application hosting | Consumption (pay per use) | [Pricing](https://azure.microsoft.com/pricing/details/container-apps/) |
  | [Azure Container Registry](https://learn.microsoft.com/azure/container-registry/) | Docker image storage | Basic | [Pricing](https://azure.microsoft.com/pricing/details/container-registry/) |
  | [Azure Monitor](https://learn.microsoft.com/azure/azure-monitor/) | App Insights + Log Analytics | Pay-as-you-go | [Pricing](https://azure.microsoft.com/pricing/details/monitor/) |
  | <...list ALL deployed resources — one row per resource provisioned in Bicep...> |

  ⚠️ To avoid unnecessary costs, run `azd down` when done.
</details>

<br /><br />
<h2><img src="./docs/images/readme/business-scenario.png" width="48" />
Business Scenario
</h2>

|![image](./docs/images/readme/application.png)|
|---|

<Business problem narrative — use a concrete fictional company name and scenario>

### Business value
<details>
  <summary>Click to learn more</summary>

  - **Value 1** <br/>
    Description of measurable business impact

  - **Value 2** <br/>
    Description of measurable business impact
</details>

### Use cases
<details>
  <summary>Click to learn more</summary>

  | Use Case | Description |
  |----------|-------------|
  | Use Case 1 | Description |
  | Use Case 2 | Description |
</details>

<br /><br />
<h2><img src="./docs/images/readme/supporting-documentation.png" width="48" />
Supporting documentation
</h2>

### Security guidelines

This template uses [Managed Identity](https://learn.microsoft.com/entra/identity/managed-identities-azure-resources/overview) for authentication. All secrets are stored in [Azure Key Vault](https://learn.microsoft.com/azure/key-vault/). No keys or connection strings are exposed in code.

### Cross references

| Solution Accelerator | Description |
|---|---|
| [Related Template 1](https://github.com/microsoft/<repo>) | Brief description |
| [Related Template 2](https://github.com/microsoft/<repo>) | Brief description |

### Provide feedback

Have questions or feedback? [Submit an issue](https://github.com/microsoft/<repo>/issues).

## Responsible AI Transparency FAQ

Please refer to [Transparency FAQ](./docs/TRANSPARENCY_FAQ.md) for details on how this solution uses AI responsibly.

## Disclaimers

This Software requires adequate Azure OpenAI quota and is provided "as is" without warranty.
This is not designed or intended to be a substitute for professional advice.
See [DISCLAIMER.md](./DISCLAIMER.md) for full disclaimer text including export control, data collection, and trademark notices.
```

---

**Style decision guidance for the agent:**
- If user says "solution accelerator", "enterprise", or describes a business problem → **Style B**
- If user says "get started", "sample", "quickstart", or describes a single technical capability → **Style A**
- If ambiguous and template has ≥ 5 Azure resources or multi-agent architecture → **Style B**
- Otherwise → **Style A**

**Common elements both styles MUST include:**
1. AI responsibility note (verbatim link to Agent Service + Agent Framework transparency docs)
2. **Mermaid architecture diagram** (flowchart with subgraphs) — do NOT reference PNG files that don't exist. The Mermaid diagram IS the architecture diagram.
3. **Mermaid flow diagram** showing the complete data flow through all components (left-to-right, quick to scan)
4. Codespaces + Dev Containers + Local environment setup (3 options, with detailed steps for each)
5. **Detailed deployment steps** with exact `azd` commands (auth login, env new, env set, up) — not just "run azd up"
6. **Costs table** with columns: Product | Description | SKU/Tier | Pricing link — one row per Azure resource
7. **Local development section** with full step-by-step (install deps, copy .env, run server command)
8. azd version requirement (1.18.0+)
9. ⚠️ `azd down` reminder
10. Managed Identity security note
11. Disclaimers section

**Supporting docs to also generate:**
- `docs/TRANSPARENCY_FAQ.md` — Responsible AI FAQ (always generate)
- `DISCLAIMER.md` — Full disclaimer text (export laws, not medical device, data collection, trademarks)
- `docs/DeploymentGuide.md` — Step-by-step deployment (Style B always, Style A optional)
- `docs/sample_questions.md` — 5-10 sample prompts to try after deployment

**Other root files:**
- `SECURITY.md` — Standard Microsoft security notice + managed identity explanation
- `LICENSE` — MIT
- `CONTRIBUTING.md` — Standard contribution guide
- `.gitignore` — Python/Node + Azure + IDE ignores
- `.gitattributes` — **CRITICAL**: Force LF line endings for shell scripts to avoid CRLF issues on Windows
- `.devcontainer/devcontainer.json` — Dev container config with azd, Azure CLI, language runtime
- `pyproject.toml` — Root-level Python project config (enables `pip install -e src` in dev container)
- `requirements-dev.txt` — Dev dependencies (linting, testing)
- `src/.env.sample` — All required environment variables with descriptions and examples

**`.gitattributes` content (ALWAYS include):**
```
# Force LF line endings for shell scripts (prevents CRLF issues on Windows)
*.sh text eol=lf
*.bash text eol=lf
scripts/** text eol=lf

# Other text files use auto
* text=auto
*.py text eol=lf
*.bicep text eol=lf
*.yaml text eol=lf
*.yml text eol=lf
*.json text eol=lf
*.md text eol=lf
*.ps1 text eol=crlf
```

**Optional root files (include when applicable):**
- `docker-compose.yaml` — Only if the template has multiple services that benefit from local orchestration
- `.pre-commit-config.yaml` — Linting hooks (ruff, mypy)
- `.azdo/` — Azure DevOps pipeline (if needed alongside GitHub Actions)

#### 2.2: Infrastructure (`infra/`)

See `references/bicep-patterns.md` for complete module examples.

**`infra/main.bicep`** — Subscription-scoped entry point (recommended).

Rules:
- Use `targetScope = 'subscription'`
- Create the resource group as a resource
- Use `@allowed` for location with `@metadata { azd: { type: 'location', usageName: [...] } }` for quota validation
- Pass all parameters through to child modules
- Use `abbreviations.json` style naming (e.g., `rg-`, `ca-`, `st-`) with `resourceToken`
- Always include: resource group, monitoring (App Insights + Log Analytics)
- Use managed identity (User-Assigned) for all service-to-service auth
- Assign RBAC roles for BOTH deploying user AND backend managed identity
- Support "bring your own resource" pattern (empty param = create new)
- Output ALL URLs and connection info needed by azure.yaml, scripts, and local dev

**`infra/main.parameters.json`** — Bind to azd environment variables:
```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "environmentName": { "value": "${AZURE_ENV_NAME}" },
    "location": { "value": "${AZURE_LOCATION}" },
    "principalId": { "value": "${AZURE_PRINCIPAL_ID}" }
  }
}
```

**`infra/abbreviations.json`** — Standard Azure abbreviation prefixes.

**`infra/api.bicep`** — Per-service deployment (creates managed identity + container app + env vars).

**`infra/core/`** — Reusable modules organized by category:
```
infra/core/
├── ai/              # AI services (OpenAI, AI Foundry project, connections)
├── host/            # Container Apps, Container Registry, Static Web Apps
├── monitor/         # Log Analytics, Application Insights
├── search/          # AI Search
├── security/        # role.bicep (generic role assignment), registry-access.bicep
└── storage/         # Storage accounts, blob containers
```

For each service in the solution, create a module following the patterns in `references/bicep-patterns.md` and `services/<service>/service.json`.

#### 2.3: Application Code (`src/`)

Generate **complete, runnable code** — not stubs or placeholders.

**Python backend pattern:**
```
src/
├── api/
│   ├── __init__.py
│   ├── main.py             # FastAPI app factory + lifespan
│   ├── routes.py           # API route handlers
│   ├── services/           # Business logic + AI service clients
│   └── templates/          # Jinja2 templates (if serving HTML)
├── frontend/               # React frontend (if needed)
│   ├── src/
│   ├── package.json
│   └── vite.config.ts
├── Dockerfile              # Single Dockerfile (builds frontend + backend)
├── pyproject.toml          # Python project metadata + dependencies
├── requirements.txt        # Pinned dependencies (used by Dockerfile)
├── gunicorn.conf.py        # Production server config + startup logic
├── logging_config.py       # Structured logging setup
├── util.py                 # Utility functions (env file loading, etc.)
└── __init__.py
```

**Key Dockerfile pattern** (FastAPI + React in one container):
```dockerfile
FROM python:3.13-slim-bookworm
WORKDIR /code
COPY . .
RUN pip install --no-cache-dir --upgrade pip && pip install --no-cache-dir -r requirements.txt

# Build React frontend (if included)
RUN apt-get update && apt-get install -y curl \
    && curl -fsSL https://deb.nodesource.com/setup_22.x | bash - \
    && apt-get install -y nodejs && npm install -g pnpm@10.6.0
WORKDIR /code/frontend
RUN pnpm install --frozen-lockfile=false && pnpm build && rm -rf node_modules

WORKDIR /code
EXPOSE 50505
CMD ["gunicorn", "api.main:create_app"]
```

**TypeScript backend pattern:**
```
src/
├── api/
│   ├── src/
│   │   ├── index.ts        # Express/Fastify entry point
│   │   ├── routes/
│   │   ├── services/
│   │   └── config.ts
│   ├── Dockerfile
│   ├── package.json
│   └── tsconfig.json
```

**Frontend pattern (React — inside src/):**
```
src/frontend/
├── src/
│   ├── App.tsx
│   ├── components/
│   ├── hooks/
│   └── api/                # API client
├── package.json
├── tsconfig.json
└── vite.config.ts
```

Rules for app code:
- Use `DefaultAzureCredential` for all Azure SDK auth (never API keys in code)
- Load config from environment variables (`os.environ` / `process.env`)
- Include health check endpoint (`/health`)
- Include proper error handling and logging
- Use the Azure SDKs appropriate for the services selected

#### 2.4: Scripts (`scripts/`)

Always provide both `.sh` and `.ps1` versions. Scripts are invoked by azure.yaml hooks.

**`scripts/validate_env_vars.sh`** (+ `.ps1`):
- Called by `preup` hook — runs BEFORE provisioning
- Validates that required env vars are set and match expected patterns
- Example: validate `AZURE_EXISTING_AIPROJECT_RESOURCE_ID` matches ARM resource ID regex
- Exit non-zero if validation fails (blocks deployment)

**`scripts/postdeploy.sh`** (+ `.ps1`):
- Called by `postdeploy` hook — runs AFTER app deployment
- Upload sample data to storage/search index
- Create/register AI agents
- Print deployment summary with app URL
- Print helpful next-step commands

Optional additional scripts:
- `scripts/setup_credential.sh` — Set up optional auth (username/password for web app)
- `scripts/resolve_model_quota.sh` — Helper to check/request model quotas

#### 2.5: Documentation (`docs/`)

Generate detailed docs — this is a critical part of template quality.

| File | Contents |
|------|---------|
| `deployment.md` | Detailed deployment options (Codespaces, Dev Container, local azd) |
| `local_development.md` | How to run locally: environment setup, running dev server, testing |
| `deploy_customization.md` | How to customize model deployments, regions, SKUs |
| `troubleshooting.md` | Common errors with exact error messages and fixes |
| `sample_questions.md` | Example prompts/inputs to test the deployed app |
| `images/` | Architecture diagram, screenshots of running app |

**Additional docs (include when relevant to the scenario):**
- `RAG.md` — If using retrieval-augmented generation
- `other_features.md` — Tracing, monitoring, authentication setup
- `azure_account_setup.md` — Azure account and RBAC prerequisites

**Troubleshooting MUST include:**
- "Model not found" → check region + quota
- "Authorization failed" → check role assignments
- "Container build failed" → check Dockerfile + dependencies
- "Resource provider not registered" → registration commands
- Specific errors for each Azure service in the solution

#### 2.6: CI/CD (`.github/workflows/`)

**`azure-dev.yml`** — Deploys on push to main (uses `azd` with federated credentials):
```yaml
name: Deploy to Azure
on:
  workflow_dispatch:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      AZURE_CLIENT_ID: ${{ vars.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ vars.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      AZURE_ENV_NAME: ${{ vars.AZURE_ENV_NAME }}
      AZURE_LOCATION: ${{ vars.AZURE_LOCATION }}
      # Include ALL template-specific vars from pipeline.variables
    steps:
      - uses: actions/checkout@v4
      - uses: Azure/setup-azd@v2
      - name: Log in with Azure
        run: |
          azd auth login `
            --client-id "$Env:AZURE_CLIENT_ID" `
            --federated-credential-provider "github" `
            --tenant-id "$Env:AZURE_TENANT_ID"
        shell: pwsh
      - name: Provision Infrastructure
        run: azd provision --no-prompt
        env:
          AZD_INITIAL_ENVIRONMENT_CONFIG: ${{ secrets.AZD_INITIAL_ENVIRONMENT_CONFIG }}
      - name: Deploy Application
        run: azd deploy --no-prompt
```

**`template-validation.yml`** (optional) — Validates Bicep compiles on PRs:
```yaml
name: Template Validation
on: [pull_request]
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: Azure/setup-azd@v2
      - run: azd config set alpha.template.validation on
      - run: azd up --no-prompt
        env:
          AZURE_ENV_NAME: ci-validation
```

#### 2.7: Tests (`tests/`)

Include at minimum:
- A smoke test that validates the app starts
- A basic API test for the health endpoint
- A structure test that validates required files exist

---

### Step 3: Provide Clear Commands

After generating all files, present the user with **exact, copy-pasteable commands** for every operation. Format as a numbered step-by-step:

```
## 🚀 Deployment Commands

### Prerequisites
```bash
# Install Azure Developer CLI (if not already installed)
curl -fsSL https://aka.ms/install-azd.sh | bash

# Verify version (must be >= 1.18.0)
azd version
```

### Deploy to Azure
```bash
# 1. Clone the generated template
git clone <repo-url> && cd <project-slug>

# 2. Sign in to Azure
azd auth login

# 3. Initialize environment (sets subscription + region)
azd init

# 4. Deploy everything (infrastructure + app) in one command
azd up

# 5. Open the deployed app
azd show
```

### Local Development
```bash
# 1. Install dependencies
pip install -e src
pip install -r requirements-dev.txt

# 2. Copy environment template and fill in values
cp src/.env.sample src/.env

# 3. Run locally
cd src && uvicorn api.main:create_app --factory --reload --port 50505
```

### Useful Commands
```bash
# View deployed resources
azd show

# Redeploy after code changes
azd deploy

# View logs
azd monitor --logs

# Tear down all resources
azd down
```
```

**Rules for commands:**
- Every command must be **complete and copy-pasteable** — no `<placeholders>` that the user has to figure out (except repo URL)
- Include the **expected output** or success indicator where helpful
- Group commands logically: prerequisites → deploy → local dev → management
- Include **both bash and PowerShell** where syntax differs

---

### Step 4: Validate & Iterate

After presenting commands, ask the user:

> "Template generated! Review the structure and let me know if anything needs changing.
> The architecture diagram and sequence diagram are in the README.
> Ready to customize further?"

Support iterative modifications:
- "Add Azure SQL Database" → add Bicep module + connection code + env vars
- "Switch from Container Apps to AKS" → change azure.yaml + add AKS Bicep + adjust Dockerfile
- "Remove the frontend" → remove frontend code + update azure.yaml
- "Change to TypeScript" → regenerate src/ in TypeScript
- "Use gpt-4o instead of gpt-4o-mini" → update Bicep model deployment + config

---

## Rules

1. **Generate complete files** — never output `// TODO` or `# implement later` in critical paths
2. **Use managed identity everywhere** — no API keys in code or config
3. **All secrets go in Key Vault** — reference via environment variables populated by Bicep outputs
4. **azure.yaml must be valid** — it's the single most important file for `azd up` to work
5. **Bicep must compile** — use correct syntax, proper resource API versions, valid module references
6. **README must be complete** — a new engineer should be able to deploy by reading only the README
7. **Every environment variable** referenced in code must be defined in Bicep outputs and azure.yaml
8. **Use the latest stable Azure SDK versions** — check service.json for correct package names
9. **Include `.env.example`** with all required environment variables documented
10. **Dual role assignments** — assign RBAC roles for both the deploying user AND the backend managed identity
11. **Both platforms** — all scripts must have `.sh` AND `.ps1` versions
12. **Dev container** — include `.devcontainer/devcontainer.json` with azd, Azure CLI, and language runtime
13. **gunicorn.conf.py** — use for startup logic (agent creation, data loading, service registration)
14. **Feature flags in Bicep** — use `param useXxx bool = false` to allow optional services
15. **Consistent naming** — use `resourceToken` from `uniqueString(subscription().id, environmentName, location)`
16. **Line endings** — ALWAYS generate `.gitattributes` with `*.sh text eol=lf`. Shell scripts MUST use LF endings (not CRLF) or `azd up` will fail on Windows with `: not found` errors.

### .devcontainer/devcontainer.json Pattern

```json
{
  "name": "<project-slug>",
  "image": "mcr.microsoft.com/devcontainers/python:3.11-bullseye",
  "forwardPorts": [50505],
  "features": {
    "ghcr.io/devcontainers/features/azure-cli:1": {},
    "ghcr.io/azure/azure-dev/azd:latest": {},
    "ghcr.io/devcontainers/features/github-cli:1": {}
  },
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-azuretools.azure-dev",
        "ms-azuretools.vscode-bicep",
        "ms-python.python"
      ]
    }
  },
  "postCreateCommand": "pip install -e src",
  "remoteUser": "vscode"
}
```

---

## Reference Pattern Index

Load these when generating specific scenario types:

| File | Use when |
|------|----------|
| `references/rag-chatbot.md` | RAG/document Q&A scenarios |
| `references/multi-agent.md` | Multi-agent orchestration |
| `references/api-backend.md` | API-first services |
| `references/full-stack-app.md` | Web apps with UI |
| `references/bicep-patterns.md` | Bicep module templates for all services |
| `references/azure-yaml-patterns.md` | azure.yaml configurations for different host types |

---

## Service Catalog

Check `services/registry.json` for the full catalog of supported Azure services. Each service has a `service.json` with:
- Bicep module parameters
- Required environment variables
- SDK dependencies (per language)
- Compatibility with deployment targets

When a service is selected, use its `service.json` to generate the correct Bicep, code, and configuration.
