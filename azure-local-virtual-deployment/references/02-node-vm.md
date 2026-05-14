# Phase 2: Node VM Creation + Azure Stack HCI OS Setup

> **Source documentation**:
> - Microsoft Learn — [Deploy Azure Local using virtual machines](https://learn.microsoft.com/en-us/azure/azure-local/deploy/deployment-virtual), sections "Create the virtual host" and "Install the OS on the virtual host VMs".
> - [Download Azure Local software (ISO)](https://learn.microsoft.com/en-us/azure/azure-local/deploy/download-23h2-software) — where to get the OS image.
>
> The procedure below adds VHDX disk-distribution strategy, MAC-based NIC renaming using captured variables, and explicit DNS bootstrapping (which is changed again in Phase 5).

This is the longest phase. You create the Node VM, install Azure Stack HCI OS, rename network adapters by MAC address, set a static IP, and enable the Hyper-V role.

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

## Step 7: OS installation (manual via console)

In the VM console:

1. Language → Next
2. Install Azure Stack HCI
3. Accept license
4. **Custom: Install the newer version of Azure Stack HCI only (advanced)**
5. **Choose the 127 GB disk** (do NOT pick the 1024 GB s2d disks)
6. Wait ~10-20 min for install (multiple auto-reboots)
7. At first boot, set the **Administrator password** (14+ chars, all 4 character classes)
8. SConfig will start. Press `15` → exit to command line
9. Restart SConfig: type `SConfig` and press Enter
10. Press `2` → set computer name to `Node1` → confirm restart

## Step 8: From host, connect via PowerShell Direct

After Node1 reboots and SConfig is up again:

```powershell
# On host
$cred = Get-Credential  # User: Administrator, Password: what you set in step 7

# Test connection
Invoke-Command -VMName "Node1" -Credential $cred -ScriptBlock { hostname }
# Expected: Node1
```

## Step 9: Capture MAC addresses and rename NICs

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

```powershell
1..4 | ForEach-Object {
    Invoke-Command -VMName "Node1" -Credential $cred -ScriptBlock {
        param($n)
        Set-NetIPInterface -InterfaceAlias "NIC$n" -Dhcp Disabled
    } -ArgumentList $_
}
```

## Step 11: Set NIC1 static IP (initial, will be updated in Phase 5)

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
