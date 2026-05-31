# Ansible with NETCONF + BGP Configuration

## Overview

In this section, you'll learn how to use Ansible with NETCONF to configure BGP on IOS XE devices. This module demonstrates the fundamentals of using NETCONF with YANG data models through Ansible, focusing on a practical BGP routing configuration use case.

NETCONF provides transaction support and candidate configuration capabilities, making it ideal for structured configuration tasks like routing protocol setup.

## What You'll Accomplish 

By the end of this section, you will:

* **Configure BGP** using NETCONF with XML-based YANG models
* **Learn NETCONF operations**: `netconf_config`, `netconf_get` Ansible modules
* **Use transactional configuration**: Atomic operations with validation
* **Create BGP neighbors**: Configure both iBGP and eBGP peer relationships
* **Advertise networks**: Add network statements to BGP
* **Retrieve BGP state**: Query operational data via NETCONF
* **Understand XML/YANG**: Work with Cisco-IOS-XE-bgp YANG model
* **Verify configuration**: Validate BGP setup on the device

**Single Device Lab**: All tasks work on **one IOS XE device** - no additional routers needed!

## Important Limitations 

**What This Lab CANNOT Do:**

*  **Establish working BGP peerings**: BGP neighbors (10.0.0.2, 10.0.0.3) are example configurations only. They will remain in "Idle" state without actual peer devices.
*  **Exchange BGP routes**: Without real neighbors, no route exchange occurs.
*  **Test BGP convergence**: Cannot demonstrate BGP failover or path selection.

**Why This Doesn't Matter:**

This lab teaches **API automation mechanics**, not BGP protocol operation. You're learning:
- How to construct NETCONF XML payloads
- How to use YANG models for configuration
- How Ansible interacts with NETCONF APIs
- How to validate and retrieve configuration

The BGP neighbors showing as "Idle" is **expected and perfectly fine for learning**. The configuration is valid and demonstrates real-world NETCONF automation patterns.


## NETCONF Confidence Checkpoint (GET Before SET)

Goal: confirm NETCONF authentication and read access before applying configuration.

Run from `~/src/ansible-iosxe-api`:

```bash
cat <<EOF > src/playbooks/netconf_checkpoint.yml
---
- name: NETCONF read-only checkpoint
  hosts: iosxe_devices
  gather_facts: false
  tasks:
    - name: Read hostname using NETCONF
      ansible.netcommon.netconf_get:
        source: running
        filter: |
          <filter>
            <native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
              <hostname/>
            </native>
          </filter>
      register: netconf_checkpoint

    - name: Show checkpoint result
      ansible.builtin.debug:
        var: netconf_checkpoint.stdout
EOF

ANSIBLE_CONFIG=src/ansible.cfg ansible-playbook src/playbooks/netconf_checkpoint.yml
```

Expected: successful play recap and hostname XML returned.

## What is NETCONF?

NETCONF (Network Configuration Protocol) is a network management protocol that provides mechanisms to:
* **Install, manipulate, and delete** configuration of network devices
* **Use XML** for data encoding
* **Operate** over secure transport (SSH)
* **Support** transactional operations with rollback
* **Validate** configurations before applying them

### NETCONF Key Features

1. **Operations**: `<get>`, `<get-config>`, `<edit-config>`, `<copy-config>`, `<delete-config>`
2. **Datastores**: Running, Candidate, Startup
3. **Capabilities**: Device advertises supported features
4. **Error Handling**: Structured error responses

## Understanding YANG Data Models

YANG (Yet Another Next Generation) is a data modeling language used to model configuration and state data for NETCONF:

* **Hierarchical structure**: Data organized in a tree
* **Types and constraints**: Strong typing and validation rules
* **Reusable**: Modules can be imported and extended
* **Self-documenting**: Models describe their structure

### Example YANG Hierarchy for BGP

```
module: Cisco-IOS-XE-bgp
  +--rw native
     +--rw router
        +--rw bgp
           +--rw id              // BGP AS number
           +--rw bgp
           |  +--rw log-neighbor-changes?
           |  +--rw router-id?
           +--rw neighbor[]      // BGP neighbors
           |  +--rw id
           |  +--rw remote-as
           |  +--rw description?
           |  +--rw update-source?
           +--rw network[]       // Networks to advertise
              +--rw number
              +--rw mask?
```

## Use Case: Basic BGP Configuration

We'll configure a basic BGP setup suitable for an enterprise edge router:

1. **Enable BGP** with an AS number
2. **Configure router-id** for BGP identification
3. **Add BGP neighbors** (peers)
4. **Advertise networks** into BGP
5. **Verify the configuration**

### Network Scenario

```
                 ┌─────────────────────┐
                 │  Your Sandbox       │
                 │  IOS XE Device      │
                 │  C9000V             │
                 │  Switch-ID: 1.1.1.1 │
                 └─────────────────────┘
```

## Step 1: Understanding NETCONF + YANG for BGP

The BGP configuration uses the **Cisco-IOS-XE-bgp** YANG model. Key paths:

```xml
/native/router/bgp
  /id                        <!-- AS number -->
  /bgp/router-id             <!-- BGP router ID -->
  /bgp/log-neighbor-changes  <!-- Log neighbor state changes -->
  /neighbor                  <!-- BGP neighbor configuration -->
    /id                      <!-- Neighbor IP address -->
    /remote-as               <!-- Neighbor AS number -->
    /description             <!-- Neighbor description -->
  /network                   <!-- Networks to advertise -->
    /number                  <!-- Network address -->
    /mask                    <!-- Network mask -->
```

### How NETCONF Works with YANG

1. **YANG defines** the data structure (what can be configured)
2. **XML encoding** is used to represent the data
3. **NETCONF carries** the XML payload over SSH
4. **Device validates** against YANG model
5. **Configuration applied** if valid

## Step 2: Create BGP Variables File

Create `src/vars/bgp_config.yml` with your BGP parameters:

```bash
cat <<EOF > src/vars/bgp_config.yml
---
# BGP Configuration
bgp_asn: 65001

# BGP Neighbors
bgp_neighbors:
  - ip_address: "10.0.0.2"
    remote_as: 65001
    description: "iBGP Peer Router-2"
    
  - ip_address: "10.0.0.3"
    remote_as: 65002
    description: "eBGP Peer to AS 65002"

# Networks to Advertise
bgp_networks:
  - network: "192.168.1.0"
    mask: "255.255.255.0"
    
  - network: "192.168.2.0"
    mask: "255.255.255.0"
EOF
```

**Note:** These are example values. Adjust them based on your actual environment or use them as-is for learning purposes.

## Step 3: Create NETCONF Playbook for BGP

Create `src/playbooks/netconf_bgp.yml`:

```bash
cat <<EOF > src/playbooks/netconf_bgp.yml
---
- name: Configure BGP using NETCONF and YANG
  hosts: iosxe_devices
  gather_facts: no
  vars_files:
    - ../vars/bgp_config.yml
  
  tasks:
    # =======================================================
    # Task 1: Enable BGP and Configure Basic Parameters
    # =======================================================
    
    - name: Configure BGP process with AS number and router-id
      ansible.netcommon.netconf_config:
        content: |
          <config xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
            <native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
              <router>
                <bgp xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-bgp">
                  <id>{{ bgp_asn }}</id>
                  <bgp>
                    <log-neighbor-changes/>
                  </bgp>
                </bgp>
              </router>
            </native>
          </config>
      register: bgp_basic_result

    - name: Display BGP basic configuration result
      debug:
        msg: " BGP AS {{ bgp_asn }} configured"
      when: bgp_basic_result.changed

    # =======================================================
    # Task 2: Configure BGP Neighbors
    # =======================================================
    
    - name: Configure BGP neighbors
      ansible.netcommon.netconf_config:
        content: |
          <config xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
            <native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
              <router>
                <bgp xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-bgp">
                  <id>{{ bgp_asn }}</id>
                  <neighbor>
                    <id>{{ item.ip_address }}</id>
                    <remote-as>{{ item.remote_as }}</remote-as>
                    <description>{{ item.description }}</description>
                  </neighbor>
                </bgp>
              </router>
            </native>
          </config>
      loop: "{{ bgp_neighbors }}"
      loop_control:
        label: "{{ item.ip_address }} (AS {{ item.remote_as }})"
      register: bgp_neighbor_result

    - name: Display BGP neighbor configuration result
      debug:
        msg: " {{ bgp_neighbors | length }} BGP neighbor(s) configured"

    # =======================================================
    # Task 3: Advertise Networks into BGP
    # =======================================================
    
    - name: Configure BGP networks to advertise
      ansible.netcommon.netconf_config:
        content: |
          <config xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
            <native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
              <router>
                <bgp xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-bgp">
                  <id>{{ bgp_asn }}</id>
                  <address-family>
                    <no-vrf>
                      <ipv4>
                        <af-name>unicast</af-name>
                        <ipv4-unicast>
                          <network>
                            <number>{{ item.network }}</number>
                            <mask>{{ item.mask }}</mask>
                          </network>
                        </ipv4-unicast>
                      </ipv4>
                    </no-vrf>
                  </address-family>
                </bgp>
              </router>
            </native>
          </config>
      loop: "{{ bgp_networks }}"
      loop_control:
        label: "{{ item.network }}/{{ item.mask }}"
      register: bgp_network_result

    - name: Display BGP network advertisement result
      debug:
        msg: " {{ bgp_networks | length }} network(s) configured for BGP advertisement"

    # =======================================================
    # Final Summary
    # =======================================================
    
    - name: Display configuration summary
      debug:
        msg:
          - "══════════════════════════════════════════════════════"
          - "    BGP Configuration Complete via NETCONF"
          - "══════════════════════════════════════════════════════"
          - "BGP AS Number: {{ bgp_asn }}"
          - "BGP Neighbors: {{ bgp_neighbors | length }}"
          - "Networks Advertised: {{ bgp_networks | length }}"
          - "══════════════════════════════════════════════════════"
EOF
```

## Step 4: Run the BGP Configuration Playbook

Execute the playbook to configure BGP:

```bash
# Navigate to your project directory
cd ~/src/ansible-iosxe-api

# Run the playbook
ansible-playbook src/playbooks/netconf_bgp.yml -v
```

### Expected Output

You should see output similar to:

```
PLAY [Configure BGP using NETCONF and YANG] **********************************

TASK [Configure BGP process with AS number and router-id] ********************
changed: [sandbox-iosxe]

TASK [Display BGP basic configuration result] *********************************
ok: [sandbox-iosxe] => {
    "msg": " BGP AS 65001 configured"
}

TASK [Configure BGP neighbors] ************************************************
changed: [sandbox-iosxe] => (item=10.0.0.2 (AS 65001))
changed: [sandbox-iosxe] => (item=10.0.0.3 (AS 65002))

TASK [Display BGP neighbor configuration result] ******************************
ok: [sandbox-iosxe] => {
    "msg": " 2 BGP neighbor(s) configured"
}

TASK [Configure BGP networks to advertise] ************************************
changed: [sandbox-iosxe] => (item=192.168.1.0/255.255.255.0)
changed: [sandbox-iosxe] => (item=192.168.2.0/255.255.255.0)

TASK [Display BGP network advertisement result] *******************************
ok: [sandbox-iosxe] => {
    "msg": " 2 network(s) configured for BGP advertisement"
}

TASK [Display configuration summary] ******************************************
ok: [sandbox-iosxe] => {
    "msg": [
        "══════════════════════════════════════════════════════",
        "    BGP Configuration Complete via NETCONF",
        "══════════════════════════════════════════════════════",
        "BGP AS Number: 65001",
        "BGP Neighbors: 2",
        "Networks Advertised: 2",
        "══════════════════════════════════════════════════════"
    ]
}

PLAY RECAP ********************************************************************
sandbox-iosxe : ok=7 changed=3 unreachable=0 failed=0 skipped=0
```

## Step 5: Verify BGP Configuration

**Create Verification Playbook**

Create `src/playbooks/verify_bgp.yml`:

```bash
cat <<EOF > src/playbooks/verify_bgp.yml
---
- name: Verify BGP Configuration via NETCONF
  hosts: iosxe_devices
  gather_facts: no
  
  tasks:
    - name: Retrieve BGP configuration using NETCONF
      ansible.netcommon.netconf_get:
        source: running
        filter: |
          <filter>
            <native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
              <router>
                <bgp xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-bgp"/>
              </router>
            </native>
          </filter>
        display: pretty
      register: bgp_config

    - name: Display retrieved BGP configuration
      debug:
        var: bgp_config.stdout

    - name: Save BGP configuration to file
      copy:
        content: "{{ bgp_config.stdout }}"
        dest: "./bgp_config_{{ ansible_date_time.epoch }}.xml"
      delegate_to: localhost
EOF
```

Run the verification:

```bash
ansible-playbook src/playbooks/verify_bgp.yml
```

## Understanding NETCONF Operations

This playbook demonstrates several NETCONF concepts:

### 1. XML Encoding with YANG Namespaces

```xml
<config xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
    ...
  </native>
</config>
```

* **NETCONF namespace**: `urn:ietf:params:xml:ns:netconf:base:1.0`
* **YANG model namespace**: `http://cisco.com/ns/yang/Cisco-IOS-XE-native`

### 2. The `<edit-config>` Operation

`ansible.netcommon.netconf_config` uses the NETCONF `<edit-config>` operation:
* **Target**: Running datastore (immediate effect)
* **Operation**: Merge (adds to existing config, doesn't replace)
* **Validation**: YANG model validates before applying

### 3. The `<get-config>` Operation

`ansible.netcommon.netconf_get` uses the NETCONF `<get-config>` operation:
* **Source**: Running configuration
* **Filter**: XML subtree filter to retrieve specific data
* **Output**: XML formatted configuration

### 4. Idempotency

Running the playbook multiple times produces the same result:
* First run: Creates BGP configuration
* Subsequent runs: NETCONF recognizes existing configuration, no changes made
* Result: Safe to re-run without breaking config

## Comparison: NETCONF vs CLI

|  | **CLI** | **NETCONF** |
|---|---|---|
| **Format** | Text commands | Structured XML |
| **Validation** | After execution | Before execution |
| **Error Handling** | Text parsing required | Structured error codes |
| **Rollback** | Manual | Built-in support |
| **Transactions** | No | Yes (atomic operations) |
| **Programmatic** | Screen scraping | Native API |


---



## Next Steps

Congratulations! You've learned how to:
*  Understand NETCONF protocol fundamentals  
*  Work with Cisco Native YANG model for BGP
*  Use Ansible to configure devices via NETCONF
*  Structure XML payloads based on YANG hierarchy
*  Verify configurations programmatically

**In the next section**, you'll learn how to use **RESTCONF** with Cisco Native and IETF models for single-device sandbox interface and VLAN configuration.

Continue to [05-restconf-native.md](./05-restconf-native.md).

## Navigation

- Previous: [02-lab-setup.md](./02-lab-setup.md)
- Next: [05-restconf-native.md](./05-restconf-native.md)
