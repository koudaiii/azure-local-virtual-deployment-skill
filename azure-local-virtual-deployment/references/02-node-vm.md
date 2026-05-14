# Phase 2: Node VM Creation + Azure Stack HCI OS Setup

> **Source documentation**:
> - Microsoft Learn — [Deploy Azure Local using virtual machines](https://learn.microsoft.com/en-us/azure/azure-local/deploy/deployment-virtual), sections "Create the virtual host" and "Install the OS on the virtual host VMs".
> - [Download Azure Local software (ISO)](https://learn.microsoft.com/en-us/azure/azure-local/deploy/download-23h2-software) — where to get the OS image.
>
> The procedure below adds VHDX disk-distribution strategy, MAC-based NIC renaming using captured variables, and explicit DNS bootstrapping (which is changed again in Phase 5).
>
> **Teaching reminder**: Apply the four teaching principles from `SKILL.md` — (1) show before/after state for every change, (2) point the user at the source docs above, (3) accept screenshots and links the user shares, (4) state which layer/credential each command runs as and preview any dialog before it appears. See `references/00-accounts-and-context.md`.

> **Context shifts in this phase** — read this once before starting:
>
> | Steps | Where you act |
> |---|---|
> | 1–6 (VM creation) | `[Host PowerShell — azureuser]` |
> | 7 (OS install) | `[Inside Node1 console]` via `vmconnect.exe` window |
> | 8 (capture `$cred`) | `[Host PowerShell — azureuser]` — `Get-Credential` dialog appears |
> | 9–12 (NIC rename, IP, Hyper-V) | `[Host PowerShell → Node1 via $cred]` |
>
> Each command block below begins with its badge so you can scan for shifts.

This is the longest phase. You create the Node VM, install Azure Stack HCI OS, rename network adapters by MAC address, set a static IP, and enable the Hyper-V role.

### Why Generation 2 + Secure Boot + vTPM are mandatory (not optional)

Azure Stack HCI / Azure Local requires UEFI firmware, Secure Boot, and a TPM 2.0 for the OS to install and for BitLocker to engage during cluster deployment. In a nested Hyper-V VM these requirements map to:

| Azure Local requirement | Hyper-V VM setting | Why |
|---|---|---|
| UEFI firmware | **Generation 2** | Gen 1 VMs only emulate BIOS; Gen 2 emulates UEFI. There is no way to retrofit a Gen 1 VM. |
| Secure Boot | Gen 2 default (enabled) | Required for the HCI installer to accept the disk and for Secure Boot validation chain. |
| TPM 2.0 | **vTPM** (Step 4) | Cluster deployment uses BitLocker; BitLocker requires a TPM. vTPM only exists on Gen 2 VMs. |
| Nested virt | `ExposeVirtualizationExtensions $true` (Step 2) | Node1 itself runs Hyper-V to host the cluster's infra VMs. |

So the chain is: **Azure Local needs BitLocker → BitLocker needs TPM → TPM means vTPM → vTPM means Gen 2 → therefore everything in this phase is Gen 2.** This is not a "best practice", it is a hard prerequisite — Gen 1 fails at the installer screen.

> **Lab caveat**: vTPM in this lab is backed by `UntrustedGuardian` with `-AllowUntrustedRoot` (Step 4). That is deliberately a *weakened* trust posture suitable only for a lab — in production you would tie the guardian to a real HGS attestation service. We accept the weaker posture because the alternative (deploying HGS) is out of scope for a learning environment.

## What you build

```
Node1 VM (on Internal switch)
├─ OS: Azure Stack HCI 24H2 (Build 26100+)
├─ 32 GB RAM, 4 vCPU, vTPM, nested virtualization enabled
├─ 4 NICs (NIC1-NIC4) all on Internal switch
│   └─ NIC1 = 192.168.44.201/24 (management)
├─ 3 VHDX disks: OS (127GB) + s2d1 (1TB) + s2d2 (1TB)
└─ Hyper-V role enabled
```

## Before you start

Confirm Phase 1 is complete. Decide where VHDX files will live based on available drives:

```powershell
# Check free space
Get-Volume | Where-Object DriveLetter -in 'C','E','F','G' |
    Format-Table DriveLetter, FileSystemLabel,
        @{N='SizeGB';E={[math]::Round($_.Size/1GB,1)}},
        @{N='FreeGB';E={[math]::Round($_.SizeRemaining/1GB,1)}}
```

**VHDX placement strategy** (recommended for I/O distribution):
- OS VHDX (127 GB) → smallest available drive with >150 GB free
- s2d1.vhdx (1 TB) and s2d2.vhdx (1 TB) → separate drives for I/O parallelism

Example: if E: has 500 GB and F:, G: have 1 TB each → OS on E:, s2d1 on F:, s2d2 on G:.

## Step 1: Create Node VM with OS disk

```powershell
# Adjust paths to match your drive layout
New-Item -ItemType Directory -Path "E:\DCVM\Node1" -Force
New-Item -ItemType Directory -Path "F:\DCVM\Node1" -Force
New-Item -ItemType Directory -Path "G:\DCVM\Node1" -Force

New-VHD -Path "E:\DCVM\Node1\OS.vhdx" -SizeBytes 127GB
New-VM -Name "Node1" `
    -MemoryStartupBytes 32GB `
    -VHDPath "E:\DCVM\Node1\OS.vhdx" `
    -Generation 2 `
    -Path "E:\DCVM\Node1"
```

## Step 2: VM tuning

`[Host PowerShell — azureuser]`

```powershell
# Disable dynamic memory (Azure Local wants static)
Set-VMMemory -VMName "Node1" -DynamicMemoryEnabled $false

# Disable automatic checkpoints (not supported on cluster nodes)
Set-VM -Name "Node1" -CheckpointType Disabled

# Set 4 vCPUs
Set-VMProcessor -VMName "Node1" -Count 4

# Enable nested virtualization (REQUIRED for Azure Local)
Set-VMProcessor -VMName "Node1" -ExposeVirtualizationExtensions $true

# Disable time sync (the cluster manages its own time)
Disable-VMIntegrationService -VMName "Node1" -Name "Time Synchronization"
```

### Why we *disable* checkpoints here (against Hyper-V's general advice)

Generic Hyper-V guidance recommends **Production checkpoints** for any production VM, so disabling checkpoints entirely on Node1 looks wrong at first glance. It is correct for this specific case because Node1 is going to be a **Storage Spaces Direct (S2D) cluster node**, and S2D is fundamentally incompatible with Hyper-V checkpoints:

- A checkpoint freezes the VM's disk state at a point in time, but S2D is a distributed storage layer that expects its physical disks to keep moving forward consistently with the cluster. Reverting a node to a checkpoint puts its S2D metadata out of sync with the rest of the cluster, which the cluster cannot recover from.
- Microsoft's Azure Local deployment validator explicitly checks `CheckpointType` on the host VM that will become a node — leaving the default Production value will cause validation warnings.
- Automatic checkpoints (introduced in Windows 10 / Server 2019) are even worse because they trigger silently on every VM start.

So the rule is: **for *normal* Hyper-V VMs, keep Production checkpoints on. For VMs that will become Azure Local / S2D nodes, set `-CheckpointType Disabled`.** This applies only to Node1 in our lab — the DC VM (Phase 5) keeps default checkpoint behavior because it is not part of the storage cluster.

If you ever need a "save point" for Node1 during the lab, **stop the VM and copy the VHDX files** instead — that gives you a real snapshot that does not break S2D.

## Step 3: Network adapters (4 NICs)

```powershell
# Remove the default adapter
Get-VMNetworkAdapter -VMName "Node1" | Remove-VMNetworkAdapter

# Add 4 NICs named NIC1-NIC4
1..4 | ForEach-Object {
    Add-VMNetworkAdapter -VMName "Node1" -Name "NIC$_"
}

# Connect all to Internal switch
Get-VMNetworkAdapter -VMName "Node1" | Connect-VMNetworkAdapter -SwitchName "Internal"

# Enable MAC address spoofing (required for cluster guest VMs later)
Get-VMNetworkAdapter -VMName "Node1" | Set-VMNetworkAdapter -MacAddressSpoofing On
```

## Step 4: vTPM (key protector + enable)

```powershell
# Create a local guardian if one doesn't already exist
# (This error is harmless on retry - subsequent commands still execute)
New-HgsGuardian -Name "UntrustedGuardian" -GenerateCertificates -ErrorAction SilentlyContinue

$owner = Get-HgsGuardian -Name "UntrustedGuardian"
$kp = New-HgsKeyProtector -Owner $owner -AllowUntrustedRoot
Set-VMKeyProtector -VMName "Node1" -KeyProtector $kp.RawData
Enable-VMTPM -VMName "Node1"
```

⚠️ If you see `A guardian with the same name 'UntrustedGuardian' already exists`, that's fine — the guardian was created on a previous run. Run the remaining 3 lines (`Get-HgsGuardian`, `New-HgsKeyProtector`, `Set-VMKeyProtector`, `Enable-VMTPM`) to apply key protector to this specific VM.

## Step 5: S2D storage disks

```powershell
# Create 2 large VHDX for Storage Spaces Direct (1 TB each, dynamic)
New-VHD -Path "F:\DCVM\Node1\s2d1.vhdx" -SizeBytes 1024GB
New-VHD -Path "G:\DCVM\Node1\s2d2.vhdx" -SizeBytes 1024GB

# Attach to VM (OS disk was attached at New-VM time; don't re-attach it)
Add-VMHardDiskDrive -VMName "Node1" -Path "F:\DCVM\Node1\s2d1.vhdx"
Add-VMHardDiskDrive -VMName "Node1" -Path "G:\DCVM\Node1\s2d2.vhdx"
```

## Step 6: Mount the ISO and boot

`[Host PowerShell — azureuser]`

```powershell
$isoPath = "C:\Users\azureuser\Downloads\AzureStackHCI.iso"  # Adjust to your path

Add-VMDvdDrive -VMName "Node1" -Path $isoPath

# Set DVD as first boot device
$dvd = Get-VMDvdDrive -VMName "Node1"
Set-VMFirmware -VMName "Node1" -FirstBootDevice $dvd

# Start the VM
Start-VM -Name "Node1"

# Connect to the console for OS install
vmconnect.exe localhost Node1
```

The last line opens a **separate window** — the Hyper-V virtual machine console. From this point your keystrokes go into Node1, not the host:

```
┌─ Node1 on AZLOCAL - Virtual Machine Connection ──────────[_][□][×]┐
│ File  Action  Media  Clipboard  View  Help                        │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│                                                                   │
│           Loading files...                                        │
│           Windows is starting up                                  │
│                                                                   │
│                                                                   │
├───────────────────────────────────────────────────────────────────┤
│ Status: Running                                                   │
└───────────────────────────────────────────────────────────────────┘
```

## Step 7: OS installation (manual via console)

`[Inside Node1 console]` — all interaction in this step happens in the vmconnect window opened in Step 6.

In the VM console:

1. Language → Next
2. Install Azure Stack HCI
3. Accept license
4. **Custom: Install the newer version of Azure Stack HCI only (advanced)**
5. **Choose the 127 GB disk** (do NOT pick the 1024 GB s2d disks)
6. Wait ~10-20 min for install (multiple auto-reboots)
7. At first boot, set the **Administrator password** (14+ chars, all 4 character classes)

   ```
   ┌─ Node1 console ─────────────────────────────────────────────────┐
   │ The user's password must be changed before signing in.          │
   │                                                                 │
   │ New password:    [ ********************            ]            │
   │ Confirm pwd:     [ ********************            ]            │
   │                                                                 │
   │ → 14+ chars, must include upper + lower + digit + special.      │
   │   Write this password down — captured into $cred in Step 8      │
   │   and re-used as the "Local administrator" in Phase 6 Tab 4.    │
   └─────────────────────────────────────────────────────────────────┘
   ```

8. SConfig will start. The screen looks like this:

   ```
   ┌─ Node1 console — SConfig ──────────────────────────────────────┐
   │ Server Configuration                                            │
   │                                                                 │
   │  1) Domain/Workgroup:        Workgroup: WORKGROUP               │
   │  2) Computer name:           WIN-XXXXXXXXXXX                    │
   │  3) Add local administrator                                     │
   │  4) Configure remote management                                 │
   │  5) Windows Update settings                                     │
   │  6) Install updates                                             │
   │  7) Remote desktop                                              │
   │  8) Network settings                                            │
   │  9) Date and time                                               │
   │ 10) Telemetry settings                                          │
   │ 11) Windows activation                                          │
   │ 12) Log off user                                                │
   │ 13) Restart server                                              │
   │ 14) Shut down server                                            │
   │ 15) Exit to command line (PowerShell)                           │
   │                                                                 │
   │ Enter number to select an option: _                             │
   └─────────────────────────────────────────────────────────────────┘
   ```

   Press `15` → exit to command line.
9. Restart SConfig: type `SConfig` and press Enter
10. Press `2` → at the "Enter new computer name" prompt type `Node1` → confirm restart. The VM reboots; SConfig will reappear after login as `Node1`.

## Step 8: From host, connect via PowerShell Direct

`[Host PowerShell — azureuser]` — switch back to the host's own PowerShell window (the one from Steps 1–6, **not** the vmconnect console).

After Node1 reboots and SConfig is up again:

```powershell
$cred = Get-Credential
```

Running this opens a credential dialog on the host:

```
┌─ Windows PowerShell credential request ─────────────────────────┐
│                                                                 │
│ Enter your credentials.                                         │
│                                                                 │
│ User name:  [ Administrator                          ]   ▼     │
│ Password:   [ ********************                   ]         │
│                                                                 │
│                                       [   OK   ]  [ Cancel ]    │
└─────────────────────────────────────────────────────────────────┘
```

- **User name**: just `Administrator` (no domain prefix — Node1 is NOT domain-joined yet; that happens automatically in Phase 6).
- **Password**: the one you set in Step 7.

After the dialog closes, confirm `$cred` holds the expected account before continuing:

```powershell
$cred.UserName     # → Administrator
```

```powershell
# Test connection
Invoke-Command -VMName "Node1" -Credential $cred -ScriptBlock { hostname }
# Expected: Node1
```

If the test returns a credential error, see `references/00-accounts-and-context.md` § "What to do when a credential is rejected".

## Step 9: Capture MAC addresses and rename NICs

`[Host PowerShell — azureuser]` for the MAC capture, then `[Host PowerShell → Node1 via $cred]` for the rename inside the VM.

The 4 NICs inside Node1 are named `Ethernet`, `Ethernet 2`, etc. We rename them to `NIC1-NIC4` matching the Hyper-V names by MAC address:

```powershell
# Get and format each MAC
1..4 | ForEach-Object {
    $i = $_
    $mac = (Get-VMNetworkAdapter -VMName "Node1" -Name "NIC$i").MacAddress
    $macFormatted = $mac.Insert(2,"-").Insert(5,"-").Insert(8,"-").Insert(11,"-").Insert(14,"-")
    Set-Variable -Name "Node1mac$i" -Value $macFormatted -Scope Global
    Write-Host "NIC$i = $macFormatted"
}
```

Now rename adapters inside Node1 (one Invoke-Command per NIC keeps it readable):

```powershell
foreach ($i in 1..4) {
    $macVar = Get-Variable "Node1mac$i" -ValueOnly
    Invoke-Command -VMName "Node1" -Credential $cred -ScriptBlock {
        param($targetMac, $newName)
        Get-NetAdapter -Physical |
            Where-Object {$_.MacAddress -eq $targetMac} |
            Rename-NetAdapter -NewName $newName
    } -ArgumentList $macVar, "NIC$i"
}

# Verify
Invoke-Command -VMName "Node1" -Credential $cred -ScriptBlock {
    Get-NetAdapter | Format-Table Name, MacAddress, Status
}
```

## Step 10: Disable DHCP on all NICs

`[Host PowerShell → Node1 via $cred]`

```powershell
1..4 | ForEach-Object {
    Invoke-Command -VMName "Node1" -Credential $cred -ScriptBlock {
        param($n)
        Set-NetIPInterface -InterfaceAlias "NIC$n" -Dhcp Disabled
    } -ArgumentList $_
}
```

## Step 11: Set NIC1 static IP (initial, will be updated in Phase 5)

`[Host PowerShell → Node1 via $cred]`

```powershell
Invoke-Command -VMName "Node1" -Credential $cred -ScriptBlock {
    New-NetIPAddress -InterfaceAlias "NIC1" `
        -IPAddress "192.168.44.201" -PrefixLength 24 `
        -AddressFamily IPv4 -DefaultGateway "192.168.44.1"

    # Temporary DNS — will change to AD DC in Phase 5
    Set-DnsClientServerAddress -InterfaceAlias "NIC1" -ServerAddresses "8.8.8.8"
}

# Verify internet access
Invoke-Command -VMName "Node1" -Credential $cred -ScriptBlock {
    Test-NetConnection -ComputerName "www.microsoft.com" -Port 443
}
# Expected: TcpTestSucceeded : True
```

## Step 12: Enable Hyper-V role inside Node1

`[Host PowerShell → Node1 via $cred]`

```powershell
Invoke-Command -VMName "Node1" -Credential $cred -ScriptBlock {
    Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
    Install-WindowsFeature -Name Hyper-V -IncludeManagementTools
}
```

Azure Stack HCI ships with Hyper-V largely pre-enabled, so `RestartNeeded` is often `False`.

## How to verify this phase is done

Run this comprehensive check from the host:

```powershell
Invoke-Command -VMName "Node1" -Credential $cred -ScriptBlock {
    Write-Host "=== Hostname ===" -ForegroundColor Cyan
    hostname

    Write-Host "=== NICs ===" -ForegroundColor Cyan
    Get-NetAdapter | Format-Table Name, Status, MacAddress

    Write-Host "=== IP ===" -ForegroundColor Cyan
    Get-NetIPAddress -InterfaceAlias "NIC1" -AddressFamily IPv4 |
        Format-Table InterfaceAlias, IPAddress, PrefixLength

    Write-Host "=== Hyper-V ===" -ForegroundColor Cyan
    Get-WindowsFeature -Name Hyper-V* | Format-Table Name, InstallState
}
```

✅ Hostname = `Node1`
✅ 4 NICs named `NIC1-NIC4`, all `Up`
✅ NIC1 has `192.168.44.201/24`
✅ Hyper-V and Hyper-V-PowerShell `Installed`
✅ `Test-NetConnection www.microsoft.com -Port 443` returns `TcpTestSucceeded : True`

→ Proceed to **Phase 3** (`references/03-azure-prep.md`)
