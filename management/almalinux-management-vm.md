<!--
Organized from: STEP 2 almalinux_installation_notes(3).txt
Source wording and technical details were preserved as closely as possible.
-->

> [!CAUTION]
> This is a historical implementation record. Verify versions, addresses, paths, and commands against the live environment before applying changes.

# AlmaLinux VM on Proxmox for Terraform + Ansible

### Goal
Create an AlmaLinux management VM inside Proxmox. This VM will later be used to run Terraform and Ansible.

Recommended order:
1. Install AlmaLinux VM first
2. Install Terraform second
3. Install Ansible third

Why this order?
- Terraform creates/provisions infrastructure, such as VMs.
- Ansible configures machines after they exist.

### Recommended VM Purpose
This VM is only a management/control VM. It will mainly run:
- Terraform
- Ansible
- SSH
- Git
- Small scripts

It does not need heavy resources.

### Recommended VM Specs
CPU:
- Sockets: 1
- Cores: 2
- Type: host

Why 2 cores?
- 1 core works, but can feel slow during updates/package installs.
- 2 cores is smooth and lightweight.
- 4 cores is fine, but usually wasted for this VM.

Why CPU type host?
- It lets the VM use the real CPU features from the physical server.
- Better performance for a homelab.
- Avoid only if you plan to live-migrate VMs between different CPU models.

Memory:
- 4096 MB recommended
- 2048 MB minimum

Disk:
- 32 GB is enough
- 64 GB is safer/lazier if you do not want to think about space later

Network:
- Bridge: vmbr0
- Model: VirtIO paravirtualized
- Firewall: checked

### QEMU Guest Agent
qemu-guest-agent lets Proxmox talk to the VM from the inside.

It helps with:
- Showing the VM IP address in Proxmox
- Cleaner shutdown/reboot from Proxmox
- Better backup/snapshot handling
- Filesystem freeze during backups to reduce corruption risk
- More accurate VM status reporting

In Proxmox, check:
- Qemu Agent: enabled

Inside AlmaLinux, install it later with:

```text
sudo dnf install -y qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent
```

Then reboot:

```text
sudo reboot
```

### ISO Upload Notes
Normal Proxmox upload path:

Datacenter -> pve node -> local (pve) -> ISO Images -> Upload

If browser upload fails with an error like:

Error '0' occurred while receiving the document.

Use the Proxmox shell instead.

Download ISO directly on Proxmox:

```text
cd /var/lib/vz/template/iso
wget https://repo.almalinux.org/almalinux/9/isos/x86_64/AlmaLinux-9-latest-x86_64-minimal.iso
```

Check the ISO:

```text
ls -lh /var/lib/vz/template/iso
```

You should see something like:

AlmaLinux-9-latest-x86_64-minimal.iso

If wget is missing:

```text
apt update
apt install wget -y
```

### ISO on Another Drive
That is fine. Proxmox just needs the ISO to be on a storage location that supports ISO images.

The easiest route is usually to copy/move it into:

```text
/var/lib/vz/template/iso
```

Then refresh:

local (pve) -> ISO Images

### Recommended ISO
Use AlmaLinux 9 Minimal ISO x86_64.

AlmaLinux 10 may work, but for a Terraform/Ansible management VM, AlmaLinux 9 is safer and has better RHEL 9/EPEL compatibility.

### Creating the VM with SeaBIOS
Use SeaBIOS if OVMF/UEFI gets stuck booting the ISO.

## Step 1: Click Create VM

Top right:

Create VM

## Step 2: General tab

Name:

Alma-MGMT

Start at boot:

unchecked for now

Click Next.

## Step 3: OS tab

Use:

Use CD/DVD disc image file: checked
Storage: wherever the ISO is stored
ISO image: AlmaLinux 9 minimal ISO
Guest OS Type: Linux
Version: 6.x - 2.6 Kernel

Click Next.

## Step 4: System tab

Use:

Graphic card: Default
Machine: q35
BIOS: SeaBIOS
SCSI Controller: VirtIO SCSI single
Qemu Agent: checked
Add TPM: unchecked

Since this uses SeaBIOS, there should be no EFI disk.

Click Next.

## Step 5: Disks tab

Use:

