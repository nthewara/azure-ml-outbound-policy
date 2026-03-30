# Azure ML / AI Studio — Outbound Rules Custom Policy

Custom Azure Policy to govern outbound rules on Azure Machine Learning and AI Studio managed networks.

## Overview

When Azure ML workspaces use **Managed VNet** with `AllowOnlyApprovedOutbound` isolation mode, outbound rules control what the workspace can reach. This policy lets you enforce which outbound rules are permitted — blocking any rule that doesn't match your approved list.

## How it works

The policy evaluates the **child resource type** `Microsoft.MachineLearningServices/workspaces/outboundRules`. Every time an outbound rule is created or modified, the policy checks it against your approved lists.

### Rule types governed

| Rule Type | Parameter | What it checks |
|-----------|-----------|----------------|
| **FQDN** | `allowedFQDNs` | Destination FQDN (e.g. `pypi.org`, `*.anaconda.com`) |
| **ServiceTag** | `allowedServiceTags` | Azure Service Tag (e.g. `AzureActiveDirectory`, `AzureMonitor`) |
| **PrivateEndpoint** | `allowedPrivateEndpointSubresources` | Subresource target (e.g. `blob`, `vault`, `amlworkspace`) |

### System rules

By default (`allowSystemRules: true`), rules with category `SystemDefault` or `Required` are always permitted. Azure creates these automatically for workspace functionality. Set to `false` if you want full control.

## Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `effect` | String | `Audit` | `Audit`, `Deny`, or `Disabled` |
| `allowedFQDNs` | Array | `[]` | Approved FQDN destinations |
| `allowedServiceTags` | Array | `[]` | Approved Azure Service Tags |
| `allowedPrivateEndpointSubresources` | Array | `[]` | Approved PE subresource targets |
| `allowSystemRules` | String | `true` | Allow system-managed rules to pass |

## Deployment

### Azure CLI

```bash
# Create the policy definition
az policy definition create \
  --name "ml-outbound-rules-governance" \
  --display-name "Azure ML / AI Studio - Enforce approved outbound rules only" \
  --rules policy.json \
  --mode All

# Assign with parameters
az policy assignment create \
  --name "ml-outbound-rules" \
  --policy "ml-outbound-rules-governance" \
  --params '{
    "effect": {"value": "Deny"},
    "allowedFQDNs": {"value": ["pypi.org", "*.anaconda.com", "mcr.microsoft.com"]},
    "allowedServiceTags": {"value": ["AzureActiveDirectory", "AzureMonitor", "BatchNodeManagement"]},
    "allowedPrivateEndpointSubresources": {"value": ["blob", "vault", "amlworkspace"]},
    "allowSystemRules": {"value": "true"}
  }'
```

### Terraform

```hcl
resource "azurerm_policy_definition" "ml_outbound_rules" {
  name         = "ml-outbound-rules-governance"
  policy_type  = "Custom"
  mode         = "All"
  display_name = "Azure ML / AI Studio - Enforce approved outbound rules only"

  policy_rule = file("${path.module}/policy.json")
}
```

## Key aliases used

- `Microsoft.MachineLearningServices/workspaces/outboundRules/type`
- `Microsoft.MachineLearningServices/workspaces/outboundRules/FQDN.destination`
- `Microsoft.MachineLearningServices/workspaces/outboundRules/ServiceTag.destination.serviceTag`
- `Microsoft.MachineLearningServices/workspaces/outboundRules/PrivateEndpoint.destination.subresourceTarget`
- `Microsoft.MachineLearningServices/workspaces/outboundRules/category`

## Reference

- [Built-in policy: Managed VNet isolation mode](https://www.azadvertizer.net/azpolicyadvertizer/6ddb1705-c8cf-450e-aa4b-19ad6703c440.html) — the reference policy this is based on
- [Azure ML managed network docs](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-managed-network)
- [Azure Policy aliases for ML workspaces](https://github.com/maciejporebski/azure-policy-aliases/blob/master/aliases/Microsoft.MachineLearningServices/workspaces.md)
