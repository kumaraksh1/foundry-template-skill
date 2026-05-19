# Bicep Patterns

Standard Bicep module patterns for all infrastructure. Use these exact patterns when generating Bicep files.

## Infra Folder Organization

Two conventions exist in real templates. Use either consistently:

### Option A: `core/` with subfolders (used by get-started-with-ai-agents)
```
infra/
├── main.bicep
├── main.parameters.json
├── abbreviations.json
├── api.bicep                 # Per-service deployment (Container App + identity)
└── core/
    ├── ai/                   # AI Foundry hub, project, connections
    ├── host/                 # Container Apps env, container app, registry
    ├── monitor/              # Log Analytics, App Insights
    ├── search/               # AI Search
    ├── security/             # role.bicep, registry-access.bicep
    └── storage/              # Storage accounts
```

### Option B: `modules/` flat (used by Build-your-own-copilot, Multi-Agent)
```
infra/
├── main.bicep
├── main.parameters.json
├── abbreviations.json
└── modules/
    ├── ai-services.bicep
    ├── container-apps.bicep
    ├── cosmos-db.bicep
    ├── monitoring.bicep
    ├── keyvault.bicep
    ├── role-assignment.bicep
    └── storage.bicep
```

## main.bicep Structure (Subscription Scope — Recommended)

```bicep
targetScope = 'subscription'

@minLength(1)
@maxLength(64)
@description('Name of the environment which is used to generate a short unique hash used in all resources.')
param environmentName string

@description('Location for all resources')
@allowed([
  'eastus'
  'eastus2'
  'swedencentral'
  'westus'
  'westus3'
])
@metadata({
  azd: {
    type: 'location'
    usageName: [
      'OpenAI.GlobalStandard.gpt-4o-mini,30'
    ]
  }
})
param location string

// Optional: support for using existing resources
@description('Use existing AI project resource ID (empty = create new)')
param azureExistingAIProjectResourceId string = ''
@description('The Azure resource group name. If empty, will be generated.')
param resourceGroupName string = ''

// Model parameters
@description('Name of the chat model to deploy')
param chatModelName string = 'gpt-4o-mini'
@description('Version of the chat model to deploy')
param chatModelVersion string = '2024-07-18'
@description('Chat model deployment capacity (TPM)')
param chatDeploymentCapacity int = 30

// Feature flags — allow turning services on/off
@description('Enable Application Insights')
param useApplicationInsights bool = true
@description('Enable Azure AI Search')
param useSearchService bool = false

// Identity
@description('Principal type for role assignments')
param principalTypeOverride string = 'User'
@description('Principal ID for role assignments (set by azd)')
param principalId string = ''

// Generate unique suffix for resource names
var abbrs = loadJsonContent('./abbreviations.json')
var resourceToken = toLower(uniqueString(subscription().id, environmentName, location))
var tags = { 'azd-env-name': environmentName }

// Resource Group
resource rg 'Microsoft.Resources/resourceGroups@2021-04-01' = {
  name: !empty(resourceGroupName) ? resourceGroupName : '${abbrs.resourcesResourceGroups}${environmentName}'
  location: location
  tags: tags
}

// Monitoring (always included)
module monitoring './core/monitor/monitoring.bicep' = {
  name: 'monitoring'
  scope: rg
  params: {
    location: location
    tags: tags
    logAnalyticsName: '${abbrs.operationalInsightsWorkspaces}${resourceToken}'
    applicationInsightsName: '${abbrs.insightsComponents}${resourceToken}'
  }
}

// Container Apps host (including container registry)
module containerApps './core/host/container-apps.bicep' = {
  name: 'container-apps'
  scope: rg
  params: {
    name: 'app'
    location: location
    tags: tags
    containerAppsEnvironmentName: 'containerapps-env-${resourceToken}'
    containerRegistryName: '${abbrs.containerRegistryRegistries}${resourceToken}'
    logAnalyticsWorkspaceName: monitoring.outputs.logAnalyticsWorkspaceName
  }
}

// API app (container app + managed identity)
module api './api.bicep' = {
  name: 'api'
  scope: rg
  params: {
    name: 'ca-api-${resourceToken}'
    location: location
    tags: tags
    identityName: '${abbrs.managedIdentityUserAssigned}api-${resourceToken}'
    containerAppsEnvironmentName: containerApps.outputs.environmentName
    containerRegistryName: containerApps.outputs.registryName
    // Pass service-specific params...
  }
}

// Role assignments — BOTH for user AND for backend identity
module userRoleAzureAIDeveloper './core/security/role.bicep' = {
  name: 'user-role-azureai-developer'
  scope: rg
  params: {
    principalType: principalTypeOverride
    principalId: principalId
    roleDefinitionId: '64702f94-c441-49e6-a78b-ef80e0188fee'
  }
}

module backendRoleAzureAIDeveloper './core/security/role.bicep' = {
  name: 'backend-role-azureai-developer'
  scope: rg
  params: {
    principalType: 'ServicePrincipal'
    principalId: api.outputs.SERVICE_API_IDENTITY_PRINCIPAL_ID
    roleDefinitionId: '64702f94-c441-49e6-a78b-ef80e0188fee'
  }
}

// Outputs (consumed by azure.yaml, scripts, and local dev)
output AZURE_RESOURCE_GROUP string = rg.name
output AZURE_TENANT_ID string = tenant().tenantId
output AZURE_CONTAINER_ENVIRONMENT_NAME string = containerApps.outputs.environmentName
output AZURE_CONTAINER_REGISTRY_ENDPOINT string = containerApps.outputs.registryLoginServer
output SERVICE_API_IDENTITY_PRINCIPAL_ID string = api.outputs.SERVICE_API_IDENTITY_PRINCIPAL_ID
output SERVICE_API_NAME string = api.outputs.SERVICE_API_NAME
output SERVICE_API_URI string = api.outputs.SERVICE_API_URI
// Add service-specific outputs (endpoints, connection strings, etc.)...
```

