# Introduction to Ansible with IOS XE APIs

## Objectives

After completing this Learning Lab, you will be able to:

* Understand why API-based automation is superior to CLI-based approaches
* Configure Ansible to use NETCONF, RESTCONF, and gNMI with IOS XE devices
* Use YANG data models for configuration management
* Create Ansible playbooks that leverage modern network programmability interfaces

## Lab Overview: Single Device, Three APIs

**This is a hands-on API automation lab** designed to teach you the mechanics of modern network programmability using **one IOS XE device**.

### What Makes This Lab Unique

You'll work with **three different API protocols** on the same device, experiencing how each handles network automation differently:

| **API** | **Protocol** | **Data Format** | **Ansible Module** | **Use Case** |
|---------|-------------|-----------------|-------------------|--------------|
| **NETCONF** | XML over SSH | XML | `netconf_config`, `netconf_get` | Transactional BGP config |
| **RESTCONF** | HTTP/HTTPS | JSON | `uri` module | Interface/VLAN provisioning |
| **gNMI** | gRPC | Protocol Buffers | `cisco.gnmi.*` | Telemetry subscriptions |

### Single Device = Complete Learning Experience

**You only need ONE IOS XE sandbox device** to complete this entire lab. Each section is self-contained:

* **NETCONF BGP**: Configure routing protocols (neighbors won't establish, but config is valid)
* **RESTCONF Interfaces**: Create VLANs, configure interfaces, apply QoS
* **gNMI Telemetry**: Set up streaming telemetry, collect operational data

### What You'll Learn (The Important Part!)

This lab focuses on **API interaction patterns**, not building production networks:

* How to construct **XML payloads** for NETCONF
* How to use **HTTP methods** with RESTCONF (GET, PUT, POST, PATCH, DELETE)
* How to work with **gRPC subscriptions** for telemetry
* How **YANG data models** structure network configuration
* How Ansible modules abstract API complexity
* How to validate configuration and retrieve operational state

### What This Lab Is NOT

* Building a multi-device network topology  
* Testing BGP convergence or failover  
* Deploying production monitoring stacks  
* Validating traffic flow or security policies  

**This is an API automation skills lab**, teaching you how to programmatically interact with network devices using modern protocols.
## Why APIs Instead of CLI?

Traditional CLI-based network automation has significant limitations:

### CLI Limitations
* **Screen scraping**: Parsing unstructured text output is brittle and error-prone
* **No data validation**: CLI doesn't validate against a schema
* **Human-centric**: Commands designed for human interaction, not programmatic access
* **Inconsistent output**: Format changes between software versions break automation

### API Advantages
* **Structured data**: JSON/XML responses with predictable formats
* **Schema validation**: YANG models ensure data integrity
* **Machine-optimized**: Designed for programmatic interaction
* **Version independence**: YANG models provide consistent abstraction
* **Transaction support**: Atomic operations with rollback capabilities
* **Easier to scale**: Reusable playbooks and data models let you apply the same workflow across many devices consistently
* **Faster troubleshooting**: Structured error responses and deterministic payloads make failures easier to isolate and fix
* **Better observability**: APIs provide clear state/config data retrieval paths for validation and post-change checks

## IOS XE Programmability Interfaces

Modern IOS XE devices support three primary programmability interfaces:

### 1. NETCONF (Network Configuration Protocol)
* **Transport**: SSH (port 830)
* **Encoding**: XML
* **Use Cases**: Configuration management, operational data retrieval
* **Strengths**: Transaction support, candidate configuration, rollback

### 2. RESTCONF (RESTful NETCONF)
* **Transport**: HTTPS (port 443)
* **Encoding**: JSON or XML
* **Use Cases**: RESTful operations, integration with web applications
* **Strengths**: HTTP-based, easy to debug, stateless

### 3. gNMI (gRPC Network Management Interface)
* **Transport**: gRPC (port 9339)
* **Encoding**: Protocol Buffers (protobuf)
* **Use Cases**: High-performance telemetry, streaming configuration
* **Strengths**: Efficient binary protocol, streaming, bidirectional

## YANG Data Models

All three interfaces use **YANG** (Yet Another Next Generation) data models:

* **Standardized**: IETF and OpenConfig define common models
* **Vendor-specific**: Cisco provides IOS XE native models
* **Self-documenting**: Models describe structure, types, and constraints
* **Tool-friendly**: Models can generate code and documentation

## Ansible and Network APIs

Ansible provides modules to interact with each programmability interface:

* `netconf_config` / `netconf_get`: NETCONF operations
* `ansible.netcommon.restconf_config` / `restconf_get`: RESTCONF operations  
* `gnmi_config` / `gnmi_get`: gNMI operations (via community collections)

## What's Next?

Next, complete environment setup on Page 2, then continue through the protocol labs:

1. **Page 2**: Environment setup and tooling checks
2. **Section 3**: Configure BGP using NETCONF
3. **Section 4**: Configure interfaces using RESTCONF  
4. **Section 5/6**: Configure OSPF using gNMI (Native and OpenConfig)

Each section demonstrates real API automation workflows with runnable Ansible playbooks.

Let's begin!

## Navigation

- Next: [02-lab-setup.md](./02-lab-setup.md)
