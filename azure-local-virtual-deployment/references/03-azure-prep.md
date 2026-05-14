# Phase 3: Azure-Side Preparation

> **Source documentation**: Microsoft Learn — [Register to Arc and assign permissions for deployment](https://learn.microsoft.com/en-us/azure/azure-local/deploy/deployment-arc-register-server-permissions).
>
> The procedure below captures the 13 required resource providers and the 6 role assignments (2 subscription-level + 4 resource-group-level) in a single PowerShell flow.

This phase is **entirely on the host's PowerShell** (no Node interaction). You register the 13 required resource providers, create the resource group, and assign roles needed for cluster deployment.

## What you set up

```
Azure Subscription
├─ 13 Resource Providers registered
├─ Resource Group: "azurelocalconnected" (or your choice)
│   └─ Role assignments to your user:
│       ├─ Key Vault Data Access Administrator
│       ├─ Key Vault Secrets Officer
│       ├─ Key Vault Contributor
│       └─ Storage Account Contributor
└─ Subscription-level role assignments:
    ├─ Azure Stack HCI Administrator
    └─ Reader
```

## Before you start

- Phase 1 and 2 complete
- Az PowerShell module installed on the host (`Get-Module -ListAvailable Az.Accounts, Az.Resources`)
- An Azure subscription where you have **Owner**, or **Contributor + User Access Administrator**

## Step 1: Sign in to Azure

```powershell
Connect-AzAccount
```

If multiple subscriptions exist, set the right one:

```powershell
Get-AzSubscription | Format-Table Name, Id, State
Set-AzContext -SubscriptionId "<your-subscription-id>"
```

Capture key IDs into variables for reuse:

```powershell
$context = Get-AzContext
$subscriptionId = $context.Subscription.Id
$tenantId = $context.Tenant.Id
$resourceGroupName = "azurelocalconnected"   # adjust as needed
$location = "japaneast"                       # use any Azure Local-supported region

Write-Host "Subscription: $subscriptionId"
Write-Host "Tenant:       $tenantId"
```

## Step 2: Register all 13 resource providers

```powershell
$providers = @(
    "Microsoft.HybridCompute",
    "Microsoft.GuestConfiguration",
    "Microsoft.HybridConnectivity",
    "Microsoft.AzureStackHCI",
    "Microsoft.Kubernetes",
    "Microsoft.KubernetesConfiguration",
    "Microsoft.ExtendedLocation",
    "Microsoft.ResourceConnector",
    "Microsoft.HybridContainerService",
    "Microsoft.Attestation",
    "Microsoft.Storage",
    "Microsoft.Insights",
    "Microsoft.KeyVault"
)

foreach ($p in $providers) {
    Write-Host "Registering: $p" -ForegroundColor Yellow
    Register-AzResourceProvider -ProviderNamespace $p | Out-Null
}
```

Registration is asynchronous. Poll until all show `Registered`:

```powershell
foreach ($p in $providers) {
    $status = (Get-AzResourceProvider -ProviderNamespace $p | Select-Object -First 1).RegistrationState
    $color = if ($status -eq "Registered") { "Green" } else { "Yellow" }
    Write-Host ("{0,-40} {1}" -f $p, $status) -ForegroundColor $color
}
```

⚠️ Don't proceed until all 13 are `Registered`. Some take 1-3 minutes.

## Step 3: Create the resource group

```powershell
$existingRG = Get-AzResourceGroup -Name $resourceGroupName -ErrorAction SilentlyContinue
if (-not $existingRG) {
    New-AzResourceGroup -Name $resourceGroupName -Location $location
}
```

## Step 4: Assign roles (Azure portal recommended)

Role assignment via portal is more reliable than PowerShell, especially for multi-step assignments. From the [Azure portal](https://portal.azure.com):

### Subscription-level roles

1. Search → **Subscriptions** → select your subscription
2. **Access control (IAM)** → **+ Add** → **Add role assignment**
3. Add each role to your user:
   - **Azure Stack HCI Administrator**
   - **Reader**

### Resource-group-level roles

1. Search → **Resource groups** → select `azurelocalconnected`
2. **Access control (IAM)** → **+ Add** → **Add role assignment**
3. Add each role to your user:
   - **Key Vault Data Access Administrator**
   - **Key Vault Secrets Officer**
   - **Key Vault Contributor**
   - **Storage Account Contributor**

## Step 5: Verify assignments

```powershell
# Subscription level
Get-AzRoleAssignment -SignInName (Get-AzContext).Account.Id `
    -Scope "/subscriptions/$subscriptionId" |
    Where-Object {$_.RoleDefinitionName -in @('Azure Stack HCI Administrator', 'Reader', 'Owner')} |
    Format-Table RoleDefinitionName, Scope

# Resource group level
Get-AzRoleAssignment -SignInName (Get-AzContext).Account.Id `
    -ResourceGroupName $resourceGroupName |
    Format-Table RoleDefinitionName, Scope
```

## Why all these roles?

- **Azure Stack HCI Administrator** — required to create the cluster resource
- **Reader** — provides baseline list-permissions on the subscription
- **Key Vault Contributor/Secrets Officer/Data Access Administrator** — the cluster deployment creates a Key Vault and writes cryptographic material into it
- **Storage Account Contributor** — the cluster creates 2 storage accounts (cloud witness + Key Vault audit log)

If you're already **Owner** on the subscription, technically all of these are inherited. But assigning them explicitly avoids "permission inherited from parent" edge cases that sometimes confuse the deployment validator.

## Save these values for later

You will need them in subsequent phases:

| Variable | Used in |
|---|---|
| `$subscriptionId` | Phase 4 (Arc registration), Phase 5 portal wizard |
| `$tenantId` | Phase 4 (Arc registration) |
| `$resourceGroupName` | Phase 4, Phase 5 portal wizard |
| `$location` | Phase 4, Phase 5 portal wizard |

## How to verify this phase is done

✅ All 13 resource providers show `Registered`
✅ Resource group exists in the chosen region
✅ Subscription IAM lists your user with `Azure Stack HCI Administrator` and `Reader`
✅ Resource group IAM lists your user with 4 Key Vault and Storage roles

→ Proceed to **Phase 4** (`references/04-arc-registration.md`)
