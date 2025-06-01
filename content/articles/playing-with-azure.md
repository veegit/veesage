+++
title = 'Agentic Platform - Chapter 4: Deploy to Azure'
date = 2025-05-27T00:55:00-07:00
ShowToc = true
TocOpen = false
+++

Now that the application is [working locally](/articles/playing-with-windsurf/), let's deploy it to the cloud. I chose Azure since I get a generous $150 credit allowance as a Microsoft employee (thanks, employer!). The process was surprisingly straightforward. I remember my AWS deployment days from more than a decade ago—fun, but somewhat tedious. With the `az CLI, deployment was a breeze.

There were few deployment options available (arent we spoiled with choices). 
I chose Azure Container Apps since it offers auto-scaling, managed certificates, and seamless container registry integration with lower operational overhead than Kubernetes(AKS). AKS provides maximum control and full Kubernetes capabilities but adds complexity, while ACI is best for simple deployments and App Service works well for primarily web/API-focused applications. 

To refresh the memory, our Agentic platform was divided into 4 services with a Redis Storage

{{< mermaid >}}
graph TD
    User[User] --> API[API Service]
    API --> Agent[Agent Service]
    API --> Lifecycle[Agent Lifecycle Service]
    Agent --> Skill[Skill Service]
    Agent --> Redis[(Redis Cache)]
    Lifecycle --> Redis
    Skill --> Redis
    
    subgraph "Three-Step Process"
    Reasoning[Reasoning Phase] --> SkillExecution[Skill Execution Phase]
    SkillExecution --> Response[Response Formulation Phase]
    end
    
    Agent --> Reasoning
    Skill --> SkillExecution
    Agent --> Response
{{< /mermaid >}}

We need to make some changes before we could deploy the application to Azure.
1. Redis Storage - Move it to Azure Cache for Redis
2. Use KeyVault - Move the passwords/secrets to Azure KeyVault
3. Change the Client code to refer to the remote URLs.

### Preparing for Azure
We created a Dockerfile that could handle all our services:
```Dockerfile
FROM python:3.11-slim

# Set working directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    curl \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements file
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Install LangGraph
RUN pip install --no-cache-dir "langgraph==0.0.15"

# Copy project files
COPY . .

# Create non-root user
RUN adduser --disabled-password --gecos "" appuser
RUN chown -R appuser:appuser /app

# Switch to non-root user
USER appuser

# Get service to run from build arg (default to API service)
ARG SERVICE=api_service
ENV SERVICE_ENV=${SERVICE}

# Dynamic service selection
CMD if [ "$SERVICE_ENV" = "api_service" ]; then \
        python -m services.api.main; \
    elif [ "$SERVICE_ENV" = "agent_service" ]; then \
        python -m services.agent_service.main; \
    elif [ "$SERVICE_ENV" = "agent_lifecycle" ]; then \
        python -m services.agent_lifecycle.main; \
    elif [ "$SERVICE_ENV" = "skill_service" ]; then \
        python -m services.skill_service.main; \
    else \
        python -m services.api.main; \
    fi
```

### Setting Up Azure Authentication and Prerequisites
```
# Login to Azure CLI
az login

# Verify the login and list available subscriptions
az account list --output table

# Set the subscription to use (if you have multiple)
az account set --subscription "your-subscription-id"
```

### Setting up Required Resource Providers
Its a one time setup of resource providers before we start creating the resources.

```
# Register resource providers
az provider register --namespace Microsoft.App
az provider register --namespace Microsoft.ContainerRegistry
az provider register --namespace Microsoft.Cache
az provider register --namespace Microsoft.KeyVault

# Verify registration status
az provider show --namespace Microsoft.App --query "registrationState"
```

### Setting Up Docker for Azure Container Registry and Eventual Resources
```
# Log in to Azure Container Registry
az acr login --name agentplatformacr

docker pull hello-world
docker tag hello-world agentplatformacr.azurecr.io/hello-world
docker push agentplatformacr.azurecr.io/hello-world
```

