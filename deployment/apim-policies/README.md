# APIM Policies for TCCC Hub

This folder contains API Management policies that enforce the Hub-Spoke architecture and provide security controls.

## Files

### apim-policies-complete.xml
Full policy set including:
- Basic security and rate limiting
- MCP-specific controls
- Extended timeouts and quotas
- Hub-Spoke enforcement

**Use this for production deployments**

### apim-policies-basic.xml
Basic policy set with:
- Standard rate limiting
- Basic Hub-Spoke enforcement
- Security headers

**Use this for initial testing**

## Critical Hub-Spoke Enforcement

Both policies include the critical validation that prevents direct bottler-to-bottler communication:

```xml
<choose>
    <when condition="@(context.Request.Headers.GetValueOrDefault("X-Target-Bottler","") != "" && 
                       context.Request.Headers.GetValueOrDefault("X-Target-Bottler","") != context.Variables.GetValueOrDefault<string>("bottlerId"))">
        <return-response>
            <set-status code="403" reason="Forbidden" />
            <set-body>{"error": "Hub-Spoke violation: Direct bottler-to-bottler communication is not allowed"}</set-body>
        </return-response>
    </when>
</choose>
```

## How to Apply

After deploying the TCCC Hub infrastructure:

1. Go to Azure Portal → API Management → APIs
2. Select your API → All operations
3. Click on "Inbound processing" → Code editor
4. Copy the content from the appropriate XML file
5. Save and test

Or use Azure CLI:
```bash
az apim api policy set \
  --resource-group <rg-name> \
  --service-name <apim-name> \
  --api-id <api-id> \
  --policy-file apim-policies-complete.xml
```