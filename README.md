# Foundry AI Solution Template Skill

A GitHub Copilot Skill that generates complete, deployable Azure AI solution templates from natural language descriptions.

## Quick Start

### 1. Invoke the Skill

**VS Code Chat (recommended):**
Open Copilot Chat (Ctrl+Shift+I) and type your prompt:
```
Create a Linux App Service to show weather on UI and use redis as a sidecar to cache the api calls
```

**Copilot CLI (interactive):**
```bash
copilot chat
```
Then type your prompt at the `>` prompt. The skill auto-matches based on your description.

**Copilot CLI (one-shot):**
```bash
echo "Build an HR policy chatbot using Azure OpenAI and AI Search" | copilot chat
```

### 2. Answer Clarifying Questions

The skill asks batched questions about scenario type, backend language, AI model, deployment target, etc. Answer them or accept the defaults.

### 3. Template Gets Generated

Copilot generates all files into a new directory:

```
your-template/
├── azure.yaml
├── .gitattributes
├── .devcontainer/devcontainer.json
├── infra/
│   ├── main.bicep
│   ├── main.parameters.json
│   └── core/
├── src/
│   ├── api/
│   ├── frontend/
│   ├── Dockerfile
│   └── gunicorn.conf.py
├── scripts/
│   ├── validate_env_vars.sh / .ps1
│   └── postdeploy.sh / .ps1
├── docs/
├── .github/workflows/
├── tests/
├── README.md
├── SECURITY.md
└── LICENSE
```

### 4. Open in Dev Container

```bash
cd your-template
code .
```

When VS Code opens, click **"Reopen in Container"** (or use `Dev Containers: Reopen in Container` from the command palette). The devcontainer installs:
- Azure Developer CLI (`azd`)
- Azure CLI (`az`)
- Python 3.11+ (or Node.js for TypeScript templates)
- GitHub CLI

### 5. Deploy

Inside the devcontainer terminal:

```bash
# Authenticate
azd auth login

# Deploy everything (infra + app)
azd up
```

Follow the prompts to select subscription and region. The `preup` hook validates environment variables, then `azd` provisions infrastructure and deploys the app.

### 6. Verify

After deployment completes:
- The `postdeploy` hook prints the app URL and next steps
- Open the URL in your browser
- Try the sample prompts in `docs/sample_questions.md`

### 7. Clean Up

```bash
azd down
```

---

## Alternative: GitHub Codespaces

Click the Codespaces badge in the generated template's README, then run `azd auth login && azd up` — no local setup needed.

---

## Skill Setup (for first-time users)

The skill must be available in `~/.copilot/skills/`:

```
~/.copilot/skills/foundry-template-generator/
├── SKILL.md
├── references/
└── services/
```

If using [skill-sync](https://github.com/your-org/skill-sync), add the source repo and run:
```bash
copilot chat
> sync skills
```

Or manually symlink:
```powershell
New-Item -ItemType SymbolicLink -Path "$env:USERPROFILE\.copilot\skills\foundry-template-generator" -Target "path\to\foundry-template-skill"
```

---

## Supported Scenarios

| Scenario | Description | Reference |
|----------|-------------|-----------|
| RAG Chatbot | Document Q&A with Azure OpenAI + AI Search | `references/rag-chatbot.md` |
| Multi-Agent | Multiple Foundry agents with orchestration | `references/multi-agent.md` |
| API Backend | REST API with database and AI integration | `references/api-backend.md` |
| Full-Stack Web App | Frontend + backend + AI services | `references/full-stack-app.md` |

## Repo Structure

```
├── SKILL.md              # Core skill instructions for Copilot
├── references/           # Architecture patterns & code recipes
│   ├── rag-chatbot.md
│   ├── multi-agent.md
│   ├── api-backend.md
│   ├── full-stack-app.md
│   ├── bicep-patterns.md
│   └── azure-yaml-patterns.md
├── services/             # Azure service catalog
│   ├── registry.json
│   └── <service>/service.json
└── README.md             # This file
```
