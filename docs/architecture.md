# Architecture

## Topology

```
                         AWS VPC (10.60.0.0/24)
                    ┌───────────────────────────────┐
                    │                                │
   AAP Controller   │   dc01 (10.60.0.10)            │
   (WinRM 5986) ─────────► Forest root DC             │
                    │     DNS, DHCP (authoritative)  │
                    │     FSMO: all 5 roles at build  │
                    │                                │
                    │   dc02 (10.60.0.11)             │
   AAP Controller ──────► Additional DC               │
                    │     DNS, DFS replication peer  │
                    │     FSMO target after transfer │
                    │                                │
                    └───────────────────────────────┘
```

Both instances are public (`assign_public_ip: true`) so AAP, which runs outside this VPC, can reach them over WinRM. AD service ports (Kerberos, LDAP, SMB, dynamic RPC, etc.) are scoped to the VPC CIDR only - never exposed to the internet. See `roles/network` for the exact rule set.

## Build order

```
01 Provision Network
  -> 02 Provision DC Instances (dc01, dc02)
  -> 03 Install AD DS / DNS / DHCP / DFS roles (both hosts)
  -> 04 Promote Forest Root (dc01 creates the forest)
  -> 05 Promote Additional DC (dc02 joins, becomes a second DC)
  -> 06 Configure DNS (forwarders + sample records)
  -> 07 Configure DHCP (dc01 only - see docs/aap_setup.md)
  -> 08 Configure DFS (namespace + replication group)
  -> 09 FSMO Report (who holds what, before)
  -> [Approval: "Approve FSMO Transfer"]
  -> 10 FSMO Transfer (move role(s) to dc02)
  -> 09 FSMO Report (who holds what, after - proves the move happened)
```

One AAP workflow template runs this whole sequence end to end. Each step is also its own job template, so any single stage can be re-run on its own during troubleshooting.

## Credential model

| Credential | Type | Used by | Identity |
|---|---|---|---|
| `AWS` | Amazon Web Services | 01, 02, Teardown | n/a |
| `DC Build Admin` | Machine | 02-05 | local `Administrator` on each EC2 instance |
| `DC Operations Admin` | Machine | 06-10 | the domain `Administrator` account (same password as above, but a distinct AAP credential object) |
| `AD Domain Secrets` | AD Domain Secrets (custom type) | 04, 05 | injects `safe_mode_password` and `domain_admin_password` as extra vars at run time |

See `docs/fsmo_notes.md` and `docs/aap_setup.md` for why the build/ops split is real (scoping, not cosmetic) and how the secrets are generated and never committed to git.

## RBAC model

Two AAP teams, two different levels of access to the same project:

- **Platform-Eng** - Admin on templates 01-05 and the workflow itself. Can rebuild the environment from scratch.
- **AD-Ops** - Execute-only on templates 06-10. Can run day-2 AD operations (DNS/DHCP/DFS/FSMO) but cannot provision, re-promote, or tear down anything.

This is the direct answer to "how does AAP handle credentials/permissions" - the AD team gets exactly enough access to do their job, nothing more, enforced by AAP rather than by convention.