## main.bicep (ResourceGroup Scope — Alternative)

Some templates use `targetScope = 'resourceGroup'` when the RG is created externally:
```bicep
targetScope = 'resourceGroup'

@minLength(3)
@maxLength(16)
param solutionName string = 'myapp'

@maxLength(5)
param solutionUniqueText string = take(uniqueString(subscription().id, resourceGroup().name, solutionName), 5)

@metadata({ azd: { type: 'location' } })
param location string

// Resources go directly without scope: since we're already in the RG
module monitoring './modules/monitoring.bicep' = {
  name: 'monitoring'
  params: { ... }
}
```

## main.parameters.json

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
Note: `${VAR_NAME}` syntax is azd's parameter substitution from `.env` file.

## Module: Generic Role Assignment (`core/security/role.bicep`)

This is the most important reusable module — used by ALL templates for RBAC:

```bicep
metadata description = 'Creates a role assignment for a service principal.'
param principalId string

@allowed(['Device', 'ForeignGroup', 'Group', 'ServicePrincipal', 'User', ''])
param principalType string

param roleDefinitionId string

resource role 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(subscription().id, resourceGroup().id, principalId, roleDefinitionId)
  properties: {
    principalId: principalId
    principalType: principalType
    roleDefinitionId: resourceId('Microsoft.Authorization/roleDefinitions', roleDefinitionId)
  }
}
```

## Module: api.bicep (Container App + Identity Pattern)

Each deployable service gets its own top-level `.bicep` that creates the managed identity and container app:

```bicep
param name string
param location string = resourceGroup().location
param tags object = {}
param containerRegistryName string
param identityName string
param containerAppsEnvironmentName string

// Service-specific params
param projectEndpoint string = ''
param chatModelDeploymentName string = ''

// Create user-assigned managed identity
resource apiIdentity 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: identityName
  location: location
}

// Build environment variables array
var env = [
  { name: 'AZURE_CLIENT_ID', value: apiIdentity.properties.clientId }
  { name: 'PROJECT_ENDPOINT', value: projectEndpoint }
  { name: 'CHAT_MODEL_DEPLOYMENT_NAME', value: chatModelDeploymentName }
  { name: 'RUNNING_IN_PRODUCTION', value: 'true' }
]

// Deploy container app
module app 'core/host/container-app-upsert.bicep' = {
  name: 'container-app-module'
  params: {
    name: name
    location: location
    tags: union(tags, { 'azd-service-name': 'api' })
    identityName: apiIdentity.name
    containerRegistryName: containerRegistryName
    containerAppsEnvironmentName: containerAppsEnvironmentName
    targetPort: 8000
    env: env
  }
}

output SERVICE_API_IDENTITY_PRINCIPAL_ID string = apiIdentity.properties.principalId
output SERVICE_API_NAME string = app.outputs.name
output SERVICE_API_URI string = app.outputs.uri
```

## Module: Container Apps (`core/host/container-apps.bicep`)

