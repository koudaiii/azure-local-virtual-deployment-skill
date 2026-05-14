# Phase 6: Cluster Deployment via Azure Portal

> **Source documentation**: Microsoft Learn — [Deploy Azure Local using Azure portal](https://learn.microsoft.com/en-us/azure/azure-local/deploy/deploy-via-portal).
>
> Related: [Network reference patterns for Azure Local](https://learn.microsoft.com/en-us/azure/azure-local/plan/cluster-deployment-network-considerations) explains the different "Group all/some traffic" intent patterns referenced in Tab 3.
>
> The procedure below adds the specific values that work for the IP plan established in this skill (192.168.44.0/24) and flags fields most likely to trip up first-time users.
>
> **Teaching reminder**: Apply the three teaching principles from `SKILL.md` — (1) show before/after state for every change, (2) point the user at the source doc above, (3) accept screenshots and links the user shares. This phase is almost entirely GUI — **expect a screenshot for every tab** and confirm the visible field values against the expected values before telling the user to click Next.

The final phase: run the Azure portal's "Deploy Azure Local" wizard, validate, and trigger the actual cluster build. This phase has an **8-tab wizard** followed by a **2-hour automated deployment**.

## Before you start

- Phase 1-5 complete
- The LCM user password is 14+ characters (verify if uncertain)
- You're signed in to the Azure portal with the user that has the roles assigned in Phase 3
- You have ~3 hours: 30 min input + 15 min validation + 1.5-2 hour deploy

## Reference: IP plan for the wizard

| IP | Role |
|---|---|
| 192.168.44.1 | NAT gateway (host) |
| 192.168.44.10 | Domain Controller |
| **192.168.44.50–55** | **Cluster infrastructure (reserved during wizard, 6 IPs)** |
| 192.168.44.201 | Node1 management |

Avoid overlap. The wizard expects a contiguous block of at least 6 IPs not used by any machine.

## Open the wizard

1. Azure portal → search **Azure Local**
2. **Get started** tab → **Deploy Azure Local** tile → **Create instance**

## Tab 1: Basics

| Field | Value |
|---|---|
| Subscription | Same as Phase 3 |
| Resource group | `azurelocalconnected` |
| Instance name | `azlocal-cluster` (3-15 chars, immutable) |
| Region | Same as Phase 3 (e.g., `Japan East`) |
| Cluster options | **Standard** |
| Storage options | **Storage Spaces Direct (S2D)** |
| ID provider | **Active Directory** |

### Add the machine

Click **+ Add machines** → select `Node1` → **Add**.

Wait for Arc extensions to install (5-15 minutes). Status changes from `Not validated` → `Ready`.

Click **Validate selected machines**. Wait for the green checkmark.

### Key Vault

Choose **Create a new Key Vault**:
- Accept the suggested name (or customize)
- Retention: 7-90 days (default 90 OK)
- Purge protection: **Disabled** (easier cleanup for lab)
- If "Grant Key Vault permissions" appears, click it (Phase 3 should have made this unnecessary, but click anyway if shown)

Click **Create**, wait, then **Next: Configuration**.

## Tab 2: Configuration

Select **New configuration** → **Next: Networking**.

## Tab 3: Networking ⚠️ Most error-prone tab

### Storage switch
Select **No switch for storage** (single-node lab doesn't need one).

### Network pattern
Select **Group all traffic** (simplest for single-node).

This creates one network intent named like `Compute_Management_Storage`.

### Intent configuration
| Field | Value |
|---|---|
| Intent name | `default` (or accept suggestion) |
| Traffic types | Compute, Management, Storage (all 3) |
| Network adapter 1 | `NIC1` |
| Network adapter 2 | `NIC2` |
| Network adapter 3 | `NIC3` |
| Network adapter 4 | `NIC4` |

⚠️ **Adding all 4 NICs is recommended**. Azure Local validation may warn if you leave NICs unused.

Storage VLAN IDs default to 711-714. Leave defaults — they're logical-only on a single-node setup.

### IP allocation
| Field | Value |
|---|---|
| Allocation method | **Manual** |
| Starting IP | **192.168.44.50** |
| Ending IP | **192.168.44.55** |
| Subnet mask | **255.255.255.0** |
| Default gateway | **192.168.44.1** |
| DNS server | **192.168.44.10** |

Click **Validate subnet** — should show a green checkmark.

→ **Next: Management**.

## Tab 4: Management

| Field | Value |
|---|---|
| Custom location name | `azlocal-location` (or any name) |
| Witness | **No witness** (single-node) or create a new storage account |
| Domain | **azurelocal.com** |
| OU | **OU=AzureLocal,DC=azurelocal,DC=com** |
| Deployment account: User name | **lcmuser** (no domain prefix!) |
| Deployment account: Password | LCM user password (14+ chars) |
| Local administrator: User name | **Administrator** |
| Local administrator: Password | Node1's local Administrator password from Phase 2 |

### Common errors here

**"Password must be 14-72 chars with all 4 character classes"** under the deployment account → the LCM user's password isn't strong enough. Either fix it (Phase 5 Step C4) or enter a known-good password if you already reset it.

→ **Next: Security**.

## Tab 5: Security

Select **Recommended security settings** → **Next: Advanced**.

This enables BitLocker, Secure Boot, VBS, Credential Guard, HVCI, and Windows Defender Application Control. Default is fine for a lab.

## Tab 6: Advanced

Select **Create workload volumes and required infrastructure volumes (Recommended)**.

For single-node with S2D:
- 1 infrastructure volume (cluster's own use)
- 1+ workload volume (where you'll deploy VMs/AKS)
- Resiliency: Two-way mirror

→ **Next: Tags**.

## Tab 7: Tags

Optional. Skip (or add `Environment=Lab`).

→ **Next: Validation**.

## Tab 8: Validation

Click **Start validation**. Pre-validation resources are created (system resources, Key Vault audit log, etc.), then the actual validation runs (~15 minutes).

Sub-tasks include:
- PrepareEnvironment
- BootstrapEnvironment
- Download content for deployment ← largest time consumer
- InvokeEnvironmentChecker

If validation succeeds, the wizard moves to **Review + create**.

### If validation fails
- Review the error message
- Most errors point to a specific Phase 5 misstep (DC unreachable, OU missing, password too weak, etc.)
- Don't click "Try again" while a task is still running (it can produce inaccurate results)
- See `references/troubleshooting.md`

## Review + create

Final summary screen. Verify:
- Instance name correct
- Region correct
- IP range correct
- Domain correct

Click **Create**.

## Deployment (~1.5-2 hours)

The deployment runs ~30+ sub-steps. Major ones:
- Check and resolve requirements
- Validate environment
- Resolve requirement
- Install OpenSSH Client
- Clean up after update
- Evaluate proxy configuration
- Configure required proxy bypass
- Adjust the number of infrastructure VMs
- Configure DNS if identity provider
- Set up certificates
- Reload certificate
- Full AgentLifecycleManager Deployment ← ~30 min
- Validate network settings for servers
- Configure settings on servers
- Set Azure cloud for AzureLocal
- **Create the cluster** ← Failover Cluster created here
- Configure networking
- Configure Cloud Management
- ... and more

You can close the browser tab — the deployment continues in Azure. Reopen the resource later to check progress at:

> Resource group → `azlocal-cluster` → Deployments

## What you get on success

In the resource group, expect:

| Resource type | Count | Description |
|---|---|---|
| Machine - Azure Arc | 1 | Node1 (existing from Phase 4) |
| **Azure Local** | 1 | The cluster |
| **Arc Resource Bridge** | 1 | Hosts AKS / VM control plane |
| **Infrastructure logical network** | 1 | `<clustername>-InfraLNET` |
| Key Vault | 1 | Created in Tab 1 |
| **Custom location** | 1 | For deploying VMs/AKS later |
| Storage account | 2 | Cloud witness + Key Vault audit |
| **Azure Local storage path** | 1+ | Per workload volume |

## Verify cluster health from Node1

```powershell
Invoke-Command -VMName "Node1" -Credential $cred -ScriptBlock {
    Write-Host "=== Cluster ===" -ForegroundColor Cyan
    Get-Cluster | Format-List Name, Domain, ClusterFunctionalLevel

    Write-Host "=== Node ===" -ForegroundColor Cyan
    Get-ClusterNode | Format-Table Name, State, Type

    Write-Host "=== S2D ===" -ForegroundColor Cyan
    Get-ClusterS2D

    Write-Host "=== Volumes ===" -ForegroundColor Cyan
    Get-Volume | Where-Object FileSystemLabel | Format-Table DriveLetter, FileSystemLabel,
        @{N='SizeGB';E={[math]::Round($_.Size/1GB,1)}},
        @{N='FreeGB';E={[math]::Round($_.SizeRemaining/1GB,1)}}
}
```

⚠️ **The local Administrator account is renamed after deployment** for security. If `$cred` no longer works, find the new admin name at:
> Resource group → Cluster → Settings → Local built-in user accounts

## How to verify this phase is done

✅ Cluster resource exists with status Connected in Azure portal
✅ All ~8 expected resource types are in the resource group
✅ Cluster has at least 1 workload storage path
✅ `Get-Cluster` from Node1 returns the cluster name and domain

## What's next (optional)

After successful deployment:

- Deploy VMs onto the cluster via Azure portal: Cluster → **Virtual machines** → + Create
- Enable AKS on Azure Local: Cluster → **Kubernetes clusters** → + Add
- Enable RDP on Node1 (disabled by default after deploy): see Microsoft Learn "Enable RDP" post-deployment task
- Set up health monitoring: Cluster → **Insights** → Enable

For cleanup, see `references/troubleshooting.md` § Cleanup.
