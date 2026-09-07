pyATS/Genie learn integration
=============================

What this does
---------------

This repository contains Ansible tasks and helper scripts to build a pyATS testbed from inventory and run a Genie "learn" on devices to collect:

- Platform/version information
- Interface operational status
- VLAN configuration

Files added
-----------

- `pyats_learn_allhosts.yml` - playbook that renders a testbed for all inventory hosts and runs `clay584.genie.learn_genie` per host (when available).
- `pyats_learn_single_host.yml` - example playbook that runs the `pyats_learn` role for a single host (set `target_host` variable).
- `roles/pyats_learn/tasks/main.yml` - role tasks used by the single-host playbook; it will try `clay584.genie.learn_genie` and fall back to `scripts/pyats_learn.py`.
- `roles/pyats_learn/templates/pyats_testbed_host.j2` - testbed template for a single host.
- `pyats_testbed.j2` - testbed template for all hosts.
- `scripts/pyats_learn.py` - fallback script that loads the testbed, connects to the device, runs Genie learn for platform, interfaces, and vlans, and writes JSON outputs into `backups/`.

Requirements
------------

- Python with pyATS and Genie installed. The repository includes a `virt/` virtualenv; use `virt/bin/python` which should have pyats and genie installed if you've created it.
- Optional: the Ansible collection `clay584.genie` provides the `learn_genie` module used by the role. If that collection is not installed, the role will fall back to the local script.

Quick run
---------

1. To run for all hosts (uses `clay584.genie.learn_genie` per host when available):

```bash
ansible-playbook pyats_learn_allhosts.yml -i hosts
```

1. To run for a single host (edit `pyats_learn_single_host.yml` and set `target_host` to the inventory name):

```bash
ansible-playbook pyats_learn_single_host.yml -i hosts
```

Notes
-----

- The generated JSON files will be written to `backups/` with filenames like `<host>-learn.json`.
- If pyATS/Genie are not installed in your default Python, set the environment variable `VIRT_PYTHON` to point to the Python interpreter you want to use before running the playbook, e.g.:

```bash
export VIRT_PYTHON=/home/swt/Automation/Ansible_demo/software_update/virt/bin/python
ansible-playbook pyats_learn_single_host.yml -i hosts
```

- The templates attempt to read `ansible_host`, `ansible_port`, `ansible_user`, and `ansible_password` from hostvars. Ensure your inventory provides the connection data or the default will be used.
pyATS/Genie learn integration
=============================

What this does
---------------

This repository contains Ansible tasks and helper scripts to build a pyATS testbed from inventory and run a Genie "learn" on devices to collect:

- Platform/version information
- Interface operational status
- VLAN configuration

Files added
-----------

- `pyats_learn.yml` - playbook that renders a testbed for all inventory hosts and runs `scripts/pyats_learn.py` for each.
- `pyats_learn_single_host.yml` - example playbook that runs the role for a single host (set `target_host` variable).
- `roles/pyats_learn/tasks/main.yml` - role tasks used by the single-host playbook.
- `roles/pyats_learn/templates/pyats_testbed_host.j2` - testbed template for a single host.
- `scripts/pyats_learn.py` - script that loads the testbed, connects to the device, runs Genie learn for platform, interfaces, and vlans, and writes JSON outputs into `backups/`.

Requirements
------------

- Python with pyATS and Genie installed. The repository includes a `virt/` virtualenv; use `virt/bin/python` which should have pyats and genie installed if you've created it.

Quick run
---------

1. To run for all hosts (creates a testbed from inventory and runs learn per host):

```bash
ansible-playbook pyats_learn.yml -i hosts
```

1. To run for a single host (edit `pyats_learn_single_host.yml` and set `target_host` to the inventory name):

```bash
ansible-playbook pyats_learn_single_host.yml -i hosts
```

Notes
-----

- The generated JSON files will be written to `backups/` with filenames like `<host>-learn.json`.
- If pyATS/Genie are not installed in your default Python, set the environment variable `VIRT_PYTHON` to point to the Python interpreter you want to use before running the playbook, e.g.:

```bash
export VIRT_PYTHON=/home/swt/Automation/Ansible_demo/software_update/virt/bin/python
ansible-playbook pyats_learn_single_host.yml -i hosts
```

- The templates attempt to read `ansible_host`, `ansible_port`, `ansible_user`, and `ansible_password` from hostvars. Ensure your inventory provides the connection data or the default will be used.