```bicep
@description('Location for resources')
param location string

@description('Tags for resources')
param tags object

@description('Name prefix')
param name string

@description('Name of the Container Apps Environment')
param containerAppsEnvironmentName string

@description('Name of the Container Registry')
param containerRegistryName string

@description('Log Analytics workspace name for environment')
param logAnalyticsWorkspaceName string

resource logAnalyticsWorkspace 'Microsoft.OperationalInsights/workspaces@2023-09-01' existing = {
  name: logAnalyticsWorkspaceName
}

resource containerAppsEnvironment 'Microsoft.App/managedEnvironments@2024-03-01' = {
  name: containerAppsEnvironmentName
  location: location
  tags: tags
  properties: {
    appLogsConfiguration: {
      destination: 'log-analytics'
      logAnalyticsConfiguration: {
        customerId: logAnalyticsWorkspace.properties.customerId
        sharedKey: logAnalyticsWorkspace.listKeys().primarySharedKey
      }
    }
  }
}

resource containerRegistry 'Microsoft.ContainerRegistry/registries@2023-07-01' = {
  name: containerRegistryName
  location: location
  tags: tags
  sku: { name: 'Basic' }
  properties: {
    adminUserEnabled: true
  }
}

output environmentName string = containerAppsEnvironment.name
output registryLoginServer string = containerRegistry.properties.loginServer
output registryName string = containerRegistry.name
```

## Module: Container App Upsert (`core/host/container-app-upsert.bicep`)

Creates/updates a single container app (called by api.bicep):

```bicep
param name string
param location string = resourceGroup().location
param tags object = {}
param identityName string
param containerRegistryName string
param containerAppsEnvironmentName string
param targetPort int = 8000
param env array = []

resource identity 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' existing = {
  name: identityName
}

resource containerRegistry 'Microsoft.ContainerRegistry/registries@2023-07-01' existing = {
  name: containerRegistryName
}

resource containerAppsEnvironment 'Microsoft.App/managedEnvironments@2024-03-01' existing = {
  name: containerAppsEnvironmentName
}

resource containerApp 'Microsoft.App/containerApps@2024-03-01' = {
  name: name
  location: location
  tags: tags
  identity: {
    type: 'UserAssigned'
    userAssignedIdentities: {
      '${identity.id}': {}
    }
  }
  properties: {
    managedEnvironmentId: containerAppsEnvironment.id
    configuration: {
      ingress: {
        external: true
        targetPort: targetPort
        transport: 'auto'
      }
      registries: [
        {
          server: containerRegistry.properties.loginServer
          identity: identity.id
        }
      ]
    }
    template: {
      containers: [
        {
          name: 'main'
          image: 'mcr.microsoft.com/azuredocs/containerapps-helloworld:latest'
          env: env
          resources: {
            cpu: json('1.0')
            memory: '2Gi'
          }
        }
      ]
      scale: {
        minReplicas: 0
        maxReplicas: 10
      }
    }
  }
}

output name string = containerApp.name
output uri string = 'https://${containerApp.properties.configuration.ingress.fqdn}'
```

## Module: Azure OpenAI — LEGACY (`core/ai/openai.bicep`)

> **NOTE**: This standalone pattern uses `kind: 'OpenAI'`. For new Foundry-based templates, use the **AI Foundry Composition Module** pattern above (`kind: 'AIServices'` with `allowProjectManagement: true`). Only use this legacy pattern when explicitly needing a standalone OpenAI resource without AI Foundry project management.

```bicep
@description('Location for resources')
param location string

@description('Tags for resources')
param tags object

@description('Name of the Azure OpenAI resource')
param openAiName string

@description('Chat model name')
param chatModelName string = 'gpt-4o-mini'

@description('Chat model version')
param chatModelVersion string = '2024-07-18'

@description('Deployment capacity in TPM (thousands)')
param chatDeploymentCapacity int = 30

@description('Embedding model name (empty to skip)')
param embeddingModelName string = ''

resource openAi 'Microsoft.CognitiveServices/accounts@2024-04-01-preview' = {
  name: openAiName
  location: location
  tags: tags
  kind: 'OpenAI'
  sku: {
    name: 'S0'
  }
  properties: {
    customSubDomainName: openAiName
    publicNetworkAccess: 'Enabled'
  }
}

resource chatDeployment 'Microsoft.CognitiveServices/accounts/deployments@2024-04-01-preview' = {
  parent: openAi
  name: chatModelName
  sku: {
    name: 'Standard'
    capacity: chatDeploymentCapacity
  }
  properties: {
    model: {
      format: 'OpenAI'
      name: chatModelName
      version: chatModelVersion
    }
  }
}

resource embeddingDeployment 'Microsoft.CognitiveServices/accounts/deployments@2024-04-01-preview' = if (!empty(embeddingModelName)) {
  parent: openAi
  name: embeddingModelName
  sku: {
    name: 'Standard'
    capacity: 30
  }
  properties: {
    model: {
      format: 'OpenAI'
      name: embeddingModelName
      version: '2'
    }
  }
  dependsOn: [chatDeployment]
}

output openAiEndpoint string = openAi.properties.endpoint
output openAiName string = openAi.name
output openAiId string = openAi.id
```

