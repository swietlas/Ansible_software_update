# Cisco IOS Software Update Automation - AI Agent Instructions

## Project Overview
This is an Ansible automation framework for managing Cisco network device software updates and configuration backups. It orchestrates firmware distribution, validation, and device reload with pre/post health checks. The project integrates pyATS/Genie for network device fact learning and state validation.

## Architecture Patterns

### Playbook Structure
- **Multi-play design**: Each major workflow (backup, distribute, upgrade, validate) is organized as separate plays in sequential playbooks
- **Tag-based task organization**: Tasks are tagged (`hwinfo`, `backup`, `md5sum`, `cp_ios`, etc.) to enable selective execution
- **Localhost coordination plays**: Always include a "Create Backup Dir for today" play at the start to establish timestamped backup paths using `DTG` (Date-Time Group) fact
- **Network device plays**: Use device group targets (`3560series`, `1800series`, `2900series`, `2800series`) defined in inventory

### Configuration Management
- **Group vars pattern**: Device-type-specific config in `group_vars/<group_name>.yml` (e.g., `group_vars/3560series.yml`)
- **Host vars pattern**: Device-specific overrides in `host_vars/<hostname>.yml`
- **Key variables to define per device group**:
  - `flashname`: Device flash memory name (e.g., `"flash:"`)
  - `upgrade_ios_version`: Target IOS semantic version
  - `upgrade_ios_image`: Binary filename on flash
  - `ios_md5sum`: MD5 checksum for verification
  - `ios_size`: Approximate size in KB for free-space checks
  - `ansible_connection`: Must be `network_cli` for Cisco devices
  - `ansible_network_os`: Must be `ios`

### Common Workflow Pattern
1. Create timestamped backup directory at localhost
2. Collect device facts (hardware info, free flash space)
3. Backup running configuration to timestamped file
4. Execute upgrade logic in a `block` with error handling
5. Validate post-upgrade state with assertions
6. Wait for device reboot with appropriate timeout (typically 120-900 second delay)

## Key Files & Their Roles

| File | Purpose |
|------|---------|
| `activate_software.yml` | Primary upgrade workflow: backup → distribute → verify MD5 → set boot → reload → validate |
| `upgrade.yml` | Legacy upgrade variant for 1800series routers (similar workflow, different device group) |
| `distribute_software_scp.yml` | SCP-based firmware distribution with device-side space validation |
| `distributeSoftware_ver_tftp.yml` | TFTP-based firmware distribution variant |
| `md5sum.yml` | Standalone MD5 verification task |
| `pre-post-check.yml` | Health check playbook (run before/after upgrades) |
| `pyats_learn.yml` / `pyats_learn_allhosts.yml` | Genie learn integration: collects interface/VLAN/ARP facts to JSON |
| `group_vars/3560series.yml` | Cisco 3560 switch config (connection, credentials, target IOS) |
| `scripts/pyats_learn.py` | Fallback Python script for Genie learning when `clay584.genie` collection unavailable |

## Critical Workflows & Commands

### Run full upgrade workflow
```bash
ansible-playbook activate_software.yml -i hosts
```

### Run with specific device group
```bash
# Edit activate_software.yml, change "hosts: 3560series" to desired group
ansible-playbook activate_software.yml -i hosts
```

### Run only specific tasks (by tag)
```bash
ansible-playbook activate_software.yml -i hosts --tags backup
ansible-playbook activate_software.yml -i hosts --tags md5sum
ansible-playbook activate_software.yml -i hosts --tags hwinfo
```

### Run pyATS/Genie learning (fact collection)
```bash
# Requires virt/bin/python with pyats/genie installed
export VIRT_PYTHON=/path/to/virt/bin/python
ansible-playbook pyats_learn_allhosts.yml -i hosts
# Output: backups/<host>-learn.json files with parsed device facts
```

### Check for unused imports/configs
```bash
# Review group_vars and host_vars for commented-out variables (common in this project)
grep -r "^#" group_vars/ host_vars/
```

## Project-Specific Conventions

### Variable Naming & Hostvars Access
- **DTG (Date-Time Group)**: Centralized fact set by localhost play, accessed by all other plays via `hostvars.localhost.DTG`
- **Flashname**: Device-specific flash device name (e.g., `flash:` on 3560s), must be defined per device group
- **Command timeout**: Use `vars.ansible_command_timeout` (300-600 sec) for reliability; reload command needs 600+ sec

### MD5 Verification Pattern
The upgrade workflow **must verify MD5 before reload**:
```yaml
- name: Check md5sum
  cisco.ios.ios_command:
    commands:
      - 'verify /md5 flash:{{ upgrade_ios_image }} {{ ios_md5sum }}'
  register: md5_output
  
- name: Assert MD5 verified
  ansible.builtin.assert:
    that:
      - '"Verified" in md5_output.stdout[0]'
```
Only proceed with reload if "Verified" appears in stdout.

### Reload Pattern
- Use `cisco.ios.ios_command` with prompt/answer for interactive reload
- Add 120-900 second `delay` before `wait_for` (device needs time to shut down)
- Set `timeout: 900` (15 min) for devices to fully reboot and SSH to be ready
- Delegate `wait_for` to localhost: `delegate_to: localhost`

### Backup Pattern
Backups stored in `./backups/{{ hostvars.localhost.DTG }}/{{ inventory_hostname }}-{{ hostvars.localhost.DTG }}-config.txt`

### Assertions vs Debug
- Use `assert` for validation gates (MD5, version match) that **stop execution on failure**
- Use `debug` for informational messages (hardware info, status checks)

## Integration Points

### Cisco IOS Collection
- Playbooks use `cisco.ios` collection modules (`ios_facts`, `ios_command`, `ios_config`)
- Requires `ansible_connection: network_cli` and `ansible_network_os: ios`
- Connection credentials defined in group_vars (plaintext storage - consider vault in production)

### pyATS/Genie Integration
- Optional: `clay584.genie` Ansible collection provides `learn_genie` module
- Fallback: Local `scripts/pyats_learn.py` runs if collection unavailable
- Testbed templates: `pyats_testbed.j2` and `roles/pyats_learn/templates/pyats_testbed_host.j2`
- Output: JSON files written to `backups/` with learned facts (interfaces, VLANs, ARP)

### Virtual Environment
- Project includes `virt/` Python virtualenv with ansible, pyats, genie pre-installed
- Use `virt/bin/python` or `virt/bin/ansible-playbook` to ensure correct dependencies
- Export `VIRT_PYTHON` when running scripts that need specific Python version

## Common Failure Points & Debugging

1. **Free space check**: If device has insufficient flash space, old IOS is deleted before transfer
2. **MD5 mismatch**: Transfer failed or corrupted; re-distribute firmware
3. **Boot variable syntax**: Use `no boot system <old>` + `boot system flash:<new>` pattern exactly
4. **Timeout during reload**: Increase `wait_for.timeout` (current: 900 sec = 15 min)
5. **Version mismatch post-reload**: Check upload completed before asserting final version

## When Modifying Workflows
- **Add backup dir creation** at start of any new playbook (see `activate_software.yml` play 1)
- **Tag all tasks** with logical categories for selective runs
- **Use device-group plays** rather than hardcoded hostnames for reusability
- **Add assertions after state-changing tasks** (reload, config changes)
- **Test with single device first** before rolling out to device groups