After authentication and resource provider registration, we created the necessary Azure resources:
```
# Create a resource group
az group create --name agent-platform-rg --location eastus

# Create Azure Container Registry (ACR)
az acr create --resource-group agent-platform-rg --name agentplatformacr --sku Basic

# Create Azure Container Apps environment
az containerapp env create \
  --name agent-platform-env \
  --resource-group agent-platform-rg \
  --location eastus

# Create Azure Redis Cache
az redis create \
  --name agent-platform-redis \
  --resource-group agent-platform-rg \
  --location eastus \
  --sku Basic \
  --vm-size C0
```

### Managing Secrets with Azure Key Vault
```
# Create a Key Vault
az keyvault create --name agent-platform-kv --resource-group agent-platform-rg --location eastus

# Add secrets to the Key Vault
az keyvault secret set --vault-name agent-platform-kv --name "REDIS-PASSWORD" --value "your-redis-password"
az keyvault secret set --vault-name agent-platform-kv --name "SERP-API-KEY" --value "your-serpapi-key"
az keyvault secret set --vault-name agent-platform-kv --name "GROQ-API-KEY" --value "your-groq-api-key"
```

### Setting Up Managed Identity for Accessing Secrets
```
# Create a user-assigned managed identity
az identity create --name agent-platform-identity --resource-group agent-platform-rg --location eastus

# Get the identity client ID and resource ID
IDENTITY_CLIENT_ID=$(az identity show --name agent-platform-identity --resource-group agent-platform-rg --query clientId -o tsv)
IDENTITY_RESOURCE_ID=$(az identity show --name agent-platform-identity --resource-group agent-platform-rg --query id -o tsv)

# Grant the identity access to Key Vault secrets
az keyvault set-policy --name agent-platform-kv --object-id $(az identity show --name agent-platform-identity --resource-group agent-platform-rg --query principalId -o tsv) --secret-permissions get list
```

### Referencing Key Vault Secrets in Container Apps

```
# Get Key Vault ID
KEYVAULT_ID=$(az keyvault show --name agent-platform-kv --resource-group agent-platform-rg --query id -o tsv)

# Create Container App with Key Vault references
az containerapp create \
  --name agent-platform-api \
  --resource-group agent-platform-rg \
  --environment agent-platform-env \
  --image agentplatformacr.azurecr.io/agent-platform-api:latest \
  --target-port 8000 \
  --ingress external \
  --registry-server agentplatformacr.azurecr.io \
  --user-assigned $IDENTITY_RESOURCE_ID \
  --secrets "REDIS_PASSWORD=keyvaultref:${KEYVAULT_ID}/secrets/REDIS-PASSWORD" \
            "SERPAPI_API_KEY=keyvaultref:${KEYVAULT_ID}/secrets/SERP-API-KEY" \
            "GROQ_API_KEY=keyvaultref:${KEYVAULT_ID}/secrets/GROQ-API-KEY"
```

In the container code, these secrets are accessed as regular environment variables

```Python
import os

# Access secrets as environment variables
redis_password = os.environ.get("REDIS_PASSWORD")
serpapi_key = os.environ.get("SERPAPI_API_KEY")
groq_api_key = os.environ.get("GROQ_API_KEY")
```

### Deploying the Services
```
# Build images for each service
docker build --platform linux/amd64 -t agentplatformacr.azurecr.io/agent-platform-api:latest -f Dockerfile .
docker build --platform linux/amd64 -t agentplatformacr.azurecr.io/agent-platform-agent:latest -f Dockerfile . --build-arg SERVICE=agent_service
docker build --platform linux/amd64 -t agentplatformacr.azurecr.io/agent-platform-lifecycle:latest -f Dockerfile . --build-arg SERVICE=agent_lifecycle
docker build --platform linux/amd64 -t agentplatformacr.azurecr.io/agent-platform-skill:latest -f Dockerfile . --build-arg SERVICE=skill_service

# Push all images to ACR
docker push agentplatformacr.azurecr.io/agent-platform-api:latest
docker push agentplatformacr.azurecr.io/agent-platform-agent:latest
docker push agentplatformacr.azurecr.io/agent-platform-lifecycle:latest
docker push agentplatformacr.azurecr.io/agent-platform-skill:latest
```

