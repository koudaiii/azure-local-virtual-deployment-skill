# Phase 4: Register Node with Azure Arc

> **Source documentation**: Microsoft Learn — [Register your Azure Local machines with Azure Arc (without Azure Arc gateway)](https://learn.microsoft.com/en-us/azure/azure-local/deploy/deployment-without-azure-arc-gateway).
>
> Multiple variants exist in the doc based on proxy/gateway combinations. This skill uses the **without-proxy, without-Arc-gateway, ARM-token** variant — the cleanest path for a nested-VM lab where the host is already signed into Azure.
>
> **Teaching reminder**: Apply the four teaching principles from `SKILL.md` — (1) show before/after state for every change, (2) point the user at the source doc above, (3) accept screenshots and links the user shares, (4) state which layer/credential each command runs as and preview any dialog before it appears. See `references/00-accounts-and-context.md`. The "before" here is "no Node in the resource group"; the "after" is the Arc machine resource appearing — confirm both explicitly.

> **Context shifts in this phase**:
>
> | Steps | Where you act |
> |---|---|
> | 1 (cmdlet probe) | `[Host PowerShell → Node1 via $cred]` |
> | 2 (ARM token) | `[Host PowerShell — azureuser]` (uses the host's Azure sign-in from Phase 3) |
> | 3 (bootstrap) | `[Host PowerShell → Node1 via $cred]` — token is passed in |
> | 4–5 (monitor / recovery) | `[Host PowerShell → Node1 via $cred]` |
> | 6 (verify) | `[Host PowerShell — azureuser]` + `[Azure portal]` |

This phase runs the **Azure Local bootstrap** on Node1 to register it as an Arc-enabled machine. After this, Azure can see the Node and you can run the cluster deployment wizard.

## What you set up

```
Azure
└─ Resource Group "azurelocalconnected"
    └─ Microsoft.HybridCompute/machines  "Node1"
       └─ Arc extensions installed:
           ├─ AzureEdgeTelemetryAndDiagnostics
           ├─ AzureEdgeDeviceManagement
           └─ AzureEdgeLifecycleManager
```

## Before you start

- Phase 1-3 complete
- Node1 has internet access via NAT (verified in Phase 2)
- You're signed into Azure on the host (`Get-AzContext` returns a valid context)
- The `$cred` variable still holds Node1's Administrator credentials

## Step 1: Verify the bootstrap cmdlet exists on Node1

The `Invoke-AzStackHciArcInitialization` cmdlet is pre-installed on Azure Stack HCI OS via the `AzureEdgeBootstrap` module.

```powershell
Invoke-Command -VMName "Node1" -Credential $cred -ScriptBlock {
    $cmd = Get-Command Invoke-AzStackHciArcInitialization -ErrorAction SilentlyContinue
    if ($cmd) {
        Write-Host "OK: $($cmd.Module) v$($cmd.Version)"
    } else {
        Write-Host "MISSING - bootstrap module not installed"
    }
}
```

If the cmdlet is missing (very rare on a fresh Azure Stack HCI install), troubleshoot before proceeding.

## Step 2: Get an ARM access token on the host

Two authentication options exist for `Invoke-AzStackHciArcInitialization`:

1. **Device code flow** — interactive, requires browser auth from inside Node1's session
2. **ARM access token** — non-interactive, uses the host's existing Azure context ⭐

The ARM token approach is much cleaner. Generate the token on the host where you're already signed in:

```powershell
$armTokenResponse = Get-AzAccessToken -WarningAction SilentlyContinue
$ArmAccessToken = [System.Net.NetworkCredential]::new("", $armTokenResponse.Token).Password

# Sanity check
Write-Host "Token length: $($ArmAccessToken.Length) chars"
# Expected: ~3000+ chars
```

⚠️ **The token expires in ~1 hour.** Run Step 3 within an hour of generating the token. If the bootstrap fails due to expired token, regenerate.

## Step 3: Run the bootstrap on Node1

```powershell
# Display parameters first to verify
Write-Host "Tenant:        $tenantId"
Write-Host "Subscription:  $subscriptionId"
Write-Host "RG:            $resourceGroupName"
Write-Host "Region:        $location"

Invoke-Command -VMName "Node1" -Credential $cred -ScriptBlock {
    param($Tenant, $Subscription, $RG, $Region, $Token)

    Invoke-AzStackHciArcInitialization `
        -TenantId $Tenant `
        -SubscriptionID $Subscription `
        -ResourceGroup $RG `
        -Region $Region `
        -Cloud "AzureCloud" `
        -ArmAccessToken $Token

} -ArgumentList $tenantId, $subscriptionId, $resourceGroupName, $location, $ArmAccessToken
```

## Step 4: Monitor progress

The bootstrap runs through 10 sub-stages. Expected timeline:

- **Without OS update needed**: 5-10 minutes (typical for fresh Azure Stack HCI 24H2)
- **With OS update needed**: 50-60 minutes total (OS update + bootstrap)

The output streams sub-stage status:

```
Microsoft image detected.
No target solution version provided. Skipping update.
Starting Arc registration process...
Triggering bootstrap on the device...
Waiting for bootstrap to complete... Current Status: InProgress
  NetworkConfig : NotStarted/InProgress/Succeeded
  RemoteConfig : ...
  WebProxy : ...
  TimeServer : ...
  HostName : ...
  ArcConfiguration : ...
      ArtifactsUpload : ...
      ConnectivityValidation : ...
      ArcRegistration : ...
      ArcExtensionInstall : ...
...
Bootstrap succeeded.
```

The final line should be **`Bootstrap succeeded.`**.

> **Note about `NotApplicable` results**: If you pre-configured network, hostname, etc. (as we did in Phase 2), those sub-stages may report `NotApplicable` rather than `Succeeded`. This is fine — it means the bootstrap detected the configuration was already correct and skipped it.

## Step 5: Recovery from interrupted bootstrap

If your PowerShell session disconnects mid-bootstrap, the process **continues running on Node1**. Reconnect and check status:

```powershell
Invoke-Command -VMName "Node1" -Credential $cred -ScriptBlock {
    $status = Get-ArcBootstrapStatus
    $status.Response.Status
}
```

Possible values: `InProgress`, `Succeeded`, `Failed`.

For detailed logs:

```powershell
Invoke-Command -VMName "Node1" -Credential $cred -ScriptBlock {
    Collect-ArcBootstrapSupportLogs
}
```

## Step 6: Verify in Azure

```powershell
Get-AzResource -ResourceGroupName $resourceGroupName `
    -ResourceType "Microsoft.HybridCompute/machines" |
    Format-Table Name, Location, ResourceType
```

Expected: `Node1` of type `Microsoft.HybridCompute/machines` in your chosen region.

In the Azure portal:
1. Resource group `azurelocalconnected` → `Node1`
2. **Status**: `Connected` (green)
3. **Last seen**: within last few minutes

The portal blade should look approximately like this:

```
┌─ Node1  |  Machine - Azure Arc ───────────────────────────────────┐
│ ⟳ Refresh  ✎ Edit  ⌫ Delete  + Tags                              │
├───────────────────────────────────────────────────────────────────┤
│  Essentials                                                       │
│  Resource group     :  azurelocalconnected                        │
│  Location           :  Japan East                                 │
│  Subscription       :  MyAzureSub                                 │
│  OS                 :  Azure Stack HCI                            │
│  Status             :  ● Connected            ← green dot         │
│  Last seen          :  Just now                                   │
│  Computer name      :  Node1                                      │
│  Azure Arc agent    :  1.x.x                                      │
└───────────────────────────────────────────────────────────────────┘
```

If `Status` shows `Disconnected` or `Expired`, do not move on — re-run Step 5 to inspect the bootstrap state.

## Common failures

### `ConnectivityValidation: Failed`
Node1 can't reach an Azure endpoint. Confirm from Node1:

```powershell
Invoke-Command -VMName "Node1" -Credential $cred -ScriptBlock {
    Test-NetConnection -ComputerName "login.microsoftonline.com" -Port 443
    Test-NetConnection -ComputerName "management.azure.com" -Port 443
    Test-NetConnection -ComputerName "his.arc.azure.com" -Port 443
}
```

If any fail, verify DNS resolution and the host's NAT is working. Phase 1's `New-NetNat` must exist on the host.

### `Token expired` or `Authentication failed`
Regenerate the ARM token (Step 2) and re-run Step 3.

### Bootstrap hangs at one stage for >30 minutes
Collect logs (Step 5) and inspect. Common cause: a transient Azure regional issue. Wait 10 minutes and retry.

## How to verify this phase is done

✅ `Bootstrap succeeded.` printed
✅ `Get-AzResource` shows Node1 of type `Microsoft.HybridCompute/machines`
✅ Azure portal shows Node1 with Status = `Connected`

→ Proceed to **Phase 5** (`references/05-ad-setup.md`)