## Module: Monitoring (`core/monitor/monitoring.bicep`)

```bicep
@description('Location for resources')
param location string

@description('Tags for resources')
param tags object

@description('Name of the Log Analytics Workspace')
param logAnalyticsName string

@description('Name of Application Insights')
param applicationInsightsName string

resource logAnalytics 'Microsoft.OperationalInsights/workspaces@2023-09-01' = {
  name: logAnalyticsName
  location: location
  tags: tags
  properties: {
    sku: {
      name: 'PerGB2018'
    }
    retentionInDays: 30
  }
}

resource applicationInsights 'Microsoft.Insights/components@2020-02-02' = {
  name: applicationInsightsName
  location: location
  tags: tags
  kind: 'web'
  properties: {
    Application_Type: 'web'
    WorkspaceResourceId: logAnalytics.id
  }
}

output logAnalyticsWorkspaceId string = logAnalytics.id
output applicationInsightsConnectionString string = applicationInsights.properties.ConnectionString
output applicationInsightsName string = applicationInsights.name
```

## Module: Key Vault (`core/security/keyvault.bicep`)

```bicep
@description('Location for resources')
param location string

@description('Tags for resources')
param tags object

@description('Name of the Key Vault')
param keyVaultName string

resource keyVault 'Microsoft.KeyVault/vaults@2024-04-01-preview' = {
  name: keyVaultName
  location: location
  tags: tags
  properties: {
    sku: {
      family: 'A'
      name: 'standard'
    }
    tenantId: subscription().tenantId
    enableRbacAuthorization: true
    enableSoftDelete: true
    softDeleteRetentionInDays: 7
  }
}

output keyVaultName string = keyVault.name
output keyVaultUri string = keyVault.properties.vaultUri
```

## Module: Storage Account (`core/storage/storage.bicep`)

```bicep
@description('Location for resources')
param location string

@description('Tags for resources')
param tags object

@description('Name of the storage account')
param storageAccountName string

@description('Blob container names to create')
param containerNames array = ['documents']

resource storage 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: storageAccountName
  location: location
  tags: tags
  kind: 'StorageV2'
  sku: {
    name: 'Standard_LRS'
  }
  properties: {
    accessTier: 'Hot'
    allowBlobPublicAccess: false
    minimumTlsVersion: 'TLS1_2'
  }
}

resource blobServices 'Microsoft.Storage/storageAccounts/blobServices@2023-05-01' = {
  parent: storage
  name: 'default'
}

resource containers 'Microsoft.Storage/storageAccounts/blobServices/containers@2023-05-01' = [for name in containerNames: {
  parent: blobServices
  name: name
}]

output storageAccountName string = storage.name
output storageAccountId string = storage.id
output storageBlobEndpoint string = storage.properties.primaryEndpoints.blob
```

## abbreviations.json

```json
{
  "resourcesResourceGroups": "rg-",
  "containerAppsEnvironments": "cae-",
  "containerApps": "ca-",
  "containerRegistries": "cr",
  "cognitiveServicesAccounts": "oai-",
  "searchServices": "srch-",
  "storageAccounts": "st",
  "keyVaultVaults": "kv-",
  "insightsComponents": "appi-",
  "operationalInsightsWorkspaces": "log-",
  "managedIdentityUserAssigned": "id-",
  "webStaticSites": "stapp-",
  "cosmosDbAccounts": "cosmos-",
  "sqlServers": "sql-",
  "serviceBusNamespaces": "sb-"
}
```

## Role Assignments Pattern

CRITICAL: Always assign roles to BOTH the deploying user (for local dev) AND the backend managed identity (for production):

