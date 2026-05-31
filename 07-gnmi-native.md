# Ansible with gNMI + OSPF Configuration

## Overview

In this section, you'll learn how to use Ansible with gNMI (gRPC Network Management Interface) to configure OSPF routing on IOS XE devices. gNMI provides high-performance, efficient network management using Protocol Buffers over gRPC, making it ideal for modern automation workflows.

This section demonstrates OSPF configuration using BOTH Cisco Native and OpenConfig YANG models, showing you the portability advantages of vendor-neutral approaches.

## What You'll Accomplish 

By the end of this section, you will:

* **Configure OSPF** using gNMI Set operations
* **Work with two model types**: Cisco Native and OpenConfig
* **Use gNMI operations**: Get, Set operations via Ansible
* **Configure OSPF process**: Router ID, area configuration
* **Enable OSPF on interfaces**: Network statements and interface-level config
* **Retrieve OSPF state**: Query neighbor and route information
* **Compare models**: Understand trade-offs between Native and OpenConfig
* **Use cisco.gnmi collection**: Work with Cisco gNMI modules

**Single Device Lab**: All configuration applied to **one IOS XE device**!

## Important Limitations 

**What This Lab CANNOT Do:**

*  **Form OSPF adjacencies**: Without additional routers, no OSPF neighbors will form
*  **Exchange OSPF routes**: No route learning without actual OSPF peers
*  **Test OSPF convergence**: Cannot demonstrate failover or SPF calculations

**Why This Doesn't Matter:**

This lab teaches **gNMI API automation**, not OSPF protocol operation. You're learning:
- How to use gNMI Set operations for configuration
- How to construct Protocol Buffer payloads
- How OpenConfig models provide vendor portability
- How to retrieve operational state via gNMI Get

The OSPF process will run, interfaces will be enabled, but neighbors won't form - this is **expected and fine for learning gNMI mechanics**.


---

## What is gNMI?

gNMI is a modern network management protocol that:
* **Uses gRPC**: High-performance binary protocol
* **Protocol Buffers**: Compact, typed data serialization
* **Streaming capable**: Subscribe to configuration/state changes
* **Unified**: Single protocol for config and operational data
* **Vendor-agnostic**: OpenConfig initiative, multi-vendor support

### gNMI Key Operations

1. **Capabilities**: Discover supported YANG models
2. **Get**: Retrieve configuration or operational state
3. **Set**: Configure device (update, replace, delete)
4. **Subscribe**: Stream configuration changes or telemetry (not covered here)

### Why gNMI for Configuration?

- **Efficiency**: Binary protocol reduces overhead vs XML/JSON
- **Atomicity**: Transactions ensure consistent state
- **Model-driven**: YANG validation before applying config
- **Modern**: Cloud-native, containerized automation friendly

---

## Understanding YANG Models for OSPF

We'll work with TWO different YANG models for OSPF:

### 1. Cisco Native Model

**Module**: `Cisco-IOS-XE-ospf`

The Cisco Native model provides complete feature parity with IOS XE CLI commands, giving you access to all OSPF capabilities available on Cisco devices. This model is well-documented and mature, making it ideal when working exclusively in Cisco environments.

**Path Structure:**
```
/Cisco-IOS-XE-native:native/router/ospf
  /id                    # OSPF process ID
  /router-id            # OSPF router ID
  /network[]            # Network statements
    /ip                 # Network address
    /mask               # Wildcard mask
    /area               # OSPF area
```

### 2. OpenConfig Model

**Module**: `openconfig-ospf`

The OpenConfig model is a vendor-neutral standard that works across Cisco, Juniper, Arista, and other vendors. It provides a standardized structure and portable playbooks, making it valuable for multi-vendor environments. Coverage focuses on commonly-used OSPF features.

**Path Structure:**
```
/openconfig-network-instance:network-instances/network-instance/protocols/protocol/ospfv2
  /global/config
    /router-id          # OSPF router ID
  /areas/area
    /config
      /identifier       # Area ID
```

### Model Comparison

| Feature | Cisco Native | OpenConfig |
|---------|-------------|------------|
| **Router ID** | Supported | Supported |
| **Area Config** | Full support | Basic support |
| **Network Statements** | Network-based | Interface-based |
| **Authentication** | All types | Basic types |
| **Virtual Links** | Supported | Not available |
| **Stub Areas** | Supported | Supported |
| **Multi-vendor** | Cisco only | Cross-platform |

