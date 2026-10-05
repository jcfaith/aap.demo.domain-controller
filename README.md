# AAP Domain Controller Deployment Demo

An AAP-driven demo that deploys a two-node Windows Active Directory domain and automates the operations an AD team actually does day to day:

- Provision two Windows Server 2022 EC2 instances
- Install the required Windows/AD roles and features
- Promote the first to a forest-root domain controller, join and promote the second as an additional DC
- Automate DNS (forwarders, records), DHCP (authorization, scope), and DFS (namespace + replication) configuration
- Report on and transfer FSMO roles between the two domain controllers
- Do all of the above under AAP's credential and RBAC model - scoped build vs. operations credentials, a custom AD Domain Secrets credential (no secrets file anywhere in the repo), and two AAP teams with genuinely different access

See `docs/architecture.md` for the full topology and build order, `docs/demo_script.md` for how to actually run and narrate the demo, `docs/fsmo_notes.md` for what is and isn't automated around FSMO, and `docs/aap_setup.md` for how to stand this up in your own AAP instance.

## Quick start

```
ansible-galaxy collection install -r collections/requirements.yml
```

Then follow `docs/aap_setup.md` end to end - it covers setting your own secrets, running the one-time `setup_demo.yml` config-as-code playbook (which creates the organization, project, credentials, templates, workflow and teams), and launching the workflow.

## Repo layout

```
playbooks/        One playbook per workflow stage, plus setup_demo.yml (Day 0 config-as-code) and site_delete.yml (teardown)
roles/             One role per playbook - network, dc_instances, ad_prereqs, dc_promote_root,
                   dc_promote_additional, dns_config, dhcp_config, dfs_config, fsmo_report, fsmo_transfer
docs/              architecture.md, aap_setup.md, demo_script.md, fsmo_notes.md
```

## Cost note

This provisions real EC2 instances. Run the **DC - Teardown** job template when you're done; it removes every AWS resource the demo creates. A **DC - Nightly Teardown** schedule (11 PM Pacific) is created as a safety net - see `docs/aap_setup.md`.
