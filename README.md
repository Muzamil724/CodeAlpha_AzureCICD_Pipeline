# CodeAlpha DevOps Internship — Task 1: CI/CD Pipeline using Azure

## Overview
An automated CI/CD pipeline built with **Azure Pipelines** that builds a containerized ASP.NET Core API, pushes it to **Azure Container Registry (ACR)**, and deploys it automatically to **Azure App Service** on every push to `main`.

## Architecture
```
GitHub (main branch push)
        │
        ▼
Azure Pipelines (self-hosted agent)
        │
   ┌────┴────┐
   │  Build  │  → docker build + push → Azure Container Registry
   └────┬────┘
        ▼
   ┌─────────┐
   │ Deploy  │  → pulls image from ACR → Azure App Service (Linux container)
   └─────────┘
```

## Stack
- **App**: Minimal ASP.NET Core API (.NET 10) with a `/health` endpoint
- **Containerization**: Multi-stage Dockerfile (SDK image for build, ASP.NET runtime image for execution)
- **Registry**: Azure Container Registry — `acrdevopstask1muzamil` (Basic SKU)
- **CI/CD**: Azure Pipelines, using a **self-hosted agent** (local machine, pool `local-agents`) instead of Microsoft-hosted agents
- **Hosting**: Azure App Service — `muzamil-task1-api` (Linux, container-based, F1 Free tier)
- **Resource Group**: `rg-devops-task1` (region: `uaenorth`)
- **Auth**: Managed Identity + `AcrPull` role assignment (no stored registry credentials)

## Setup Steps

1. **Build the app and Dockerfile** — multi-stage build, `dotnet publish` in a build stage, copied into a lightweight ASP.NET runtime stage.

2. **Create Azure resources**:
   ```bash
   az group create --name rg-devops-task1 --location uaenorth
   az acr create --resource-group rg-devops-task1 --name acrdevopstask1muzamil --sku Basic
   az appservice plan create --name asp-devops-task1 --resource-group rg-devops-task1 --is-linux --sku F1
   az webapp create --resource-group rg-devops-task1 --plan asp-devops-task1 --name muzamil-task1-api \
     --deployment-container-image-name acrdevopstask1muzamil.azurecr.io/muzamilcodealphaazurecicdpipeline:5
   ```

3. **Grant App Service pull access to ACR via Managed Identity**:
   ```bash
   az webapp identity assign --resource-group rg-devops-task1 --name muzamil-task1-api --query principalId --output tsv

   az acr show --name acrdevopstask1muzamil --query id --output tsv

   az role assignment create \
     --assignee <principalId-from-first-command> \
     --scope <acr-resource-id-from-second-command> \
     --role "AcrPull"

   az webapp config set --resource-group rg-devops-task1 --name muzamil-task1-api \
     --generic-configurations '{"acrUseManagedIdentityCreds": true}'
   ```

4. **Set the container port** — App Service defaults to port 80; this app's container listens on 8080:
   ```bash
   az webapp config appsettings set --resource-group rg-devops-task1 --name muzamil-task1-api --settings WEBSITES_PORT=8080
   ```

5. **Set up a self-hosted Azure Pipelines agent** (used instead of Microsoft-hosted agents, which require a parallelism grant not available by default on new/student orgs):
   ```bash
   mkdir ~/myagent && cd ~/myagent
   tar zxvf ~/Downloads/vsts-agent-linux-x64-5.279.0.tar.gz
   ./config.sh   # pool: local-agents
   ./run.sh
   ```

6. **Pipeline YAML** (`azure-pipelines.yml`) — two stages:

   ```yaml
   trigger:
   - main

   resources:
   - repo: self

   variables:
     dockerRegistryServiceConnection: 'aeae1097-6c17-47d1-9a92-d875730ef686'
     imageRepository: 'muzamilcodealphaazurecicdpipeline'
     containerRegistry: 'acrdevopstask1muzamil.azurecr.io'
     dockerfilePath: '$(Build.SourcesDirectory)/api/Dockerfile'
     tag: '$(Build.BuildId)'

   stages:
   - stage: Build
     displayName: Build and push stage
     jobs:
     - job: Build
       pool:
         name: local-agents
       steps:
       - task: Docker@2
         inputs:
           command: buildAndPush
           repository: $(imageRepository)
           dockerfile: $(dockerfilePath)
           containerRegistry: $(dockerRegistryServiceConnection)
           tags: |
             $(tag)

   - stage: Deploy
     displayName: Deploy stage
     dependsOn: Build
     jobs:
     - job: Deploy
       pool:
         name: local-agents
       steps:
       - task: AzureWebAppContainer@1
         inputs:
           azureSubscription: 'azure-rg-devops-task1-connection'
           appName: 'muzamil-task1-api'
           containers: '$(containerRegistry)/$(imageRepository):$(tag)'
   ```

   - **Build**: `Docker@2` task — builds the image and pushes to ACR, tagged with `$(Build.BuildId)`
   - **Deploy**: `AzureWebAppContainer@1` task — deploys the same build's image to App Service, using an Azure Resource Manager service connection scoped to `rg-devops-task1`

## Result
Every push to `main` automatically builds, containerizes, pushes to ACR, and redeploys the live app — verified working end-to-end:
```bash
curl https://muzamil-task1-api.azurewebsites.net/health
# → OK
```

## Key Concepts Demonstrated
- Multi-stage Docker builds (build vs. runtime image separation)
- Azure Container Registry as a private image registry
- Pipeline-as-code with Azure Pipelines YAML
- Self-hosted CI/CD agents vs. Microsoft-hosted agents
- Managed Identity for credential-free service-to-service auth
- Container port configuration on Azure App Service

## Part of
CodeAlpha DevOps Internship — one of three completed tasks (alongside a Dockerized Nginx web server and a Jenkins master/remote-agent setup).
