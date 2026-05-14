# Phase 0: Accounts, Contexts, and "Where Am I?"

> **Why this file exists**: Every other phase assumes you know which machine you are on and which credential variable corresponds to which `Administrator`. The username `Administrator` is reused in three completely independent places — confusing them is the single most common source of "the command ran but nothing changed" or "credentials rejected" symptoms in this deployment.
>
> Open this file in a side window. Refer to it whenever a step says "on the host" or "inside Node1" or "as `$dcCred`" and you are not 100% sure what that means.

## The 3-Administrator map

You will deal with **three independent administrator accounts** spread across three layers. They share a username but they are not the same account.

```
┌──────────────────────────────────────────────────────────────────┐
│  CLOUD LAYER — Azure VM "AZLOCAL" (the host)                     │
│  ├─ User:     azureuser  (or whatever you set when creating VM)  │
│  ├─ Password: set in Azure portal during VM creation             │
│  ├─ Login:    RDP into the Azure VM's public IP                  │
│  └─ Used for: running every PowerShell command in the host       │
│               window. Owns Hyper-V, NAT, switches, $cred,        │
│               $dcCred, $lcmCred variables.                       │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  NESTED LAYER 1 — Node1 VM (Hyper-V guest)                 │  │
│  │  ├─ User:     Administrator  (local; NOT a domain user)    │  │
│  │  ├─ Password: set during Azure Stack HCI install (Phase 2  │  │
│  │  │            Step 7). 14+ chars, all 4 character classes. │  │
│  │  ├─ Login:    `vmconnect.exe localhost Node1` for console; │  │
│  │  │            from host, `Invoke-Command -VMName Node1     │  │
│  │  │            -Credential $cred`                           │  │
│  │  └─ Captured into:  $cred                                  │  │
│  │                                                            │  │
│  │  ⚠ After cluster deploy (Phase 6) this account is renamed  │  │
│  │    for security. Look up the new name in:                  │  │
│  │      Resource group → Cluster → Local built-in user accts. │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  NESTED LAYER 2 — ActiveDirectory VM (Hyper-V guest)       │  │
│  │  ├─ User:     Administrator  (after DC promotion, this is  │  │
│  │  │            the Domain Administrator of azurelocal.com)  │  │
│  │  ├─ Password: set during Windows Server install (Phase 5   │  │
│  │  │            Step A2). 14+ chars recommended.             │  │
│  │  ├─ Login:    `vmconnect.exe localhost ActiveDirectory`    │  │
│  │  │            for console; from host, `Invoke-Command      │  │
│  │  │            -VMName ActiveDirectory -Credential $dcCred` │  │
│  │  └─ Captured into:  $dcCred                                │  │
│  │                                                            │  │
│  │  Also living inside this DC, in AD itself:                 │  │
│  │  ┌─────────────────────────────────────────────────────┐   │  │
│  │  │ DOMAIN USER — lcmuser                               │   │  │
│  │  │ ├─ Lives in: OU=AzureLocal,DC=azurelocal,DC=com     │   │  │
│  │  │ ├─ Created by: New-HciAdObjectsPreCreation          │   │  │
│  │  │ │              (Phase 5 Step C2)                    │   │  │
│  │  │ ├─ Password: 14+ chars, all 4 classes — required by │   │  │
│  │  │ │            Azure Local portal wizard (Phase 6).   │   │  │
│  │  │ ├─ Used by:  cluster deployment wizard to           │   │  │
│  │  │ │            domain-join Node1 automatically        │   │  │
│  │  │ └─ Captured into:  $lcmCred                         │   │  │
│  │  └─────────────────────────────────────────────────────┘   │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

## Credential-variable cheat sheet

When a phase says "run as `$cred`" or "use `$dcCred`", this is what each refers to:

| Variable | Defined in | What it holds | Used to |
|---|---|---|---|
| (none — RDP) | Azure portal at VM creation | `azureuser` / Azure VM password | Log in to the host with RDP, run host PowerShell |
| `$cred` | Phase 2 Step 8 (`Get-Credential`) | Node1 local `Administrator` + Phase 2 password | `Invoke-Command -VMName Node1 -Credential $cred`; Phase 4 Arc bootstrap |
| `$dcCred` | Phase 5 Step B2 (`Get-Credential`) | DC VM `Administrator` (Domain Admin after promo) | `Invoke-Command -VMName ActiveDirectory -Credential $dcCred`; AD object creation |
| `$lcmCred` | Phase 5 Step C2 (`Get-Credential`) | `lcmuser` + 14+ char password | Passed into `New-HciAdObjectsPreCreation`; later typed into Phase 6 Tab 4 |

⚠ **`$lcmCred` is the only credential that maps to a *domain* user.** `$cred` and `$dcCred` are both local accounts on their respective VMs. The portal wizard (Phase 6 Tab 4) expects `lcmuser` **without** any `azurelocal\` or `@azurelocal.com` prefix.

## "Where am I right now?" — quick detection snippet

Whenever you lose track of which window or session you are in, paste this into the prompt:

```powershell
"$(hostname) | $env:USERDOMAIN\$env:USERNAME | PSDirect=$(if ($PSSenderInfo) { $PSSenderInfo.ConnectionInfo.ComputerName } else { 'no' })"
```

Expected outputs by layer:

| You are on… | Output |
|---|---|
| Host Azure VM (host PowerShell window) | `AZLOCAL \| AZLOCAL\azureuser \| PSDirect=no` |
| Inside Node1 console (vmconnect) | `Node1 \| NODE1\Administrator \| PSDirect=no` |
| Inside DC console (vmconnect) | `ActiveDirectory \| AZURELOCAL\Administrator \| PSDirect=no` (after AD promo)<br>or `ACTIVEDIRECTORY\Administrator` (before promo) |
| Inside `Invoke-Command -VMName Node1 …` block | `Node1 \| NODE1\Administrator \| PSDirect=Node1` |
| Inside `Invoke-Command -VMName ActiveDirectory …` block | `ActiveDirectory \| AZURELOCAL\Administrator \| PSDirect=ActiveDirectory` |

If the hostname does not match what the phase says it should, **stop and back out** — you are in the wrong window.

## Visual badge convention used in the reference files

Each command block in `01-…` through `06-…` is prefixed with a one-line badge so you can scan for context shifts at a glance:

- `[Host PowerShell — azureuser]` — host's own PowerShell window, running as the Azure VM admin
- `[Inside Node1 console — Administrator]` — typed directly into the `vmconnect` console window for Node1
- `[Inside DC console — Administrator]` — typed directly into the `vmconnect` console window for the ActiveDirectory VM
- `[Host PowerShell → Node1 via $cred]` — `Invoke-Command -VMName Node1 -Credential $cred { … }` from the host
- `[Host PowerShell → DC via $dcCred]` — `Invoke-Command -VMName ActiveDirectory -Credential $dcCred { … }` from the host
- `[Azure portal — signed in as your Azure user]` — browser action against `portal.azure.com`

Any block without a badge defaults to **`[Host PowerShell — azureuser]`**.

## What to do when a credential is rejected

The error is almost always one of these four causes. Walk down the list in order:

1. **Wrong window.** Run the detection snippet above. If you are in PSDirect into the wrong VM, exit and reconnect.
2. **Wrong account.** Re-read the badge: did the block want `$cred` (Node1) or `$dcCred` (DC)? Typing Node1's password into the DC dialog is the #1 mistake.
3. **Username format.** For `$dcCred` after AD promotion, try in this order: `Administrator`, then `azurelocal\Administrator`, then `Administrator@azurelocal.com`, then `.\Administrator`. PowerShell Direct's cache is sometimes picky.
4. **Switch swap not followed by reboot.** If you just moved a VM between External and Internal switches, `Restart-VM` first — PSDirect's credential cache breaks until reboot.

If all four fail, see `references/troubleshooting.md` § "PowerShell Direct credential failures".