### Common Deployment Challenges
#### Challenge #1: Container Startup Issues
Our first major challenge was containers failing to start with exit code 1. The logs showed:

```bash
Container was terminated with exit code '1' and reason 'ProcessExited'
/usr/local/bin/python: Error while finding module specification for 'services.skill.main' (ModuleNotFoundError: No module named 'services.skill')
```

We discovered our Dockerfile was using incorrect module paths. We fixed it by ensuring the PYTHONPATH environment variable was correctly set and adjusting our module import paths.

```bash
# Fixed Dockerfile with correct Python path setup
ENV PYTHONPATH=/app
```

#### Challenge #2: Service Communication Failure
Services couldn't talk to each other. The logs showed:

```bash
Error getting agent: All connection attempts failed
Requesting URL: http://localhost:8001/agents/b73f7a1f-72f6-4833-8a5d-8d82b056ae37
```

We found two issues:

1. Environment variable naming mismatch:
```python
# Code was looking for:
self.base_url = base_url or os.environ.get("AGENT_LIFECYCLE_URL", "http://localhost:8001")

# But we were setting:
AGENT_LIFECYCLE_SERVICE_URL=https://agent-platform-lifecycle.purplewater-c0416f2a.eastus.azurecontainerapps.io
```

2. We needed to explicitly set service URLs in the environment:
```bash 
# Update service URLs for inter-service communication
az containerapp update --name agent-platform-api --resource-group agent-platform-rg \
  --set-env-vars "AGENT_LIFECYCLE_URL=https://agent-platform-lifecycle.purplewater-c0416f2a.eastus.azurecontainerapps.io"
```

#### Challenge #3: Frontend Configuration
The frontend JavaScript was hardcoding localhost URLs:
`fetch('http://localhost:8001/agents')`
We created a dynamic configuration system that detects the environment:

```javascript
const CONFIG = {
  // Default to local development URLs
  API_URL: 'http://localhost:8000',
  LIFECYCLE_URL: 'http://localhost:8001',
  AGENT_URL: 'http://localhost:8003',
  SKILL_URL: 'http://localhost:8002',
  
  // Check if we're running in Azure environment
  init: function() {
    // Detect Azure environment
    if (window.location.hostname.includes('azurecontainerapps.io')) {
      this.API_URL = 'https://agent-platform-api.purplewater-c0416f2a.eastus.azurecontainerapps.io';
      this.LIFECYCLE_URL = 'https://agent-platform-lifecycle.purplewater-c0416f2a.eastus.azurecontainerapps.io';
      this.AGENT_URL = 'https://agent-platform-agent.purplewater-c0416f2a.eastus.azurecontainerapps.io';
      this.SKILL_URL = 'https://agent-platform-skill.purplewater-c0416f2a.eastus.azurecontainerapps.io';
    }
  }
};

// Initialize configuration
CONFIG.init();
```

Then we updated our frontend JavaScript to use this configuration:

```javascript
// Before:
fetch(`http://localhost:8003/conversations/${conversationId}/messages`, {/*...*/})

// After:
fetch(CONFIG.API_URL + `/conversations/${conversationId}/messages`, {/*...*/})
```

#### Challenge #4: Resource Costs
After a day of running the service, I was noticing the costs were going up and it was matching the 1 vCPU cost+memory. 

We then reduced the min replicas to 0 from default 1

```bash
az containerapp update --name agent-platform-lifecycle --resource-group $RESOURCE_GROUP \
  --min-replicas 0 \
  --max-replicas 3
```

### Demo of the Agent Platform in Azure
With some prereading of good azure documentation + relying on Claude Chat, I was able to get the service running in Azure.

_Look on my works, ye Mighty, and despair_
![](/images/url-azure.png)
 <button class="clipboard-btn" onclick="copyToClipboard('https://agent-platform-api.purplewater-c0416f2a.eastus.azurecontainerapps.io/')">Copy URL📋</button>