<!--
Organized from: STEP 3 terraform_ansible_ubuntu2604_setup_and_troubleshooting(2).txt
Source wording and technical details were preserved as closely as possible.
-->

> [!CAUTION]
> This is a historical implementation record. Verify versions, addresses, paths, and commands against the live environment before applying changes.

# Terraform + Ansible Ubuntu Server 26.04 VM Setup and Troubleshooting

### Purpose
This document records the setup process for using Terraform and Ansible from an Alma Linux VM to create and configure a fresh Ubuntu Server 26.04 VM in Proxmox.

The goal was to copy the Alma Linux VM's SETTINGS, not clone the whole VM or copy the virtual disk contents.

Terraform was used to create the VM shell/hardware settings.
Ubuntu Server 26.04 ISO was used to install a fresh OS manually.
Ansible was used after the VM was installed and reachable over SSH.

### Environment
Control VM:
  Alma Linux
  Runs Terraform and Ansible

Hypervisor:
  Proxmox VE

Target VM:
  Ubuntu Server 26.04

Storage:
  ISO file is stored in Proxmox storage named: VMs
  New VM virtual disks are also stored in: VMs

Main concept:
  Terraform creates a new empty VM with similar settings.
  Terraform does NOT clone Alma Linux disk contents.
  Ansible configures Ubuntu after installation and SSH setup.

### Correct Workflow
1. Prepare Proxmox API access.
2. Install Terraform and Ansible on Alma Linux.
3. Create Terraform files.
4. Use Terraform to create a fresh VM shell.
5. Boot from Ubuntu Server 26.04 ISO.
6. Install Ubuntu manually to the blank disk.
7. Install and enable SSH inside Ubuntu.
8. Install and enable qemu-guest-agent inside Ubuntu.
9. Test SSH from Alma Linux to Ubuntu.
10. Configure Ansible inventory.
11. Run Ansible playbook.
12. Fix sudo/become issues if needed.

### Important Distinction
Terraform is for VM infrastructure settings:
  - VM ID
  - VM name
  - CPU cores
  - RAM
  - disk size
  - disk location
  - network bridge
  - ISO mount
  - boot order

Ansible is for operating system configuration:
  - packages
  - SSH
  - qemu-guest-agent
  - users
  - firewall
  - services
  - security hardening

Terraform should not be used here to install packages inside Ubuntu.
Ansible should not be used here to create the VM shell in Proxmox.

### Part 1: Check Alma Linux VM Settings
Before writing the Terraform config, check the existing Alma Linux VM settings in Proxmox.

Write down:
  - Node name
  - VM ID
  - BIOS type: SeaBIOS or OVMF
  - Machine type: i440fx or q35
  - CPU sockets
  - CPU cores
  - CPU type
  - Memory
  - Network bridge
  - Network model
  - Disk bus
  - Disk size
  - Storage location
  - Boot order

Example settings:
  Node name: pve
  BIOS: SeaBIOS
  Machine type: i440fx / pc
  CPU sockets: 1
  CPU cores: 2
  Memory: 4096 MB
  Network bridge: vmbr0
  Network model: VirtIO
  Disk bus: SCSI
  Disk size: 32 GB
  Storage: VMs

Do not copy or clone the Alma Linux disk.
Only recreate the settings.

### Part 2: Upload Ubuntu Server 26.04 ISO
In the Proxmox GUI:

```text
  Datacenter
  -> Node
  -> VMs storage
  -> ISO Images
  -> Upload
```

Upload the Ubuntu Server 26.04 ISO.

The ISO ID should look similar to:

  VMs:iso/ubuntu-26.04-live-server-amd64.iso

The exact filename must match what Proxmox shows.

### Part 3: Create Proxmox API Token
In the Proxmox GUI:

  Datacenter
  -> Permissions
  -> API Tokens
  -> Add

Example:
  User: terraform@pve
  Token ID: terraform
  Privilege Separation: checked

Then add API token permission:

  Datacenter
  -> Permissions
  -> Add
  -> API Token Permission

