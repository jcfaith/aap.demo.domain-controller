# Extra setup notes

The step-by-step setup is in the [README](../README.md). This page covers the details behind it.

## Why the setup collections aren't in `collections/requirements.yml`

`ansible.platform`, `ansible.controller` and `infra.aap_configuration` are only used by `setup_demo.yml`, which runs once from your workstation and talks to the AAP API. Keeping them out of `collections/requirements.yml` means AAP never has to download them when it syncs the project, and the project sync doesn't need Automation Hub credentials.

## Execution environment

The job templates use AAP's built-in **Default execution environment** (`ee-supported`). On AAP 2.5 and newer it already includes everything the demo needs: `ansible.windows`, `microsoft.ad`, `amazon.aws`, `pywinrm` with CredSSP support, and `boto3`. Collections listed in `collections/requirements.yml` are installed on top at project sync time.

If your AAP uses a different default EE and jobs fail with `pywinrm`, `credssp` or `boto3` import errors, build a custom EE with `ansible-builder` that adds those Python packages. Then set `execution_environment` on each template in `playbooks/files/config_as_code/controller_templates.yml` to its name and rerun `setup_demo.yml`.

## Credentials and the domain password

One password (`DC_ADMIN_PASSWORD`) is used in three places, because Windows makes them the same account:

- It's set on the built-in local `Administrator` account at first boot (`roles/dc_instances/scripts/aws_userdata`).
- Promoting `dc01` turns that local account into the domain `Administrator`, so it's also the domain admin password.
- It's used as the DSRM (safe mode) password during promotion.

AAP holds it in three credentials:

| Credential | Used by | Logs in as |
|---|---|---|
| `DC Build Admin` | 02-05 | `Administrator` (local, before the domain exists) |
| `DC Operations Admin` | 06-10 | `DEMOAD\Administrator` (domain-qualified, so steps that reach the other DC authenticate correctly) |
| `AD Domain Secrets` | 04, 05 | not a login: hands `safe_mode_password` and `domain_admin_password` to the promotion steps as variables |

The build and operations credentials hold the same secret today but are separate AAP objects, so access to each can be granted to a different team and revoked or rotated independently.

## Teams and permissions

`playbooks/files/config_as_code/controller_teams_roles.yml` grants `Platform-Eng` admin on templates 01-05, the workflow and teardown, and `AD-Ops` execute on templates 06-10. The field names (`aap_teams`, `controller_roles`) match `infra.aap_configuration` 4.x. If you use a different major version and `setup_demo.yml` errors on that file, check the collection's README for the current names.

## Rerunning setup

`setup_demo.yml` is safe to rerun. It updates anything that differs from the files in `config_as_code/` and leaves everything else alone. Rerun it after changing anything in that folder, or to reset a template someone edited by hand in the UI.