```bicep
// In main.bicep — assign roles for the DEPLOYING USER
module userRoleCognitiveServicesUser 'core/security/role.bicep' = {
  name: 'user-role-cognitive-services-user'
  scope: rg
  params: {
    principalType: principalTypeOverride   // 'User' during azd up
    principalId: principalId               // Set by azd from AZURE_PRINCIPAL_ID
    roleDefinitionId: 'a97b65f3-24c7-4388-baec-2e87135dc908'
  }
}

// Also assign roles for the BACKEND IDENTITY (Container App managed identity)
module backendRoleCognitiveServicesUser 'core/security/role.bicep' = {
  name: 'backend-role-cognitive-services-user'
  scope: rg
  params: {
    principalType: 'ServicePrincipal'
    principalId: api.outputs.SERVICE_API_IDENTITY_PRINCIPAL_ID
    roleDefinitionId: 'a97b65f3-24c7-4388-baec-2e87135dc908'
  }
}
```

### Conditional role assignments (only when service is enabled):
```bicep
module backendRoleSearchContributor 'core/security/role.bicep' = if (useSearchService) {
  name: 'backend-role-search-service-contributor'
  scope: rg
  params: {
    principalType: 'ServicePrincipal'
    principalId: api.outputs.SERVICE_API_IDENTITY_PRINCIPAL_ID
    roleDefinitionId: '7ca78c08-252a-4471-8644-bb5ff32d4ba0'
  }
}
```

## Common Role Definition IDs

| Role | ID |
|------|-----|
| Azure AI Developer | `64702f94-c441-49e6-a78b-ef80e0188fee` |
| Azure AI User | `53ca6127-db72-4b80-b1b0-d745d6d5456d` |
| Cognitive Services User | `a97b65f3-24c7-4388-baec-2e87135dc908` |
| Cognitive Services OpenAI User | `5e0bd9bd-7b93-4f28-af87-19fc36ad61bd` |
| Search Index Data Reader | `1407120a-92aa-4202-b7e9-c0e197c71c8f` |
| Search Index Data Contributor | `8ebe5a00-799e-43f5-93ac-243d3dce84a7` |
| Search Service Contributor | `7ca78c08-252a-4471-8644-bb5ff32d4ba0` |
| Storage Blob Data Reader | `2a2b9908-6ea1-4ae2-8e65-a410df84e7d1` |
| Storage Blob Data Contributor | `ba92f5b4-2d11-453d-a403-e96b0029c9fe` |
| Storage Account Contributor | `17d1049b-9a84-46fb-8f53-869881c3d3ab` |
| Key Vault Secrets User | `4633458b-17de-408a-b874-0445c86b69e6` |
| AcrPull | `7f951dda-4ed3-4680-a7ca-43fe172d538d` |
| Monitoring Metrics Publisher | `3913510d-42f4-4e42-8a64-420c390055eb` |

---

## AI Foundry Composition Module (`core/host/ai-environment.bicep`)

This is the **recommended pattern for all new AI templates**. It replaces the standalone `core/ai/openai.bicep` module. It orchestrates Storage + Monitoring + AI Services (with Project + Connections) + Search in one composition module.

### When to Use
- Any template using Azure AI Foundry (AI Projects, Foundry Tools)
- Any template that needs model deployments (chat, embeddings)
- Templates that support "bring your own existing AI project"

### Architecture
```
ai-environment.bicep (composition)
├── core/storage/storage-account.bicep    → Storage with containers + files + queues + tables
├── core/monitor/loganalytics.bicep       → Log Analytics workspace
├── core/monitor/applicationinsights.bicep → App Insights (linked to Log Analytics)
├── core/ai/cognitiveservices.bicep       → AI Services + Project + Connections + Deployments
├── core/search/search-services.bicep     → AI Search (conditional)
└── core/security/role.bicep              → Storage RBAC for AI Services identity
```

### ai-environment.bicep