Path:

```text
  /
```

Role:
  PVEVMAdmin

The API token should look similar to:

  terraform@pve!terraform=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

Save the token secret immediately because Proxmox only shows it once.

### Part 4: Install Terraform and Ansible on Alma Linux
On the Alma Linux control VM:

```text
  sudo dnf update -y
  sudo dnf install -y dnf-plugins-core curl wget unzip git python3 python3-pip
```

Install Terraform:

```text
  sudo dnf config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
  sudo dnf install -y terraform
```

Check Terraform:

```text
  terraform version
```

Install Ansible:

```text
  sudo dnf install -y ansible-core
```

Check Ansible:

```text
  ansible --version
```

### Part 5: Create Terraform Project Folder
On Alma Linux:

```text
  mkdir -p ~/terraform/proxmox-ubuntu2604
  cd ~/terraform/proxmox-ubuntu2604
  touch main.tf variables.tf terraform.tfvars
```

### Part 6: variables.tf
Create/edit variables.tf:

```text
  nano variables.tf
```

Paste:

  variable "proxmox_endpoint" {
    description = "Proxmox API endpoint"
    type        = string
  }

```text
  variable "proxmox_api_token" {
    description = "Proxmox API token"
    type        = string
    sensitive   = true
  }
```

  variable "proxmox_node" {
    description = "Proxmox node name"
    type        = string
  }

  variable "vm_id" {
    description = "New VM ID"
    type        = number
  }

  variable "vm_name" {
    description = "New VM name"
    type        = string
  }

  variable "iso_file_id" {
    description = "ISO file ID from Proxmox storage"
    type        = string
  }

Save:
  CTRL + O
  Enter
  CTRL + X

### Part 7: terraform.tfvars
Create/edit terraform.tfvars:

```text
  nano terraform.tfvars
```

Paste and edit:

```text
  proxmox_endpoint  = "https://YOUR-PROXMOX-IP:8006/"
  proxmox_api_token = "terraform@pve!terraform=PASTE_YOUR_TOKEN_SECRET_HERE"
```

  proxmox_node = "pve"

```text
  vm_id   = 2604
  vm_name = "ubuntu-server-2604"
```

  iso_file_id = "VMs:iso/ubuntu-26.04-live-server-amd64.iso"

Example:

  proxmox_endpoint  = "https://192.168.1.242:8006/"
  proxmox_api_token = "terraform@pve!terraform=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

  proxmox_node = "pve"

```text
  vm_id   = 2604
  vm_name = "ubuntu-server-2604"
```

  iso_file_id = "VMs:iso/ubuntu-26.04-live-server-amd64.iso"

Important:
  The storage name "VMs" must match Proxmox exactly.
  The ISO filename must match Proxmox exactly.

### Part 8: main.tf
Create/edit main.tf:

```text
  nano main.tf
```

Paste:

  terraform {
    required_providers {
      proxmox = {
        source  = "bpg/proxmox"
        version = ">= 0.45.0"
      }
    }
  }

  provider "proxmox" {
    endpoint  = var.proxmox_endpoint
    api_token = var.proxmox_api_token
    insecure  = true
  }

  resource "proxmox_virtual_environment_vm" "ubuntu_server" {
    name      = var.vm_name
    node_name = var.proxmox_node
    vm_id     = var.vm_id

    description = "Ubuntu Server 26.04 VM created by Terraform. Fresh disk only, not cloned from Alma Linux."

```text
    started = false
    on_boot = false
```

```text
    machine = "pc"
    bios    = "seabios"
```

    operating_system {
      type = "l26"
    }

    cpu {
      sockets = 1
      cores   = 2
      type    = "host"
    }

    memory {
      dedicated = 4096
    }

    agent {
      enabled = true
    }

    cdrom {
      enabled   = true
      file_id   = var.iso_file_id
      interface = "ide2"
    }

    disk {
      datastore_id = "VMs"
      interface    = "scsi0"
      size         = 32
      file_format  = "qcow2"
    }

    network_device {
      bridge   = "vmbr0"
      model    = "virtio"
      firewall = true
    }

    boot_order = ["ide2", "scsi0"]
  }

