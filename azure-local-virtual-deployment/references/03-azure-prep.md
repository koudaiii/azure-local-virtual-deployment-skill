# Phase 3: Azure-Side Preparation

> **Source documentation**: Microsoft Learn — [Register to Arc and assign permissions for deployment](https://learn.microsoft.com/en-us/azure/azure-local/deploy/deployment-arc-register-server-permissions).
>
> The procedure below captures the 13 required resource providers and the 6 role assignments (2 subscription-level + 4 resource-group-level) in a single PowerShell flow.
>
> **Teaching reminder**: Apply the four teaching principles from `SKILL.md` — (1) show before/after state for every change, (2) point the user at the source doc above, (3) accept screenshots and links the user shares, (4) state which layer/credential each command runs as and preview any dialog before it appears. See `references/00-accounts-and-context.md`. The role-assignment steps are GUI-driven — actively invite the user to share Azure portal screenshots so you can confirm each assignment landed.

> **Context shifts in this phase**:
>
> | Steps | Where you act |
> |---|---|
> | 1–3 (RP registration, RG) | `[Host PowerShell — azureuser]` + browser device-code login |
> | 4 (role assignments) | `[Azure portal — signed in as your Azure user]` |
> | 5 (verify) | `[Host PowerShell — azureuser]` |
>
> Note: the Azure user you sign into the portal as is the **same identity** as the device-code login in Step 1, NOT `azureuser` on the host VM. `azureuser` is just a local OS account on the Azure VM — it has no relationship to Azure RBAC.

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

## Choose your tenant strategy (decide before Step 1)

This phase is the only point in the entire deployment where the tenant choice matters. **You have two valid plans**, and the rest of this skill works identically for both — you just need to pick one consciously before running `Connect-AzAccount`.

| | Plan A — Same tenant (default) | Plan B — Cross-tenant |
|---|---|---|
| **Where the host Azure VM lives** | Tenant T | Tenant A (your primary work tenant) |
| **Where the Azure Local cluster is registered** | Tenant T | Tenant B (separate registration tenant) |
| **What you sign into in Step 1 below** | `Connect-AzAccount` (no `-TenantId`) | `Connect-AzAccount -TenantId <Tenant-B-guid>` |
| **When to choose this** | You own / have Owner on a tenant that permits `Connect-AzAccount` from an Azure VM and where you want the cluster billed. The simplest configuration. | Your work tenant's Conditional Access blocks Azure VM sign-in, you lack Owner there, or you want billing isolation. |
| **Read first** | Skip to **Step 1** below. The "Plan B details" subsection is informational only for you. | Read **Plan B details** below, then proceed to **Step 1**. |

> The Microsoft Learn tutorial implicitly assumes **Plan A**. This skill supports both — Plan B is documented because it is the realistic configuration when the engineer's own work tenant cannot host the sign-in (a frequent situation for engineers in tightly-managed enterprise tenants, including many Microsoft engineers).

If you are unsure: try `Connect-AzAccount` (Plan A) first. If your sign-in is rejected by CA, fall back to Plan B and re-read this section. The two plans share every command below; only the `Connect-AzAccount` invocation differs.

## Plan B details — cross-tenant registration

*Skip this subsection if you chose Plan A above.*

When does Plan B apply?

- **Conditional Access / sign-in policies block `Connect-AzAccount` from the host VM.** Tightly-managed work tenants (including many Microsoft engineer tenants) frequently require device compliance, managed devices, MFA on a registered authenticator, or block device-code flow entirely. An Azure VM you spun up for the lab will fail those checks.
- **You lack Owner / `Contributor + User Access Administrator` in the host VM's tenant** but have it in another tenant (MSDN / Visual Studio Enterprise, partner, dev/test, personal).
- **You want lab resources separated from production billing / governance** in your primary work tenant.

### How tenant separation works

The bootstrap in Phase 4 only needs egress to the public Azure control-plane endpoints (`login.microsoftonline.com`, `management.azure.com`, `*.arc.azure.com`, …). It does **not** require any trust, federation, or B2B relationship between the host VM's tenant and the registration tenant. Concretely:

```
┌──────────────────────────────────────────────────────────────┐
│ Tenant A (your primary work tenant — e.g., a tightly-        │
│            managed enterprise tenant)                        │
│  └─ Subscription A                                           │
│      └─ Host Azure VM "AZLOCAL"   ← runs Hyper-V + Node1     │
│         (only used for compute; CA policy may block          │
│          Connect-AzAccount from here)                        │
└──────────────────────────────────────────────────────────────┘
                          │
                          │   public Azure endpoints over NAT
                          ▼
┌──────────────────────────────────────────────────────────────┐
│ Tenant B (your registration / lab tenant — e.g., MSDN /      │
│            Visual Studio Enterprise / partner / personal)    │
│  └─ Subscription B                                           │
│      └─ Resource group "azurelocalconnected"                 │
│          ├─ Microsoft.HybridCompute/machines  Node1          │
│          ├─ Microsoft.AzureStackHCI/clusters  <cluster>      │
│          └─ … (all cluster resources land here)              │
└──────────────────────────────────────────────────────────────┘
```

`$tenantId` / `$subscriptionId` captured in Step 1 below are bound to **Tenant B**. Phase 4 passes them straight into `Invoke-AzStackHciArcInitialization`, so the Arc machine, cluster, Key Vault, storage accounts — everything — are created in Tenant B.

### What you need in the registration tenant (Tenant B)

- An Azure user account that can sign in **from the host VM** (i.e., its CA policy permits sign-in from an arbitrary Azure VM IP, or you have a break-glass / unconditional account)
- **Owner**, or **Contributor + User Access Administrator**, on the target subscription in Tenant B
- The subscription must be in (or your home tenant has access to) a region that supports Azure Local (e.g., `japaneast`, `eastus`, `westeurope`)

### Practical recipe

1. Open the host VM via RDP as usual.
2. Run `Connect-AzAccount -TenantId <tenant-B-guid>` (or just `Connect-AzAccount` and pick Tenant B from the consent screen).
3. Complete device-code sign-in with the **Tenant B** account in the browser — *not* your corporate account.
4. Continue from Step 1 below. Everything else (RP registration, RG, role assignments, Arc bootstrap) targets Tenant B automatically.

If `Connect-AzAccount` itself fails because of CA policy on Tenant A (e.g., the browser auth flow gets intercepted), see `references/troubleshooting.md` § "`Connect-AzAccount` blocked by Conditional Access" for the workarounds (explicit `-TenantId`, an alternate browser profile, or running the device-code flow from a different device).

### Cost split between the two tenants (Subscription A vs Subscription B)

Even when you split tenancy, the **60-day free trial of the Azure Local service fee** still applies — it starts when the cluster resource is created in **Subscription B**, regardless of where the host VM lives. Knowing exactly which bill each cost lands on is critical for chargeback paperwork and avoiding surprise charges.

| Cost item | Bills against | Notes |
|---|---|---|
| Host Azure VM compute (`Standard_E32-8s_v5` running Hyper-V + Node1) | **Subscription A** | Largest cost in this lab. Hourly rate varies by region — roughly **\$700–\$1,100 / month** running 24×7 in `japaneast` / `eastus` as of 2026. Deallocate (`Stop-AzVM -Force`) when not actively building to halt compute charges. |
| Host Azure VM managed disks (OS + 3 data disks for the lab) | **Subscription A** | Continues while the VM is deallocated — only deleting the disks stops it. Premium SSD v2 at ~1 TB ≈ \$80–\$120 / month. |
| Host VM outbound bandwidth | **Subscription A** | First 100 GB / month free; the bootstrap + ISO traffic is well under that. |
| **Azure Local service fee** (per physical core / month) | **Subscription B** | **Free for the first 60 days** after the cluster resource is registered. After the trial: **\$10 / physical core / month** (8 vCPUs in this lab → ~\$80 / month if you keep running it). Verify current rates at the [Azure Local pricing page](https://azure.microsoft.com/pricing/details/azure-local/). |
| Cluster-side Azure resources (Key Vault, 2 × Storage Accounts, Arc machine, Resource Bridge, custom location, etc.) | **Subscription B** | Pennies / month at lab scale; usually under \$5 / month. |

> ⚠️ **The 60-day trial counter starts the moment the cluster resource is created**, not when you start using it. If you provision the cluster and walk away, you are burning trial days against zero workload. To stop Azure Local service charges in Subscription B, **delete the `Microsoft.AzureStackHCI/clusters` resource** — stopping the host VM in Subscription A halts only compute, not the service fee. Full teardown: `references/troubleshooting.md` § "Cleanup procedure".

> 💡 If a single-bill view is required (e.g., one cost center owning the whole lab for chargeback), the alternative is to provision the **host VM directly in Subscription B**, assuming CA on Tenant B permits it. Most MSDN / Visual Studio Enterprise subscriptions are fine for both roles, and many corporate procurement teams accept this single-tenant lab arrangement.

### Guidance for engineers signing in from a tightly-managed work tenant

This cross-tenant pattern is the supported way to build an Azure Local lab when your primary work tenant's CA blocks `Connect-AzAccount` from arbitrary Azure VMs — a situation many Microsoft engineers (and engineers at other organizations with similar tenant hardening) run into on day one. Keep the host VM in your work subscription if billing visibility there is required, then register the cluster to an MSDN / Visual Studio Enterprise / partner subscription whose CA policy permits the sign-in. If your team's lab paperwork or compliance review asks why the cluster resources live in a separate subscription, the answer is simply that the work-tenant CA policy does not permit interactive Azure sign-in from a self-managed lab VM, and Azure Local's bootstrap is bound to whichever tenant the engineer signs into during Phase 3.

## Step 1: Sign in to Azure

`[Host PowerShell — azureuser]`

Pick the line that matches the plan you chose above:

```powershell
# Plan A — same tenant (default). No -TenantId needed.
Connect-AzAccount

# Plan B — cross-tenant. Pin the registration tenant explicitly so the dialog
# does NOT default back to the host VM's home tenant.
# Connect-AzAccount -TenantId "<registration-tenant-guid>"
```

If you are a guest in multiple tenants, always pass `-TenantId` (Plan B) — otherwise the consent screen will default to your account's home tenant and silently land the cluster there.

Expected output — a device-code instruction in the terminal:

```
WARNING: To sign in, use a web browser to open the page
         https://microsoft.com/devicelogin and enter the code ABCD-EFGH
         to authenticate.
```

Open `https://microsoft.com/devicelogin` (in any browser — easiest from the host's own Edge), paste the code, and sign in with the Azure user that has Owner / Contributor + User Access Administrator on the target subscription. That dialog looks like:

```
┌─ Sign in to your account ───────────────────────────────────────┐
│ Microsoft                                                       │
│                                                                 │
│ Enter code                                                      │
│                                                                 │
│ Code:  [ ABCD-EFGH                            ]                 │
│                                                                 │
│                                       [ Next ]                  │
└─────────────────────────────────────────────────────────────────┘
```

Back in PowerShell after the browser flow completes, you should see your account printed:

```
Account               SubscriptionName   TenantId      Environment
-------               ----------------   --------      -----------
you@contoso.com       MyAzureSub         <guid>        AzureCloud
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

`[Azure portal — signed in as your Azure user]`

Role assignment via portal is more reliable than PowerShell, especially for multi-step assignments. From the [Azure portal](https://portal.azure.com):

The IAM blade you'll be working in looks like this — `+ Add` opens a 4-step "Add role assignment" wizard:

```
┌─ Access control (IAM)  —  azurelocalconnected ────────────────────┐
│ + Add ▼  ↓ Download  ⟳ Refresh  ? Got feedback                    │
│ ┌─────────────────────────────────────────────────────────────┐   │
│ │ Add role assignment     ← click this entry from the + Add   │   │
│ │ Add co-administrator      menu                              │   │
│ │ Add deny assignment                                         │   │
│ └─────────────────────────────────────────────────────────────┘   │
│                                                                   │
│ [ Check access ] [ Role assignments ] [ Roles ] [ Deny assign… ]  │
│                                                                   │
│  Showing 12 of 28 role assignments…                               │
└───────────────────────────────────────────────────────────────────┘
```

The wizard itself is a 4-tab flow — for each role assignment you'll touch tabs **Role → Members → Review + assign**:

```
┌─ Add role assignment ─────────────────────────────────────────────┐
│ Role  >  Members  >  Conditions  >  Review + assign               │
│                                                                   │
│ Search:  [ Key Vault Contributor                          ] 🔍    │
│                                                                   │
│ ◉ Key Vault Contributor                                           │
│ ○ Key Vault Secrets Officer                                       │
│ ○ Key Vault Data Access Administrator                             │
│ ○ Storage Account Contributor                                     │
│ …                                                                 │
│                                                                   │
│                                       [ Back ] [ Next > ]         │
└───────────────────────────────────────────────────────────────────┘
```

On the **Members** tab, leave "User, group, or service principal" selected and add yourself (the Azure user signed in to the portal — same as Step 1).

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

`[Host PowerShell — azureuser]`

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
