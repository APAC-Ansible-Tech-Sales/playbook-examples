# Overview

Example playbook that gathers host data and publishes it to Ansible Automation Platform (AAP) job artifacts via `ansible.builtin.set_stats`.

It collects filesystem capacity (`df`) and the local user list (`/etc/passwd`), prints them with `debug`, then exposes the values under **Artifacts** in the AAP job view when `aggregate` is `false`.

# Resources

- https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/set_stats_module.html

# Requirements

- Ansible (runs on `localhost`; no inventory hosts required)
- Suitable privileges to read `/etc/passwd` and run `df`

# Usage

From this folder:

```bash
ansible-playbook main.yml
```

In AAP, create a job template pointing at this playbook. After a successful run, open the job and check the **Artifacts** tab for `storage_usage` and `users`.
