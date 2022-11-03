# Ansible_software_update
Ansible script to update software on Cisco devices. 

### How it works:
- Script ensures backup directory exists and save config before applying any change
```yaml
 - name: Get ansible date/time facts
      ansible.builtin.setup:
        filter: "ansible_date_time"
        gather_subset: "!all"

    - name: Store DTG as fact
      ansible.builtin.set_fact:
        DTG: "{{ ansible_date_time.date }}"

    - name: Create Directory
      ansible.builtin.file:
        path: ./backups/{{ hostvars.localhost.DTG }}
        state: directory
        mode: u+rw,g-wx,o-rwx
  run_once: true
  
  # Later in script:
  
      - name: Backup Running Config
      ios_command:
        commands: show run
      register: config
      tags: backup

    - name: Save output of sh run to ./backups/
      ansible.builtin.copy:
        content: "{{ config.stdout[0] }}"
        dest: "./backups/{{ hostvars.localhost.DTG }}/{{ inventory_hostname }}-{{ hostvars.localhost.DTG }}-config.txt"
        mode: u+rw,g-wx,o-rwx
      tags: backup

  
  ```


- Script checks whether current software version matches targer IOS release
  - if maches then skip for current host
  - if not then runs tasks
  
  ```yaml
   - name: Collect ios facts
      ios_facts:
        gather_subset:
          - hardware
      tags: hwinfo

    - name: display hardware info
      ansible.builtin.debug:
        msg:
          - "*** HARDWARE INFO ***"
          - "Free Space is: {{ ansible_net_filesystems_info[ flashname ]['spacefree_kb'] }} kb of {{ ansible_net_filesystems_info[ flashname ]['spacetotal_kb'] }} kb"
          - "Current IOS image is: {{ ansible_net_image }}"
          - "Target IOS image is: {{ upgrade_ios_image }}"
          - "Current IOS version is: {{ ansible_net_version }}"
          - "Target IOS version is: {{ upgrade_ios_version }}"
      tags: hwinfo

    - name: Check if current IOS release eq target release Cl
      ansible.builtin.meta: end_host
      when: ansible_net_version == upgrade_ios_version
      tags: checkver
  ```
  
- It uses variables defined in group_vars and host_vars:
   - upgrade_ios_image (target release)
   - ios_md5sum (for target release)
   - ios_size  (for target release)
   - upgrade_ios_version

```yaml
---
ansible_connection: network_cli
ansible_become: yes
ansible_become_method: enable
ansible_network_os: ios
ansible_user: ansi
ansible_password: C1sc0123!
flashname: "flash:"
#upgrade_ios_version: 15.1(4)M12a
#upgrade_ios_image: c1861-adventerprisek9-mz.151-4.M12a.bin
#ios_md5sum: 43b0f86a6627b3bb320ed4b3c96ddcb7
#ios_size: 49644
upgrade_ios_version: 15.2(4)M11
upgrade_ios_image: c1861-adventerprisek9-mz.152-4.M11.bin
ios_md5sum: 21abb3fccde22f65dd134d53f3970a0d
ios_size: 56056
...
```

 - Before transfering new IOS file script verifies whether or not there is enought space available for new IOS image:
   - if not then remove old image
 
 ```yaml
     - name: Not enough free space info check
      ansible.builtin.debug:
        msg:
          - "[Error] there not enought free space for new IOS version"
      when: ansible_net_filesystems_info[ flashname ]['spacefree_kb'] < ios_size
      tags: space

    - name: Delete old IOS release if not enought free space
      ios_command:
        commands: delete /force {{ ansible_net_image }}
      when: ansible_net_filesystems_info[ flashname ]['spacefree_kb'] < ios_size
      tags: del_old
```

- Copy new image to flash:

```yaml
 - name: Copy IOS to device
      ios_command:
        commands:
          - command: "copy tftp://{{ tftp_server }}/{{ upgrade_ios_image }} flash:"
            prompt: 'Destination filename \[{{ upgrade_ios_image }}\]?'
            answer: "\r"
          - command: show flash
      when: ansible_net_filesystems_info[ flashname ]['spacefree_kb'] > ios_size
      until: result[1] contains upgrade_ios_image
      retries: 3
      vars:
        ansible_command_timeout: 600
      tags: cp_ios

    - name: Check md5sum of IOS on device
      ios_command:
        commands:
          - 'verify /md5 flash:{{ upgrade_ios_image }} {{ ios_md5sum }}'
        # - command: 'verify /md5 flash:{{ upgrade_ios_image }} {{ ios_md5sum }}'
      register: md5_output
      vars:
        ansible_command_timeout: 600
      tags: md5sum
```      
      
 - Final step is to verify if new image has been transferred successfully and reload.
 
 ```yaml
 - name: Change config and reload if md5 checksum matches
      block:
        - name: Set new boot variable
          ios_config:
            lines:
              - no boot system {{ ansible_net_image }}
              - boot system flash:{{ upgrade_ios_image }}
            save_when: always

        - name: Reload device
          ios_command:
            commands:
              - command: reload
                prompt: '\[confirm\]'
                answer: "\r"

        - name: Wait for device to reboot
          ansible.builtin.wait_for:
            host: "{{ inventory_hostname }}"
            port: 22
            delay: 120
            timeout: 600
          delegate_to: localhost

        - name: Gather facts of the switch again
          ios_facts:

        - name: Assert that the running IOS version is correct
          ansible.builtin.assert:
            that:
              - upgrade_ios_version == ansible_net_version
            success_msg: "Upgrade successful."
            fail_msg: "IOS version does not match compliant version. Upgrade unsuccessful."

      when: '"Verified" in md5_output.stdout[0]'
      ```

