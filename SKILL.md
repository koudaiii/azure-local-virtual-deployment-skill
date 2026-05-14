---
name: azure-local-virtual-deployment
description: Step-by-step guide to deploy a single-node Azure Local (formerly Azure Stack HCI) cluster on a nested virtualization environment running inside an Azure VM. Use this skill whenever the user wants to build an Azure Local lab, follow Microsoft's "Deploy Azure Local using virtual machines" documentation, set up a learning/testing Azure Local environment, deploy Azure Stack HCI on Azure VMs, configure nested Hyper-V for Azure Local, or work through any part of the Azure Local virtual deployment journey (host networking, node VM creation, OS installation, Azure Arc registration, Active Directory preparation, or cluster deployment via Azure portal). Also triggers on questions about Azure Local prerequisites, troubleshooting nested virtualization setups, or pitfalls encountered when following the Microsoft Learn `deployment-virtual` tutorial.
---

# Azure Local Virtual Deployment (Lab Environment)

A practical, battle-tested walkthrough for deploying a single-node **Azure Local** (formerly Azure Stack HCI) cluster in a **nested virtualization** environment, typically running on a single large Azure VM. This skill captures both the official Microsoft Learn flow and the **real corrections** needed to make it work end-to-end.

## When to use this skill

- The user wants to follow Microsoft's [Deploy Azure Local using VMs](https://learn.microsoft.com/en-us/azure/azure-local/deploy/deployment-virtual) tutorial
- The user is building an Azure Local lab for learning, certification prep, or proof-of-concept
- The user encounters errors in the official tutorial and needs corrected commands
- The user wants the complete journey from a fresh Azure VM to a working Azure Local cluster registered with Azure Arc

This skill is **NOT** for production Azure Local deployment on physical hardware. For production, see Microsoft's main deployment overview.

## Prerequisites

Before starting, confirm:

- A running **Azure VM with nested virtualization enabled** (Standard_E16s_v5 or larger recommended — needs >= 64 GB RAM, 16+ vCPUs)
- The user has **Owner or Contributor + User Access Administrator** on an Azure subscription
- A **Windows Server ISO** (2022 or 2025) for the Domain Controller VM
- **At least 3-4 hours** of available time for the full walkthrough
- The user understands this will incur **Azure costs** (Azure VM hosting + Azure Local service fee after 60-day trial)

## High-level phase map

This deployment has 5 phases. Each has a dedicated reference file with detailed commands.

| Phase | What happens | Reference | Approx. time |
|---|---|---|---|
| 1 | Host networking (External + Internal switches + NAT) | `references/01-networking-host.md` | 15 min |
| 2 | Node VM creation, OS install, NIC config, Hyper-V role | `references/02-node-vm.md` | 60-90 min |
| 3 | Azure side: resource providers, RG, role assignments | `references/03-azure-prep.md` | 15 min |
| 4 | Register Node with Azure Arc | `references/04-arc-registration.md` | 15 min |
| 5A | Domain Controller VM + AD DS + DNS | `references/05-ad-setup.md` (part 1) | 45 min |
| 5B | OU + LCM user via `AsHciADArtifactsPreCreationTool` | `references/05-ad-setup.md` (part 2) | 10 min |
| 5C | Azure portal cluster deploy wizard | `references/06-cluster-portal.md` | 30 min input + 2 hour deploy |

**Total: 4-5 hours** (much of it waiting on automated processes).

For errors and gotchas, always consult `references/troubleshooting.md`.

## Critical principles (always follow)

These are the **non-negotiable rules** that prevent the most common failures:

### Networking
- The **Internal switch with NAT** is the network the cluster uses. The External switch is only for the host's own internet access.
- **All cluster VMs (Node, DC) connect to the Internal switch**, never External.
- Use a clean private IP range (e.g., `192.168.44.0/24`) with these reservations:
  - `.1` — NAT gateway (host)
  - `.10` — Domain Controller
  - `.50–.55` — Cluster infrastructure (6 IPs reserved during portal wizard)
  - `.201` — Node1 management IP

### Active Directory
- **NEVER join the Azure Local node to AD before cluster deployment.** The cluster deployment process domain-joins it automatically.
- The DC's DNS must include a forwarder (e.g., `8.8.8.8`) so the DC itself can resolve external names.
- The Node's DNS must point at the DC (`192.168.44.10`), NOT at `8.8.8.8` — switch this after AD is up.

### Passwords
- The LCM (Lifecycle Manager) deployment user requires **14+ characters with all 4 character classes** (upper/lower/digit/special). AD's default policy allows shorter passwords, so a password that passes `New-HciAdObjectsPreCreation` may still fail in the Azure portal wizard. Always use 14+ characters.

### PowerShell Direct
- After changing a VM's virtual switch, **restart the VM** before relying on `Invoke-Command -VMName`. The PowerShell Direct credential cache can break otherwise.

### vSwitch creation
- For an **External** switch, do **NOT** pass `-SwitchType External`. Use only `-NetAdapterName <NIC>`. The `-SwitchType` parameter only accepts `Internal` or `Private`.

## Workflow: how to drive this skill

When a user invokes this skill, follow this loop:

1. **Identify which phase they're in.** Ask if unclear. If they say "I just started", begin at Phase 1.
2. **Read the relevant reference file** before issuing commands. Each phase has its own.
3. **Verify prerequisites** for the phase before running its commands. Most phases have a "Before you start" section in their reference.
4. **Run commands one block at a time** and ask the user to share the output. Many steps depend on output values (MAC addresses, NIC names, etc.) being captured into variables.
5. **On errors, consult `references/troubleshooting.md`** before improvising. Most common errors have known fixes there.
6. **Validate before moving to the next phase.** Each reference has a "How to verify this phase is done" checklist.

## Cost awareness

Communicate these to the user before they commit to deployment:

- The **Azure VM hosting Azure Local** is the largest cost (~$300-800/month for an E16s_v5).
- **Azure Local service fee**: ~$10 per physical core per month, **free for the first 60 days** after registration.
- Stopping (deallocating) the host Azure VM stops compute charges. To stop **Azure Local** charges, you must **delete the Azure Local cluster resource** — shutting down the Node VM is not enough.
- See the official [Azure Local pricing page](https://azure.microsoft.com/pricing/details/azure-local/) for current rates.

## Verification at the end

A successful deployment results in the following resources in the resource group:

- `Microsoft.HybridCompute/machines` — Node1 (Arc-enabled)
- `Microsoft.AzureStackHCI/clusters` — the cluster itself
- `Microsoft.ResourceConnector/appliances` — Arc Resource Bridge
- `Microsoft.AzureStackHCI/logicalnetworks` — `<clustername>-InfraLNET`
- `Microsoft.KeyVault/vaults` — the Key Vault
- `Microsoft.ExtendedLocation/customLocations` — custom location for VM/AKS deployment
- `Microsoft.Storage/storageAccounts` (x2) — cloud witness + Key Vault audit log
- `Microsoft.AzureStackHCI/storagepaths` — at least one workload storage path

If all of these exist and the cluster shows as Connected in Azure portal, deployment is complete.

## Cleanup (when finished with the lab)

To stop all Azure Local charges:

1. Delete the Azure Local cluster resource via Azure portal (or delete the entire resource group).
2. The Node OS becomes unusable after cluster deletion — destroy the Node VM in Hyper-V.
3. Stop or delete the host Azure VM to stop compute charges.

A more detailed cleanup procedure is in `references/troubleshooting.md` under "Cleanup".
