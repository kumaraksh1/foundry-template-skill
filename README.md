# Foundry AI Solution Template Skill

A GitHub Copilot Skill that generates complete, deployable Azure AI solution templates from natural language descriptions.

## What This Does

You describe what you want to build. Copilot asks clarifying questions. Then it generates a **complete repo** — Bicep infrastructure, app code, CI/CD, docs — that deploys with `azd up`.

## Usage

1. Add this repo as a Copilot skill in VS Code
2. Describe your project:
   ```
   Build an HR policy chatbot that answers questions from uploaded PDF handbooks
   using Azure OpenAI and AI Search, deployed to Container Apps
   ```
3. Answer Copilot's clarifying questions
4. Get a complete, deployable template repo
5. Run `azd up` to deploy

## What Gets Generated

```
your-template/
├── azure.yaml              # azd deployment config + hooks
├── infra/
│   ├── main.bicep          # Infrastructure orchestrator
│   ├── main.parameters.json
│   └── modules/            # Per-service Bicep modules
├── src/
│   ├── api/                # Backend (Python/TypeScript)
│   ├── frontend/           # UI (React, or omitted if API-only)
│   └── Dockerfile
├── scripts/
│   ├── preprovision.sh     # Pre-deploy validation
│   └── postprovision.sh    # Post-deploy setup
├── docs/
│   ├── deployment.md
│   ├── architecture.md
│   ├── troubleshooting.md
│   └── local_development.md
├── .github/workflows/
│   ├── ci.yml
│   └── deploy.yml
├── tests/
├── README.md
├── SECURITY.md
└── LICENSE
```

## Repo Structure (This Repo)

```
├── SKILL.md                # Core skill instructions for Copilot
├── references/             # Architecture patterns & code recipes
│   ├── rag-chatbot.md
│   ├── multi-agent.md
│   ├── api-backend.md
│   └── ...
├── services/               # Azure service catalog (structured)
│   ├── registry.json
│   └── <service>/service.json
└── examples/               # Validated reference outputs
```

## Supported Scenarios

| Scenario | Description |
|----------|-------------|
| RAG Chatbot | Document Q&A with Azure OpenAI + AI Search |
| Multi-Agent | Multiple Foundry agents with orchestration |
| API Backend | REST API with database and AI integration |
| Full-Stack Web App | Frontend + backend + AI services |

## How It Works

1. **SKILL.md** tells Copilot how to gather requirements and generate files
2. **references/** provides architecture patterns and proven code recipes for each scenario type
3. **services/** provides structured metadata about Azure services (Bicep, dependencies, env vars)
4. Copilot combines these to produce a complete, consistent template
