# Lab Environment Setup

## Important: Single Device Lab 

**You only need ONE IOS XE device** (from DevNet Sandbox) to complete this entire lab. All three sections (NETCONF, RESTCONF, gNMI) use the same single device. You do **not** need multiple routers, switches, or external collectors.

**What You'll Set Up:**
*  One IOS XE sandbox device with APIs enabled
*  Ansible on your local machine or lab environment
*  Required Ansible collections for NETCONF, RESTCONF, and gNMI
*  Inventory file pointing to your single sandbox device

## Understanding Your Lab Environment

### Where Will You Run Commands?

For this lab, you'll need access to a **terminal/command line** where you can:
- Run bash/shell commands
- Create files and directories
- Run Ansible playbooks

**DevNet Sandbox Environment:**

Use only the IOS XE 8k/9k single instance reservable sandbox for this lab. Run commands from the sandbox terminal/developer workstation provided by that reservation.

**Throughout this lab:** Commands marked with  should be run on your Ansible control machine (terminal/workstation), NOT on the IOS XE device.

**Connect to Your Sandbox**
- Open a new terminal tab in your editor using the + symbol. Use the credentials from your reservation email or the I/O tab in your reservation window 

---

##  Quick Reference Card

Keep this handy as you work through the lab:

| Task | Command | What it does |
|------|---------|--------------|
| Check current directory | `pwd` | Shows where you are in the filesystem |
| Change directory | `cd ~/src/ansible-iosxe-api` | Navigate to working directory |
| List files | `ls -lh` | Show files with details |
| View file contents | `cat filename.yml` | Display entire file |
| Check first lines | `head -5 filename.yml` | Show first 5 lines |
| Edit file in lab editor | Open file from Explorer panel | Edit directly in the browser/workstation editor |
| Update sandbox credentials | Open `src/inventory/hosts.yml` in Explorer | Replace placeholders with your sandbox host/user/password |
| Create directory | `mkdir dirname` | Make a new folder |
| Validate YAML | `python3 -c "import yaml; yaml.safe_load(open('file.yml'))"` | Check YAML syntax |

**File Creation Pattern Used in This Lab:**

```
cat <<EOF > src/filename.yml
[content goes here]
EOF
```
With an arrow that you can click to easily run the block of code in the terminal window.

 **What this does:** Creates a file and writes everything between the `cat >` line and `EOF` into it. Just copy the entire block and paste into your terminal.

## Step 1: Working with Ansible

**Note:** The DevNet Sandbox pre-configured environment, already has Ansible installed:

Note: ensure you are back in your FIRST terminal tab!

```bash
# Confirm the installed Ansible version
ansible --version
```

**Ansible is already installed**, so you'll see an output similar to the fofllowing where "..." denotes additional ansible modules and their versions:
```
ansible [core 2.19.5]
  ...
  python version = 3.11.14
  ...
```

Versions used for this lab:
- `ansible-core` `2.19.5`
- Python `3.11.14`

## Step 2: Review Required Ansible Collections

Ansible collections provide modules for NETCONF, RESTCONF, and gNMI. The Ansible NETCONF and RESTCONF (and util) modules are already installed for you.

Install them now:

```bash
# # Install network common collection (required for NETCONF/RESTCONF) already installed for you in this lab
# ansible-galaxy collection install ansible.netcommon:8.2.0

# # Install utilities collection (provides helpful filters) already installed for you in this lab
# ansible-galaxy collection install ansible.utils:6.0.0

# Install Cisco gNMI collection from source
git clone https://github.com/CiscoDevNet/ansible-gnmi.git
cd ansible-gnmi
ansible-galaxy collection build
ansible-galaxy collection install cisco-gnmi-*.tar.gz
cd ..

# Python dependencies for cisco.gnmi collection
pip3 install grpcio grpcio-tools protobuf cryptography
```

**What these do:**
- `ansible.netcommon` - Provides `netconf_config`, `netconf_get` modules
- `ansible.utils` - Data manipulation and validation utilities
- `cisco.gnmi` - gNMI GET/SET/SUBSCRIBE modules used for connectivity testing

###  Verify Collections are Installed

```bash
# List all installed collections
ansible-galaxy collection list
```

