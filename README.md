# Active Directory Domain Controller Automation with Ansible Automation Platform

This repo builds a working two-server Active Directory domain in AWS and then runs the day-to-day work an AD team does on it, all from **Red Hat Ansible Automation Platform (AAP)**:

- Builds two Windows Server 2022 servers in AWS
- Installs the AD DS, DNS, DHCP and DFS roles
- Creates a new forest on the first server (`dc01`) and promotes the second (`dc02`) as an additional domain controller
- Configures DNS forwarders and records, authorizes DHCP and creates a scope, and sets up a DFS namespace with replication
- Reports which DC holds each FSMO role, waits for a person to approve, transfers the role(s) you picked, and reports again to prove it moved
- Tears the whole thing down again when you're finished, so you only pay for AWS while you're using it

---

## Contents

1. [AAP in five minutes](#1-aap-in-five-minutes)
2. [What you need before you start](#2-what-you-need-before-you-start)
3. [Set up your workstation](#3-set-up-your-workstation)
4. [Load the demo into AAP](#4-load-the-demo-into-aap)
5. [Run the demo](#5-run-the-demo)
6. [Clean up (and avoid AWS charges)](#6-clean-up-and-avoid-aws-charges)
7. [Run it again](#7-run-it-again)
8. [Try the access controls](#8-try-the-access-controls)
9. [Changing the defaults](#9-changing-the-defaults)
10. [Troubleshooting](#10-troubleshooting)
11. [What's in this repo](#11-whats-in-this-repo)

---

## 1. AAP in five minutes

| AAP term | What it is | In this demo |
|---|---|---|
| **Playbook** | A YAML file describing tasks to run (install a feature, run a PowerShell command, create a VPC...) | `playbooks/01_provision_network.yml` through `10_fsmo_transfer.yml` |
| **Project** | A link from AAP to a Git repository holding playbooks. AAP pulls the latest code from it. | `aap.demo.domain-controller`, pointing at this repo |
| **Inventory** | The list of machines AAP can run against | `DC Demo Inventory`, which discovers `dc01`/`dc02` from AWS automatically |
| **Credential** | A stored secret (AWS keys, a Windows password). AAP injects it at run time. Users can *use* it without ever *seeing* it. | `DC Demo AWS`, `DC Build Admin`, `DC Operations Admin`, `AD Domain Secrets` |
| **Job template** | A saved "run this playbook, against this inventory, with these credentials" button | `DC - 01 Provision Network` ... `DC - 10 FSMO Transfer`, `DC - Teardown` |
| **Workflow template** | Several job templates chained together, with approvals and branching | `DC Deployment & AD Operations`, which runs steps 01-10 in order |
| **Survey** | A short form AAP shows when you launch something | Asks which FSMO role(s) to move and to which DC |
| **Approval node** | A pause in a workflow until a person clicks Approve | `Approve FSMO Transfer`, before any FSMO role moves |
| **Execution environment (EE)** | The container image a job runs in, with Ansible and its plugins inside | AAP's built-in `Default execution environment` |
| **Team / role** | Groups of users and what they're allowed to do | `Platform-Eng` can build; `AD-Ops` can only run day-2 operations |

The idea behind the demo: your AD knowledge and your PowerShell stay the same. AAP adds the parts around them, like who's allowed to run what, where the passwords live, approvals, logs of every run, and the ability to rerun any step the same way every time.

---

## 2. What you need before you start

| You need | Notes |
|---|---|
| **An AAP environment, version 2.5 or newer** | Tested on AAP 2.7. You need the URL and an admin username/password. |
| **An AWS account** | Must be able to create VPCs, subnets, security groups, internet gateways and EC2 instances, and read the public Windows AMI parameter from SSM. The demo uses `us-east-1` by default. |
| **A Red Hat account with Automation Hub access** | Comes with an AAP subscription. Needed to download two certified collections for the setup step. |
| **A GitHub account** | So you can fork this repo and have AAP pull from your copy. |
| **A workstation with Python 3.9+ and `ansible-core` 2.16+** | Linux or macOS. This is only used once, to load the demo into AAP. The demo itself runs inside AAP. |

**AWS cost:** the demo runs two `m5.xlarge` Windows instances while it's up. A typical build-demo-teardown session costs a few dollars. Step 6 shows how to remove everything, and a nightly clean-up schedule is included as a safety net.

---

## 3. Set up your workstation

### 3.1 Fork and clone the repo

1. On GitHub, click **Fork** on this repo to make your own copy.
2. Clone your fork:

```bash
git clone https://github.com/<your-github-user>/aap.demo.domain-controller.git
cd aap.demo.domain-controller
```

### 3.2 Install Ansible

```bash
python3 -m pip install --user ansible-core
ansible --version     # should report core 2.16 or newer
```

### 3.3 Connect to Automation Hub

Two of the collections the setup step uses (`ansible.platform` and `ansible.controller`) are Red Hat certified content, so they come from Automation Hub, not public Galaxy.

1. Go to <https://console.redhat.com/ansible/automation-hub/token> and click **Load token**. Copy it.
2. Tell Ansible to use Automation Hub first and public Galaxy second:

```bash
export ANSIBLE_GALAXY_SERVER_LIST=automation_hub,galaxy
export ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_URL=https://console.redhat.com/api/automation-hub/content/published/
export ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_AUTH_URL=https://sso.redhat.com/auth/realms/redhat-external/protocol/openid-connect/token
export ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_TOKEN='<paste your token>'
export ANSIBLE_GALAXY_SERVER_GALAXY_URL=https://galaxy.ansible.com/
```

### 3.4 Install the collections

```bash
ansible-galaxy collection install -r collections/requirements.yml
ansible-galaxy collection install ansible.platform ansible.controller infra.aap_configuration
```

The first line installs what the demo playbooks use. AAP also installs these automatically when it syncs the project. The second line installs what the one-time setup step needs to talk to AAP.

---

## 4. Load the demo into AAP

One playbook, `playbooks/setup_demo.yml`, creates everything in AAP for you. This is "configuration as code": the whole AAP setup lives in `playbooks/files/config_as_code/`, so it's repeatable and reviewable like any other code.

### 4.1 Pick a domain password

Choose one strong password (at least 12 characters, with upper case, lower case, a number and a symbol). It becomes:

- the local `Administrator` password on both new servers,
- the Directory Services Restore Mode (safe mode) password, and
- after promotion, the `DEMOAD\Administrator` domain password.

It's stored only inside AAP credentials. Nothing secret is ever written to this repo. Keep it somewhere safe; you'll need it if you want to RDP into the servers.

### 4.2 Set your connection details

```bash
export CONTROLLER_HOST=https://<your-aap-hostname>
export CONTROLLER_USERNAME=admin
export CONTROLLER_PASSWORD='<your AAP admin password>'
export AWS_ACCESS_KEY_ID='<your AWS access key>'
export AWS_SECRET_ACCESS_KEY='<your AWS secret key>'
export DC_ADMIN_PASSWORD='<the domain password from 4.1>'
```

Use single quotes so your shell doesn't interpret special characters in the passwords.

### 4.3 Run the setup

Point AAP at **your fork** (replace `<your-github-user>`):

```bash
ansible-playbook playbooks/setup_demo.yml \
  -e my_project_scm_url=https://github.com/<your-github-user>/aap.demo.domain-controller.git
```

It takes a few minutes and should finish with `failed=0`. It's safe to run again at any time; it only changes what's different.

### 4.4 Look at what it created

Log in to the AAP web UI. Most of these are under **Automation Execution** in the left menu; Organizations, Users and Teams are under **Access Management**. You'll find:

| Where | What was created |
|---|---|
| Access Management → Organizations | `IT Service Automation` (everything below belongs to it) |
| Projects | `aap.demo.domain-controller`, synced from your fork |
| Inventories | `DC Demo Inventory`, with an AWS source that finds the DCs by tag |
| Infrastructure → Credential Types | `AD Domain Secrets`, a custom type that hands the domain password to the promotion steps |
| Infrastructure → Credentials | `DC Demo AWS`, `DC Build Admin`, `DC Operations Admin`, `AD Domain Secrets` |
| Templates | Job templates `DC - 01` ... `DC - 10`, `DC - Teardown`, and the workflow `DC Deployment & AD Operations` |
| Schedules | `DC - Nightly Teardown` (see step 6) |
| Access Management → Teams | `Platform-Eng` and `AD-Ops` |

---

## 5. Run the demo

### 5.1 Launch the workflow

1. Go to **Automation Execution → Templates**.
2. Click the rocket icon next to **DC Deployment & AD Operations**.
3. The survey asks two questions. The defaults are fine for a first run:
   - **Which DC should receive the FSMO role(s)?** `dc02`
   - **Which FSMO role(s) should move?** `PDCEmulator`
4. Click **Launch**. AAP opens a live view of the workflow.

### 5.2 What happens

| Step | What it does | Roughly |
|---|---|---|
| 01 Provision Network | Creates a VPC, subnet, internet gateway, route table and security group. AD ports are open only inside the VPC. | 1 min |
| 02 Provision DC Instances | Launches `dc01` (10.60.0.10) and `dc02` (10.60.0.11) and waits for WinRM | 5-10 min |
| 03 Install AD Roles | Renames the servers to `dc01`/`dc02`, installs AD DS, DNS, DHCP, DFS and the RSAT tools, rebooting as needed | 10 min |
| 04 Promote Forest Root | `dc01` creates the `ad.demo.local` forest | 10 min |
| 05 Promote Additional DC | `dc02` joins the domain and becomes a second DC | 10 min |
| 06 Configure DNS | Sets forwarders and adds sample A records | 1 min |
| 07 Configure DHCP | Authorizes `dc01` in AD and creates a scope with router/DNS options | 1 min |
| 08 Configure DFS | Creates the `\\ad.demo.local\CorpData` namespace on both DCs and a replication group between them | 2 min |
| 09 FSMO Report (Before) | Shows all five FSMO roles on `dc01` | 1 min |
| **Approve FSMO Transfer** | **The workflow stops here and waits for you** | - |
| 10 FSMO Transfer | Moves the role(s) you picked to the target DC and waits for AD replication | 1 min |
| 09 FSMO Report (After) | Shows the role now on `dc02`, read straight from AD | 1 min |

Click any box in the workflow view to watch that step's output live.

### 5.3 Approve the FSMO transfer

When the workflow reaches the approval:

1. Open **Workflow Approvals** under **Automation Execution** in the left menu. You can also click the paused approval box in the workflow view.
2. Open **Approve FSMO Transfer** and click **Approve**.

The approval waits **1 hour**. If nobody approves in that time, the workflow stops there. The DCs stay up and nothing breaks; you can run `DC - 10 FSMO Transfer` on its own afterwards.

### 5.4 Check the result

Open the last step, **DC - 09 FSMO Report (After)**. You should see:

```
Schema Master:          dc01.ad.demo.local
Domain Naming Master:    dc01.ad.demo.local
PDC Emulator:            dc02.ad.demo.local
RID Master:              dc01.ad.demo.local
Infrastructure Master:   dc01.ad.demo.local
```

To look around yourself, RDP to either DC's public IP (shown in step 02's output, or in the AWS console) as `DEMOAD\Administrator` with your domain password, and open Active Directory Users and Computers, DNS, DHCP or DFS Management.

---

## 6. Clean up (and avoid AWS charges)

When you're done, go to **Templates** and launch **DC - Teardown**. It removes everything the demo created in AWS:

- both DC instances (in any state), waiting until they're fully terminated
- the route table(s), internet gateway, subnet and security group
- the VPC

It only touches resources tagged `Environment: ad-demo` or inside the demo's `addemo` VPC. It's safe to run at any time, even if a build only got halfway or everything is already gone.

**Safety net:** the `DC - Nightly Teardown` schedule runs `DC - Teardown` every night at **11 PM US Pacific**. If you want the environment to survive overnight, turn that schedule off under **Automation Execution → Schedules**, and turn it back on afterwards. To use a different time zone or time, see [section 9](#9-changing-the-defaults).

---

## 7. Run it again

The whole demo is designed to be rebuilt as often as you like:

1. Run **DC - Teardown** (or let the nightly schedule do it).
2. Launch **DC Deployment & AD Operations** again.

Every step checks what already exists before changing anything. That means you can also relaunch the workflow against an environment that's already built, or one where a run stopped halfway. It skips what's done and carries on from there.

Each step is also its own job template, so you can rerun just one: for example, `DC - 10 FSMO Transfer` with target `dc01` to move the PDC Emulator back.

---

## 8. Try the access controls

AAP decides who can run what. The setup created two teams:

| Team | Can do |
|---|---|
| `Platform-Eng` | Build and rebuild: steps 01-05, the full workflow, and teardown |
| `AD-Ops` | Day-2 operations only: steps 06-10 (DNS, DHCP, DFS, FSMO). Can't provision, promote or tear down anything. |

To see it in action:

1. In **Access Management → Users**, create a user (for example `adops-user`).
2. Add them to the **AD-Ops** team.
3. Log in as that user. They can launch `DC - 06` through `DC - 10`, but `DC - 01` to `DC - 05`, `DC - Teardown` and the full workflow aren't available to them.

Also note: nobody, including admins, can read the stored passwords back out of AAP's credentials. Jobs use them; people don't see them.

---

## 9. Changing the defaults

All defaults live in plain YAML files. Change them in your fork, commit, push, then sync the project in AAP (**Projects → aap.demo.domain-controller → Sync**). For anything under `config_as_code/`, also rerun `setup_demo.yml` (step 4.3).

| What | Where |
|---|---|
| AWS region | Change `us-east-1` in all three places: `network_region` in `roles/network/vars/main.yml`, the `dc_region` default in `roles/dc_instances/vars/main.yml`, and the `regions` list in `playbooks/files/config_as_code/controller_inventories.yml` |
| Instance size | `dc_instance_type` in `roles/dc_instances/vars/main.yml` |
| Domain name / NetBIOS name | `domain_name` on `DC - 04` in `controller_templates.yml`, and `my_domain_netbios_name` in `playbooks/setup_demo.yml` |
| DNS forwarders / sample records | `roles/dns_config/vars/main.yml` |
| DHCP scope | `roles/dhcp_config/vars/main.yml` |
| DFS share / namespace names | `roles/dfs_config/vars/main.yml` |
| Nightly teardown time / time zone | `rrule` in `playbooks/files/config_as_code/controller_schedules.yml` |
| Approval timeout | `timeout` (seconds) on the approval node in `controller_templates_workflow.yml` |
| Organization name | `my_organization` in `playbooks/setup_demo.yml` |

---

## 10. Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| `setup_demo.yml` fails right at the start with "required environment variables are missing" | One of the `export` lines in step 4.2 wasn't run in this terminal. Rerun them, then rerun setup. |
| `ansible-galaxy` can't find `ansible.platform` or `ansible.controller` | The Automation Hub settings in step 3.3 aren't set in this terminal, or the token has expired. Load a fresh token. |
| `setup_demo.yml` fails with "returned 2 items, expected 1" | Your AAP already has an object with the same name in another organization (often a credential called `AWS` or similar). Credential names can be changed with the `my_*` variables at the top of `setup_demo.yml`; other names are in `playbooks/files/config_as_code/`. |
| Project sync fails | Check the project's URL points at your fork and that the fork is public (or add a Source Control credential to the project for a private repo). |
| Step 02 fails waiting for port 5986 | The instances can't be reached from AAP. Check the AWS account allows public IPs and inbound 5986, and that you have enough EC2 vCPU quota for two `m5.xlarge`. Run `DC - Teardown`, then launch again. |
| Step 04 says `safe_mode_password is not set` | The `AD Domain Secrets` credential is missing from `DC - 04`. Rerun `setup_demo.yml`. |
| A job fails partway through | Open the failed step in the workflow view and read the red task output. After fixing the cause, relaunch the workflow. It skips what's already done. |
| The approval expired | Launch `DC - 10 FSMO Transfer` and then `DC - 09 FSMO Report` on their own. |
| Anything else | Run `DC - Teardown` and start a fresh build. A clean rebuild takes about an hour. |

---

## 11. What's in this repo

```
playbooks/
  01_provision_network.yml ... 10_fsmo_transfer.yml   one playbook per workflow step
  site_delete.yml                                       teardown
  setup_demo.yml                                        one-time AAP setup (config as code)
  files/config_as_code/                                 everything setup_demo.yml creates in AAP
roles/                                                  the real work, one role per step
  network, dc_instances, ad_prereqs, dc_promote_root, dc_promote_additional,
  dns_config, dhcp_config, dfs_config, fsmo_report, fsmo_transfer
collections/requirements.yml                            collections AAP installs on project sync
docs/
  architecture.md     network layout, build order, credential model
  demo_script.md      how to present the demo, talking points, likely questions
  fsmo_notes.md       what's automated around FSMO and what deliberately isn't (seizing)
  aap_setup.md        extra setup notes: execution environments, RBAC details
```

Where no dedicated Ansible module exists (DHCP scopes, DFS, FSMO), the roles run the same PowerShell cmdlets an AD admin would use (`Add-DhcpServerv4Scope`, `New-DfsnRoot`, `Move-ADDirectoryServerOperationMasterRole`...). Each one checks the current state first, so it only changes what needs changing.