---

## Use Case: Campus OSPF Deployment

We'll configure OSPF for a typical campus network scenario:

**Single-device layout:**
```
Campus Network - Single IOS XE Device
┌──────────────────────────────┐
│  Distribution Switch         │
│  OSPF Process 1              │
│  Router ID: 1.1.1.1          │
│  Area 0 (Backbone)           │
│                              │
│  GigabitEthernet1: 10.1.1.1  │
│  Loopback0: 1.1.1.1          │
└──────────────────────────────┘
```

**Configuration Goals:**
1. Enable OSPF process 1
2. Set router ID to 1.1.1.1  
3. Configure Area 0 (backbone)
4. Enable OSPF on Loopback0
5. Enable OSPF on GigabitEthernet1
6. Verify OSPF is running

---

## Step 1: Verify gNMI Collection Availability

 **Working Directory**: `~/src/ansible-iosxe-api`

```bash
# Verify cisco.gnmi is already available from setup page
ansible-galaxy collection list | grep cisco.gnmi
```

**Expected output:**
```
cisco.gnmi    2.0.1
```

If this does not appear, return to [02-lab-setup.md](./02-lab-setup.md) and complete the gNMI collection installation workflow there.

---

## gNMI Confidence Checkpoint (GET Before SET)

Run a read-only GET before applying OSPF configuration.

```bash
cat <<EOF > src/playbooks/gnmi_checkpoint.yml
---
- name: gNMI read-only checkpoint
  hosts: iosxe_devices
  gather_facts: false
  tasks:
    - name: Read hostname over gNMI
      cisco.gnmi.gnmi_get:
        host: "{{ ansible_host }}"
        port: "{{ gnmi_port }}"
        username: "{{ ansible_user }}"
        password: "{{ ansible_password }}"
        insecure: true
        encoding: "json_ietf"
        path:
          - "/Cisco-IOS-XE-native:native/hostname"
      delegate_to: localhost
      register: gnmi_checkpoint

    - name: Show checkpoint result
      ansible.builtin.debug:
        var: gnmi_checkpoint
EOF

ANSIBLE_CONFIG=src/ansible.cfg ansible-playbook src/playbooks/gnmi_checkpoint.yml
```

Expected: successful play recap and returned hostname data.

---

## Step 2: Create OSPF Variables File

 **Working Directory**: `~/src/ansible-iosxe-api`

Create variables for OSPF configuration:

```bash
cat <<EOF > src/vars/ospf_config.yml
---
# OSPF Configuration Variables

# OSPF Process Settings
ospf_process_id: 1
ospf_router_id: "1.1.1.1"
ospf_area: 0

# Networks to Include in OSPF (for Native model)
ospf_networks:
  - network: "1.1.1.1"
    wildcard: "0.0.0.0"
    area: 0
    description: "Loopback0 - Router ID"
  
  - network: "10.1.1.0"
    wildcard: "0.0.0.255"
    area: 0
    description: "GigabitEthernet1 subnet"

# Interfaces for OSPF (for OpenConfig model)
ospf_interfaces:
  - name: "Loopback0"
    area: "0.0.0.0"
    enabled: true
    
  - name: "GigabitEthernet1"
    area: "0.0.0.0"
    enabled: true

# OpenConfig Network Instance
network_instance_name: "default"
EOF
```

**What these variables do:**
- `ospf_process_id`: OSPF process number (1-65535)
- `ospf_router_id`: Unique identifier for this router
- `ospf_networks`: Cisco Native model uses network statements
- `ospf_interfaces`: OpenConfig model configures OSPF per-interface
- Both approaches accomplish the same goal!

###  Verify Variables File

```bash
# Check file exists
ls -lh src/vars/ospf_config.yml

# Validate YAML syntax
python3 -c "import yaml; yaml.safe_load(open('src/vars/ospf_config.yml'))" && echo " Valid YAML"

# View contents
cat src/vars/ospf_config.yml
```

---

## Step 3: Configure OSPF Using Cisco Native Model

 **Working Directory**: `~/src/ansible-iosxe-api`

Create playbook using Cisco Native YANG model:

```bash
cat <<EOF > src/playbooks/gnmi_ospf_native.yml
---
- name: Configure OSPF using gNMI with Cisco Native Model
  hosts: iosxe_devices
  gather_facts: no
  vars_files:
    - ../vars/ospf_config.yml
  
  tasks:
    # ========================================================
    # Task 1: Enable OSPF Process and Set Router ID
    # ========================================================
    
    - name: Configure OSPF process with router ID (Native Model)
      cisco.gnmi.gnmi_set:
        host: "{{ ansible_host }}"
        port: "{{ gnmi_port }}"
        username: "{{ ansible_user }}"
        password: "{{ ansible_password }}"
        insecure: true
        encoding: "json_ietf"
        update:
          - path: "/Cisco-IOS-XE-native:native/router/Cisco-IOS-XE-ospf:ospf/process-id={{ ospf_process_id }}"
            val:
              id: "{{ ospf_process_id }}"
              router-id: "{{ ospf_router_id }}"
      delegate_to: localhost
      register: ospf_process_result
    
    - name: Display OSPF process configuration result
      debug:
        msg: " OSPF process {{ ospf_process_id }} configured with router-id {{ ospf_router_id }}"
      when: ospf_process_result is succeeded
    
    # ========================================================
    # Task 2: Configure OSPF Networks
    # ========================================================
    
    - name: Add networks to OSPF (Native Model)
      cisco.gnmi.gnmi_set:
        host: "{{ ansible_host }}"
        port: "{{ gnmi_port }}"
        username: "{{ ansible_user }}"
        password: "{{ ansible_password }}"
        insecure: true
        encoding: "json_ietf"
        update:
          - path: "/Cisco-IOS-XE-native:native/router/Cisco-IOS-XE-ospf:ospf/process-id={{ ospf_process_id }}/network"
            val:
              - ip: "{{ item.network }}"
                mask: "{{ item.wildcard }}"
                area: "{{ item.area }}"
      loop: "{{ ospf_networks }}"
      loop_control:
        label: "{{ item.description }}"
      delegate_to: localhost
      register: ospf_networks_result
    
    - name: Display OSPF networks configuration result
      debug:
        msg: " {{ ospf_networks | length }} network(s) added to OSPF"
    
    # ========================================================
    # Summary
    # ========================================================
    
    - name: Configuration summary
      debug:
        msg:
          - "════════════════════════════════════════════"
          - "   OSPF Configuration Complete (Native)"
          - "════════════════════════════════════════════"
          - "OSPF Process: {{ ospf_process_id }}"
          - "Router ID: {{ ospf_router_id }}"
          - "Area: {{ ospf_area }}"
          - "Networks: {{ ospf_networks | length }}"
          - "Model: Cisco-IOS-XE-native + Cisco-IOS-XE-ospf"
EOF
```

### Run the Playbook

```bash
ansible-playbook src/playbooks/gnmi_ospf_native.yml -v
```

**Expected output:**
```
PLAY [Configure OSPF using gNMI with Cisco Native Model] ****

TASK [Configure OSPF process with router ID (Native Model)] ****
ok: [sandbox]

TASK [Display OSPF process configuration result] ****
ok: [sandbox] => {
    "msg": " OSPF process 1 configured with router-id 1.1.1.1"
}

TASK [Add networks to OSPF (Native Model)] ****
ok: [sandbox] => (item=Loopback0 - Router ID)
ok: [sandbox] => (item=GigabitEthernet1 subnet)

TASK [Display OSPF networks configuration result] ****
ok: [sandbox] => {
    "msg": " 2 network(s) added to OSPF"
}

PLAY RECAP ****
sandbox : ok=4  changed=0  failed=0
```

---



## Next Steps

Congratulations! You've learned how to:
*  Understand gNMI protocol and gRPC fundamentals
*  Work with Cisco Native YANG model for OSPF
*  Use Ansible with gNMI Set and Get operations
*  Configure OSPF routing via modern APIs
*  Retrieve configuration and operational data

**In the next section**, you'll learn how to use **OpenConfig models** with gNMI for vendor-portable OSPF configuration that works across multiple network vendors.

Continue to [08-gnmi-openconfig.md](./08-gnmi-openconfig.md).

## Navigation

- Previous: [05-restconf-native.md](./05-restconf-native.md)
- Next: [08-gnmi-openconfig.md](./08-gnmi-openconfig.md)