**Expected output** (example):
```
developer:src > ansible-galaxy collection list

# /usr/local/lib/python3.11/site-packages/ansible/_internal/ansible_collections
Collection                               Version
---------------------------------------- -------
ansible._protomatter                     2.19.5

# /usr/local/lib/python3.11/site-packages/ansible_collections
Collection                               Version
---------------------------------------- -------
amazon.aws                               10.1.2
ansible.netcommon                        8.2.0
ansible.posix                            2.1.0
ansible.utils                            6.0.0
ansible.windows                          3.3.0
arista.eos                               12.0.0
awx.awx                                  24.6.1
```

**Version checks to confirm (required for this lab):**
- `ansible.netcommon` must be `8.2.0`
- `ansible.utils` must be `6.0.0`

---

## Step 3: Verify APIs on IOS XE Device (Optional)

**Note:** DevNet Sandbox devices typically have NETCONF, RESTCONF, and gNMI pre-configured and enabled. You can **skip this step** if using a sandbox.

If you want to confirm API services directly on the device:

###  Check on IOS XE Device

SSH to your IOS XE device:

<!-- add back in after CL 26 -->

<!-- ```bash
# SSH to the device using your sandbox reservation values
ssh <USERNAME>@<SANDBOX_HOSTNAME>
```

![Example login to sandbox from Learning Lab](./assets/images/access-sandbox-from-learning-lab.png) -->

Once logged in to the IOS XE CLI, check API status:

```cisco
! Check NETCONF
show running-config | include netconf

! Check RESTCONF  
show running-config | include restconf

! Check gNMI
show running-config | include gnxi
```

**Expected output:**
```
netconf-yang
restconf
gnxi
 gnxi secure-server
 gnxi secure-port 9339
```

###  Enable APIs (If Needed)

Only run these if APIs are NOT already enabled:

```cisco
! Enter configuration mode
configure terminal

! Enable NETCONF
netconf-yang
netconf-yang ssh port 830

! Enable RESTCONF
restconf

! Enable gNMI
gnxi
 gnxi secure-init
 gnxi secure-server
 gnxi secure-port 9339
 
! Enable HTTPS (required for RESTCONF)
ip http secure-server
ip http authentication local

! Save configuration
end
write memory
```

To return to the `developer:src > `   
prompt, type "exit" + ENTER. Now you'll be ready to continue the lab.

---
## Step 4: Create Project Directory Structure

We'll create a dedicated directory for this lab with subdirectories for organization.

```bash
# Create main project directory
mkdir -p ~/src/ansible-iosxe-api

# Navigate into it
cd ~/src/ansible-iosxe-api

# Verify you're in the correct location
pwd
```

**Expected output:**
```
/home/developer/src/ansible-iosxe-api
```

Now create subdirectories:

```bash
# Create subdirectories for organization
mkdir -p ~/src/inventory ~/src/playbooks ~/src/vars
```

**What these directories are for:**
- `~/src/inventory/` - Device inventory and credentials
- `~/src/playbooks/` - Ansible playbooks for NETCONF, RESTCONF, gNMI
- `~/src/vars/` - Configuration variables (BGP, interfaces, telemetry)

###  Verify Directory Structure

```bash
# List the directories
ls -lh
```

**Example output (note the date will vary):**
```
drwxr-xr-x  2 user user 4096 May  7 10:30 src/inventory
drwxr-xr-x  2 user user 4096 May  7 10:30 src/playbooks
drwxr-xr-x  2 user user 4096 May  7 10:30 src/vars
```

###  Complete Directory Tree

By the end of this lab, your directory will look like this:

```
~/src/ansible-iosxe-api/
├── ansible.cfg          # Ansible configuration file
├── inventory/
│   └── hosts.yml            # Device inventory with credentials
├── vars/
│   ├── bgp_config.yml      # BGP variables (Section 2: NETCONF)
│   ├── interface_config.yml # Interface variables (Section 3: RESTCONF)
│   └── telemetry_config.yml # Telemetry variables (Section 4: gNMI)
└── playbooks/
    ├── netconf_bgp.yml     # NETCONF BGP playbook
    ├── restconf_*.yml      # RESTCONF playbooks
    └── gnmi_*.yml          # gNMI playbooks
```

**Pro Tip:** Keep this terminal window open for the entire lab. All future commands assume you're in `~/src/ansible-iosxe-api`.

---

## Step 5: Create Inventory File (Device Credentials)

The inventory file tells Ansible about your IOS XE device and how to connect to it.

 **Make sure you're in:** `~/src/ansible-iosxe-api`

```bash
# Verify working directory
pwd
# Should show: /home/developer/src/ansible-iosxe-api
```



**Create the Inventory Template**

First, create the inventory file with placeholder values. 


