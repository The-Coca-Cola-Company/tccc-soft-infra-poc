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
3. **No VNet for Agents**: Agents connect via HTTPS, not VNet peering
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

*Last updated: December 2024*  
*Architecture: Multi-Tenant API Management with Private Backend*  
*Deployment Time: ~45 minutes (API Management provisioning)*
