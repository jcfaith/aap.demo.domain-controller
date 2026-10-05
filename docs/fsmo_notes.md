# FSMO roles - what's automated here and what isn't

Active Directory has five Flexible Single Master Operation roles. Exactly one DC in the forest/domain holds each at any time:

| Role | Scope | What breaks if it's unavailable |
|---|---|---|
| Schema Master | Forest | Schema extensions (new attributes/classes) |
| Domain Naming Master | Forest | Adding/removing domains from the forest |
| PDC Emulator | Domain | Time sync, password changes, GPO authoring, legacy NT4 BDC emulation |
| RID Master | Domain | Issuing new RID pools (new objects eventually can't be created) |
| Infrastructure Master | Domain | Cross-domain object reference updates |

`dc01` holds all five at forest creation (`playbooks/04_promote_forest_root.yml`). `roles/fsmo_report` queries and reports current placement at any time via `Get-ADForest`/`Get-ADDomain`.

## Transfer vs. seize - this demo only automates transfer

`roles/fsmo_transfer` calls `Move-ADDirectoryServerOperationMasterRole` **without** `-Force`. That's a **graceful transfer**: it requires the current role holder (`dc01`) to be online and reachable, and it hands the role off cleanly.

`-Force` turns the same cmdlet into a **seize** - a disaster-recovery operation you perform when the current holder is gone for good (hardware failure, unrecoverable corruption) and will never come back online. A seize is destructive if the original holder ever does come back online with stale role data, and it's not something that belongs behind a single playbook run without a lot more guardrails (metadata cleanup, verifying the old holder is actually dead, etc.).

This repo deliberately stops at transfer. If a customer wants to see seize/DR automation as a follow-up engagement, that's a separate, more careful conversation - not something to bolt onto this demo.

## Why there's an approval node before the transfer

The workflow pauses for a manual approval right before `DC - 10 FSMO Transfer`. FSMO placement is exactly the kind of change an AD team wants a human to consciously sign off on, even when the automation is mechanically trivial. It also doubles as a live demo of AAP's workflow approval nodes, which is a useful feature on its own regardless of the AD content.
