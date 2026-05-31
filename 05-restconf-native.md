# Ansible with RESTCONF + Native Models

## Overview

In this section, you will use Ansible and RESTCONF to configure VLAN and interface settings on a single IOS XE sandbox device.

This page is focused on hands-on API usage with clear, repeatable examples.

## What You Will Accomplish

By the end of this section, you will:

- Use RESTCONF over HTTPS (port 443) with Ansible
- Build JSON payloads for IOS XE native YANG paths
- Create VLANs and update interface descriptions
- Verify applied configuration with Ansible-based read checks
- Understand idempotent rerun behavior for RESTCONF automation

This section assumes setup is already complete from [02-lab-setup.md](./02-lab-setup.md).

## RESTCONF Confidence Checkpoint (Read-Only)

Goal: confirm credentials + RESTCONF connectivity before writing any config.

Run from ~/src/ansible-iosxe-api:

```bash
cat <<EOF > src/playbooks/restconf_checkpoint.yml
---
- name: RESTCONF read-only checkpoint
  hosts: iosxe_devices
  gather_facts: false
  tasks:
    - name: Read current hostname via RESTCONF
      ansible.builtin.uri:
        url: "https://{{ ansible_host }}/restconf/data/Cisco-IOS-XE-native:native/hostname"
        method: GET
        user: "{{ ansible_user }}"
        password: "{{ ansible_password }}"
        force_basic_auth: true
        validate_certs: false
        headers:
          Accept: application/yang-data+json
      delegate_to: localhost
      register: hostname_result

    - name: Show checkpoint result
      ansible.builtin.debug:
        var: hostname_result.json
EOF

ANSIBLE_CONFIG=src/ansible.cfg ansible-playbook src/playbooks/restconf_checkpoint.yml
```

Expected: playbook returns hostname JSON and no auth/connection errors.

## Step 1: Create RESTCONF Variables

Create src/vars/interface_config.yml:

```bash
cat <<EOF > src/vars/interface_config.yml
---
vlan_list:
  - id: 110
    name: USERS
  - id: 120
    name: VOICE

interface_updates:
  - name: GigabitEthernet1
    description: "Configured by RESTCONF Ansible"
  - name: Loopback10
    description: "RESTCONF API test loopback"
EOF
```

## Step 2: Create RESTCONF Configuration Playbook

Create src/playbooks/restconf_interfaces.yml:

```bash
cat <<EOF > src/playbooks/restconf_interfaces.yml
---
- name: Configure VLANs and interfaces via RESTCONF
  hosts: iosxe_devices
  gather_facts: false
  vars_files:
    - ../vars/interface_config.yml

  tasks:
    - name: Create VLANs
      ansible.builtin.uri:
        url: "https://{{ ansible_host }}/restconf/data/Cisco-IOS-XE-native:native/vlan"
        method: PATCH
        user: "{{ ansible_user }}"
        password: "{{ ansible_password }}"
        force_basic_auth: true
        validate_certs: false
        headers:
          Content-Type: application/yang-data+json
          Accept: application/yang-data+json
        body_format: json
        body:
          Cisco-IOS-XE-native:vlan:
            vlan-list:
              - id: "{{ item.id }}"
                name: "{{ item.name }}"
      loop: "{{ vlan_list }}"
      loop_control:
        label: "VLAN {{ item.id }}"
      delegate_to: localhost
      register: vlan_patch_results

    - name: Update interface descriptions
      ansible.builtin.uri:
        url: "https://{{ ansible_host }}/restconf/data/Cisco-IOS-XE-native:native/interface"
        method: PATCH
        user: "{{ ansible_user }}"
        password: "{{ ansible_password }}"
        force_basic_auth: true
        validate_certs: false
        headers:
          Content-Type: application/yang-data+json
          Accept: application/yang-data+json
        body_format: json
        body:
          Cisco-IOS-XE-native:interface:
            {{ 'GigabitEthernet' if 'GigabitEthernet' in item.name else 'Loopback' }}:
              - name: "{{ item.name | regex_replace('^GigabitEthernet|^Loopback', '') }}"
                description: "{{ item.description }}"
      loop: "{{ interface_updates }}"
      loop_control:
        label: "{{ item.name }}"
      delegate_to: localhost
      register: interface_patch_results

    - name: Summary
      ansible.builtin.debug:
        msg:
          - "RESTCONF writes complete"
          - "VLAN tasks: {{ vlan_patch_results.results | length }}"
          - "Interface tasks: {{ interface_patch_results.results | length }}"
EOF
```

Run it:

```bash
ANSIBLE_CONFIG=src/ansible.cfg ansible-playbook src/playbooks/restconf_interfaces.yml -v
```

## Step 3: Create RESTCONF Verification Playbook (Ansible-Based)

Create src/playbooks/restconf_verify.yml:

```bash
cat <<EOF > src/playbooks/restconf_verify.yml
---
- name: Verify RESTCONF-applied configuration
  hosts: iosxe_devices
  gather_facts: false

  tasks:
    - name: Read VLAN config
      ansible.builtin.uri:
        url: "https://{{ ansible_host }}/restconf/data/Cisco-IOS-XE-native:native/vlan"
        method: GET
        user: "{{ ansible_user }}"
        password: "{{ ansible_password }}"
        force_basic_auth: true
        validate_certs: false
        headers:
          Accept: application/yang-data+json
      delegate_to: localhost
      register: vlan_state

    - name: Read native interface config
      ansible.builtin.uri:
        url: "https://{{ ansible_host }}/restconf/data/Cisco-IOS-XE-native:native/interface"
        method: GET
        user: "{{ ansible_user }}"
        password: "{{ ansible_password }}"
        force_basic_auth: true
        validate_certs: false
        headers:
          Accept: application/yang-data+json
      delegate_to: localhost
      register: intf_state

    - name: Show verification highlights
      ansible.builtin.debug:
        msg:
          - "VLAN payload retrieved: {{ vlan_state.status }}"
          - "Interface payload retrieved: {{ intf_state.status }}"
EOF
```

Run it:

```bash
ANSIBLE_CONFIG=src/ansible.cfg ansible-playbook src/playbooks/restconf_verify.yml
```

Expected: HTTP status 200 responses and JSON payload returned.

## Rerun Behavior (Confidence Builder)

If you rerun src/playbooks/restconf_interfaces.yml with the same variables:

- It should complete successfully again
- Existing values should remain stable
- You should not need manual cleanup between runs

This is the expected workflow for idempotent automation.

## Troubleshooting

If auth fails:

- Re-check src/inventory/hosts.yml username/password/hostname
- Re-run the checkpoint playbook first

If RESTCONF path fails:

- Confirm RESTCONF is enabled from setup page checks
- Verify Accept/Content-Type headers are application/yang-data+json

If TLS validation errors occur:

- Keep validate_certs: false in this lab environment

## Next Step

Continue to [07-gnmi-native.md](./07-gnmi-native.md) for gNMI Native model automation.

## Navigation

- Previous: [03-netconf-native.md](./03-netconf-native.md)
- Next: [07-gnmi-native.md](./07-gnmi-native.md)
