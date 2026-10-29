# Overview

Promotes a **pinned Chocolatey release manifest** through Dev → Test on Windows hosts (`os_windows`).

The same manifest (Chocolatey agent version + managed package pins) is preflight-checked, discovered, applied (agent first, then packages), and verified. Versions never float on `latest` across environments.

Stages are selected with Ansible **tags** (most stages also use `never`, so they only run when that tag is requested—for example via an AAP job template tag):

| Stage | Tag | Purpose |
|-------|-----|---------|
| Preflight | `preflight` | Chocolatey binary present; SSH session is Administrator; fail closed |
| Discover | `discover` | Report agent + managed package versions; flag drift; list unmanaged installs; publish Artifacts |
| Upgrade agent | `upgrade_agent` | Pin `chocolatey` to `chocolatey_target_version` |
| Upgrade packages | `upgrade_packages` | Install, upgrade, or downgrade each entry in `managed_packages` to the pinned version |
| Verify | `verify` | Assert agent matches manifest; assert managed packages when `upgrade_packages` was also in the run tags |

Optional companion playbook `manage_itsm.yml` (tags `update_itsm_ticket`, `close_itsm_ticket`) is a stub for ITSM status updates in a workflow.

# Layout

| Path | Role |
|------|------|
| `main.yml` | Stage entry playbook (`hosts: os_windows`) |
| `group_vars/all/manifest.yml` | Release pins (agent + packages) — what gets approved and promoted |
| `group_vars/all/main.yml` | Feed URL, serial, unmanaged reporting |
| `manage_itsm.yml` | Optional ITSM ticket update/close stubs for workflow nodes |

# Requirements

- Windows hosts in inventory group `os_windows`, reachable over **SSH** (OpenSSH Server) with an Administrator credential
- Inventory / credential use the SSH connection (`ansible_connection: ssh`), not WinRM
- Chocolatey already installed on targets (preflight fails closed if `choco.exe` is missing)
- Play uses `become: true` (ensure SSH user / become settings are valid for elevated Chocolatey work)
- Collections from `../../collections/requirements.yml`:
  - `chocolatey.chocolatey`
  - `ansible.windows`
- Execution Environment that includes those collections (or install on the controller project)

# AAP setup

Inventories, credentials, and workflow objects stay outside this repo. Suggested wiring:

## Inventories / credentials

- Inventory groups: hosts under `os_windows` (separate Dev / Test inventories, or limits)
- Connection: SSH to Windows OpenSSH Server (`ansible_connection: ssh`)
- Credential: Machine / SSH credential for a Windows Administrator; configure become as required for elevated installs
- Project: this git repo; playbook path `playbooks/server_promotion/main.yml`. Allow the branch to be overwritten

## Job templates

Create **one JT** and set the stage **tag** on each workflow node.

JT settings:

- Inventory: Dev or Test inventory
- Credentials: Windows admin (SSH)
- Ask for limit on launch to target different hosts
- Ask for branch on launch to target a dev branch with packages changes
