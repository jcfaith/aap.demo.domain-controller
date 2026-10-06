# Changelog

## 1.1.0 - 2026-10-06

Verified end to end on AAP 2.7: full build, FSMO transfer behind approval, and teardown leaving nothing in AWS.

- Teardown removes instances in any state, every route table, IGW, subnet, security group and VPC; nightly teardown schedule added
- Day-2 jobs connect as the domain-qualified Administrator
- PowerShell steps stop on error instead of reporting false success
- FSMO transfer is idempotent and waits for replication before reporting
- Security group opens AD Web Services (9389) and NTP (123) between DCs
- Instances renamed to dc01/dc02 before promotion; WinRM configured inline at boot
- Moved to microsoft.ad modules; setup creates org, project and survey-driven workflow

## 1.0.0 - 2026-10-05

Initial build: two-DC AWS deployment, AD DS/DNS/DHCP/DFS automation, FSMO report + transfer behind an approval node, scoped build/ops credentials, Platform-Eng/AD-Ops RBAC split.