```bash
cat <<EOF > src/inventory/hosts.yml
all:
  children:
    iosxe_devices:
      hosts:
        sandbox:
          # REPLACE THESE THREE VALUES WITH YOUR SANDBOX CREDENTIALS
          ansible_host: <SANDBOX_HOSTNAME>
          ansible_user: <USERNAME>
          ansible_password: <SANDBOX_PASSWORD>
          
          # NETCONF settings (don't change these)
          ansible_network_os: cisco.ios.ios
          ansible_connection: ansible.netcommon.netconf
          netconf_port: 830
          
          # RESTCONF settings (don't change these)
          restconf_port: 443
          restconf_root: /restconf/data
          
          # gNMI settings (don't change these)
          gnmi_port: 9339
          gnmi_insecure: true
EOF
```

**What this command does:**
- Creates `src/inventory/hosts.yml` file
- Contains placeholders that you'll replace in the next step
- Uses YAML format (indentation matters!)

**Update with Your Sandbox Credentials**

Note: Receive your sandbox credentails from your proctor.

<!-- ADD back in after CL US 26

Now edit the file to add your **actual** credentials from your sandbox reservation email. -->

**Find these in your sandbox reservation:**
- `<SANDBOX_HOSTNAME>` - Hostname (e.g., `devnetsandboxiosxec9k.cisco.com`,  `10.1.1.4`) (this is the host value, not the username)
- `<USERNAME>` - Your username (e.g., `developer`, `admin`)
- `<SANDBOX_PASSWORD>` - Your password (from reservation email or in your DevNet Sandbox Reservation I/O tab)

**Update using the Learning Lab editor (recommended):**
1. In the Explorer panel, open `src/inventory/hosts.yml`.
2. Find the three placeholder lines under `sandbox:`.
3. Replace placeholders with your real values:
  - Replace `<SANDBOX_HOSTNAME>` with your hostname
  - Replace `<USERNAME>` with your username
  - Replace `<SANDBOX_PASSWORD>` with your password
4. Save the file from the editor.

**Editor workflow used in this lab:**
- Use the Learning Lab editor (Explorer panel) for credential and file edits.
- Use terminal commands only for validation and playbook execution.

**Verify Your Inventory File**
 
```bash
# View the file (your password will be visible - this is OK for lab purposes)
cat src/inventory/hosts.yml
```

**Expected output** (with YOUR actual values, not placeholders):

```yaml
all:
  children:
    iosxe_devices:
      hosts:
        sandbox:
          ansible_host: devnetsandboxiosxec9k.cisco.com  # ← Should show YOUR hostname
          ansible_user: developer                          # ← Should show YOUR username
          ansible_password: C1sco12345                     # ← Should show YOUR password
          
          # NETCONF settings (don't change these)
          ansible_network_os: cisco.ios.ios
          ansible_connection: ansible.netcommon.netconf
          netconf_port: 830
          
          # RESTCONF settings (don't change these)
          restconf_port: 443
          restconf_root: /restconf/data
          
          # gNMI settings (don't change these)
          gnmi_port: 9339
          gnmi_insecure: true
```

**Important checks:**
- The three values should NOT have `<angle brackets>` anymore
-  Values should match your sandbox reservation exactly
-  YAML indentation should be intact (2 spaces per level)

**Test YAML syntax:**

```bash
# Validate YAML format (if this runs without errors, you're good!)
python3 -c "import yaml; yaml.safe_load(open('src/inventory/hosts.yml'))" && echo "✓ Valid YAML syntax"
```

**If successful, you'll see:** `✓ Valid YAML syntax`  
**If there's an error:** Python will show a syntax error message

###  Troubleshooting

**Problem:** File still has `<PLACEHOLDERS>`

**Solution:** Reopen `src/inventory/hosts.yml` in the Learning Lab editor and replace the three placeholder values with your sandbox credentials, then save.

**Problem:** YAML syntax error

**Solution:** Check that:
- Indentation uses spaces (not tabs)
- Colons have a space after them (`: ` not `:`)
- No extra characters were added
- Delete and recreate the file if needed

---

## Step 7: Create Ansible Configuration File

 **Make sure you're in:** `~/src/ansible-iosxe-api`

```bash
# Verify working directory
pwd
```

Create the Ansible configuration file:

```bash
cat <<EOF > src/ansible.cfg
[defaults]
inventory = src/inventory/hosts.yml
host_key_checking = False
timeout = 60
gathering = explicit
retry_files_enabled = False

[persistent_connection]
connect_timeout = 60
command_timeout = 60
EOF
```

