# Azure ML / AI Studio — Outbound Rules Custom Policy

Custom Azure Policies to govern outbound rules on Azure Machine Learning and AI Studio managed networks.

## Overview

When Azure ML workspaces use **Managed VNet** with `AllowOnlyApprovedOutbound` isolation mode, outbound rules control what the workspace can reach. These policies enforce which outbound rules are permitted — blocking any rule that doesn't match your approved list.

## Policies

### 1. Deny Policy (`policy.json`) — Real-time Prevention

Evaluates the **child resource type** `Microsoft.MachineLearningServices/workspaces/outboundRules`. Every time an outbound rule is created or modified via CLI, REST API, ARM template, or Portal, the policy checks it against your approved lists.

**Effect:** `Deny` (blocks in real-time)

### 2. DINE Policy (`dine-policy.json`) — Remediation Safety Net

Uses `DeployIfNotExists` on the workspace to detect non-compliant outbound rules and remediate by redeploying the workspace managed network config. Acts as a safety net for any rules that might slip through during policy propagation delays.

**Effect:** `DeployIfNotExists` (auto-remediate)

## Rule Types Governed

| Rule Type | Parameter | What it checks |
|-----------|-----------|----------------|
| **FQDN** | `allowedFQDNs` | Destination FQDN (e.g. `pypi.org`, `*.anaconda.com`) |
| **ServiceTag** | `allowedServiceTags` | Azure Service Tag (e.g. `AzureActiveDirectory`, `AzureMonitor`) |
| **PrivateEndpoint** | `allowedPrivateEndpointSubresources` | Subresource target (e.g. `blob`, `vault`, `amlworkspace`) |

### System Rules

By default (`allowSystemRules: true` in Deny policy), rules with category `SystemDefault` or `Required` are always permitted. Azure creates these automatically for workspace functionality.

## Parameters

### Deny Policy (`policy.json`)

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `effect` | String | `Audit` | `Audit`, `Deny`, or `Disabled` |
| `allowedFQDNs` | Array | `[]` | Approved FQDN destinations |
| `allowedServiceTags` | Array | `[]` | Approved Azure Service Tags |
| `allowedPrivateEndpointSubresources` | Array | `[]` | Approved PE subresource targets |
| `allowSystemRules` | String | `true` | Allow system-managed rules to pass |

### DINE Policy (`dine-policy.json`)

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `effect` | String | `DeployIfNotExists` | `DeployIfNotExists` or `Disabled` |
| `allowedFQDNs` | Array | `[]` | Approved FQDN destinations |
| `allowedServiceTags` | Array | `[]` | Approved Azure Service Tags |
| `allowedPrivateEndpointSubresources` | Array | `[]` | Approved PE subresource targets |

## Deployment

### Deploy Both Policies (Recommended)

```bash
# 1. Create Deny policy definition
az policy definition create \
  --name "ml-outbound-rules-governance" \
  --display-name "Azure ML / AI Studio - Enforce approved outbound rules only" \
  --rules policy.json \
  --params <(jq '.properties.parameters' policy.json) \
  --mode All

# 2. Create DINE policy definition
az policy definition create \
  --name "ml-outbound-rules-dine-remediate" \
  --display-name "Azure ML / AI Studio - Remediate unapproved outbound rules (DINE)" \
  --rules <(jq '.properties.policyRule' dine-policy.json) \
  --params <(jq '.properties.parameters' dine-policy.json) \
  --mode All

# 3. Assign Deny policy
az policy assignment create \
  --name "ml-outbound-deny" \
  --policy "ml-outbound-rules-governance" \
  --scope "/subscriptions/<sub-id>/resourceGroups/<rg>" \
  --params '{
    "effect": {"value": "Deny"},
    "allowedFQDNs": {"value": ["pypi.org", "*.anaconda.com", "mcr.microsoft.com"]},
    "allowedServiceTags": {"value": ["AzureActiveDirectory", "AzureMonitor"]},
    "allowedPrivateEndpointSubresources": {"value": ["blob", "vault", "amlworkspace"]},
    "allowSystemRules": {"value": "true"}
  }'

# 4. Assign DINE policy (requires managed identity)
az policy assignment create \
  --name "ml-outbound-dine" \
  --policy "ml-outbound-rules-dine-remediate" \
  --scope "/subscriptions/<sub-id>/resourceGroups/<rg>" \
  --mi-system-assigned \
  --location <location> \
  --params '{
    "effect": {"value": "DeployIfNotExists"},
    "allowedFQDNs": {"value": ["pypi.org", "*.anaconda.com", "mcr.microsoft.com"]},
    "allowedServiceTags": {"value": ["AzureActiveDirectory", "AzureMonitor"]},
    "allowedPrivateEndpointSubresources": {"value": ["blob", "vault", "amlworkspace"]}
  }'

# 5. Grant DINE managed identity Contributor role on the scope
PRINCIPAL_ID=$(az policy assignment show --name ml-outbound-dine --resource-group <rg> --query identity.principalId -o tsv)
az role assignment create \
  --assignee-object-id "$PRINCIPAL_ID" \
  --assignee-principal-type ServicePrincipal \
  --role Contributor \
  --scope "/subscriptions/<sub-id>/resourceGroups/<rg>"
```