Bus/Device: SCSI
Storage: VMs
Disk size: 32 GB or 64 GB
Cache: Default
Discard: checked if SSD-backed storage
IO thread: checked

Click Next.

## Step 6: CPU tab

Use:

```text
Sockets: 1
Cores: 2
Type: host
```

Total cores should show 2.

Click Next.

## Step 7: Memory tab

Use:

Memory: 4096 MB
Ballooning Device: optional

Click Next.

## Step 8: Network tab

Use:

```text
Bridge: vmbr0
Model: VirtIO paravirtualized
Firewall: checked
```

Click Next.

## Step 9: Confirm tab

Verify:

BIOS: SeaBIOS
Machine: q35
SCSI Controller: VirtIO SCSI single
Qemu Agent: enabled
Disk: SCSI
CPU Type: host
Network: VirtIO

Click Finish.

### Booting the VM
Click the new VM:

Start -> Console

You should see the AlmaLinux boot menu.

Choose:

Install AlmaLinux 9.x

Press Enter.

### If VM Gets Stuck on Boot Screen
If you see messages like:

```text
BdsDxe: failed to load Boot0003 "UEFI QEMU HARDDISK"
BdsDxe: loading Boot0002 "UEFI QEMU DVD-ROM"
BdsDxe: starting Boot0002 "UEFI QEMU DVD-ROM"
```

That usually means:
- Hard disk has no OS yet, which is normal before install
- It is trying to boot the DVD/ISO
- If it freezes there, UEFI/ISO boot may be stuck

Fixes:
1. Stop the VM
2. Check the CD/DVD Drive has the ISO attached
3. Check Boot Order has the CD/DVD drive first
4. If using OVMF/UEFI, switch to SeaBIOS
5. Try AlmaLinux 9 instead of AlmaLinux 10
6. Redownload ISO if needed

### Force Stop a VM
In the Proxmox UI:

VM -> Shutdown dropdown -> Stop

Use Stop, not Shutdown, if the VM is frozen before install.

From shell:

```text
qm list
qm stop VMID
```

Example:

```text
qm stop 100
```

If locked:

```text
qm unlock 100
qm stop 100
```

### Boot Order
Correct boot order while installing:

1. ide2 / CD/DVD / ISO
2. scsi0 / VM disk
3. net0 / network boot

After installation, set hard disk first:

1. scsi0 / VM disk
2. ide2 / CD/DVD, or disable CD/DVD boot

### Deleting a VM in Proxmox
1. Stop the VM first:

VM -> Shutdown dropdown -> Stop

2. Click the VM on the left.
3. Click:

More -> Remove

## 4. Type the VM ID exactly when asked, for example:

100

5. Check the option to destroy/delete unreferenced disks if you want the VM disk deleted too.
6. Click Remove.

### AlmaLinux Installer Settings
Inside the installer:

Language:
- English

Installation Destination:
- Select the virtual disk
- Use automatic partitioning

Network & Hostname:
- Turn Ethernet ON
- Hostname: alma-mgmt

Software Selection:
- Minimal Install

User Creation:
- Create your normal user
- Make this user administrator

Example:

Username: gang
Make this user administrator: checked

Root Password:
- Set a root password for now since this is a lab

Then click:

Begin Installation

### After Install
When installation finishes:

Reboot System

Then in Proxmox:

VM -> Hardware -> CD/DVD Drive -> Edit -> Do not use any media

Or update boot order so the hard disk boots first:

VM -> Options -> Boot Order -> scsi0 first

### First Commands After Login
Update the system:

```text
sudo dnf update -y
sudo reboot
```

After reboot, install basic tools:

```text
sudo dnf install -y vim nano git curl wget unzip tar bash-completion qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent
```

### Static IP Setup
Check NetworkManager connection name:

```text
nmcli connection show
```

Example connection name:

Wired connection 1

Set static IP example:

sudo nmcli connection modify "Wired connection 1" \
ipv4.addresses 192.168.1.250/24 \
ipv4.gateway 192.168.1.1 \
ipv4.dns "1.1.1.1 8.8.8.8" \
ipv4.method manual

Restart the connection:

```text
sudo nmcli connection down "Wired connection 1"
sudo nmcli connection up "Wired connection 1"
```

Check network:

```text
ip a
ping 1.1.1.1
ping google.com
```

