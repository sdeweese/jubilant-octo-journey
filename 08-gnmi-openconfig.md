# Ansible with gNMI + OpenConfig OSPF Models

## Overview

In this section, you'll learn how to use **OpenConfig YANG models** with gNMI and Ansible to configure OSPF routing. OpenConfig provides vendor-neutral models that work across Cisco, Juniper, Arista, and other vendors, making your gNMI-based automation highly portable.

## What You'll Accomplish 

By the end of this section, you will:

* **Configure OSPF** using OpenConfig network-instance models
* **Compare** Cisco Native vs OpenConfig gNMI approaches
* **Understand** model portability benefits
* **Retrieve configuration** using both model types
* **Learn** when to use each model approach

This section assumes setup is already complete and `src/vars/ospf_config.yml` already exists from the previous page.

## OpenConfig Confidence Checkpoint (GET Before SET)

Run a read-only gNMI GET first to confirm connectivity before applying OpenConfig changes.

```bash
cat <<EOF > src/playbooks/gnmi_openconfig_checkpoint.yml
---
- name: OpenConfig gNMI read-only checkpoint
  hosts: iosxe_devices
  gather_facts: false
  tasks:
    - name: Read OpenConfig protocol container if present
      cisco.gnmi.gnmi_get:
        host: "{{ ansible_host }}"
        port: "{{ gnmi_port }}"
        username: "{{ ansible_user }}"
        password: "{{ ansible_password }}"
        insecure: true
        encoding: "json_ietf"
        path:
          - "/openconfig-network-instance:network-instances"
      delegate_to: localhost
      register: openconfig_checkpoint
      ignore_errors: true

    - name: Show checkpoint output
      ansible.builtin.debug:
        var: openconfig_checkpoint
EOF

ANSIBLE_CONFIG=src/ansible.cfg ansible-playbook src/playbooks/gnmi_openconfig_checkpoint.yml
```

Expected: transport/auth succeeds. If OpenConfig paths are limited on your image, continue with Native model and use OpenConfig tasks where supported.

---

## Step 1: Configure OSPF Using OpenConfig Model

 **Working Directory**: `~/src/ansible-iosxe-api`

Now configure OSPF using the vendor-neutral OpenConfig model:

```bash
cat <<EOF > src/playbooks/gnmi_ospf_openconfig.yml
---
- name: Configure OSPF using gNMI with OpenConfig Model
  hosts: iosxe_devices
  gather_facts: no
  vars_files:
    - ../vars/ospf_config.yml
  
  tasks:
    # ========================================================
    # Task 1: Configure OSPF Global Settings
    # ========================================================
    
    - name: Configure OSPF global config (OpenConfig Model)
      cisco.gnmi.gnmi_set:
        host: "{{ ansible_host }}"
        port: "{{ gnmi_port }}"
        username: "{{ ansible_user }}"
        password: "{{ ansible_password }}"
        insecure: true
        encoding: "json_ietf"
        update:
          - path: "/openconfig-network-instance:network-instances/network-instance[name={{ network_instance_name }}]/protocols/protocol[identifier=OSPF][name=ospfv2]/ospfv2/global/config"
            val:
              router-id: "{{ ospf_router_id }}"
      delegate_to: localhost
      register: ospf_global_result
    
    - name: Display OSPF global configuration result
      debug:
        msg: " OSPF router-id {{ ospf_router_id }} configured (OpenConfig)"
      when: ospf_global_result is succeeded
    
    # ========================================================
    # Task 2: Configure OSPF Area
    # ========================================================
    
    - name: Configure OSPF Area 0 (OpenConfig Model)
      cisco.gnmi.gnmi_set:
        host: "{{ ansible_host }}"
        port: "{{ gnmi_port }}"
        username: "{{ ansible_user }}"
        password: "{{ ansible_password }}"
        insecure: true
        encoding: "json_ietf"
        update:
          - path: "/openconfig-network-instance:network-instances/network-instance[name={{ network_instance_name }}]/protocols/protocol[identifier=OSPF][name=ospfv2]/ospfv2/areas/area[identifier={{ ospf_area }}]/config"
            val:
              identifier: "{{ ospf_area }}"
      delegate_to: localhost
      register: ospf_area_result
    
    - name: Display OSPF area configuration result
      debug:
        msg: " OSPF Area {{ ospf_area }} configured"
      when: ospf_area_result is succeeded
    
    # ========================================================
    # Task 3: Enable OSPF on Interfaces
    # ========================================================
    
    - name: Enable OSPF on interfaces (OpenConfig Model)
      cisco.gnmi.gnmi_set:
        host: "{{ ansible_host }}"
        port: "{{ gnmi_port }}"
        username: "{{ ansible_user }}"
        password: "{{ ansible_password }}"
        insecure: true
        encoding: "json_ietf"
        update:
          - path: "/openconfig-network-instance:network-instances/network-instance[name={{ network_instance_name }}]/protocols/protocol[identifier=OSPF][name=ospfv2]/ospfv2/areas/area[identifier={{ ospf_area }}]/interfaces/interface[id={{ item.name }}]/config"
            val:
              id: "{{ item.name }}"
              enabled: "{{ item.enabled }}"
      loop: "{{ ospf_interfaces }}"
      loop_control:
        label: "{{ item.name }}"
      delegate_to: localhost
      register: ospf_interfaces_result
    
    - name: Display OSPF interfaces configuration result
      debug:
        msg: " OSPF enabled on {{ ospf_interfaces | length }} interface(s)"
    
    # ========================================================
    # Summary
    # ========================================================
    
    - name: Configuration summary
      debug:
        msg:
          - "════════════════════════════════════════════"
          - "   OSPF Configuration Complete (OpenConfig)"
          - "════════════════════════════════════════════"
          - "Router ID: {{ ospf_router_id }}"
          - "Area: {{ ospf_area }}"
          - "Interfaces: {{ ospf_interfaces | map(attribute='name') | join(', ') }}"
          - "Model: openconfig-network-instance + openconfig-ospf"
          - " Vendor-neutral configuration (portable!)"
EOF
```

