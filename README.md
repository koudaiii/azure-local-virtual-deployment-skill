# azure-local-virtual-deployment-skill

[日本語版 README はこちら / Japanese README](./README_ja.md)

A skill that provides an end-to-end, battle-tested walkthrough for deploying a single-node **Azure Local** (formerly Azure Stack HCI) cluster in a **nested virtualization** environment on an Azure VM — for lab, learning, and PoC use.

This skill is based on Microsoft Learn's [Deploy Azure Local using virtual machines](https://learn.microsoft.com/en-us/azure/azure-local/deploy/deployment-virtual) tutorial, with **corrected commands** for the real-world pitfalls you hit when you actually run it end-to-end (command typos in the docs, DNS/NAT gotchas, password policy mismatches, and so on).

> ⚠️ This skill targets **lab / learning / PoC** scenarios only. Do not use it for production deployments on physical hardware.

## What this skill does

When invoked from AI, it walks you through the Azure Local lab build in **6 phases**, interactively, one command block at a time.

| Phase | What happens | Approx. time |
|---|---|---|
| 1 | Host networking (External + Internal vSwitches + NAT) | 15 min |
| 2 | Node VM creation, Azure Stack HCI OS install, NIC config | 60-90 min |
| 3 | Azure side: resource providers, RG, role assignments | 15 min |
| 4 | Register Node with Azure Arc | 15 min |
| 5 | Domain Controller VM + AD + LCM user | 60 min |
| 6 | Cluster deployment via Azure portal | 30 min input + ~2 hour deploy |

**Total: 4-5 hours** (much of it waiting on automated processes).

## Prerequisites

- An Azure VM with **nested virtualization enabled** (`Standard_E32-8s_v5` recommended — 8 vCPUs / 256 GiB RAM, constrained-core SKU that gives plenty of memory for the nested DC + Node VMs while keeping per-core licensing costs down)
- **Owner**, or **Contributor + User Access Administrator**, on an Azure subscription
- A **Windows Server 2022 or 2025 ISO** (for the Domain Controller VM)
- An **Azure Stack HCI 24H2 ISO** (for the Node VM)
- A contiguous **3-5 hours** of working time
- Awareness of **Azure costs** (host VM compute + Azure Local service fee after the 60-day free trial)

## Installation

Copy the skill into your skills directory under `~/[.claude|.copilot|.codex]/skills/`.

```bash
# Clone this repo
git clone https://github.com/koudaiii/azure-local-virtual-deployment-skill.git
cd azure-local-virtual-deployment-skill

# ln -s "$PWD" ~/[.copilot|.claude|.codex]/skills/azure-local-virtual-deployment
```

Restart and the skill will be picked up.

## Usage

Inside a app session, any of the following prompts will trigger the skill:

- "I want to build an Azure Local lab"
- "Help me try Azure Stack HCI on an Azure VM"
- "I got stuck following Microsoft's `deployment-virtual` tutorial"
- "Walk me through deploying Azure Local on nested virtualization"

The skill first asks which phase you're in, then reads the matching `references/0X-*.md` and hands you commands one block at a time. The expected workflow is to share command output back with app as you go, so it can verify each step before moving on.

When errors come up, the skill consults `references/troubleshooting.md` — known pitfalls and their fixes are documented there.

## Repository layout

```
.
├── README.md                                  ← this file (English)
├── README_ja.md                               ← Japanese version
├── SKILL.md                                   ← skill entry point (principles + workflow)
└── azure-local-virtual-deployment/
    └── references/
        ├── 01-networking-host.md             ← Phase 1: host networking
        ├── 02-node-vm.md                     ← Phase 2: Node VM
        ├── 03-azure-prep.md                  ← Phase 3: Azure prep
        ├── 04-arc-registration.md            ← Phase 4: Arc registration
        ├── 05-ad-setup.md                    ← Phase 5: AD setup
        ├── 06-cluster-portal.md              ← Phase 6: cluster deploy
        └── troubleshooting.md                ← cross-phase troubleshooting
```

## Examples of pitfalls this skill fixes

Following the public docs verbatim, you tend to hit these. The skill ships corrected commands for each.

- `New-VMSwitch -SwitchType External` does not actually work — `-SwitchType` only accepts `Internal` or `Private`. Use `-NetAdapterName` instead.
- The LCM user password can satisfy AD's default policy (7+ chars) and still be rejected by the portal wizard, which requires **14+ chars across all 4 character classes**.
- Joining the Node to AD **before** cluster deployment makes deployment fail — the cluster wizard domain-joins it automatically.
- Swapping a VM's virtual switch breaks PowerShell Direct's credential cache until you restart the VM.
- `New-NetNat` allows **only one NAT instance per host** — pre-existing ones must be removed first.

See "Critical principles" in `SKILL.md` and `references/troubleshooting.md` for the full list.

## Cleanup

End-of-lab cleanup steps live in `references/troubleshooting.md` under "Cleanup procedure". Note: to stop the **Azure Local service charges**, you must **delete the cluster resource** — shutting down the Node VM is not enough.

## License

MIT License

## Disclaimer

This skill is not an official Microsoft resource. It captures personal lab notes from running through the Microsoft Learn tutorial end-to-end, and may become out of date as Azure Local and its documentation evolve. It is not intended for production use.