```bicep
@minLength(1)
@description('Primary location for all resources')
param location string

@description('The AI Project resource name.')
param aiProjectName string
@description('The Storage Account resource name.')
param storageAccountName string
@description('The AI Services resource name.')
param aiServicesName string
@description('The AI Services model deployments.')
param aiServiceModelDeployments array = []
@description('The Log Analytics resource name.')
param logAnalyticsName string = ''
@description('The Application Insights resource name.')
param applicationInsightsName string = ''
@description('The Azure Search resource name.')
param searchServiceName string = ''
@description('The Application Insights connection name.')
param appInsightConnectionName string
param tags object = {}
param runnerPrincipalId string
param runnerPrincipalType string

module storageAccount '../storage/storage-account.bicep' = {
  name: 'storageAccount'
  params: {
    location: location
    tags: tags
    name: storageAccountName
    containers: [{ name: 'default' }]
    files: [{ name: 'default' }]
    queues: [{ name: 'default' }]
    tables: [{ name: 'default' }]
  }
}

module logAnalytics '../monitor/loganalytics.bicep' = if (!empty(logAnalyticsName)) {
  name: 'logAnalytics'
  params: {
    location: location
    tags: tags
    name: logAnalyticsName
  }
}

module applicationInsights '../monitor/applicationinsights.bicep' = if (!empty(applicationInsightsName) && !empty(logAnalyticsName)) {
  name: 'applicationInsights'
  params: {
    location: location
    tags: tags
    name: applicationInsightsName
    logAnalyticsWorkspaceId: !empty(logAnalyticsName) ? logAnalytics.outputs.id : ''
  }
}

module cognitiveServices '../ai/cognitiveservices.bicep' = {
  name: 'cognitiveServices'
  params: {
    location: location
    tags: tags
    aiServiceName: aiServicesName
    aiProjectName: aiProjectName
    deployments: aiServiceModelDeployments
    appInsightsId: applicationInsights.outputs.id
    appInsightConnectionName: appInsightConnectionName
    appInsightConnectionString: applicationInsights.outputs.connectionString
    storageAccountId: storageAccount.outputs.id
    storageAccountBlobEndpoint: storageAccount.outputs.primaryEndpoints.blob
    storageAccountConnectionName: storageAccount.outputs.name
  }
}

// AI Services system identity needs blob access for project storage
module backendStorageBlobDataContributorRoleAssignment '../security/role.bicep' = {
  name: 'backend-role-storage-blob-data-contributor'
  params: {
    principalType: 'ServicePrincipal'
    principalId: cognitiveServices.outputs.accountPrincipalId
    roleDefinitionId: 'ba92f5b4-2d11-453d-a403-e96b0029c9fe'
  }
}

module userStorageBlobDataContributorRoleAssignment '../security/role.bicep' = {
  name: 'user-role-storage-blob-data-contributor'
  params: {
    principalType: runnerPrincipalType
    principalId: runnerPrincipalId
    roleDefinitionId: 'ba92f5b4-2d11-453d-a403-e96b0029c9fe'
  }
}

module searchService '../search/search-services.bicep' = if (!empty(searchServiceName)) {
  dependsOn: [cognitiveServices]
  name: 'searchService'
  params: {
    location: location
    tags: tags
    name: searchServiceName
    semanticSearch: 'free'
    authOptions: { aadOrApiKey: { aadAuthFailureMode: 'http401WithBearerChallenge' } }
  }
}

// Outputs
output storageAccountId string = storageAccount.outputs.id
output storageAccountName string = storageAccount.outputs.name
output applicationInsightsName string = !empty(applicationInsightsName) && !empty(logAnalyticsName) ? applicationInsights!.outputs.name : ''
output logAnalyticsWorkspaceName string = !empty(logAnalyticsName) ? logAnalytics!.outputs.name : ''
output aiServicesName string = cognitiveServices.outputs.name
output aiServiceEndpoint string = cognitiveServices.outputs.endpoints['OpenAI Language Model Instance API']
output aiProjectEndpoint string = cognitiveServices.outputs.projectEndpoint
output aiServicePrincipalId string = cognitiveServices.outputs.accountPrincipalId
output searchServiceEndpoint string = !empty(searchServiceName) ? searchService!.outputs.endpoint : ''
output searchServiceName string = !empty(searchServiceName) ? searchService!.outputs.name : ''
output projectResourceId string = cognitiveServices.outputs.projectResourceId
```

### core/ai/cognitiveservices.bicep (AI Services + Project + Connections + Deployments)

