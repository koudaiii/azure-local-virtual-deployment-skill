# Phase 5: Active Directory Setup (DC VM + OU + LCM User)

> **Source documentation**: Microsoft Learn — [Prepare Active Directory environment for Azure Local deployment](https://learn.microsoft.com/en-us/azure/azure-local/deploy/deployment-prep-active-directory).
>
> The official doc assumes an existing AD domain. This skill adds Part A (building a Domain Controller VM from scratch on the Internal switch) since lab environments rarely have a pre-existing domain.
>
> **Teaching reminder**: Apply the four teaching principles from `SKILL.md` — (1) show before/after state for every change, (2) point the user at the source doc above, (3) accept screenshots and links the user shares, (4) state which layer/credential each command runs as and preview any dialog before it appears. See `references/00-accounts-and-context.md`. This phase has the most before/after-critical steps in the whole skill (switch swap, static IP, DNS repoint) and introduces **two new credential variables** (`$dcCred`, `$lcmCred`) — bookend every step with checks and confirm the right credential is being used each time.

> ⚠ **Three Administrators are now in play.** Open `references/00-accounts-and-context.md` side-by-side. The Get-Credential dialog and the username `Administrator` will appear several times in this phase, and they refer to **different accounts** each time.
>
> Quick recap for this phase:
>
> ```
> azureuser           = local OS account on the host Azure VM (RDP login)
> $cred               = Node1 local Administrator (set in Phase 2)
> $dcCred  ← NEW      = DC VM Administrator (becomes Domain Admin of azurelocal.com)
> $lcmCred ← NEW      = lcmuser, the AD domain user the cluster wizard will use
> ```
>
> **Context shifts in this phase**:
>
> | Steps | Where you act |
> |---|---|
> | A1 (build DC VM) | `[Host PowerShell — azureuser]` |
> | A2 (Windows install) | `[Inside DC console]` via `vmconnect` |
> | A3 (install AD DS) | `[Inside DC console]` — opens PowerShell as Admin inside the DC |
> | B1 (verify switch) | `[Host PowerShell — azureuser]` |
> | B2 (capture `$dcCred`) | `[Host PowerShell — azureuser]` — `Get-Credential` dialog appears |
> | B3–B5 (IP, DNS, Node DNS) | `[Host PowerShell → DC via $dcCred]` and `[Host PowerShell → Node1 via $cred]` |
> | C1–C3 (OU + lcmuser) | `[Host PowerShell → DC via $dcCred]` — `Get-Credential` for `$lcmCred` appears on the host |
> | C4 (reset LCM pwd, if needed) | `[Host PowerShell — azureuser]` + `[Host PowerShell → DC via $dcCred]` |

Azure Local **requires an Active Directory domain** for the cluster, even in a lab. This phase creates a Domain Controller VM, sets up DNS, and pre-creates the OU and LCM (Lifecycle Manager) user that the cluster deployment will use.

## What you build

```
DC VM "ActiveDirectory" (on Internal switch)
├─ Windows Server 2022 or 2025
├─ IP: 192.168.44.10
├─ Roles: AD DS, DNS
├─ Domain: azurelocal.com (or your choice)
├─ DNS forwarder: 8.8.8.8 (for external resolution)
└─ AD objects:
    ├─ OU=AzureLocal,DC=azurelocal,DC=com
    └─ User: lcmuser (in that OU, with permissions)

Node1 NIC1 DNS → 192.168.44.10 (was 8.8.8.8 in Phase 2)
```

## ⚠️ Critical principle: Node MUST NOT be domain-joined yet

> *"Machines must not be joined to Active Directory before deployment."* (Microsoft Learn)

The cluster deployment wizard domain-joins Node1 automatically using `lcmuser`. If you join Node1 to the domain manually first, deployment will fail. **Just create the domain and the OU/user; don't touch Node1's domain status.**

## Part A: Create the DC VM

### Step A1: Build the VM

```powershell
# 8 GB RAM, 4 vCPU, ~60 GB OS disk
New-Item -ItemType Directory -Path "E:\DCVM\ActiveDirectory" -Force
New-VHD -Path "E:\DCVM\ActiveDirectory\OS.vhdx" -SizeBytes 60GB

New-VM -Name "ActiveDirectory" `
    -MemoryStartupBytes 8GB `
    -VHDPath "E:\DCVM\ActiveDirectory\OS.vhdx" `
    -Generation 2 `
    -Path "E:\DCVM\ActiveDirectory" `
    -SwitchName "Internal"

Set-VMMemory -VMName "ActiveDirectory" -DynamicMemoryEnabled $false
Set-VMProcessor -VMName "ActiveDirectory" -Count 4

# Enable MAC spoofing
Get-VMNetworkAdapter -VMName "ActiveDirectory" | Set-VMNetworkAdapter -MacAddressSpoofing On

# Mount Windows Server ISO
Add-VMDvdDrive -VMName "ActiveDirectory" -Path "C:\Users\azureuser\Downloads\WindowsServer.iso"
$dvd = Get-VMDvdDrive -VMName "ActiveDirectory"
Set-VMFirmware -VMName "ActiveDirectory" -FirstBootDevice $dvd

Start-VM -Name "ActiveDirectory"
vmconnect.exe localhost ActiveDirectory
```

### Step A2: Install Windows Server (via console)

`[Inside DC console]` — all interaction in this step happens in the vmconnect window that opens from `vmconnect.exe localhost ActiveDirectory` in Step A1.

Standard Windows Server install:
1. Choose **Windows Server 2022/2025 Standard (Desktop Experience)** or Datacenter
2. Custom install on the 60 GB disk
3. Set Administrator password. The "Customize settings" prompt looks like:

   ```
   ┌─ DC console — Customize settings ───────────────────────────────┐
   │                                                                 │
   │ Type a password for the built-in administrator account that     │
   │ you can use to sign in to this computer.                        │
   │                                                                 │
   │ Password:          [ ********************              ]        │
   │ Reenter password:  [ ********************              ]        │
   │                                                                 │
   │ → 14+ chars recommended. This is the DC VM's local Admin,       │
   │   which becomes Domain Admin of azurelocal.com after Step A3.   │
   │   Captured into $dcCred in Step B2.                             │
   │                                                                 │
   │                                                  [ Finish ]     │
   └─────────────────────────────────────────────────────────────────┘
   ```
4. After install, sign in to the desktop with the password you just set.

### Step A3: Install AD DS and promote to DC (inside the DC VM)

`[Inside DC console]` — open an elevated PowerShell *inside the DC VM* (Start → right-click PowerShell → Run as Administrator). Do **not** run these in the host's PowerShell window — they must execute on the DC itself.

```powershell
# Install AD DS
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

# Promote to DC, creating a new forest
$dsrmPassword = Read-Host -Prompt "DSRM password (14+ chars)" -AsSecureString

Install-ADDSForest `
    -DomainName "azurelocal.com" `
    -DomainNetbiosName "AZURELOCAL" `
    -SafeModeAdministratorPassword $dsrmPassword `
    -InstallDns `
    -Force
```

The VM auto-reboots after promotion. After reboot, log in as `Administrator` (now Domain Admin).

## Part B: Network configuration

### ⚠️ Common gotcha: DC VM on wrong switch → APIPA

If the DC VM is on the **External** switch (Azure VM's network) instead of Internal, it can't get a DHCP address and falls back to `169.254.x.x` (APIPA).

### Step B1: Verify the DC VM is on Internal switch

```powershell
# On host
Get-VMNetworkAdapter -VMName "ActiveDirectory" |
    Format-Table Name, SwitchName, MacAddress, MacAddressSpoofing
```

If `SwitchName` is `External`, fix it:

```powershell
Get-VMNetworkAdapter -VMName "ActiveDirectory" |
    Connect-VMNetworkAdapter -SwitchName "Internal"
```

**Then restart the DC VM** — PowerShell Direct sometimes refuses credentials after a switch swap until restart:

```powershell
Restart-VM -Name "ActiveDirectory" -Force
```

### Step B2: Connect to DC via PowerShell Direct

`[Host PowerShell — azureuser]`

```powershell
$dcCred = Get-Credential
```

The credential dialog that appears here looks identical to the one used for `$cred`, but it is a **different account** — the DC VM's Administrator, which became the Domain Admin of `azurelocal.com` after `Install-ADDSForest` ran. Do **not** type Node1's password here.

```
┌─ Windows PowerShell credential request ─────────────────────────┐
│                                                                 │
│ DC VM Administrator (now Domain Admin of azurelocal.com)        │
│                                                                 │
│ User name:  [ Administrator                          ]   ▼     │
│ Password:   [ ********************                   ]         │
│                                                                 │
│                                       [   OK   ]  [ Cancel ]    │
└─────────────────────────────────────────────────────────────────┘
```

- **User name**: `Administrator`. If that fails when used with `Invoke-Command` below, retry with `azurelocal\Administrator`, then `.\Administrator`, then `Administrator@azurelocal.com`.
- **Password**: the one you set during the Windows Server install in Step A2.

Verify the variable holds the expected account before continuing:

```powershell
$dcCred.UserName        # → Administrator (or azurelocal\Administrator, etc.)
```

Then test:

```powershell
Invoke-Command -VMName "ActiveDirectory" -Credential $dcCred -ScriptBlock { hostname }
# Expected: ActiveDirectory
```

If credential fails, see `references/00-accounts-and-context.md` § "What to do when a credential is rejected".

### Step B3: Set static IP, gateway, and DNS forwarder

`[Host PowerShell → DC via $dcCred]`

```powershell
Invoke-Command -VMName "ActiveDirectory" -Credential $dcCred -ScriptBlock {
    # Remove any APIPA
    Get-NetIPAddress -InterfaceAlias "Ethernet" -AddressFamily IPv4 -ErrorAction SilentlyContinue |
        Remove-NetIPAddress -Confirm:$false
    Get-NetRoute -InterfaceAlias "Ethernet" -DestinationPrefix "0.0.0.0/0" -ErrorAction SilentlyContinue |
        Remove-NetRoute -Confirm:$false

    # Static IP
    New-NetIPAddress -InterfaceAlias "Ethernet" `
        -IPAddress "192.168.44.10" -PrefixLength 24 `
        -DefaultGateway "192.168.44.1" -AddressFamily IPv4

    # DC's own DNS is itself (127.0.0.1), but we need a forwarder for external names
    Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses "127.0.0.1"

    if ((Get-DnsServerForwarder).IPAddress.IPAddressToString -notcontains "8.8.8.8") {
        Add-DnsServerForwarder -IPAddress "8.8.8.8"
    }
}
```

### Step B4: Verify DNS resolution

`[Host PowerShell → DC via $dcCred]`

```powershell
Invoke-Command -VMName "ActiveDirectory" -Credential $dcCred -ScriptBlock {
    Resolve-DnsName -Name "azurelocal.com" -Type A     # Internal
    Resolve-DnsName -Name "www.microsoft.com"          # External (via forwarder)
}
```

Both should succeed.

### Step B5: Point Node1's DNS at the DC

`[Host PowerShell → Node1 via $cred]` — note this step jumps **back** to using `$cred` (Node1's credential), not `$dcCred`. The DNS we are updating belongs to Node1.

```powershell
Invoke-Command -VMName "Node1" -Credential $cred -ScriptBlock {
    Set-DnsClientServerAddress -InterfaceAlias "NIC1" -ServerAddresses "192.168.44.10"
}

# Verify
Invoke-Command -VMName "Node1" -Credential $cred -ScriptBlock {
    Test-NetConnection -ComputerName "192.168.44.10" -InformationLevel Quiet
    Resolve-DnsName -Name "azurelocal.com" -Type A
    Resolve-DnsName -Name "www.microsoft.com"
}
```

All three should succeed.

## Part C: Create OU and LCM user

### Step C1: Install the helper module on the DC

`[Host PowerShell → DC via $dcCred]` — back to talking to the DC.

```powershell
Invoke-Command -VMName "ActiveDirectory" -Credential $dcCred -ScriptBlock {
    # Accept NuGet prompt automatically with -Force
    Install-PackageProvider -Name NuGet -MinimumVersion 2.8.5.201 -Force -Scope AllUsers

    Set-PSRepository -Name PSGallery -InstallationPolicy Trusted

    Install-Module AsHciADArtifactsPreCreationTool -Repository PSGallery -Force -AllowClobber

    Get-Command -Module AsHciADArtifactsPreCreationTool
}
```

Expected commands: `New-HciAdObjectsPreCreation`, `Remove-AsHciOU`.

### Step C2: Create OU and LCM user

`[Host PowerShell — azureuser]` for the Get-Credential, then `[Host PowerShell → DC via $dcCred]` for the AD object creation.

⚠️ **Password requirement**: The LCM user password must be **14+ chars, with upper/lower/digit/special**. AD's default policy allows 7+ chars, so a too-short password passes `New-HciAdObjectsPreCreation` but later fails in the Azure portal wizard (Phase 6). **Always use 14+ chars.**

`Get-Credential` here creates a **new domain user** — you are not signing in as someone existing, you are *defining* the username and password that will be created inside AD. The dialog looks like:

```
┌─ Windows PowerShell credential request ─────────────────────────┐
│                                                                 │
│ LCM credentials (user: lcmuser, password: 14+ chars)            │
│                                                                 │
│ User name:  [ lcmuser                                ]   ▼     │
│ Password:   [ ********************                   ]         │
│                                                                 │
│                                       [   OK   ]  [ Cancel ]    │
└─────────────────────────────────────────────────────────────────┘
```

- **User name**: literally `lcmuser` (no domain prefix). This becomes a domain user inside `OU=AzureLocal,DC=azurelocal,DC=com`.
- **Password**: invent a fresh 14+ char password with all 4 character classes. Write it down — you will type it again into the Azure portal wizard in Phase 6 Tab 4 ("Deployment account").

```powershell
# Capture LCM credentials (user = "lcmuser", password = 14+ chars, all 4 classes)
$lcmCred = Get-Credential -Message "LCM credentials (user: lcmuser, password: 14+ chars)"

$lcmCred.UserName       # → lcmuser  (sanity check before running the command below)

Invoke-Command -VMName "ActiveDirectory" -Credential $dcCred -ScriptBlock {
    param($lcmCredential)
    New-HciAdObjectsPreCreation `
        -AzureStackLCMUserCredential $lcmCredential `
        -AsHciOUName "OU=AzureLocal,DC=azurelocal,DC=com"
} -ArgumentList $lcmCred
```

Expected verbose output:
```
VERBOSE: Successfully verified DC=azurelocal,DC=com
VERBOSE: Successfully created AzureLocal organization unit within the 'DC=azurelocal,DC=com'
VERBOSE: Successfully created 'lcmuser' within the 'OU=AzureLocal,DC=azurelocal,DC=com'
VERBOSE: Access permissions to 'OU=AzureLocal,DC=azurelocal,DC=com' have been successfully granted to 'lcmuser'
VERBOSE: Gpo inheritance blocked for 'OU=AzureLocal,DC=azurelocal,DC=com', inheritance blocked state is : True
```

### Step C3: Verify

`[Host PowerShell → DC via $dcCred]`

```powershell
Invoke-Command -VMName "ActiveDirectory" -Credential $dcCred -ScriptBlock {
    Write-Host "=== OU ===" -ForegroundColor Cyan
    Get-ADOrganizationalUnit -Filter "Name -eq 'AzureLocal'" |
        Format-List Name, DistinguishedName

    Write-Host "=== Users in OU ===" -ForegroundColor Cyan
    Get-ADObject -Filter * -SearchBase "OU=AzureLocal,DC=azurelocal,DC=com" |
        Format-Table Name, ObjectClass

    Write-Host "=== lcmuser ===" -ForegroundColor Cyan
    Get-ADUser -Identity "lcmuser" -Properties Enabled |
        Format-List Name, Enabled, DistinguishedName

    Write-Host "=== GPO inheritance ===" -ForegroundColor Cyan
    Get-GPInheritance -Target "OU=AzureLocal,DC=azurelocal,DC=com" |
        Format-List Name, GpoInheritanceBlocked
}
```

✅ OU `AzureLocal` exists
✅ `lcmuser` is in the OU and `Enabled : True`
✅ `GpoInheritanceBlocked : True`

### Step C4: Reset LCM password if it was <14 chars

If `New-HciAdObjectsPreCreation` accepted a shorter password (because AD allowed it), reset it before Phase 6:

```powershell
$newLcmPwd = Read-Host -Prompt "New 14+ char password" -AsSecureString

Invoke-Command -VMName "ActiveDirectory" -Credential $dcCred -ScriptBlock {
    param($pwd)
    Set-ADAccountPassword -Identity "lcmuser" -Reset -NewPassword $pwd
} -ArgumentList $newLcmPwd
```

## How to verify this phase is done

✅ DC VM is on Internal switch, IP `192.168.44.10`
✅ DC resolves both `azurelocal.com` (internal) and `www.microsoft.com` (external)
✅ Node1's NIC1 DNS = `192.168.44.10`, Node1 can resolve `azurelocal.com`
✅ AD has `OU=AzureLocal,DC=azurelocal,DC=com` with GPO inheritance blocked
✅ `lcmuser` exists in that OU, Enabled, with a 14+ char password
✅ **Node1 is NOT domain-joined yet** (verify with `(Get-WmiObject Win32_ComputerSystem).PartOfDomain` — should be `False`)

→ Proceed to **Phase 6** (`references/06-cluster-portal.md`)
