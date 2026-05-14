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

- A running **Azure VM with nested virtualization enabled** (`Standard_E32-8s_v5` recommended — 8 vCPUs / 256 GiB RAM; the constrained-core variant of E32s v5 keeps full memory while reducing per-core licensing costs, and was validated end-to-end for this walkthrough)
- The user has **Owner or Contributor + User Access Administrator** on an Azure subscription
- A **Windows Server ISO** (2022 or 2025) for the Domain Controller VM
- **At least 3-4 hours** of available time for the full walkthrough
- The user understands this will incur **Azure costs** (Azure VM hosting + Azure Local service fee after 60-day trial)

> **Two supported tenant plans — pick one in Phase 3.**
> - **Plan A (default)** — the host Azure VM and the Azure Local cluster both live in the same Entra ID tenant. Use this whenever you can. The Microsoft Learn tutorial implicitly assumes this plan.
> - **Plan B (cross-tenant)** — the host Azure VM lives in your primary work tenant (Tenant A) but the cluster is **registered into a different tenant** (Tenant B — e.g., MSDN / Visual Studio Enterprise / partner / personal dev). Choose this when Tenant A's Conditional Access blocks `Connect-AzAccount` / device-code flow from arbitrary VMs, or when you lack `Owner` / billing rights there. Common for engineers in tightly-managed enterprise tenants (including many Microsoft engineers).
>
> The two plans share every command in this skill; only the `Connect-AzAccount` call in Phase 3 differs. The **60-day free trial** of the Azure Local service fee applies to whichever tenant the cluster is registered in. Full plan comparison and the cross-tenant cost-split table are in `references/03-azure-prep.md` § "Choose your tenant strategy"; CA-blocked sign-in workarounds are in `references/troubleshooting.md` § "`Connect-AzAccount` blocked by Conditional Access".

## High-level phase map

This deployment has 5 phases. Each has a dedicated reference file with detailed commands.

The **"Where you act"** column says which machine/identity is in control at that phase — this matters because the same `Administrator` username means a different account in three different places (see `references/00-accounts-and-context.md`).

| Phase | What happens | Where you act | Reference | Approx. time |
|---|---|---|---|---|
| 1 | Host networking (External + Internal switches + NAT) | Host Azure VM (`azureuser`) | `references/01-networking-host.md` | 15 min |
| 2 | Node VM creation, OS install, NIC config, Hyper-V role | Host → console of Node1 → host PSDirect (`$cred`) | `references/02-node-vm.md` | 60-90 min |
| 3 | Azure side: resource providers, RG, role assignments | Host PowerShell + Azure portal browser | `references/03-azure-prep.md` | 15 min |
| 4 | Register Node with Azure Arc | Host PSDirect into Node1 (`$cred`) | `references/04-arc-registration.md` | 15 min |
| 5A | Domain Controller VM + AD DS + DNS | Host → console of DC VM → host PSDirect (`$dcCred`) | `references/05-ad-setup.md` (part 1) | 45 min |
| 5B | OU + LCM user via `AsHciADArtifactsPreCreationTool` | Host PSDirect into DC (`$dcCred`) — defines `$lcmCred` | `references/05-ad-setup.md` (part 2) | 10 min |
| 5C | Azure portal cluster deploy wizard | Azure portal browser (signed in as Azure user) | `references/06-cluster-portal.md` | 30 min input + 2 hour deploy |

**Total: 4-5 hours** (much of it waiting on automated processes).

⚠️ **Phases 2, 4, 5A, 5B all involve switching between the host, a VM console, and PowerShell Direct sessions.** Before running any command, confirm which layer you're on — see `references/00-accounts-and-context.md` for the 3-Administrator map and a one-liner that tells you "am I on host / Node1 / DC?".

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