### Run the Playbook

```bash
ansible-playbook src/playbooks/gnmi_ospf_openconfig.yml -v
```

**Expected output:**
```
PLAY [Configure OSPF using gNMI with OpenConfig Model] ****

TASK [Configure OSPF global config (OpenConfig Model)] ****
ok: [sandbox]

TASK [Display OSPF global configuration result] ****
ok: [sandbox] => {
    "msg": " OSPF router-id 1.1.1.1 configured (OpenConfig)"
}

TASK [Configure OSPF Area 0 (OpenConfig Model)] ****
ok: [sandbox]

TASK [Enable OSPF on interfaces (OpenConfig Model)] ****
ok: [sandbox] => (item=Loopback0)
ok: [sandbox] => (item=GigabitEthernet1)

TASK [Display OSPF interfaces configuration result] ****
ok: [sandbox] => {
    "msg": " OSPF enabled on 2 interface(s)"
}

PLAY RECAP ****
sandbox : ok=5  changed=0  failed=0
```

---

## Step 2: Retrieve OSPF Configuration via gNMI Get

 **Working Directory**: `~/src/ansible-iosxe-api`

Use gNMI Get to retrieve OSPF configuration:

```bash
cat <<EOF > src/playbooks/gnmi_get_ospf.yml
---
- name: Retrieve OSPF Configuration via gNMI Get
  hosts: iosxe_devices
  gather_facts: no
  vars_files:
    - ../vars/ospf_config.yml
  
  tasks:
    # ========================================================
    # Get OSPF Config - Native Model
    # ========================================================
    
    - name: Get OSPF configuration (Native Model)
      cisco.gnmi.gnmi_get:
        host: "{{ ansible_host }}"
        port: "{{ gnmi_port }}"
        username: "{{ ansible_user }}"
        password: "{{ ansible_password }}"
        insecure: true
        encoding: "json_ietf"
        path:
          - "/Cisco-IOS-XE-native:native/router/Cisco-IOS-XE-ospf:ospf"
      delegate_to: localhost
      register: ospf_native_config
    
    - name: Display OSPF configuration (Native)
      debug:
        var: ospf_native_config.notification[0].update[0].val
    
    # ========================================================
    # Get OSPF Operational State
    # ========================================================
    
    - name: Get OSPF operational state
      cisco.gnmi.gnmi_get:
        host: "{{ ansible_host }}"
        port: "{{ gnmi_port }}"
        username: "{{ ansible_user }}"
        password: "{{ ansible_password }}"
        insecure: true
        encoding: "json_ietf"
        path:
          - "/Cisco-IOS-XE-ospf-oper:ospf-oper-data"
        data_type: "state"
      delegate_to: localhost
      register: ospf_oper_state
    
    - name: Display OSPF operational state
      debug:
        var: ospf_oper_state.notification[0].update[0].val
    
    # ========================================================
    # Get OpenConfig OSPF Config
    # ========================================================
    
    - name: Get OSPF configuration (OpenConfig Model)
      cisco.gnmi.gnmi_get:
        host: "{{ ansible_host }}"
        port: "{{ gnmi_port }}"
        username: "{{ ansible_user }}"
        password: "{{ ansible_password }}"
        insecure: true
        encoding: "json_ietf"
        path:
          - "/openconfig-network-instance:network-instances/network-instance[name=default]/protocols/protocol[identifier=OSPF][name=ospfv2]"
      delegate_to: localhost
      register: ospf_openconfig_config
      ignore_errors: yes
    
    - name: Display OSPF configuration (OpenConfig)
      debug:
        var: ospf_openconfig_config.notification[0].update[0].val
      when: ospf_openconfig_config is succeeded
EOF
```