This creates:
  - 2 CPU cores
  - 4 GB RAM
  - 32 GB blank disk on VMs storage
  - Ubuntu ISO mounted from VMs storage
  - VirtIO network on vmbr0
  - SeaBIOS
  - ISO first in boot order

This does NOT clone the Alma Linux VM disk.

The key blank disk section is:

  disk {
    datastore_id = "VMs"
    interface    = "scsi0"
    size         = 32
    file_format  = "qcow2"
  }

The key ISO section is:

  cdrom {
    enabled   = true
    file_id   = var.iso_file_id
    interface = "ide2"
  }

There is no clone, no template, no source_vm_id, and no Alma disk reference.

### Part 9: Initialize Terraform
Run:

```text
  cd ~/terraform/proxmox-ubuntu2604
  terraform init
```

Terraform downloads the Proxmox provider.

### Part 10: Check Terraform Plan
Run:

```text
  terraform plan
```

Look for:

  proxmox_virtual_environment_vm.ubuntu_server

Make sure it is creating a new VM.

Bad signs:
  clone
  template
  source_vm_id

If those appear, stop and recheck the Terraform file.

### Part 11: Apply Terraform
Run:

```text
  terraform apply
```

When prompted, type:

  yes

Terraform creates the VM shell in Proxmox.

### Part 12: Install Ubuntu Manually
In Proxmox GUI:

  Click ubuntu-server-2604
  -> Console
  -> Start

The VM should boot from the Ubuntu Server 26.04 ISO.

Install Ubuntu to the blank disk created by Terraform.
Do not select any Alma Linux disk.

After Ubuntu is installed, reboot or shut down as needed.

### Part 13: Change Boot Order After Install
After Ubuntu is installed, change boot order so the VM boots from disk first.

In Proxmox GUI:

  VM
  -> Options
  -> Boot Order

Set:

  scsi0 first
  ide2 second or disabled

Recommended final boot order:

  scsi0
  ide2

You can also update Terraform later:

  boot_order = ["scsi0", "ide2"]

Then run:

```text
  terraform apply
```

### Part 14: Install qemu-guest-agent in Ubuntu
Inside the Ubuntu VM:

```text
  sudo apt update
  sudo apt install -y qemu-guest-agent
  sudo systemctl enable --now qemu-guest-agent
  sudo reboot
```

This helps Proxmox see VM details such as IP address and improves VM shutdown/status behavior.

### Part 15: Install SSH in Ubuntu
Inside Ubuntu:

```text
  sudo apt install -y openssh-server
  sudo systemctl enable --now ssh
```

Find the Ubuntu VM IP:

```text
  ip a
```

Example IP used during troubleshooting:

## 192.168.1.254

Test SSH from Alma Linux:

```text
  ssh izaman@192.168.1.254
```

### Part 16: Ansible Inventory
On Alma Linux:

```text
  mkdir -p ~/ansible/ubuntu2604
  cd ~/ansible/ubuntu2604
  nano inventory.ini
```

Example inventory:

  [ubuntu_servers]
  ubuntu-server-2604 ansible_host=192.168.1.254 ansible_user=izaman

Later, after sudo/become was being used, inventory could be written as:

  [ubuntu_servers]
  ubuntu-server-2604 ansible_host=192.168.1.254 ansible_user=izaman ansible_become=true ansible_become_method=sudo

### Part 17: Basic Ansible Playbook
Create setup.yml:

```text
  nano setup.yml
```

Paste:

  ---
  - name: Basic Ubuntu Server setup
    hosts: ubuntu_servers
    become: true

    tasks:
      - name: Update apt cache
        apt:
          update_cache: true

      - name: Install useful packages
        apt:
          name:
            - curl
            - wget
            - git
            - htop
            - qemu-guest-agent
            - openssh-server
          state: present

      - name: Enable qemu guest agent
        systemd:
          name: qemu-guest-agent
          enabled: true
          state: started

      - name: Enable SSH
        systemd:
          name: ssh
          enabled: true
          state: started