### Tenant / subscription strategy (Plan A vs Plan B)
- **Plan A — same tenant (default).** Host Azure VM and the Azure Local cluster both live in one tenant. Sign in with `Connect-AzAccount` (no `-TenantId`) in Phase 3. Choose this whenever it works.
- **Plan B — cross-tenant.** Host VM in Tenant A, cluster registered in Tenant B. The two tenants are independent: the bootstrap only needs egress to the public Azure control plane endpoints (`login.microsoftonline.com`, `management.azure.com`, `*.arc.azure.com`, …) and requires **no** trust / federation between the tenants. Sign in with `Connect-AzAccount -TenantId <Tenant-B-guid>` in Phase 3 — the `$tenantId` / `$subscriptionId` captured there flow straight into the Phase 4 Arc bootstrap, so the cluster lands in Tenant B regardless of where the host VM lives.
- **When to fall back from A to B.** Use Plan B if Tenant A's Conditional Access blocks `Connect-AzAccount` from the host VM (device-compliance, sign-in risk, named-location, etc.), if you lack `Owner` / `Contributor + User Access Administrator` in Tenant A, or if you need billing isolation. This is the typical configuration for engineers in tightly-managed enterprise tenants — Microsoft engineers in particular often must use Plan B because corporate CA prevents Plan A.
- **Cost implication of Plan B.** Host VM compute + disks bill against Tenant A's subscription; the Azure Local cluster + Key Vault + storage accounts bill against Tenant B's subscription. The **60-day free trial** of the Azure Local service fee always applies to whichever subscription holds the cluster resource. Full per-subscription cost breakdown is in `references/03-azure-prep.md` § "Cost split between the two tenants".

## Workflow: how to drive this skill

When a user invokes this skill, follow this loop:

1. **Identify which phase they're in.** Ask if unclear. If they say "I just started", begin at Phase 1.
2. **Read the relevant reference file** before issuing commands. Each phase has its own.
3. **Verify prerequisites** for the phase before running its commands. Most phases have a "Before you start" section in their reference.
4. **Run commands one block at a time** and ask the user to share the output. Many steps depend on output values (MAC addresses, NIC names, etc.) being captured into variables.
5. **On errors, consult `references/troubleshooting.md`** before improvising. Most common errors have known fixes there.
6. **Validate before moving to the next phase.** Each reference has a "How to verify this phase is done" checklist.

## Teaching principles (always apply when guiding a user)

This skill is used to *teach*, not just to execute. Whoever drives it — Claude or a human mentor — must follow these three principles on **every step**.

### 1. Before/After verification is mandatory

Never run a state-changing command without bookending it with checks:

- **Before**: run a read-only command that shows the current state, so the user sees the starting point.
- **Action**: run the change.
- **After**: run the same (or equivalent) read-only command, and explicitly compare it to the "before" so the user can see exactly what changed.

Present the comparison plainly — a small before/after table or a "was X → now Y" line. If the "after" state does not match the expected outcome, **stop** and diagnose before continuing. Do not proceed on the assumption that a command worked just because it produced no error.

Example pattern:
```
BEFORE:  Get-VMNetworkAdapter -VMName "ActiveDirectory" | ft Name, SwitchName
         → SwitchName: External
ACTION:  Connect-VMNetworkAdapter -SwitchName "Internal"
AFTER:   Get-VMNetworkAdapter -VMName "ActiveDirectory" | ft Name, SwitchName
         → SwitchName: Internal   ✅ changed as expected
```

Every reference file's "How to verify this phase is done" checklist is the **phase-level** version of this same principle.

### 2. Always surface the relevant documentation URL

When guiding a step, name and link the specific Microsoft Learn page (or section) it comes from. Each reference file lists its source doc at the top — point the user there so they can cross-check against the authoritative source, and so they learn to navigate the official docs themselves. If a step deviates from the official doc (this skill contains several deliberate corrections), say so explicitly and explain why.

### 3. Accept screenshots and links from the user — build collaboratively

This deployment is **half PowerShell, half Azure portal GUI**. For any portal-driven step (Phase 3 role assignment, Phase 6 wizard) or any step where the user is unsure:

- **Invite the user to share a screenshot** of their current screen. Reading their actual screen state is far more reliable than guessing.
- **Accept URLs the user pastes** (e.g., a specific Azure portal blade, a doc page they're looking at) and work from them directly.
- When a screenshot is shared, **read it carefully and confirm what you see** before advising — call out the specific fields, statuses, and values visible, then map them to the expected state. Treat a screenshot as the "before" or "after" evidence for principle 1.
- If a screenshot reveals something unexpected (wrong switch, APIPA address, an error banner), name it immediately and switch to `references/troubleshooting.md`.

The goal is a collaborative build: the user acts and observes, shares what they see, and the skill confirms and guides the next step — never a blind sequence of commands.

### 4. Always show the user where they are and what they're about to see

This deployment crosses **three independent OS layers** (host Azure VM → Node1 nested VM → ActiveDirectory nested VM) and four credential variables (`azureuser` on the host, `$cred`, `$dcCred`, `$lcmCred`). The same username `Administrator` exists in three of those places and means three different accounts. Users get lost easily, so before every interactive step:

- **Surface the current-layer badge.** State explicitly which machine the next command runs on, and which credential variable it uses. Example: `[Host PowerShell — running as azureuser]` or `[Inside Node1 via $cred — Administrator of Node1]`. The shared map in `references/00-accounts-and-context.md` is the canonical reference; point the user there whenever they sound unsure which Administrator is which.
- **Preview what they will see.** Before any GUI prompt (`Get-Credential`, Windows Server installer, SConfig, Azure portal blade, deployment wizard tab), include a short ASCII mockup that shows the dialog's structure and exactly what to type. Users who know what window is about to appear catch wrong-window mistakes (e.g., typing Node1's Administrator password into the DC credential dialog) before they happen.
- **Name the trigger.** Say which command opens that window and what closes it (e.g., "running `Get-Credential` opens the dialog below; click OK to return to PowerShell").
- **After interactive steps, confirm what landed.** If the user types into a dialog, immediately follow with a read-only check that proves the right credential / value was captured (`$cred.UserName`, `Get-VM`, etc.) before continuing.