### Run the Playbook

```bash
ansible-playbook src/playbooks/gnmi_get_ospf.yml -v
```

---

## Step 3: Compare Native vs OpenConfig Approaches

### Side-by-Side Comparison

| Aspect | Cisco Native Model | OpenConfig Model |
|--------|-------------------|------------------|
| **Portability** | Cisco IOS XE only | Multi-vendor |
| **Feature Coverage** | Complete | Commonly-used features |
| **Configuration Method** | Network statements | Per-interface |
| **Learning Curve** | Familiar to Cisco admins | Industry standard |
| **Automation** | Cisco-specific playbooks | Portable playbooks |
| **Updates** | Cisco release cycle | OpenConfig community |

### When to Use Each Model

**Use Cisco Native when:**
- Working exclusively with Cisco IOS XE
- Need advanced OSPF features (stub areas, virtual links, etc.)
- Team familiar with Cisco CLI/YANG models
- Migrating from CLI-based automation

**Use OpenConfig when:**
- Multi-vendor environment (Cisco + others)
- Basic OSPF requirements
- Want portable playbooks
- Following industry standards
- Cloud-native infrastructure

### Best Practice: Hybrid Approach

For real-world automation:
1. **Use OpenConfig** for common features (interfaces, basic routing)
2. **Use Native** for vendor-specific advanced features
3. **Abstract with variables** to switch models easily
4. **Test both** to understand capabilities

---

## Understanding gNMI Protocol Buffers

# What Makes gNMI Different?

Unlike NETCONF (XML) and RESTCONF (JSON), gNMI uses **Protocol Buffers**:

**Protocol Buffer Characteristics:**
- **Compact**: Binary format, smaller payloads
- **Fast**: Faster parsing than text-based formats
- **Typed**: Strong typing prevents errors
- **Versioned**: Proto schema evolution
- **Efficient**: Lower CPU/bandwidth usage

**Example Data Flow:**

```
Ansible Playbook
     ↓
JSON/YAML Variables
     ↓
cisco.gnmi module
     ↓
Protocol Buffer Encoding
     ↓
gRPC over HTTP/2
     ↓
IOS XE gNMI Server (port 9339)
     ↓
YANG Model Validation
     ↓
Configuration Applied
```

---

## Troubleshooting

### Issue: gNMI connection failed

**Symptoms:**
```
FAILED! => {"msg": "gRPC connection failed"}
```

**Solutions:**
```bash
# 1. Check port 9339 is accessible
nc -zv <SANDBOX_HOSTNAME> 9339

# 2. Verify credentials in inventory
cat src/inventory/hosts.yml

# 3. Test with Ansible gNMI module
mkdir -p src/tmp
cat <<EOF > src/tmp/gnmi_connectivity_test.yml
---
- name: Validate gNMI connectivity with Ansible
  hosts: localhost
  gather_facts: false
  tasks:
    - name: Query device hostname over gNMI
      cisco.gnmi.gnmi:
        host: "<SANDBOX_HOSTNAME>"
        port: 9339
        username: "<USERNAME>"
        password: "<SANDBOX_PASSWORD>"
        insecure: true
        platform: iosxe
        operation: get
        origin: rfc7951
        paths:
          - /Cisco-IOS-XE-native:native/hostname
      register: gnmi_result

    - debug:
        var: gnmi_result
EOF
ansible-playbook src/tmp/gnmi_connectivity_test.yml
```

