# Troubleshooting: Common Errors and Fixes

> **Related Microsoft documentation**:
> - [Get support for deployment issues](https://learn.microsoft.com/en-us/azure/azure-local/manage/get-support-for-deployment-issues) — official support escalation path and log collection
> - [Troubleshoot Azure Local deployment](https://learn.microsoft.com/en-us/azure/azure-local/manage/troubleshoot-deployment-issues) — known issues from Microsoft
> - [Azure Local known issues](https://learn.microsoft.com/en-us/azure/azure-local/known-issues) — current release-specific issues
> - [Hyper-V event logs reference](https://raw.githubusercontent.com/MicrosoftDocs/windowsserverdocs/main/WindowsServerDocs/virtualization/hyper-v/manage/Manage-Hyper-V-integration-services.md) — for hypervisor-layer symptoms below

This is the place to look when something goes wrong. Errors are grouped by phase.

## First-line Hyper-V diagnostics (use *before* the phase-specific sections below)

Most Phase 2 / 5 problems present as "the VM won't start", "PowerShell Direct keeps rejecting credentials", or "the console screen is black". These are usually Hyper-V-layer symptoms, not Azure Local symptoms — so the fastest first move is to read what Hyper-V itself recorded about the VM. Run these from the **host** before diving into the phase-specific entries.

### Step 1: What does Hyper-V think the VM is doing?

`[Host PowerShell — azureuser]`

```powershell
# Full VM state including notes and last status
Get-VM -Name "Node1" | Format-List Name, State, Status, Uptime, Heartbeat, Notes,
    ProcessorCount, MemoryAssigned, AutomaticStartAction, CheckpointType

# Was the VM running when it failed? Any obvious blocker?
Get-VM -Name "Node1" | Select-Object Name, State, OperationalStatus, PrimaryStatusDescription
```

If `State` is `Off`, the VM never started. If `Status` shows `Operating normally` but you cannot reach it via PSDirect, the *guest OS* is the problem, not Hyper-V.

### Step 2: Hyper-V Worker — why a VM failed to start

The **Hyper-V-Worker-Admin** log is the single most useful log when a VM refuses to power on. One worker process per running VM; if start-up fails, the cause is here:

```powershell
Get-WinEvent -FilterHashTable @{
    LogName = "Microsoft-Windows-Hyper-V-Worker-Admin"
} -MaxEvents 20 | Format-Table TimeCreated, Id, LevelDisplayName, Message -Wrap
```

Common message patterns and what they mean:

| Message contains… | Likely cause |
|---|---|
| `failed to power on with Error 'General access denied error'` | Host has Defender or another agent locking the VHDX file — close anything else that has the file open, then retry |
| `Key Protector could not be unwrapped` | vTPM key protector lost (Phase 2 Step 4) — re-run `Set-VMKeyProtector` |
| `'Node1' could not initialize. Synthetic SCSI Controller…` | A VHDX is missing or its path moved — check `Get-VMHardDiskDrive -VMName Node1` |
| `Cannot load a virtual machine configuration: 0x80070002` | `.vmcx` file moved or corrupted — restore from the VM's `Virtual Machines\` folder |

### Step 3: Hypervisor — host-level capability issues

The **Hyper-V-Hypervisor** log records whether the host itself can run VMs at all. This matters in a nested-virt setup because **the Azure VM's nested-virt capability is what makes any of this possible**:

```powershell
Get-WinEvent -FilterHashTable @{
    LogName = "Microsoft-Windows-Hyper-V-Hypervisor-Admin"
} -MaxEvents 10 | Format-Table TimeCreated, Id, LevelDisplayName, Message -Wrap
```

If you see errors like `Hyper-V launch failed; CPU does not support virtualization`, the host Azure VM was created without nested virtualization (wrong SKU or feature) — see SKILL Prerequisites. No amount of in-guest tweaking will fix this; the VM size has to change.

### Step 4: VMMS — broader service / configuration errors

The **Hyper-V Virtual Machine Management service (VMMS)** logs cover the Hyper-V management plane (anything `Get-VM` / `Set-VM` does goes through here):

```powershell
Get-WinEvent -FilterHashTable @{
    LogName = "Microsoft-Windows-Hyper-V-VMMS-Admin"
} -MaxEvents 20 | Format-Table TimeCreated, Id, LevelDisplayName, Message -Wrap
```

Useful for diagnosing failures in `New-VMSwitch`, `Connect-VMNetworkAdapter`, and credential-store / PowerShell Direct issues.

### Step 5: One-liner for "give me everything from the last hour"

When you do not yet know which log holds the answer, sweep all three:

```powershell
$since = (Get-Date).AddHours(-1)
@(
    "Microsoft-Windows-Hyper-V-Worker-Admin",
    "Microsoft-Windows-Hyper-V-Hypervisor-Admin",
    "Microsoft-Windows-Hyper-V-VMMS-Admin"
) | ForEach-Object {
    Get-WinEvent -FilterHashTable @{LogName = $_; StartTime = $since} -ErrorAction SilentlyContinue |
        Where-Object { $_.LevelDisplayName -in 'Error', 'Warning' } |
        Select-Object @{N='Log';E={($_.LogName -split '/')[0] -replace '^Microsoft-Windows-Hyper-V-',''}},
                       TimeCreated, Id, LevelDisplayName, Message
} | Format-Table -Wrap -AutoSize
```

This is the fastest way to see whether the failure point is the **VM** (Worker), the **host** (Hypervisor), or the **management layer** (VMMS), and to route to the right phase-specific entry below.

> **Reference**: for the full list of Hyper-V event channels and what each one logs, see `learn.microsoft.com/en-us/powershell/module/hyper-v/` and the Hyper-V hyper-v-expert reference under `manage/`.

## Phase 1: Host Networking

### Error: `The argument "External" does not belong to the set "Internal,Private"`

**Cause**: Microsoft's docs sometimes show `-SwitchType External`, but `-SwitchType` only accepts `Internal` or `Private`.

**Fix**: Omit `-SwitchType` when creating an External switch. Just supply `-NetAdapterName`:

```powershell
# Wrong (from the doc example)
New-VMSwitch -Name "External" -SwitchType External -NetAdapterName "Ethernet" -AllowManagementOS $true

# Right
New-VMSwitch -Name "External" -NetAdapterName "Ethernet" -AllowManagementOS $true
```

### Error: `Object reference not set... Failed to add NetNat`

**Cause**: A `NetNat` already exists (only one per host is allowed).

**Fix**: Either delete the existing one or choose a different IP range:

```powershell
Get-NetNat | Remove-NetNat -Confirm:$false
# Then re-run New-NetNat
```

### Host RDP drops after creating External switch

**Cause**: Normal — Azure VM's NIC is being bridged into Hyper-V.

**Fix**: Wait 30-60 seconds for the connection to come back. The session may need to reconnect. The switch creation succeeded.

## Phase 2: Node VM

### Error: `A guardian with the same name 'UntrustedGuardian' already exists`

**Cause**: A previous run already created the guardian. Not a real error.

**Fix**: PowerShell continues to the next line by default, so the key protector still gets applied. To verify:

```powershell
(Get-VMKeyProtector -VMName "Node1").Length
# If > 100 (typically 1000+), key protector was successfully applied
```

If `0` or very low, re-run the 3 lines after `New-HgsGuardian`:

```powershell
$owner = Get-HgsGuardian -Name "UntrustedGuardian"
$kp = New-HgsKeyProtector -Owner $owner -AllowUntrustedRoot
Set-VMKeyProtector -VMName "Node1" -KeyProtector $kp.RawData
```

### `Invoke-Command` fails: `The credential is invalid`

**Cause**: Wrong username format, password mismatch, or — surprisingly — recent network changes to the VM.

**Fix**:

1. Re-prompt for `$cred = Get-Credential` and try these username formats:
   - `Administrator`
   - `.\Administrator`
   - `Node1\Administrator`
2. If username/password are definitely correct, **restart the target VM**:
   ```powershell
   Restart-VM -Name "Node1" -Force
   ```
   PowerShell Direct's credential broker can get into a bad state after network configuration changes (especially switch swaps). A restart fixes it.
3. If the failure persists across restarts, the cause is usually NOT credentials — check the VMMS log via the **First-line Hyper-V diagnostics § Step 4** at the top of this file. PSDirect goes through VMMS, so authentication-broker errors land there before they surface in PowerShell.

### `Test-NetConnection` to internet fails

**Cause**: Either DNS or NAT isn't working.

**Fix**:
```powershell
# From Node1 - check each layer
Invoke-Command -VMName "Node1" -Credential $cred -ScriptBlock {
    # Layer 1: Can reach NAT gateway?
    Test-NetConnection -ComputerName "192.168.44.1" -InformationLevel Quiet
    # Layer 2: Can resolve DNS?
    Resolve-DnsName -Name "www.microsoft.com" -ErrorAction SilentlyContinue
    # Layer 3: Can reach via TCP?
    Test-NetConnection -ComputerName "8.8.8.8" -Port 443
}
```
- Layer 1 fails → host's `New-NetIPAddress` or `New-NetNat` is misconfigured
- Layer 2 fails → DNS server unreachable (e.g., if you set the DNS to AD's IP but AD isn't up yet)
- Layer 3 fails → NAT is broken or Azure NSG is blocking outbound

## Phase 3: Azure Prep

### `Connect-AzAccount` blocked by Conditional Access

**Symptoms** (any of):

- The browser auth page returns `AADSTS530003` / `AADSTS530004` / `AADSTS50158` / `AADSTS50199` / "Your sign-in was successful but does not meet the criteria to access this resource" / "You can't get there from here".
- Device-code flow accepts the code but then bounces back to a "device must be compliant / hybrid joined" page.
- `Connect-AzAccount` returns immediately with `AADSTS50005` (sign-in blocked by tenant policy) or `AADSTS50097` (device authentication required).
- A pop-up demands a managed-device certificate that the host Azure VM does not have.

**Cause**: The Entra tenant whose user account you are trying to sign in as enforces Conditional Access (CA) — device compliance, hybrid Entra join, named-location, sign-in risk, etc. — and the **host Azure VM does not satisfy those conditions**. This is the default state for tightly-managed enterprise tenants (including many Microsoft engineer accounts) and increasingly common for enterprise customers in general.

**Fix — switch from Plan A to Plan B.** The host VM tenant and the cluster registration tenant are independent (see `references/03-azure-prep.md` § "Choose your tenant strategy"). Sign in to a separate tenant (Tenant B) whose CA permits sign-in from an arbitrary Azure VM IP:

```powershell
# Example: register the cluster to an MSDN / Visual Studio Enterprise / partner /
# personal dev tenant instead of your primary work tenant.
Connect-AzAccount -TenantId "<registration-tenant-guid>"

# Confirm which tenant you actually landed in
$ctx = Get-AzContext
Write-Host "Tenant:       $($ctx.Tenant.Id)"
Write-Host "Subscription: $($ctx.Subscription.Id)"
Write-Host "Account:      $($ctx.Account.Id)"
```

The `$tenantId` / `$subscriptionId` you capture from this context flow straight into Phase 4's `Invoke-AzStackHciArcInitialization` — the cluster will be registered into this tenant, not the host VM's tenant.

**If you must use the original tenant (for billing / compliance reasons)**, ask the tenant administrator to do one of the following before retrying:

1. Add the host Azure VM's outbound public IP to a **trusted Named Location** and exempt it for the Azure Local engineer group.
2. Provide a **break-glass / unconditional** account scoped to the lab subscription.
3. Sign you in from a compliant device, then run `Connect-AzAccount -UseDeviceAuthentication` from the host and complete the code on the compliant device (works for tenants that gate on device state but not on IP).

**Guidance for engineers in tightly-managed work tenants** (including many Microsoft engineers): Plan B is the supported pattern when Plan A is impossible. Keep the host VM in your primary work subscription (Tenant A) but register the cluster into an MSDN / Visual Studio Enterprise / partner / personal-dev tenant (Tenant B). Document this split tenancy in your lab paperwork — it is the expected configuration for this scenario, not a workaround. Cost split between the two subscriptions (including the **60-day free trial** of the Azure Local service fee on the Tenant B side) is detailed in `references/03-azure-prep.md` § "Cost split between the two tenants".

### `Connect-AzAccount` succeeds but lands in the wrong tenant

**Cause**: Your account is a guest or member in multiple tenants. `Connect-AzAccount` without `-TenantId` defaults to the account's **home tenant**, which is often *not* the registration tenant you wanted.

**Fix**: Always pass `-TenantId` explicitly when doing Plan B (cross-tenant) registration:

```powershell
Disconnect-AzAccount
Connect-AzAccount -TenantId "<registration-tenant-guid>"

# Then verify before continuing to Step 2 of Phase 3:
Get-AzContext | Format-List Account, Tenant, Subscription
```

If the subscription list is still empty after sign-in, run `Get-AzSubscription -TenantId "<registration-tenant-guid>"` and `Set-AzContext -SubscriptionId "<sub-guid>"` to pick one explicitly.

### `Register-AzResourceProvider` stuck at `Registering`

**Cause**: Normal — registration is asynchronous and takes 1-5 minutes per provider.

**Fix**: Just poll until `Registered`. Don't proceed before all 13 are confirmed.

### Role assignment fails: "User does not have access"

**Cause**: You need at least Contributor + User Access Administrator (or Owner) to assign roles.

**Fix**: Have an Owner add **User Access Administrator** to your account, then retry.

## Phase 4: Arc Registration

### `Invoke-AzStackHciArcInitialization: command not found`

**Cause**: The `AzureEdgeBootstrap` module isn't on Node1. Very rare on Azure Stack HCI installs.

**Fix**: Reinstall Azure Stack HCI OS, ensuring you didn't choose a non-Azure-Local SKU.

### Bootstrap reports `ConnectivityValidation: Failed`

**Cause**: Node1 can't reach one of the required Azure endpoints.

**Fix**: Verify each from Node1:

```powershell
Invoke-Command -VMName "Node1" -Credential $cred -ScriptBlock {
    @(
        "login.microsoftonline.com",
        "management.azure.com",
        "his.arc.azure.com",
        "guestnotificationservice.azure.com",
        "*.guestconfiguration.azure.com"
    ) | ForEach-Object {
        if ($_ -notlike "*\**") {
            Test-NetConnection -ComputerName $_ -Port 443 -InformationLevel Quiet |
                Select-Object @{N='Host';E={$_}}, @{N='OK';E={$_}}
        }
    }
}
```

All should be reachable. If not, check the NSG on the host's Azure VNet, host's NAT, or any corporate proxy.

### Bootstrap fails: `Token expired`

**Cause**: The ARM token from `Get-AzAccessToken` is valid for ~1 hour. If you generated it more than an hour before running the bootstrap, it expires.

**Fix**: Regenerate the token and immediately re-run `Invoke-AzStackHciArcInitialization`:

```powershell
$armTokenResponse = Get-AzAccessToken -WarningAction SilentlyContinue
$ArmAccessToken = [System.Net.NetworkCredential]::new("", $armTokenResponse.Token).Password
# ... then immediately run the bootstrap
```

## Phase 5: Active Directory

### DC VM gets `169.254.x.x` (APIPA) instead of static IP

**Cause**: The DC VM is connected to the wrong virtual switch (typically External).

**Fix**:
```powershell
# Move to Internal switch
Get-VMNetworkAdapter -VMName "ActiveDirectory" | Connect-VMNetworkAdapter -SwitchName "Internal"
Get-VMNetworkAdapter -VMName "ActiveDirectory" | Set-VMNetworkAdapter -MacAddressSpoofing On

# Restart the VM — PowerShell Direct may refuse credentials after switch swap
Restart-VM -Name "ActiveDirectory" -Force
```

Then assign the static IP (Phase 5 Step B3).

### `Invoke-Command` to DC: credential is invalid

**Cause**: 
- DC was just promoted. Login transitioned from local Administrator to Domain Administrator.
- OR: VM network was just changed (see Phase 2 troubleshooting above).

**Fix**:
1. Try `azurelocal\Administrator` instead of just `Administrator`
2. If still failing, **restart the VM** and retry

### LCM user password "passes" New-HciAdObjectsPreCreation but fails portal wizard

**Cause**: AD's default password policy allows 7+ chars, but Azure Local requires **14+ chars with all 4 character classes**.

**Fix**: Reset the password to a stronger one:

```powershell
$newLcmPwd = Read-Host -Prompt "New 14+ char password" -AsSecureString

Invoke-Command -VMName "ActiveDirectory" -Credential $dcCred -ScriptBlock {
    param($pwd)
    Set-ADAccountPassword -Identity "lcmuser" -Reset -NewPassword $pwd
} -ArgumentList $newLcmPwd
```

### DC can't resolve external names like `www.microsoft.com`

**Cause**: No DNS forwarder set on the DC's DNS server.

**Fix**:
```powershell
Invoke-Command -VMName "ActiveDirectory" -Credential $dcCred -ScriptBlock {
    Add-DnsServerForwarder -IPAddress "8.8.8.8"
}
```

### Node can't resolve `azurelocal.com`

**Cause**: Node's DNS is still set to `8.8.8.8` from Phase 2.

**Fix**: Repoint Node's DNS to the DC:

```powershell
Invoke-Command -VMName "Node1" -Credential $cred -ScriptBlock {
    Set-DnsClientServerAddress -InterfaceAlias "NIC1" -ServerAddresses "192.168.44.10"
}
```

## Phase 6: Cluster Deployment

### Extension installation hangs on `Not validated`

**Cause**: Arc extensions take 5-15 minutes to install. Sometimes the portal UI doesn't auto-refresh.

**Fix**: Manually refresh the page. If status doesn't change after 20 minutes, check that the Arc agent on Node1 is still connected:

```powershell
Get-AzResource -ResourceGroupName "azurelocalconnected" `
    -ResourceType "Microsoft.HybridCompute/machines" -Name "Node1" |
    Select-Object -ExpandProperty Properties | ConvertTo-Json
```

Look for `status: "Connected"`.

### Validation fails: "Cannot reach domain controller"

**Cause**: Node1's DNS isn't pointing at the DC, or the DC isn't running.

**Fix**: Verify Node-to-DC connectivity:
```powershell
Invoke-Command -VMName "Node1" -Credential $cred -ScriptBlock {
    Get-DnsClientServerAddress -InterfaceAlias "NIC1"   # Should be 192.168.44.10
    Test-NetConnection -ComputerName "192.168.44.10" -Port 53
    Resolve-DnsName -Name "azurelocal.com" -Type A
}
```

### Deployment fails partway through

**Cause**: Many possibilities — network glitch, validation issue, password rotation, etc.

**Fix**:
1. Go to: Resource group → Cluster → **Deployments**
2. Click the failed step → **View details**
3. The error message usually points to a specific Phase 5 misstep or a transient Azure issue
4. Click **Resume deployment** at the top (Azure can often retry from the failed step)

### Local Administrator credentials no longer work after deployment

**Cause**: For security, Azure Local renames the local Administrator account after successful deployment.

**Fix**: Find the new admin name in the portal:
> Resource group → Cluster → Settings → Local built-in user accounts

## Cleanup procedure

To stop all Azure Local charges and remove the lab cleanly:

### Step 1: Delete cluster from Azure

In the Azure portal:
1. Resource group `azurelocalconnected` → Cluster `azlocal-cluster`
2. Click **Delete** at the top
3. Confirm

This stops Azure Local service fees. May take 10-30 minutes.

### Step 2: Delete the resource group

If you want to remove everything (Key Vault, Storage Accounts, Arc machine, etc.):

```powershell
Remove-AzResourceGroup -Name "azurelocalconnected" -Force
```

### Step 3: Destroy the Node VM

On the host:
```powershell
Stop-VM -Name "Node1" -Force
Remove-VM -Name "Node1" -Force
# Remove the VHDX files
Remove-Item -Path "E:\DCVM\Node1", "F:\DCVM\Node1", "G:\DCVM\Node1" -Recurse -Force
```

The Node1 OS is unusable after cluster deletion anyway — Azure Local locks the OS to its (now deleted) cluster registration.

### Step 4: Optionally delete the DC VM

If you don't need AD anymore:
```powershell
Stop-VM -Name "ActiveDirectory" -Force
Remove-VM -Name "ActiveDirectory" -Force
Remove-Item -Path "E:\DCVM\ActiveDirectory" -Recurse -Force
```

### Step 5: Stop or delete the host Azure VM

Either:
- **Stop (deallocate)** the host Azure VM to stop compute charges (keeps disks, you can restart later)
- **Delete** the host Azure VM and disks for full cleanup

### Step 6: Remove NetNat (optional)

If you plan to keep the host but want a clean state:
```powershell
Get-NetNat | Remove-NetNat -Confirm:$false
Get-VMSwitch | Remove-VMSwitch -Force
```

## When all else fails

- Check the deployment logs: Cluster → Deployments → expand the failed step
- Collect support logs from Node1:
  ```powershell
  Invoke-Command -VMName "Node1" -Credential $cred -ScriptBlock {
      Collect-ArcBootstrapSupportLogs
  }
  ```
- Microsoft's [Get support for deployment issues](https://learn.microsoft.com/en-us/azure/azure-local/manage/get-support-for-deployment-issues) page
- Filing a support ticket: include the deployment correlation ID from the failed step