Example pattern:
```
[Host PowerShell — running as azureuser]

About to capture Node1's local Administrator into $cred. You will see this dialog:

  ┌─────────────────────────────────────────────────┐
  │ Windows PowerShell credential request           │
  │                                                 │
  │ User name: [ Administrator              ]       │
  │ Password:  [ ********************       ]       │
  │                                                 │
  │            [ OK ]  [ Cancel ]                   │
  └─────────────────────────────────────────────────┘

→ User name MUST be just "Administrator" (no domain prefix — Node1 is
  not domain-joined yet). Password is the one set during Phase 2 OS install.
```

Mockups do not need to be pixel-accurate — they just need to communicate "which window, which fields, which value goes where".

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

## Further reading — official Microsoft documentation

This skill is built on top of (and corrects gaps in) the following Microsoft Learn pages. When the official docs change, these URLs remain the source of truth — consult them for the latest verified procedures.

### Main deployment journey

- **[Deploy Azure Local using virtual machines](https://learn.microsoft.com/en-us/azure/azure-local/deploy/deployment-virtual)** — the umbrella tutorial this skill mirrors. Covers Phases 1, 2, and parts of 5.
- **[Azure Local deployment overview](https://learn.microsoft.com/en-us/azure/azure-local/deploy/deployment-introduction)** — bigger-picture context of where virtual deployment fits.

### By phase

| Phase | Official document |
|---|---|
| 3 (Azure prep) | [Register to Arc and assign permissions for deployment](https://learn.microsoft.com/en-us/azure/azure-local/deploy/deployment-arc-register-server-permissions) |
| 4 (Arc registration) | [Register your machines with Azure Arc](https://learn.microsoft.com/en-us/azure/azure-local/deploy/deployment-without-azure-arc-gateway) |
| 5 (AD setup) | [Prepare Active Directory environment](https://learn.microsoft.com/en-us/azure/azure-local/deploy/deployment-prep-active-directory) |
| 6 (Cluster portal) | [Deploy Azure Local using Azure portal](https://learn.microsoft.com/en-us/azure/azure-local/deploy/deploy-via-portal) |

### Supporting topics

- [Azure Local requirements](https://learn.microsoft.com/en-us/azure/azure-local/concepts/system-requirements) — hardware/software prerequisites
- [Download Azure Local 23H2/24H2 software](https://learn.microsoft.com/en-us/azure/azure-local/deploy/download-23h2-software) — how to obtain the ISO
- [Network ATC reference](https://learn.microsoft.com/en-us/azure/azure-local/concepts/network-atc-overview) — how Network ATC manages cluster networking after deployment
- [Storage Spaces Direct overview](https://learn.microsoft.com/en-us/azure-stack/hci/concepts/storage-spaces-direct-overview) — S2D fundamentals
- [Get support for deployment issues](https://learn.microsoft.com/en-us/azure/azure-local/manage/get-support-for-deployment-issues) — support escalation
- [Azure Local pricing](https://azure.microsoft.com/pricing/details/azure-local/) — current cost model

### Post-deployment

- [Create VMs on Azure Local](https://learn.microsoft.com/en-us/azure/azure-local/manage/create-arc-virtual-machines) — deploying workloads after cluster is up
- [Enable AKS on Azure Local](https://learn.microsoft.com/en-us/azure/aks/aksarc/aks-overview) — Kubernetes on the cluster

Each reference file under `references/` also lists the specific Microsoft Learn page it derives from at the top of the file.