### Issue: OpenConfig model not supported

**Symptoms:**
```
FAILED! => {"msg": "path not found"}
```

**Solutions:**
- OpenConfig support varies by IOS XE version
- Check OpenConfig support with Ansible gNMI GET on an OpenConfig path and review task output for path support errors
- Use Native model as fallback
- Some features only in Native model

### Issue: OSPF not running

**Symptoms:**
- Playbook succeeds but OSPF state GET tasks return empty or unexpected data

**Solutions:**
```bash
# Validate OSPF configuration state via gNMI GET playbook
ansible-playbook src/playbooks/gnmi_get_ospf.yml -v

# Re-run OpenConfig configuration with detailed output
ansible-playbook src/playbooks/gnmi_ospf_openconfig.yml -vvv

# Re-run playbook with -vvv for details
ansible-playbook src/playbooks/gnmi_ospf_native.yml -vvv
```

---

## Best Practices

1. **Use gNMI for bulk operations**: Faster than NETCONF for large configs
2. **Leverage Protocol Buffers**: More efficient than JSON/XML
3. **Try OpenConfig first**: Portable configs, fallback to Native if needed
4. **Version control models**: Track which YANG models you depend on
5. **Test with both models**: Understand feature parity
6. **Monitor gRPC connections**: Keep alive settings for long-running operations

---

## Summary

Congratulations! You've learned how to use gNMI for OSPF configuration with both Cisco Native and OpenConfig models.

###  What You Accomplished

-  Installed cisco.gnmi Ansible collection
-  Configured OSPF using Cisco Native YANG model
-  Configured OSPF using OpenConfig YANG model  
-  Used gNMI Set operations for configuration
-  Used gNMI Get operations to retrieve state
-  Compared vendor-specific vs vendor-neutral approaches
-  Verified OSPF configuration on device

###  Key Takeaways

1. **gNMI is efficient**: Protocol Buffers provide performance advantages
2. **Two model philosophies**: Native (complete) vs OpenConfig (portable)
3. **Trade-offs exist**: Feature coverage vs portability
4. **Both have value**: Choose based on requirements
5. **gNMI Set = Configuration**: Update, replace, delete operations
6. **gNMI Get = Retrieval**: Query config and operational state

###  Three-API Journey Complete

You've now experienced all three APIs:

| API | Section | Protocol | Data Format | Use Case |
|-----|---------|----------|-------------|----------|
| **NETCONF** | BGP | SSH/XML | XML | Transactional edge routing |
| **RESTCONF** | Sandbox | HTTP/JSON | JSON | RESTful interface and VLAN automation |
| **gNMI** | OSPF | gRPC/Protobuf | Binary | High-performance campus routing |

###  What's Next?

**Continue to Mission Section:**
- Apply all three APIs in a comprehensive scenario
- Build end-to-end automation
- Practice real-world workflows

**Optional Advanced Topics:**
- gNMI Subscribe for configuration monitoring
- Multi-model playbooks (mix Native + OpenConfig)
- Error handling and rollback strategies
- Performance tuning for large-scale deployments

Continue to [09-recap-conclusion.md](./09-recap-conclusion.md).

---

## Additional Resources

**OpenConfig:**
- Website: https://openconfig.net
- Models: https://github.com/openconfig/public

**gNMI Protocol:**
- Specification: https://github.com/openconfig/gnmi
- Ansible Collection: https://github.com/CiscoDevNet/ansible-gnmi

**Cisco IOS XE YANG:**
- DevNet: https://developer.cisco.com/docs/ios-xe/
- GitHub: https://github.com/YangModels/yang

**cisco.gnmi Collection:**
- Galaxy: https://galaxy.ansible.com/cisco/gnmi
- GitHub: https://github.com/CiscoDevNet/ansible-gnmi

---

**Congratulations on completing the gNMI + OSPF lab!**

You now have hands-on experience with modern network programmability using three different APIs and multiple YANG model types. These skills are directly applicable to production network automation!

## Navigation

- Previous: [07-gnmi-native.md](./07-gnmi-native.md)
- Next: [09-recap-conclusion.md](./09-recap-conclusion.md)