```bicep
metadata description = 'Creates an Azure AI Services instance with project and Foundry connections.'
param aiServiceName string
param aiProjectName string
param location string = resourceGroup().location
param tags object = {}
param customSubDomainName string = aiServiceName
param disableLocalAuth bool = true
param deployments array = []
param appInsightsId string
param appInsightConnectionString string
param appInsightConnectionName string
param storageAccountId string
param storageAccountBlobEndpoint string
param storageAccountConnectionName string

@allowed(['Enabled', 'Disabled'])
param publicNetworkAccess string = 'Enabled'
param sku object = { name: 'S0' }

resource account 'Microsoft.CognitiveServices/accounts@2025-04-01-preview' = {
  name: aiServiceName
  location: location
  sku: sku
  kind: 'AIServices'
  identity: { type: 'SystemAssigned' }
  tags: tags
  properties: {
    allowProjectManagement: true
    customSubDomainName: customSubDomainName
    publicNetworkAccess: publicNetworkAccess
    disableLocalAuth: disableLocalAuth
  }
}

// Foundry connection to App Insights
resource appInsightConnection 'Microsoft.CognitiveServices/accounts/connections@2025-04-01-preview' = {
  name: appInsightConnectionName
  parent: account
  properties: {
    category: 'AppInsights'
    target: appInsightsId
    authType: 'ApiKey'
    isSharedToAll: true
    credentials: { key: appInsightConnectionString }
    metadata: { ApiType: 'Azure', ResourceId: appInsightsId }
  }
}

// Foundry connection to Storage
resource storageAccountConnection 'Microsoft.CognitiveServices/accounts/connections@2025-04-01-preview' = {
  name: storageAccountConnectionName
  parent: account
  properties: {
    category: 'AzureStorageAccount'
    target: storageAccountBlobEndpoint
    authType: 'AAD'
    isSharedToAll: true
    metadata: { ApiType: 'Azure', ResourceId: storageAccountId }
  }
}

// AI Project (child of account)
resource aiProject 'Microsoft.CognitiveServices/accounts/projects@2025-04-01-preview' = {
  parent: account
  name: aiProjectName
  location: location
  tags: tags
  identity: { type: 'SystemAssigned' }
  properties: {
    description: 'AI Project'
    displayName: 'AI Project'
  }
}

// Model deployments (batchSize 1 to avoid conflicts)
@batchSize(1)
resource aiServicesDeployments 'Microsoft.CognitiveServices/accounts/deployments@2024-10-01' = [for deployment in deployments: {
  parent: account
  name: deployment.name
  properties: {
    model: deployment.model
    raiPolicyName: contains(deployment, 'raiPolicyName') ? deployment.raiPolicyName : null
  }
  sku: contains(deployment, 'sku') ? deployment.sku : { name: 'Standard', capacity: 20 }
}]

output endpoint string = account.properties.endpoint
output endpoints object = account.properties.endpoints
output id string = account.id
output name string = account.name
output projectResourceId string = aiProject.id
output projectName string = aiProject.name
output projectEndpoint string = aiProject.properties.endpoints['AI Foundry API']
output accountPrincipalId string = account.identity.principalId
output projectPrincipalId string = aiProject.identity.principalId
```

### "Bring Your Own Existing Project" Pattern in main.bicep

When a user has an existing AI project, skip the ai-environment module but still create resources needed for hosting:

```bicep
// In main.bicep — support existing project OR create new
param azureExistingAIProjectResourceId string = ''

// Only create AI environment if no existing project
module ai 'core/host/ai-environment.bicep' = if (empty(azureExistingAIProjectResourceId)) {
  name: 'ai'
  scope: rg
  params: {
    location: location
    tags: tags
    storageAccountName: '${abbrs.storageStorageAccounts}${resourceToken}'
    aiServicesName: 'aoai-${resourceToken}'
    aiProjectName: 'proj-${resourceToken}'
    aiServiceModelDeployments: aiDeployments
    logAnalyticsName: '${abbrs.operationalInsightsWorkspaces}${resourceToken}'
    applicationInsightsName: '${abbrs.insightsComponents}${resourceToken}'
    searchServiceName: useSearchService ? '${abbrs.searchSearchServices}${resourceToken}' : ''
    appInsightConnectionName: 'appinsights-connection'
    runnerPrincipalId: principalId
    runnerPrincipalType: runnerPrincipalType
  }
}

// If bringing existing project, still need Log Analytics for Container Apps
module logAnalytics 'core/monitor/loganalytics.bicep' = if (!empty(azureExistingAIProjectResourceId)) {
  name: 'logAnalytics'
  scope: rg
  params: {
    location: location
    tags: tags
    name: '${abbrs.operationalInsightsWorkspaces}${resourceToken}'
  }
}

// Construct endpoint from resource ID
var existingProjEndpoint = !empty(azureExistingAIProjectResourceId)
  ? format('https://{0}.services.ai.azure.com/api/projects/{1}',
      split(azureExistingAIProjectResourceId, '/')[8],
      split(azureExistingAIProjectResourceId, '/')[10])
  : ''

// Resolve which values to use downstream
var projectResourceId = !empty(azureExistingAIProjectResourceId)
  ? azureExistingAIProjectResourceId
  : ai!.outputs.projectResourceId

var projectEndpoint = !empty(azureExistingAIProjectResourceId)
  ? existingProjEndpoint
  : ai!.outputs.aiProjectEndpoint
```

### Model Deployments Array Pattern

Build model deployment arrays conditionally (e.g., only include embedding if search is enabled):