**What this does:**
- Sets default inventory file location
- Disables SSH host key checking (OK for lab)
- Sets reasonable timeouts for API connections
- Configures persistent connection settings

**Verify ansible.cfg Was Created**

```bash
# Check file exists
ls -lh src/ansible.cfg

# View contents
cat src/ansible.cfg
```

**Expected output:**
```
[defaults]
inventory = src/inventory/hosts.yml
host_key_checking = False
timeout = 60
gathering = explicit
retry_files_enabled = False

[persistent_connection]
connect_timeout = 60
command_timeout = 60
```

**Test Your Ansible Setup**

Let's verify Ansible can read your inventory:

```bash
# List all hosts in inventory
ansible all --list-hosts
```

**Expected output:**
```
  hosts (1):
    sandbox
```

 If you see this, your setup is correct!

 **If you see errors:**
- Check that `ansible.cfg` exists in current directory
  - If needed, run with `ANSIBLE_CONFIG=src/ansible.cfg`
- Check that `src/inventory/hosts.yml` has valid YAML syntax
- Verify you're in the `~/src/ansible-iosxe-api` directory

---

##  Setup Complete!

Congratulations! Your Ansible environment is now ready.

###  What You've Accomplished

-  Installed Ansible and required collections
-  Verified APIs are accessible on your IOS XE device
-  Created organized project directory structure
-  Configured inventory with your device credentials
-  Created Ansible configuration file
-  Tested that Ansible can see your device

###  Your Working Directory

You should now have this structure:

```
~/src/ansible-iosxe-api/
├── /ansible.cfg        Created
├── /inventory/
│   └── hosts.yml          Created (with your credentials)
├── /playbooks/            Created (empty, we'll add playbooks in next sections)
└── /vars/                 Created (empty, we'll add variables in next sections)
```

### What's Next?

Now you're ready to explore the three API protocols with Ansible.

**Section 3: NETCONF + BGP with Cisco Native Models**
- Learn XML-based NETCONF operations
- Configure BGP using Cisco-IOS-XE-native YANG model
- Use `ansible.netcommon.netconf_config` module
- **Files you'll create:**
  - `src/vars/bgp_config.yml` - BGP parameters
  - `src/playbooks/netconf_bgp.yml` - Main BGP configuration playbook
  - `src/playbooks/verify_bgp.yml` - Verification playbook
- Jump Here Now: [03-netconf-native.md](03-netconf-native.md)

**Section 4: RESTCONF + Sandbox with Native/IETF Models**
- Learn JSON-based RESTCONF operations
- Configure VLANs, interfaces, QoS for sandbox
- Use Cisco-IOS-XE-native and IETF standard models
- **Files you'll create:**
  - `src/vars/sandbox_config.yml` - Sandbox configuration data
  - `src/playbooks/restconf_sandbox_deploy.yml` - Deployment playbook
  - `src/playbooks/restconf_native_model.yml` - Native model examples
  - `src/playbooks/restconf_ietf_model.yml` - IETF model examples
- Jump Here Now: [05-restconf-native.md](./05-restconf-native.md)

**Section 5: gNMI + OSPF with Cisco Native Models**
- Learn gRPC-based gNMI operations
- Configure OSPF routing with Cisco-IOS-XE-ospf model
- Use `cisco.gnmi` collection
- **Files you'll create:**
  - `src/vars/ospf_config.yml` - OSPF parameters
  - `src/playbooks/gnmi_ospf_native.yml` - Native OSPF configuration
- Jump Here Now: [07-gnmi-native.md](./07-gnmi-native.md)

**Section 6: gNMI + OSPF with OpenConfig Models**
- Configure OSPF using openconfig-network-instance model
- Compare gNMI Native vs OpenConfig approaches
- **Files you'll create:**
  - `src/playbooks/gnmi_ospf_openconfig.yml` - OpenConfig OSPF playbook
- Jump Here Now: [08-gnmi-openconfig.md](./08-gnmi-openconfig.md)

### Remember

- **Keep your terminal in `~/src/ansible-iosxe-api`** throughout the lab
- All future file creation commands assume you're in this directory
- You can always check with `pwd` command
- Each section builds on previous ones - complete them in order

**Ready? Let's dive into NETCONF!** [Continue to Section 3: NETCONF Native](./03-netconf-native.md)

## Navigation

- Previous: [01-intro.md](./01-intro.md)
- Next: [03-netconf-native.md](./03-netconf-native.md)