## Test Results

Tested on Azure subscription with:
- **RG:** `rg-policy-test` (Australia East)
- **ML Workspace:** `ws-policy-test` (AllowOnlyApprovedOutbound)
- **Allowed FQDNs:** `pypi.org`, `*.anaconda.com`, `mcr.microsoft.com`
- **Allowed ServiceTags:** `AzureActiveDirectory`, `AzureMonitor`

### Deny Policy Tests (6/6 passed)

| # | Method | Rule Type | Destination | Expected | Result |
|---|--------|-----------|-------------|----------|--------|
| 1 | CLI | FQDN | `evil.com` | ❌ Deny | ❌ **Denied** ✅ |
| 2 | CLI | FQDN | `pypi.org` | ✅ Allow | ✅ **Created** ✅ |
| 3 | CLI | ServiceTag | `Storage` | ❌ Deny | ❌ **Denied** ✅ |
| 4 | CLI | ServiceTag | `AzureMonitor` | ✅ Allow | ✅ **Created** ✅ |
| 5 | REST API | FQDN | `hacker.io` | ❌ Deny | ❌ **Denied** ✅ |
| 6 | REST API | FQDN | `mcr.microsoft.com` | ✅ Allow | ✅ **Created** ✅ |

### Portal Tests

| Test | Result |
|------|--------|
| Portal save with unapproved FQDN (after policy propagation) | ❌ **Denied** ✅ |
| Portal save with approved FQDN | ✅ **Created** ✅ |

> **Note:** Azure Policy has a 5-15 minute propagation delay after initial assignment. Rules created during this window may bypass the Deny policy — the DINE policy acts as a safety net for these cases.

## Key Aliases Used

- `Microsoft.MachineLearningServices/workspaces/outboundRules/type`
- `Microsoft.MachineLearningServices/workspaces/outboundRules/FQDN.destination`
- `Microsoft.MachineLearningServices/workspaces/outboundRules/ServiceTag.destination.serviceTag`
- `Microsoft.MachineLearningServices/workspaces/outboundRules/PrivateEndpoint.destination.subresourceTarget`
- `Microsoft.MachineLearningServices/workspaces/outboundRules/category`

## Architecture

```
                    ┌─────────────────────────────┐
                    │  Outbound Rule Change        │
                    │  (CLI / API / Portal / ARM)  │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │  Azure Resource Manager       │
                    └──────────────┬──────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                     │
    ┌─────────▼─────────┐  ┌──────▼──────┐   ┌─────────▼─────────┐
    │  Policy 1: Deny    │  │  Allowed?   │   │  Policy 2: DINE   │
    │  (Real-time block) │  │             │   │  (Safety net)     │
    └─────────┬─────────┘  └──────┬──────┘   └─────────┬─────────┘
              │                    │                     │
       ❌ Blocked            ✅ Created          🔄 Auto-remediate
       (immediate)           (passes policy)    (if non-compliant)
```

## References

- [Built-in policy: Managed VNet isolation mode](https://www.azadvertizer.net/azpolicyadvertizer/6ddb1705-c8cf-450e-aa4b-19ad6703c440.html)
- [Azure ML managed network docs](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-managed-network)
- [Azure Policy aliases for ML workspaces](https://github.com/maciejporebski/azure-policy-aliases/blob/master/aliases/Microsoft.MachineLearningServices/workspaces.md)
