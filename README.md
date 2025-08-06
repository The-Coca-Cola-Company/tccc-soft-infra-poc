
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

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FThe-Coca-Cola-Company%2Ftccc-soft-infra-poc%2Fmain%2Fdeployment%2Ftccc-soft-hub-complete-infra.json)


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
=======
# 🏢 TCCC Soft Hub - Azure API Management Deployment Guide

![tccc-soft-hub-architecture](img/image%20cross.png)

> 🚨 **CRITICAL: API Management Deployment Takes ~45 Minutes**  
> API Management provisioning is a long-running operation. Plan accordingly for production deployments.

---

## 🔄 Single-Stage Deployment Process

### Why Single Stage for Soft Architecture?
Unlike VNet-based deployments, the Soft Hub uses API Management as the central gateway. This eliminates Key Vault race conditions since agents connect via HTTPS with subscription keys, not VNet integration.

---

## Step 1: 🚀 Complete Hub Infrastructure Deployment

This deploys all hub infrastructure in a single operation:

**Includes:**
- ✅ API Management (Internal mode in VNet)
- ✅ Virtual Network with subnets
- ✅ Function App with VNet integration
- ✅ Azure AI Foundry (ML/AI orchestration)
- ✅ Cosmos DB (agent registry & state)
- ✅ Service Bus (async messaging)
- ✅ Event Grid (real-time events)
- ✅ Blob Storage (data persistence)
- ✅ Private DNS Zones (10 zones)
- ✅ Private Endpoints (all services)

