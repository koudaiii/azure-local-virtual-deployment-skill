# Phase 1: Host Networking (External Switch + Internal Switch + NAT)

> **Source documentation**: Microsoft Learn — [Deploy Azure Local using virtual machines](https://learn.microsoft.com/en-us/azure/azure-local/deploy/deployment-virtual), sections "Set up an external virtual switch" and "Deploy with internal virtual switch and NAT enabled".
>
> The procedure below corrects a known issue in the public doc (the `-SwitchType External` parameter error) and adds explicit verification steps.
>
> **Teaching reminder**: Apply the four teaching principles from `SKILL.md` — (1) show before/after state for every change, (2) point the user at the source doc above, (3) accept screenshots and links the user shares, (4) state which layer/credential each command runs as and preview any dialog before it appears. See `references/00-accounts-and-context.md` for the layer badges and credential map.

> **Context for this entire phase**: `[Host PowerShell — azureuser]`. Every command in this file runs in the **host Azure VM's own PowerShell window**, signed in as `azureuser` over RDP. No nested VMs exist yet, so there is nothing else this phase can act on.

This phase prepares the **Hyper-V virtual switches** on the host Azure VM. Two switches are needed: one for the host's own internet access, and one (with NAT) for the cluster VMs.

## What you build

```
Host Azure VM (e.g., "AZLOCAL")
├─ External Switch  ← attached to the host's NIC, used by host only
└─ Internal Switch  ← host-only software switch (NAT gateway lives here)
    └─ NAT: 192.168.44.0/24 (gateway = 192.168.44.1 on the host)
```

The Internal switch is what cluster VMs connect to. The host has a virtual NIC on this switch with IP `192.168.44.1`, acting as the NAT gateway.

## Before you start

Confirm the host Azure VM has nested virtualization enabled and the Hyper-V role installed. Verify with:

```powershell
# Should report Hyper-V is installed
Get-WindowsFeature -Name Hyper-V | Format-Table Name, InstallState

# Should show physical/host NICs
Get-NetAdapter | Format-Table Name, InterfaceDescription, Status, LinkSpeed
```

You should see at least one physical NIC (the host's own connection). On Azure VMs this is usually called `Ethernet` with description `Microsoft Hyper-V Network Adapter`. Expected shape of the output:

```
Name      InterfaceDescription                  Status  LinkSpeed
----      --------------------                  ------  ---------
Ethernet  Microsoft Hyper-V Network Adapter     Up      50 Gbps
```

## Step 1: Create the External switch

⚠️ **Common mistake**: the Microsoft Learn doc shows `-SwitchType External` but this parameter **only accepts `Internal` or `Private`**. For an External switch, omit `-SwitchType` entirely — just supply `-NetAdapterName`:

```powershell
# CORRECT: no -SwitchType when using -NetAdapterName
New-VMSwitch -Name "External" -NetAdapterName "Ethernet" -AllowManagementOS $true
```

Replace `"Ethernet"` with the actual physical NIC name from `Get-NetAdapter`.

> Why this works: when `-NetAdapterName` is specified, Hyper-V automatically infers an External switch. The `-SwitchType` parameter is only used to choose between `Internal` and `Private` (both of which have no physical NIC binding).

## Step 2: Create the Internal switch

```powershell
New-VMSwitch -Name "Internal" -SwitchType Internal
```

This creates a host-only software switch. The host automatically gets a virtual NIC named `vEthernet (Internal)`.

## Step 3: Assign the host's NAT gateway IP

```powershell
New-NetIPAddress -IPAddress 192.168.44.1 -PrefixLength 24 -InterfaceAlias "vEthernet (Internal)"
```

This makes the host listen on `192.168.44.1` for all `192.168.44.0/24` traffic from the Internal switch.

## Step 4: Create the NAT network

```powershell
# Verify no existing NetNat (only one per host is allowed!)
Get-NetNat

# Create the NAT
New-NetNat -Name "HCINAT" -InternalIPInterfaceAddressPrefix 192.168.44.0/24
```

⚠️ **Critical**: Windows allows **only one `NetNat`** per host. If `Get-NetNat` returns anything, you must `Remove-NetNat` it first (or pick a different IP range to avoid conflict).

## Step 5: Verify

```powershell
# Both switches should exist
Get-VMSwitch | Format-Table Name, SwitchType, NetAdapterInterfaceDescription

# Host should have 192.168.44.1
Get-NetIPAddress -InterfaceAlias "vEthernet (Internal)" -AddressFamily IPv4

# NAT should be active
Get-NetNat
```

Expected output (rough shape):

```
Name      SwitchType  NetAdapterInterfaceDescription
----      ----------  ------------------------------
External  External    Microsoft Hyper-V Network Adapter
Internal  Internal

IPAddress      InterfaceAlias        PrefixLength
---------      --------------        ------------
192.168.44.1   vEthernet (Internal)  24

Name    InternalIPInterfaceAddressPrefix
----    --------------------------------
HCINAT  192.168.44.0/24
```

If your output differs (missing switch, wrong IP, NetNat absent), do not move on — re-run the step that built that artifact.

## Common pitfalls

### "A guardian with the same name already exists"
Unrelated to Phase 1, but appears later in Phase 2. Track this as a Phase 2 issue.

### Host loses connectivity after creating External switch
If the host RDP session drops, the External switch creation succeeded but reassigned the host's IP. Wait 30-60 seconds for Azure VM to restore network. This is normal.

### NAT range conflicts with Azure VNet
If the host VM's Azure VNet uses `192.168.0.0/16`, then `192.168.44.0/24` may conflict. Use a different range like `172.20.0.0/24` and adjust ALL subsequent IP references accordingly.

## How to verify this phase is done

✅ `Get-VMSwitch` returns both `External` and `Internal`
✅ `Get-NetNat` returns one NAT entry
✅ Host can still RDP from outside

→ Proceed to **Phase 2** (`references/02-node-vm.md`)
