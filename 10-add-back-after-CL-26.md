# Add Back After CL 26

This file contains content that was previously inside HTML comments in active pages and was removed from the learner-facing flow.

## From 01-intro.md

### Block 1

COMMENT BACK IN AFTER CL 26

Use the **Catalyst 8/9k Single Instance Sandbox** (reservable) so you can make and validate configuration changes safely during the lab.

![Sandbox Reservation Example](./assets/images/sandbox-reservation-example.png)

### Block 2

## Prerequisites and Setup

Complete environment setup on [02-lab-setup.md](./02-lab-setup.md) before running protocol playbooks.

## From 02-lab-setup.md

### Block 1

## Prerequisites

To complete this lab, reserve the Cisco DevNet **Catalyst 8/9k Single Instance Sandbox**.

### Access to a DevNet Sandbox Environment

Follow these steps to reserve the Catalyst 8/9k single instance sandbox.

**Step 1: Navigate to DevNet Sandbox**
- Open your browser and go to [https://devnetsandbox.cisco.com](https://devnetsandbox.cisco.com)
- Sign in with your Cisco.com account

**Step 2: Locate the Sandbox**
- In the search bar, type "Single"
- Select "Catalyst 8/9k Single Instance Sandbox"
- Click the sandbox tile "Launch" button to view details

![Catalyst 8/9k Single Instance Sandbox in DevNet Sandboxes](./assets/images/find-ansible-sandbox.png)

**Step 3: Reserve Your Sandbox**
- Select your reservation duration

![Sandbox Duration](./assets/images/sandbox-duration.png)

- In the Inputs tab on the left side, select the device type

![Select Device Type](./assets/images/sandbox-device-type.png)

- Click "Review Summary"
- Click "Reserve" to confirm

**Step 4: Wait for Sandbox Activation**
- Reservable sandboxes typically take 5-15 minutes to provision
- You will receive an email when your sandbox is ready
- The email includes VPN details (if required), hostnames, and credentials

![Open a Second Terminal Tab](./assets/images/sandbox-new-terminal-tab.png)

Then, in the new Terminal window, type:

```bash
ssh <USERNAME>@devnetsandboxiosxec9k.cisco.com
```

**Example Sandbox Credentials:**
```
Hostname: devnetsandboxiosxec9k.cisco.com
NETCONF Port: 830
RESTCONF Port: 443
gNMI Port: 9339
Username: <provided in your sandbox reservation>
Password: <provided in your sandbox reservation>
```

![Sandbox Reservation Example](./assets/images/sandbox-reservation-example.png)

Note: Always-On sandboxes are shared resources. For dedicated environments and persistent configuration changes, use a reservable sandbox.

### Software Requirements

- Python 3.11.14
- ansible-core 2.19.5
- ansible.netcommon 8.2.0
- ansible.utils 6.0.0
- Basic understanding of YAML syntax
- Familiarity with YANG data models (recommended)

---

### Block 2

Remove the following after Cisco Live US 26

### Block 3

Remove the following after Cisco Live US 26

### Block 4

ADD back in after CL US 26

Now edit the file to add your **actual** credentials from your sandbox reservation email.