### 1️⃣ Production Deployment (Premium SKU)
[![Deploy TCCC Soft Hub Premium](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fyour-repo%2Fsoft-poc%2Fmain%2Ftccc-soft-manager%2Fdeployment%2Ftccc-soft-hub-main.json)

### 2️⃣ Development Deployment (Developer SKU)
[![Deploy TCCC Soft Hub Developer](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fyour-repo%2Fsoft-poc%2Fmain%2Ftccc-soft-manager%2Fdeployment%2Ftccc-soft-hub-main.json)

*Same template - choose SKU during deployment*

---

### ⏳ **Deployment takes ~45 minutes** due to API Management provisioning

---

## Step 2: 🔧 Post-Deployment Configuration

### Configure API Management
After deployment completes, import the API definition and configure agent subscriptions:

```powershell
# Get deployment outputs
$apimName = az deployment group show `
  --resource-group rg-tccc-soft-hub-prod `
  --name tccc-soft-hub-main `
  --query properties.outputs.apiManagementName.value -o tsv

$funcName = az deployment group show `
  --resource-group rg-tccc-soft-hub-prod `
  --name tccc-soft-hub-main `
  --query properties.outputs.functionAppName.value -o tsv

# Import API definition
az apim api import `
  --resource-group rg-tccc-soft-hub-prod `
  --service-name $apimName `
  --path hub `
  --api-id tccc-hub-api `
  --specification-path deployment/api-definition.json `
  --specification-format OpenApi

# Apply hub-spoke enforcement policy
az apim api policy create `
  --resource-group rg-tccc-soft-hub-prod `
  --service-name $apimName `
  --api-id tccc-hub-api `
  --policy-file-path deployment/apim-policies.xml
```

### Create Agent Subscriptions
```bash
# Create subscription for each agent
az apim subscription create `
  --resource-group rg-tccc-soft-hub-prod `
  --service-name $apimName `
  --name "agent-mexico-norte" `
  --api-id tccc-hub-api `
  --display-name "Agent Mexico Norte"

# Get subscription key to share with agent
az apim subscription show-keys `
  --resource-group rg-tccc-soft-hub-prod `
  --service-name $apimName `
  --subscription-id "agent-mexico-norte"
```

---

## 📋 Prerequisites

- Active TCCC Azure subscription
- Contributor or Owner permissions
- Azure CLI installed and configured
- PowerShell for configuration scripts
- ~45 minutes for API Management provisioning

---

## 🎯 CLI Deployment (Advanced)

### Complete Production Deployment

```bash
# Login to Azure with TCCC account
az login --tenant tccc.onmicrosoft.com

# Create Resource Group
az group create --name rg-tccc-soft-hub-prod --location eastus

# Deploy Complete Infrastructure (~45 minutes)
az deployment group create \
  --resource-group rg-tccc-soft-hub-prod \
  --template-file deployment/tccc-soft-hub-main.json \
  --parameters \
    environment=prod \
    apiManagementSku=Premium \
    publisherEmail="admin@tccc.com" \
    publisherName="The Coca-Cola Company"

# Deploy Function App Code
cd ..
func azure functionapp publish $(az deployment group show \
  --resource-group rg-tccc-soft-hub-prod \
  --name tccc-soft-hub-main \
  --query properties.outputs.functionAppName.value -o tsv)
```

### Development/Testing Deployment

```bash
# Create Resource Group
az group create --name rg-tccc-soft-hub-dev --location eastus

# Deploy with Developer SKU (cheaper, no Private Link)
az deployment group create \
  --resource-group rg-tccc-soft-hub-dev \
  --template-file deployment/tccc-soft-hub-main.json \
  --parameters \
    environment=dev \
    apiManagementSku=Developer \
    publisherEmail="dev@tccc.com" \
    publisherName="TCCC Development"
```

---

## 📊 Architecture Overview

| Component | Purpose | Access Model |
|-----------|---------|--------------|
| API Management | Central gateway for all agents | HTTPS + Subscription Keys |
| Function App | Hub orchestration logic | VNet integrated |
| Cosmos DB | Agent registry & state | Private Endpoint |
| Service Bus | Async agent messaging | Private Endpoint |
| Event Grid | Real-time notifications | Private Endpoint |
| Blob Storage | Data persistence | Private Endpoint |
| AI Foundry | ML/AI orchestration | Private Endpoint |
| Key Vault | Secrets management | Private Endpoint |

---

## 🔐 Security Architecture

### Network Security
- API Management in Internal mode (VNet deployment)
- All backend services use Private Endpoints
- 10 Private DNS Zones for name resolution
- No public access to backend services

### Agent Authentication
- Each agent has unique subscription key
- Rate limiting per agent (100 req/min)
- Daily quota per agent (10,000 req/day)
- Hub-spoke enforcement prevents agent-to-agent communication

### Multi-Tenant Support
- Hub in TCCC tenant/subscription
- Agents in separate tenants/subscriptions
- No VNet peering required
- Communication over secure HTTPS

---

## 🔧 Post-Deployment Steps

### 1. Verify deployment:
```bash
# Check API Management health
curl https://$apimName.azure-api.net/hub/health

# Test Function App (internal)
az webapp show --name $funcName --resource-group rg-tccc-soft-hub-prod
```

### 2. Share with agents:
```bash
# Information to provide each agent:
echo "API Gateway URL: https://$apimName.azure-api.net"
echo "Subscription Key: [from subscription creation above]"
echo "API Documentation: deployment/api-definition.json"
```

### 3. Monitor deployment:
```bash
# View API Management analytics
az apim show --name $apimName --resource-group rg-tccc-soft-hub-prod

# Check Application Insights
az monitor app-insights component show \
  --app $(az deployment group show \
    --resource-group rg-tccc-soft-hub-prod \
    --name tccc-soft-hub-main \
    --query properties.outputs.appInsightsName.value -o tsv) \
  --resource-group rg-tccc-soft-hub-prod
```

---

## ⚠️ Important Notes

1. **API Management Provisioning**: Takes ~45 minutes - plan accordingly
2. **Premium vs Developer SKU**: Premium required for Private Link in production
3. **No VNet for Agents**: Agents connect via HTTPS, not VNet peering (POC)
4. **Subscription Keys**: Store securely in agent Key Vaults
5. **Rate Limits**: Configure based on agent needs
6. **Hub-Spoke Enforcement**: Automatic via API Management policy

---

## 🆘 Troubleshooting

### Common Issues:

**❌ API Management deployment timeout**
- **Cause**: Normal - provisioning takes ~45 minutes
- **Solution**: Wait for completion, check activity log

**❌ Function App can't reach backend services**
- **Cause**: Private Endpoint DNS resolution issues
- **Solution**: Verify VNet integration and DNS zones

**❌ Agent gets 401 Unauthorized**
- **Cause**: Invalid or missing subscription key
- **Solution**: Verify key and subscription status

**❌ Agent gets 403 Forbidden**
- **Cause**: Hub-spoke violation (agent-to-agent attempt)
- **Solution**: All requests must target hub endpoints only

**❌ Agent gets 429 Too Many Requests**
- **Cause**: Rate limit exceeded
- **Solution**: Implement retry logic or increase limits

---

## 📁 Repository Structure

```
tccc-soft-manager/
├── deployment/
│   ├── tccc-soft-hub-main.json    # ARM template
│   ├── api-definition.json        # OpenAPI spec
│   ├── apim-policies.xml         # Hub-spoke policy
│   └── img/
│       └── image cross.png       # Architecture diagram
├── function_app.py               # Hub logic
├── requirements.txt              # Python dependencies
└── README.md                     # This file
```

---

## 🆘 Support

- **Architecture Questions**: Emerging Technology Team
- **API Management Issues**: api-support@tccc.com
- **Security Concerns**: security@tccc.com
- **Agent Onboarding**: integration@tccc.com

---

## 📚 Related Documentation

- [Multi-Tenant Setup Guide](../MULTI-TENANT-SETUP.md)
- [Agent Connection Guide](../AGENT-HUB-CONNECTION-GUIDE.md)
- [Hub-Spoke Validation](../HUB-SPOKE-VALIDATION.md)
- [Architecture Verification](../ARCHITECTURE-VERIFICATION.md)

---

## 📊 API Management SKU Comparison

| Feature | Developer SKU | Premium SKU | Impact on Your Architecture |
|---------|---------------|-------------|----------------------------|
| **Monthly Cost** | ~$50 | ~$3,000+ | Significant cost difference |
| **Deployment Purpose** | Dev/Test/Demo | Production | Choose based on environment |
| **SLA** | None | 99.95% | Availability guarantee |
| **Private Link Support** | ❌ | ✅ | Not needed (agents use HTTPS) |
| **Virtual Network** | ✅ Internal mode | ✅ Internal mode | Both support VNet deployment |
| **Subscription Keys** | ✅ | ✅ | Your authentication method |
| **Rate Limiting** | ✅ | ✅ | Per-agent limits work |
| **Custom Policies** | ✅ | ✅ | Hub-spoke enforcement works |
| **OAuth 2.0/JWT** | ✅ | ✅ | Available if needed |
| **Multi-region** | ❌ | ✅ | Geo-redundancy |
| **Auto-scaling** | Limited | ✅ Full | Performance under load |
| **Developer Portal** | ✅ | ✅ | API documentation |
| **Analytics** | ✅ | ✅ | Usage monitoring |
| **Security Features** | ✅ All features | ✅ All features | Same security capabilities |

### 🎯 Recommendation
- **For Demo/POC/Dev**: Use Developer SKU - fully functional and secure
- **For Production**: Use Premium SKU - adds SLA and enterprise features
- **Key Point**: Security features are identical - Developer SKU is secure for your use case

---

*Last updated: 8/6/2025*  
*Architecture: Multi-Tenant API Management with Private Backend*  
*Deployment Time: ~45 minutes (API Management provisioning)*

