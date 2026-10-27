# Overview

Promotes a **pinned Chocolatey release manifest** through Dev → Test on Windows hosts.

The same manifest (Chocolatey agent version + managed package pins) is discovered, preflight-checked, applied (agent first, then packages), and verified. Versions never float on `latest` across environments.

Stages are selected with the `promotion_stage` extra var and/or Ansible tags:

| Stage | `promotion_stage` / tag | Purpose |
|-------|-------------------------|---------|
| Discover | `discover` | Report agent + managed package versions; flag drift; publish Artifacts |
| Preflight | `preflight` | Choco present, admin OK, feed reachable; pins resolvable; fail closed |
| Upgrade agent | `upgrade_agent` | Pin `chocolatey` to `chocolatey_target_version` |
| Upgrade packages | `upgrade_packages` | Install/upgrade each entry in `managed_packages` |
| Verify | `verify` | Assert agent + packages match the manifest |

# Layout

| Path | Role |
|------|------|
| `main.yml` | Stage entry playbook |
| `vars/manifest.yml` | Release pins (agent + packages) — what gets approved and promoted |
| `vars/main.yml` | Feed URL, serial, target hosts, canary flags |

# Requirements

- Windows hosts with WinRM and an Administrator credential
- Chocolatey already installed on targets (preflight fails closed if missing)
- Collections from `../../collections/requirements.yml`:
  - `chocolatey.chocolatey`
  - `ansible.windows`
- Execution Environment that includes those collections (or install on the controller project)

```bash
ansible-galaxy collection install -r ../../collections/requirements.yml
```

# Manifest

Edit `vars/manifest.yml` (or override via AAP survey / extra vars):

```yaml
chocolatey_target_version: "2.4.1"
managed_packages:
  - name: git
    version: "2.47.0"
  - name: vscode
    version: "1.95.0"
  - name: 7zip
    version: "24.8.0"
```

- Only listed packages are upgraded/verified; Discover may still report unmanaged installs.
- Prefer an internal/approved feed via `chocolatey_source` in `vars/main.yml` or extra vars.

# CLI usage

```bash
# Discover only
ansible-playbook main.yml --tags discover -e promotion_stage=discover

# Full stage via extra var (no tags needed)
ansible-playbook main.yml -e promotion_stage=preflight

# Canary / serial example
ansible-playbook main.yml -e promotion_stage=upgrade_agent \
  -e promotion_serial=1 -e target_hosts=windows_dev_canary
```

Limit inventory to the right environment group (`windows_dev`, `windows_test`, etc.).

# AAP setup

Inventories, credentials, and workflow objects stay outside this repo. Suggested wiring:

## Inventories / credentials

- Groups: `windows_dev`, `windows_test` (optional `windows_dev_canary`)
- Credential: Windows Administrator (WinRM)
- Project: this git repo; playbook path `playbooks/server_promotion/main.yml`
- EE: image that includes `chocolatey.chocolatey` and `ansible.windows`

## Job templates

Create **one job template per stage** (clearest workflow graph), or **one JT** and set `promotion_stage` on each workflow node.

Shared JT settings:

- Inventory: Dev or Test inventory (per node)
- Credentials: Windows admin
- Extra vars (defaults):

```yaml
target_hosts: windows_dev   # or windows_test on the Test nodes
promotion_serial: "100%"
# chocolatey_source: https://choco.example.com/nuget/
```

| Workflow node | Extra vars / limit |
|---------------|-------------------|
| Dev Discover | `promotion_stage: discover`, `target_hosts: windows_dev` |
| Dev Preflight | `promotion_stage: preflight` |
| Dev Upgrade agent | `promotion_stage: upgrade_agent` |
| Dev Upgrade packages | `promotion_stage: upgrade_packages` |
| Dev Verify | `promotion_stage: verify` |
| **Approval** | Human approval node (see below) |
| Test Discover → … → Verify | Same stages with `target_hosts: windows_test` and **identical** manifest |

Optional canary: run Upgrade agent/packages first against `windows_dev_canary` with `promotion_serial: 1`, then the full Dev group.

## Survey (manifest promotion)

Add a JT or workflow survey so operators pin a release without editing git for every cut:

| Question | Extra var | Example |
|----------|-----------|---------|
| Chocolatey version | `chocolatey_target_version` | `2.4.1` |
| Managed packages (YAML/JSON) | `managed_packages` | list of `{name, version}` |
| Chocolatey source URL | `chocolatey_source` | internal feed (optional) |

Survey answers override `vars/manifest.yml`. The **approved** values must be the same ones passed into every Test node—do not re-resolve “newest” after Dev Verify.

Alternatively, commit the manifest for a release branch and promote that revision; still pass the same project revision into Test jobs.

## Workflow Job Template

```text
[Dev Discover] → [Dev Preflight] → [Dev Upgrade agent] → [Dev Upgrade packages] → [Dev Verify]
        → [Approval: promote manifest]
        → [Test Discover] → [Test Preflight] → [Test Upgrade agent] → [Test Upgrade packages] → [Test Verify]
```

### Approval gating (Dev → Test)

1. After **Dev Verify** succeeds, insert an **Approval** node.
2. Approver confirms the Artifacts / survey values: `manifest_chocolatey_target_version` and `manifest_managed_packages` (published via `set_stats` on each stage).
3. On approve, Test nodes run with the **same** extra vars / manifest. Do not change package versions between Dev and Test.
4. On deny, stop; fix Dev or revise the manifest and re-run from Discover.

### Artifacts

Each stage calls `ansible.builtin.set_stats` (`aggregate: false`) so the job **Artifacts** tab shows before/after and manifest pins for audit—same pattern as `playbooks/aap_artifacts/`.

## Lifecycle rules (ops checklist)

1. Discover (Dev) → review drift in Artifacts  
2. Preflight (Dev) → resolve every pin; no installs  
3. Upgrade agent → packages (Dev); serial/canary if the fleet is large  
4. Verify (Dev) → fail closed on any drift  
5. Approval → human signs off on that exact manifest  
6. Repeat stages on Test with the identical manifest  
7. Unmanaged packages: visible in Discover only unless added to the manifest  

# Guardrails

- Pin every version; never `state: latest` in Test  
- Agent before packages; package failures after a successful agent upgrade are left as package-stage failures (no silent agent rollback)  
- Idempotent re-runs succeed when hosts already match the manifest  
