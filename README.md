# 🚀 TCCC Soft Hub - Azure Infrastructure Deployment

![tccc-multiagent-azure-infra](deployment/img/image%20cross.png)

This repository contains the Azure Resource Manager (ARM) template for deploying the **TCCC Hub** - the central orchestrator in the TCCC multi-tenant hub-spoke architecture.

## 📋 Overview

The TCCC Hub is the **central orchestrator** that:
- ✅ Routes requests between bottler agents
- ✅ Enforces hub-spoke communication patterns
- ✅ Provides API Management for all bottlers
- ✅ Manages AI orchestration with Azure AI Foundry Hub

## 🏗️ Infrastructure Components

### Core Services Deployed
- **🤖 Azure AI Foundry Hub** - Central AI orchestration
- **🔐 API Management** - Gateway for all bottler communications
- **💾 Storage Account** - Primary storage
- **🗄️ Cosmos DB** - NoSQL database for configuration and state
- **📨 Service Bus** - Messaging with smart routing rules
- **⚡ Event Grid** - Event-driven architecture
- **📊 Log Analytics & Application Insights** - Monitoring
- **⚙️ Function App** (Python 3.11) - Hub orchestration logic
- **NO VNet** - Simplified deployment without virtual networks

## 🚀 Quick Deploy

### Prerequisites
- Azure subscription with appropriate permissions
- Azure CLI installed
- Publisher email for API Management

### Option 1: Using Azure Portal

Click the button below to deploy directly to Azure:

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FThe-Coca-Cola-Company%2Ftccc-hub-infrastructure%2Fmain%2Fdeployment%2Ftccc-soft-hub-complete-infra.json)

### Option 2: Using Azure CLI

```bash
# Create resource group
az group create --name rg-tccc-hub-prod --location eastus

# Deploy infrastructure
az deployment group create \
  --resource-group rg-tccc-hub-prod \
  --template-file tccc-soft-hub-complete-infra.json \
  --parameters publisherEmail="admin@coca-cola.com"
```

### Option 3: Using Parameters File

```bash
# Deploy with parameters file
az deployment group create \
  --resource-group rg-tccc-hub-prod \
  --template-file tccc-soft-hub-complete-infra.json \
  --parameters parameters.prod-complete-infra.json \
  --parameters mcpAuthToken=$MCP_AUTH_TOKEN
```

## 📝 Parameters

### Required Parameters
- `publisherEmail` - Email for API Management admin

### Optional Parameters with Defaults
- `environment` - "dev", "test", or "prod" (default: "dev")
- `location` - Azure region (default: resource group location)
- `apiManagementSku` - "Developer" or "Premium" (default: "Developer")
- `publisherName` - Organization name (default: "The Coca-Cola Company")
- `mcpAuthToken` - Token for MCP servers (can be set post-deployment)
- `enableServiceBusSmartRules` - Enable intelligent routing (default: true)

## 🔧 Post-Deployment Configuration

### 1. Apply API Management Policies

The hub-spoke enforcement policies are critical for security:

```bash
# Navigate to policies folder
cd apim-policies/

# Apply policies via Azure Portal:
# API Management → APIs → All operations → Inbound processing → Code editor
# Copy content from apim-policies-complete.xml
```

### 2. Configure Azure AI Foundry

After deployment, you'll need to:
1. Create AI model deployments in the AI Foundry Hub
2. Update Function App settings with endpoint and key:

```bash
FUNCTION_APP_NAME="tccc-soft-hub-prod-func-xxxxx"  # From deployment output

az functionapp config appsettings set \
  --name $FUNCTION_APP_NAME \
  --resource-group rg-tccc-hub-prod \
  --settings \
    "AZURE_AI_FOUNDRY_ENDPOINT=https://your-ai-foundry.openai.azure.com/" \
    "AZURE_AI_FOUNDRY_KEY=your-key"
```

### 3. Create Bottler Subscriptions in API Management

For each bottler that will connect:

```bash
APIM_NAME="tccc-soft-hub-prod-apim-xxxxx"  # From deployment output

# Create subscription for ARCA
az apim subscription create \
  --resource-group rg-tccc-hub-prod \
  --service-name $APIM_NAME \
  --name "arca-subscription" \
  --display-name "ARCA Continental" \
  --scope "/apis" \
  --state active

# Get the subscription key for the bottler
az apim subscription show \
  --resource-group rg-tccc-hub-prod \
  --service-name $APIM_NAME \
  --name "arca-subscription" \
  --query primaryKey -o tsv
```

### 4. Test the Deployment

```bash
# Get Function App URL
FUNCTION_URL=$(az deployment group show \
  --resource-group rg-tccc-hub-prod \
  --name {deployment-name} \
  --query properties.outputs.functionAppUrl.value -o tsv)

# Test health endpoint
curl https://$FUNCTION_URL/api/health
```

## 📊 Architecture

```
Bottler Request → API Management (Gateway) → TCCC Hub Function → Route to Target Bottler
                        ↓
                  Rate Limiting
                  Authentication
                  Hub-Spoke Validation
```

### Key Security Features
- **No Direct Bottler Communication** - Enforced by API Management policies
- **Rate Limiting** - Per bottler limits
- **Managed Identities** - No passwords in code
- **TLS 1.2** - Minimum across all services

## 🎯 Outputs

The deployment provides these outputs:
- `functionAppName` - Name of the Hub Function App
- `functionAppUrl` - URL for the Hub endpoints
- `apiManagementName` - Name of API Management instance
- `eventGridTopicEndpoint` - Event Grid endpoint
- `mlWorkspaceId` - AI Foundry Hub workspace ID

## 🤝 Connecting Bottlers

Each bottler needs:
1. **API Management Subscription Key** - Provided by TCCC admin
2. **Hub URL** - The API Management gateway URL
3. **Their own infrastructure** - Deployed using the bottler template

## 📚 Related Documentation

- [Bottler Infrastructure Deployment](../../soft-bottler-manager/deployment/README.md)
- [Multi-Tenant Architecture Guide](../../../../docs/architecture.md)
- [API Management Policies](apim-policies/README.md)

## ⚠️ Important Notes

1. **AI Foundry Configuration** - Must be completed post-deployment
2. **API Management Policies** - Critical for hub-spoke security
3. **No VNet** - This template uses public endpoints with security controls
4. **Consumption Plan** - Function App uses Y1 (consumption) plan for cost efficiency

---

**Version**: 1.1.0  
**Last Updated**: 2025-08-06  
**Template**: `tccc-soft-hub-complete-infra.json` (v4.1.0.0)  
**Maintained By**: TCCC Emerging Technologies (cvanegas@coca-cola.com)