Test Ansible ping:

```text
  ansible -i inventory.ini ubuntu_servers -m ping
```

Expected:

  pong

Run playbook:

```text
  ansible-playbook -i inventory.ini setup.yml
```

## Troubleshooting Section

### Error 1: SSH Permission Denied
Error:

  ubuntu-server-2604 | UNREACHABLE! => {
      "changed": false,
      "msg": "Failed to connect to the host via ssh: izaman@192.168.1.254: Permission denied (publickey,password).",
      "unreachable": true
  }

Meaning:
  Ansible reached the VM, but Ubuntu rejected the SSH login.
  The IP was reachable.
  The problem was SSH authentication.

Things checked:
  1. Test SSH manually:

```text
       ssh izaman@192.168.1.254
```

## 2. Confirm the username was correct on Ubuntu:

       whoami

## 3. Make sure SSH server was installed:

```text
       sudo apt update
       sudo apt install -y openssh-server
       sudo systemctl enable --now ssh
       systemctl status ssh
```

## 4. Try Ansible with password prompt:

```text
       ansible -i inventory.ini ubuntu_servers -m ping --ask-pass
```

## 5. Better solution: copy SSH key from Alma Linux to Ubuntu:

```text
       ls ~/.ssh/id_ed25519.pub
```

     If missing:

```text
       ssh-keygen -t ed25519
```

     Copy key:

```text
       ssh-copy-id izaman@192.168.1.254
```

     Test:

```text
       ssh izaman@192.168.1.254
       ansible -i inventory.ini ubuntu_servers -m ping
```

Expected successful result:

  ubuntu-server-2604 | SUCCESS => {
      "changed": false,
      "ping": "pong"
  }

### Error 2: sudo interactive authentication required
Error:

  TASK [Gathering Facts]
  fatal: [ubuntu-server-2604]: FAILED! => {
    "failed_modules": {
      "ansible.legacy.setup": {
        "module_stderr": "Shared connection to 192.168.1.254 closed.\r\n",
        "module_stdout": "sudo: interactive authentication is required\r\n",
        "msg": "MODULE FAILURE\nSee stdout/stderr for the exact error",
        "rc": 1
      }
    }
  }

Meaning:
  SSH was working.
  Ansible connected to the Ubuntu VM.
  But the playbook had:

    become: true

  That means Ansible needed sudo access.
  Ubuntu wanted the sudo password, but Ansible was not given it.

First attempted fix:

```text
  ansible-playbook -i inventory.ini setup.yml --ask-become-pass
```

Short version:

```text
  ansible-playbook -i inventory.ini setup.yml -K
```

If SSH password was also needed:

```text
  ansible-playbook -i inventory.ini setup.yml --ask-pass --ask-become-pass
```

Short version:

```text
  ansible-playbook -i inventory.ini setup.yml -k -K
```

### Error 3: Timeout waiting for privilege escalation prompt
Error:

  fatal: [ubuntu-server-2604]: FAILED! => {"msg": "Timeout (12s) waiting for privilege escalation prompt: "}

Another version:

  ubuntu-server-2604 | FAILED | rc=-1 >>
  Timeout (12s) waiting for privilege escalation prompt:

Meaning:
  SSH worked.
  Ansible reached the VM.
  But Ansible got stuck trying to use sudo.
  It was waiting for the privilege escalation prompt and timed out.

Possible causes:
  - Wrong sudo password
  - User is not in sudo group
  - Ansible is not being told to ask for sudo password
  - sudo prompt is not passing cleanly through Ansible
  - become settings are not clean

Tests used:

```text
  ssh izaman@192.168.1.254
```

Inside Ubuntu:

```text
  sudo whoami
```

Expected:

  root

If the user was not in sudo group, the fix would be:

```text
  sudo usermod -aG sudo izaman
  sudo reboot
```

Another Ansible test:

```text
  ansible -i inventory.ini ubuntu_servers -m command -a "whoami"
```

Expected:

  izaman

Test with sudo/become:

```text
  ansible -i inventory.ini ubuntu_servers -m command -a "whoami" -b -K
```

Expected:

  root

Another attempted timeout increase:

```text
  ansible-playbook -i inventory.ini setup.yml -K -e ansible_become_timeout=60
```

If SSH password was also needed:

```text
  ansible-playbook -i inventory.ini setup.yml -k -K -e ansible_become_timeout=60
```

### Final Working Fix: Passwordless sudo for Ansible user
The clean homelab fix was to allow the Ubuntu user to run sudo without entering a password.

SSH into Ubuntu:

```text
  ssh izaman@192.168.1.254
```

Confirm sudo works manually:

```text
  sudo whoami
```

Expected:

  root

Open sudoers safely:

```text
  sudo visudo
```

Add this exact line at the bottom:

  izaman ALL=(ALL) NOPASSWD:ALL

Save and exit.

In nano:

  CTRL + O
  Enter
  CTRL + X

Exit back to Alma Linux:

  exit

Test Ansible sudo without using -K:

```text
  ansible -i inventory.ini ubuntu_servers -m command -a "whoami" -b
```

Expected:

  ubuntu-server-2604 | CHANGED | rc=0 >>
  root

Then run the playbook normally:

```text
  ansible-playbook -i inventory.ini setup.yml
```

Important:
  After setting NOPASSWD, do not use -K or --ask-become-pass anymore.

Correct final command:

```text
  ansible-playbook -i inventory.ini setup.yml
```

Incorrect after NOPASSWD:

```text
  ansible-playbook -i inventory.ini setup.yml -K
```

## Final Working State
Terraform:
  Successfully creates the VM shell/settings.

Ubuntu Server 26.04 ISO:
  Used to install a fresh OS manually.

Ansible:
  Connects over SSH.
  Uses passwordless sudo through the izaman user.
  Runs the basic setup playbook successfully.

Final mental model:

  Terraform = creates the VM shell/settings
  Ubuntu ISO = installs the fresh OS
  Ansible = configures the VM after SSH works

## Final Commands to Remember
Create/apply VM with Terraform:

```text
  cd ~/terraform/proxmox-ubuntu2604
  terraform plan
  terraform apply
```

Test SSH:

```text
  ssh izaman@192.168.1.254
```

Test Ansible connection:

```text
  ansible -i inventory.ini ubuntu_servers -m ping
```

Test Ansible sudo after passwordless sudo:

```text
  ansible -i inventory.ini ubuntu_servers -m command -a "whoami" -b
```

Run playbook:

```text
  ansible-playbook -i inventory.ini setup.yml
```

## Reusable Future Workflow
For each new VM:

## 1. Edit Terraform variables:

```text
     nano terraform.tfvars
```

## 2. Change VM ID and name:

```text
     vm_id   = 2605
     vm_name = "ubuntu-server-test2"
```

## 3. Apply Terraform:

```text
     terraform apply
```

4. Install Ubuntu from ISO manually.
5. Install SSH and qemu-guest-agent in Ubuntu.
6. Copy SSH key from Alma Linux:

```text
     ssh-copy-id izaman@NEW_VM_IP
```

7. Add the new VM to Ansible inventory.
8. Configure passwordless sudo for the Ansible user if desired:

```text
     sudo visudo
```

     izaman ALL=(ALL) NOPASSWD:ALL

## 9. Run Ansible:

```text
     ansible-playbook -i inventory.ini setup.yml
```

## Notes
The most important mistake to avoid:
  Do not use Terraform clone settings if the goal is a fresh VM.

Avoid these in Terraform unless intentionally cloning:
  clone
  template
  source_vm_id

The correct fresh VM approach is:
  - Attach ISO
  - Create blank disk
  - Install OS manually
  - Configure with Ansible afterward
