# Overview

Builds a custom Ansible Execution Environment with `infra.ee_utilities.ee_builder` and pushes it to a container registry (for example Automation Hub).

The EE definition lives in the `ee_list` variable in `main.yml` (base image, dependencies, tags, and build steps). Edit that list to change what gets built.

The playbook prompts for the Automation Hub password at runtime and uses it as `ee_registry_password`.

# Resources

- https://github.com/redhat-cop/ee_utilities/blob/devel/roles/ee_builder/
- https://github.com/redhat-cop/infra.aap_configuration/tree/devel/roles/controller_execution_environments
- https://ansible.readthedocs.io/projects/builder/en/stable/

# Requirements

- Podman or Docker installed and running (default runtime is Podman; change with `ee_container_runtime`)
- `ansible-builder` available on the host running the playbook
- Collections from `../../collections/requirements.yml` (`infra.ee_utilities`, `containers.podman`)
- Credentials for the destination registry (`ee_registry_dest`)

# Usage

From this folder:

```bash
ansible-galaxy collection install -r ../../collections/requirements.yml
ansible-playbook main.yml --tags build,create_ee_definition
```

Update `ee_registry_dest`, `ee_registry_username`, and the `ee_list` definition in `main.yml` as needed before running.

You can pass `ee_registry_password` interactively or by other means.