```bicep
var aiChatModel = [{
  name: chatDeploymentName
  model: { format: chatModelFormat, name: chatModelName, version: chatModelVersion }
  sku: { name: chatDeploymentSku, capacity: chatDeploymentCapacity }
}]

var aiEmbeddingModel = [{
  name: embeddingDeploymentName
  model: { format: embedModelFormat, name: embedModelName, version: embedModelVersion }
  sku: { name: embedDeploymentSku, capacity: embedDeploymentCapacity }
}]

// Only deploy embedding model if search is enabled
var aiDeployments = concat(aiChatModel, useSearchService ? aiEmbeddingModel : [])
```

### Template Validation Mode Pattern

Support CI/CD template validation without real deployment:

```bicep
param templateValidationMode bool = false
@description('Random seed for deterministic resource naming in validation mode')
param seed string = newGuid()

var runnerPrincipalType = templateValidationMode ? 'ServicePrincipal' : 'User'
var resourceToken = templateValidationMode
  ? toLower(uniqueString(subscription().id, environmentName, location, seed))
  : toLower(uniqueString(subscription().id, environmentName, location))
```

### AppInsights Scoped Access Pattern (`core/security/appinsights-access.bicep`)

For granting Monitoring Metrics Contributor scoped to a specific App Insights resource:

```bicep
param principalId string
param principalType string
param appInsightsName string

resource appInsights 'Microsoft.Insights/components@2020-02-02' existing = {
  name: appInsightsName
}

resource role 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(appInsights.id, principalId, '3913510d-42f4-4e42-8a64-420c390055eb')
  scope: appInsights
  properties: {
    principalId: principalId
    principalType: principalType
    roleDefinitionId: resourceId('Microsoft.Authorization/roleDefinitions', '3913510d-42f4-4e42-8a64-420c390055eb')
  }
}
```

Usage in main.bicep:
```bicep
module monitoringMetricsRole 'core/security/appinsights-access.bicep' = if (!empty(resolvedApplicationInsightsName)) {
  name: 'monitoring-metrics-contributor-backend'
  scope: rg
  params: {
    principalType: 'ServicePrincipal'
    appInsightsName: resolvedApplicationInsightsName
    principalId: api.outputs.SERVICE_API_IDENTITY_PRINCIPAL_ID
  }
}
```

### Storage Account with All Service Types (`core/storage/storage-account.bicep`)

Required by AI Foundry — creates containers, file shares, queues, and tables:

```bicep
param name string
param location string = resourceGroup().location
param tags object = {}
param containers array = []
param files array = []
param queues array = []
param tables array = []
param deleteRetentionPolicy object = { allowPermanentDelete: false, enabled: false }
param shareDeleteRetentionPolicy object = { enabled: true, days: 7 }

resource storage 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: name
  location: location
  tags: tags
  kind: 'StorageV2'
  sku: { name: 'Standard_LRS' }
  properties: {
    accessTier: 'Hot'
    allowBlobPublicAccess: false
    allowSharedKeyAccess: true
    minimumTlsVersion: 'TLS1_2'
  }
}

resource blobServices 'Microsoft.Storage/storageAccounts/blobServices@2023-05-01' = {
  parent: storage
  name: 'default'
  properties: { deleteRetentionPolicy: deleteRetentionPolicy }
}

resource blobContainers 'Microsoft.Storage/storageAccounts/blobServices/containers@2023-05-01' = [for container in containers: {
  parent: blobServices
  name: container.name
}]

resource fileServices 'Microsoft.Storage/storageAccounts/fileServices@2023-05-01' = if (!empty(files)) {
  parent: storage
  name: 'default'
  properties: { shareDeleteRetentionPolicy: shareDeleteRetentionPolicy }
}

resource fileShares 'Microsoft.Storage/storageAccounts/fileServices/shares@2023-05-01' = [for file in files: {
  parent: fileServices
  name: file.name
}]

resource queueServices 'Microsoft.Storage/storageAccounts/queueServices@2023-05-01' = if (!empty(queues)) {
  parent: storage
  name: 'default'
}

resource storageQueues 'Microsoft.Storage/storageAccounts/queueServices/queues@2023-05-01' = [for queue in queues: {
  parent: queueServices
  name: queue.name
}]

resource tableServices 'Microsoft.Storage/storageAccounts/tableServices@2023-05-01' = if (!empty(tables)) {
  parent: storage
  name: 'default'
}

resource storageTables 'Microsoft.Storage/storageAccounts/tableServices/tables@2023-05-01' = [for table in tables: {
  parent: tableServices
  name: table.name
}]

output id string = storage.id
output name string = storage.name
output primaryEndpoints object = storage.properties.primaryEndpoints
```
