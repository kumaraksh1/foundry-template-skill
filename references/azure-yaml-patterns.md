# azure.yaml Patterns

Patterns for the `azure.yaml` file — the contract between the template and Azure Developer CLI (`azd`).

## Basic Structure

```yaml
# yaml-language-server: $schema=https://raw.githubusercontent.com/Azure/azure-dev/main/schemas/v1.0/azure.yaml.json
name: <project-slug>                    # kebab-case, matches repo name
metadata:
  template: <project-slug>@1.0.0        # Template name + version

requiredVersions:                        # Minimum azd version
  azd: ">= 1.18.0"

hooks:                                   # Lifecycle hooks
  preup|preprovision: ...                # Validation before provisioning
  postprovision|postdeploy: ...          # Setup after deployment

services:                                # What gets deployed
  <service-name>: ...

pipeline:                                # CI/CD variables (optional, for azd pipeline config)
  variables: ...
```

## Important Notes From Real Templates

1. **`requiredVersions`** — Always include. Most templates require `>= 1.18.0`.
2. **Hook naming** — Real templates use different hook points:
   - `preup` (runs before entire `azd up`) — used by `get-started-with-ai-agents`
   - `preprovision` (runs before infra deploy) — used by others
   - `postprovision` (runs after infra deploy) — most common post-hook
   - `postdeploy` (runs after app deploy) — used by multi-agent template
3. **`pipeline.variables`** — Include ALL environment variables the template uses for CI/CD.
4. **`parameters`** section — Newer templates use this for interactive prompts during `azd up`.

## Hooks (Always Include)

### Pattern A: Simple (like get-started-with-ai-agents)
```yaml
hooks:
  preup:
    posix:
      shell: sh
      run: chmod u+r+x ./scripts/validate_env_vars.sh; ./scripts/validate_env_vars.sh
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
      run: chmod u+r+x ./scripts/postdeploy.sh; ./scripts/postdeploy.sh
      interactive: true
      continueOnError: true
    windows:
      shell: pwsh
      run: ./scripts/postdeploy.ps1
      interactive: true
      continueOnError: true
```

### Pattern B: With post-provision inline output (like Build-your-own-copilot)
```yaml
hooks:
  postprovision:
    windows:
      run: |
        Write-Host "Web app URL: "
        Write-Host "$env:WEB_APP_URL" -ForegroundColor Cyan
        Write-Host ""
        Write-Host "Run the following post-deploy script:"
        Write-Host "bash ./scripts/postdeploy.sh" -ForegroundColor Cyan
      shell: pwsh
      continueOnError: false
      interactive: true
    posix:
      run: |
        echo "Web app URL: $WEB_APP_URL"
        echo ""
        echo "Run: bash ./scripts/postdeploy.sh"
      shell: sh
      continueOnError: false
      interactive: true
```

### Pattern C: With parameters (like content-generation)
```yaml
parameters:
  solutionPrefix:
    type: string
    default: myapp
    displayName: Solution Prefix
    description: A unique prefix for all resources (3-15 chars)
  enableMonitoring:
    type: boolean
    default: false
    displayName: Enable Monitoring
    description: Enable Log Analytics and Application Insights
```

## Service Patterns

### Python on Container Apps (most common)
```yaml
services:
  api:
    project: ./src
    language: py
    host: containerapp
    docker:
      image: api
      remoteBuild: true
```

### Python + React Frontend (single container — most common for AI templates)
```yaml
services:
  api_and_frontend:
    project: ./src
    language: py
    host: containerapp
    docker:
      image: api
      remoteBuild: true
```
Note: Frontend is built inside the same Dockerfile and served by the Python backend (e.g., FastAPI static mount). This is the pattern used by `get-started-with-ai-agents`.

### Python + React Frontend (separate services)
```yaml
services:
  api:
    project: ./src/api
    language: py
    host: containerapp
    docker:
      image: api
      remoteBuild: true
  frontend:
    project: ./src/frontend
    language: js
    host: staticwebapp
```

### TypeScript on Container Apps
```yaml
services:
  api:
    project: ./src
    language: ts
    host: containerapp
    docker:
      image: api
      remoteBuild: true
```

### Azure Functions (Python)
```yaml
services:
  functions:
    project: ./src
    language: py
    host: function
```

### Multiple Container Apps (Microservices)
```yaml
services:
  api-gateway:
    project: ./src/gateway
    language: py
    host: containerapp
    docker:
      image: gateway
      remoteBuild: true
  worker:
    project: ./src/worker
    language: py
    host: containerapp
    docker:
      image: worker
      remoteBuild: true
```

## Pipeline Variables (Optional)

Include when users might use `azd pipeline config`:

```yaml
pipeline:
  variables:
    - AZURE_RESOURCE_GROUP
    - AZURE_OPENAI_ENDPOINT
    - AZURE_OPENAI_DEPLOYMENT
    - AZURE_SEARCH_ENDPOINT
    - AZURE_SEARCH_INDEX_NAME
```

## Key Rules

1. **`project` path** must point to the directory containing the Dockerfile (for containerapp) or function code (for function)
2. **`host` type** must match the Bicep infrastructure being provisioned
3. **`docker.image`** name is used as the tag in Container Registry
4. **`remoteBuild: true`** means azd builds the image in Azure (no local Docker required for deploy)
5. **Service name** in `services:` must match the `azd-service-name` tag in the Bicep Container App resource
6. **Hooks** run in the repo root directory, not in the service directory
