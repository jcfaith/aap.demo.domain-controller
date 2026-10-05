# AAP setup

## 0. Prerequisites on your control node (not inside AAP)

```
ansible-galaxy collection install -r collections/requirements.yml
ansible-galaxy collection install infra.aap_configuration
ansible-galaxy collection install ansible.platform ansible.controller   # from Automation Hub (console.redhat.com token in ansible.cfg)
```

`ansible.platform` and `ansible.controller` are Red Hat certified collections - they come from Automation Hub, not public Galaxy. `infra.aap_configuration` and those two are only needed for the one-time setup below - they are deliberately not in `collections/requirements.yml`, so they never have to sync onto the AAP execution environment.

## 1. Set environment variables for the setup playbook

Pick one strong password for the domain. It becomes the built-in local `Administrator` password on both instances, the DSRM (safe mode) password, and - after dcpromo - the domain `Administrator` password (see `roles/dc_instances/scripts/aws_userdata` and `docs/architecture.md` for why these are one secret). It is only ever stored in AAP credentials; nothing secret is committed to this repo.

```
export CONTROLLER_HOST=https://aap-aap.apps.<your-cluster>
export CONTROLLER_USERNAME=admin
export CONTROLLER_PASSWORD='...'
export AWS_ACCESS_KEY_ID='...'
export AWS_SECRET_ACCESS_KEY='...'
export DC_ADMIN_PASSWORD='...'
```

## 2. Run setup

From your control node (not inside AAP - this talks to the AAP API directly):

```
ansible-playbook playbooks/setup_demo.yml
```

If you forked this repo, override the project URL: `-e my_project_scm_url=https://github.com/<you>/aap.demo.domain-controller.git`.

This creates the `IT Service Automation` organization, the `aap.demo.domain-controller` project (and syncs it), the `AD Domain Secrets` credential type, the credentials (`AWS`, `DC Build Admin`, `DC Operations Admin`, `AD Domain Secrets`), the `AD Demo Inventory` with its AWS EC2 dynamic source, every job template, the `DC Deployment & AD Operations` workflow, and the `Platform-Eng`/`AD-Ops` teams with their role grants.

## 3. Execution environment

Everything here targets Windows over WinRM and AWS over the `amazon.aws` collection. Check whether **Default execution environment** on your AAP instance already has `pywinrm`, `pyspnego`/`requests-credssp`, and `amazon.aws`'s Python deps (`boto3`/`botocore`) installed:

```
# from a job run against any job template, check the job output for
# "pywinrm" / "boto3" import errors, or inspect the EE image directly
```

If it doesn't, build a custom EE (`ansible-builder`) with those added and point the job templates at it instead of `Default execution environment` in `playbooks/files/config_as_code/controller_templates.yml`, then re-run `setup_demo.yml`.

## 4. Launch

Launch the **DC Deployment & AD Operations** workflow template. It will ask for nothing by default; `domain_name`/`domain_netbios_name` are already set on `DC - 04 Promote Forest Root` (edit that job template's extra vars to change them). The FSMO transfer step has its own survey (which role(s), which target DC) and sits behind a manual approval node.

## 5. Tear down

Run the **DC - Teardown** job template when you're done - it terminates both EC2 instances and removes the VPC. RHDP/demo-lab AWS accounts are usually billed by the hour; don't leave this running overnight.

## Validating the RBAC/credential config

- Log in as a user on the `AD-Ops` team only. Confirm you can launch `DC - 06 Configure DNS` through `DC - 10 FSMO Transfer`, and that `DC - 01` through `DC - 05` and `DC - Teardown` are not launchable.
- `controller_teams_roles.yml`'s exact field names (`aap_teams`, `controller_roles`, its `job_templates`/`role` keys) should be checked against whatever version of `infra.aap_configuration` you installed - role-module schemas have shifted between major versions. If `setup_demo.yml` errors on that file, check `ansible-doc -t role infra.aap_configuration.roles` (or the collection's README) and adjust the keys to match.