Use an IP that is free on your network.

### Install Terraform
Install HashiCorp repo:

```text
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
```

Install Terraform:

```text
sudo dnf install -y terraform
```

Check:

```text
terraform version
```

### Install Ansible
Enable EPEL:

```text
sudo dnf install -y epel-release
```

Install Ansible Core:

```text
sudo dnf install -y ansible-core
```

Check:

```text
ansible --version
```

### Create Proxmox API Token for Terraform
In Proxmox web UI:

Datacenter -> Permissions -> Users -> Add

Create user:

User name: terraform
Realm: pve

Full user becomes:

terraform@pve

Give permissions:

Datacenter -> Permissions -> Add -> User Permission

Use:

```text
Path: /
User: terraform@pve
Role: Administrator
Propagate: checked
```

For learning in a home lab, Administrator is easiest. Later, restrict permissions.

Create API token:

Datacenter -> Permissions -> API Tokens -> Add

Use:

User: terraform@pve
Token ID: terraform-token
Privilege Separation: unchecked

Copy the token secret. You only see it once.

### Terraform Project Folder
On AlmaLinux:

```text
mkdir -p ~/homelab/terraform/proxmox
cd ~/homelab/terraform/proxmox
```

```text
touch main.tf variables.tf terraform.tfvars
```

### Terraform Provider Config
Use the bpg/proxmox provider.

Edit main.tf:

```text
nano main.tf
```

Paste:

terraform {
  required_providers {
    proxmox = {
      source  = "bpg/proxmox"
      version = "~> 0.70"
    }
  }
}

provider "proxmox" {
  endpoint  = var.proxmox_endpoint
  api_token = var.proxmox_api_token
  insecure  = true

  ssh {
    agent    = true
    username = "root"
  }
}

Edit variables.tf:

```text
nano variables.tf
```

Paste:

variable "proxmox_endpoint" {
  type = string
}

variable "proxmox_api_token" {
  type      = string
  sensitive = true
}

Edit terraform.tfvars:

```text
nano terraform.tfvars
```

Paste and change values:

proxmox_endpoint  = "https://192.168.1.242:8006/api2/json"
proxmox_api_token = "terraform@pve!terraform-token=PASTE_SECRET_HERE"

Replace the IP with your Proxmox IP.

### Test Terraform
Run:

```text
terraform init
terraform plan
```

If it initializes successfully, Terraform is talking to Proxmox.

### Terraform's Job
Terraform is for:
- Creating VMs
- Cloning VM templates
- Setting CPU/RAM/disk
- Attaching networks
- Setting cloud-init user/IP/SSH key
- Destroying/rebuilding VMs

Terraform makes the machine exist.

### Ansible's Job
Ansible is for:
- Installing packages
- Creating users
- Configuring SSH
- Hardening Linux
- Installing Docker
- Configuring firewall
- Updating systems
- Deploying apps

Ansible configures the machine after it exists.

### SSH Key Setup
On the AlmaLinux management VM:

```text
ssh-keygen -t ed25519 -C "alma-mgmt"
```

Press Enter through defaults.

Show public key:

```text
cat ~/.ssh/id_ed25519.pub
```

This key can later be used for cloud-init VMs.

### Basic Ansible Test
Create Ansible folder:

```text
mkdir -p ~/homelab/ansible
cd ~/homelab/ansible
```

Create inventory:

```text
nano inventory.ini
```

Example:

[test]
192.168.1.251 ansible_user=gang

Test:

```text
ansible all -i inventory.ini -m ping
```

Expected result:

pong

### Simple Ansible Playbook
Create update.yml:

```text
nano update.yml
```

Paste:

---
- name: Update Linux servers
  hosts: all
  become: true

  tasks:
    - name: Update all packages
      ansible.builtin.dnf:
        name: "*"
        state: latest

Run it:

```text
ansible-playbook -i inventory.ini update.yml
```

### Final Workflow
The clean homelab automation workflow is:

1. Build a VM template in Proxmox
2. Terraform clones the template and sets CPU/RAM/IP/SSH key
3. Ansible logs into the new VM
4. Ansible installs and configures everything

Final reminder:
- Install AlmaLinux first
- Install Terraform second
- Install Ansible third
- Terraform provisions
- Ansible configures
