# Demo script - DC Deployment & AD Operations

Audience: a Senior AD Manager. Deep Windows/AD knowledge, new to AAP. This is the first of a planned two-part engagement - the goal today is to earn credibility on the mechanics (this really does what real AD operations require) and land the credential/permissions story, not to cover everything AAP can do.

## Framing (30 seconds, before you launch anything)

"This isn't a toy. Every step here is a real Install-ADDSForest, a real Move-ADDirectoryServerOperationMasterRole, real DHCP and DFS cmdlets - the exact same operations you run today. What changes is who can run them, what credentials they run with, and whether there's a record of it afterward."

## Walking the workflow

Launch **DC Deployment & AD Operations** and narrate as it progresses - don't just let it run silently.

1. **Provision Network / Provision DC Instances** - "Two Windows Server 2022 boxes, VPC, security group. Notice the AD service ports - Kerberos, LDAP, SMB, replication RPC - are scoped to this VPC only. Nothing AD-specific is exposed to the internet; only WinRM, for AAP itself, and RDP, for you."
2. **Install AD Roles** - AD DS, DNS, DHCP, DFS Namespaces/Replication, RSAT, installed on both boxes before anything else happens.
3. **Promote Forest Root (dc01)** - a real `Install-ADDSForest`. Point out: the safe mode password comes from a custom AAP credential, injected only at run time - it's never visible in the playbook, the job output, or this repo.
4. **Promote Additional DC (dc02)** - dc02 points its DNS at dc01, then joins and promotes. "This is the step people get nervous about automating by hand - timing the reboot, re-establishing the connection, getting the DNS order right. AAP just... waits."
5. **Configure DNS / DHCP / DFS** - "Here's an honest point worth making explicitly: there is no Ansible module for DHCP scopes, DFS namespaces, or FSMO. AAP wraps your existing PowerShell - `Add-DhcpServerv4Scope`, `New-DfsReplicationGroup`, `Move-ADDirectoryServerOperationMasterRole` - with credentials, logging, and access control around it. It's not replacing your AD knowledge, it's operationalizing it."
6. **FSMO Report** - shows current role placement (all 5 on dc01).
7. **Approval node** - the workflow stops here. "This is a manual gate. Someone has to click Approve before the FSMO transfer runs - same as it would in change management today, except now it's enforced by the platform, not by someone remembering to ask."
8. **FSMO Transfer** - uses the role(s) and target DC you picked in the survey at workflow launch; approve, and watch it move.
9. **FSMO Report (again)** - "Same report, same job template, run again - and now PDC Emulator (or whichever role you picked) shows dc02. That's not us telling you it worked. That's AD telling you."

## The credential/permissions conversation (the actual point of this demo)

Pull up **Credentials** and **Teams** in the AAP UI once the workflow finishes, or at any natural pause:

- "Four credentials. One holds the AWS keys. Two are Windows connection credentials - `DC Build Admin` and `DC Operations Admin` - and they happen to resolve to the same domain Administrator account today, but they're two separate objects in AAP. That matters because access to them is granted separately: your platform team gets `DC Build Admin`, your AD team gets `DC Operations Admin`, and revoking one doesn't touch the other."
- "The fourth is a custom credential type we defined - `AD Domain Secrets`. It holds the safe-mode/domain admin password this whole build depends on and hands it to the promotion jobs at run time. It's not in a file, not in git, and you can swap it for a CyberArk or HashiCorp Vault lookup without touching a single playbook."
- "And then Teams: `Platform-Eng` can rebuild and promote these DCs. `AD-Ops` can run DNS, DHCP, DFS, and FSMO changes - day 2 operations - but literally cannot launch the provisioning or promotion templates. That's not a convention your team has to remember to follow. AAP enforces it."

## Anticipated pushback, and honest answers

- **"Why would I automate FSMO transfer at all?"** - You might not run it often, but having it as a reviewed, approved, logged, repeatable job template beats an ad hoc PowerShell session during an actual DR event, when people are stressed and typing commands by hand.
- **"What about seize, not just transfer?"** - Deliberately out of scope here - see `docs/fsmo_notes.md`. Worth a dedicated conversation, not a checkbox on this demo.
- **"Two credentials with the same password feels redundant."** - It is the same secret today; the point is that they're independently revocable/rotatable AAP objects, and access to them is granted to different teams. In a production rollout you'd typically also separate the underlying accounts (e.g. a dedicated service account for day-2 ops instead of the built-in Administrator) - this demo keeps one account for simplicity.

## Next conversation (second half of the engagement)

This demo deliberately didn't cover: GPO management, patching/compliance, user/group lifecycle, cross-forest trusts, or integrating AD changes with a ticketing system (ServiceNow, etc. - see the separate Windows VM provisioning demo for what that integration looks like). That's the natural "part two."
