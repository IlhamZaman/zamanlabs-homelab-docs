<!--
Organized from: STEP 1.5 SSH Public Key Only Authentication(2).txt
Source wording and technical details were preserved as closely as possible.
-->

> [!CAUTION]
> This is a historical implementation record. Verify versions, addresses, paths, and commands against the live environment before applying changes.

# STEP 1.5 SSH Public Key Only Authentication

Goal

This step disables SSH password authentication on Linux VMs and only allows SSH login using public key authentication.

After this setup, SSH login will only work if the connecting machine has the correct private key.

Normal SSH login should look like this:

```text
ssh youruser@VM-IP
```

Password-based SSH login will be blocked.

Important Warning

Do not disable password authentication until public key login has already been tested and confirmed working.

Keep your current SSH session open while testing changes in a second terminal. This helps prevent getting locked out.


## Step 1: Check If You Already Have an SSH Key

On your main computer or management VM, check your SSH folder:

```text
ls -la ~/.ssh
```

Look for files like:

id_ed25519
id_ed25519.pub

The private key is:

id_ed25519

The public key is:

id_ed25519.pub

If you do not already have an SSH key, create one:

```text
ssh-keygen -t ed25519 -C "homelab-admin-key"
```

Press Enter through the default prompts unless you want to customize the file name or add a passphrase.


## Step 2: Copy Your SSH Public Key To The VM

From your main computer or management VM, run:

```text
ssh-copy-id youruser@VM-IP
```

Example:

```text
ssh-copy-id izaman@192.168.1.50
```

Enter the VM password one last time when prompted.

This installs your public key into the VM user's authorized SSH keys file.


## Step 3: Test SSH Key Login

Before changing any SSH security settings, test login:

```text
ssh youruser@VM-IP
```

Example:

```text
ssh izaman@192.168.1.50
```

If it logs in without asking for the VM password, public key authentication is working.

Do not continue until this works.


## Step 4: Confirm The Authorized Key Exists On The VM

Once logged into the VM, check the authorized keys file:

```text
cat ~/.ssh/authorized_keys
```

You should see a long public key line that starts with something like:

ssh-ed25519

Then fix the SSH folder permissions:

```text
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

These permissions are important because SSH may reject the key if the files are too open.


## Step 5: Edit The SSH Server Config

On the VM, open the SSH server config file:

```text
sudo nano /etc/ssh/sshd_config
```

Find or add these lines:

```text
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
ChallengeResponseAuthentication no
PermitRootLogin no
PermitEmptyPasswords no
UsePAM yes
X11Forwarding no
```

Recommended optional hardening settings:

```text
MaxAuthTries 3
LoginGraceTime 30
```

The most important settings are:

```text
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
```

Save the file:

CTRL + O
Enter
CTRL + X


## Step 6: Check For SSH Config Override Files

Some Linux distributions use extra SSH config files in this folder:

```text
/etc/ssh/sshd_config.d/
```

Check the folder:

```text
ls -la /etc/ssh/sshd_config.d/
```

Search for password authentication settings:

```text
sudo grep -R "PasswordAuthentication\|KbdInteractiveAuthentication\|ChallengeResponseAuthentication" /etc/ssh/sshd_config /etc/ssh/sshd_config.d/
```

If you see a file that says:

```text
PasswordAuthentication yes
```

edit that file and change it to:

```text
PasswordAuthentication no
```

Example:

```text
sudo nano /etc/ssh/sshd_config.d/50-cloud-init.conf
```

Set or add:

```text
PasswordAuthentication no
KbdInteractiveAuthentication no
```

This matters because files inside /etc/ssh/sshd_config.d/ can override the main sshd_config file.


## Step 7: Validate The SSH Configuration

Before restarting SSH, check for config errors:

```text
sudo sshd -t
```

If the command returns no output, the SSH config is valid.

If it shows an error, do not restart SSH yet. Fix the error first.


## Step 8: Restart SSH

On Debian or Ubuntu:

```text
sudo systemctl restart ssh
```

On Fedora, AlmaLinux, Rocky Linux, RHEL, or openSUSE:

```text
sudo systemctl restart sshd
```

If you are not sure which service name your VM uses, check both:

```text
systemctl status ssh
systemctl status sshd
```


## Step 9: Test From A New Terminal

Do not close your current SSH session yet.

Open a new terminal and test normal SSH login:

```text
ssh youruser@VM-IP
```

Example:

```text
ssh izaman@192.168.1.50
```

If it logs in successfully, your SSH key still works.


## Step 10: Test That Password Login Is Blocked

From your main computer or management VM, run:

```text
ssh -o PubkeyAuthentication=no youruser@VM-IP
```

Example:

```text
ssh -o PubkeyAuthentication=no izaman@192.168.1.50
```

This test disables public key authentication on purpose.

It should fail with something like:

Permission denied

That means password authentication is disabled successfully.


## Step 11: Repeat For Each VM

Repeat the same process for each Linux VM:

1. Copy the SSH public key to the VM.
2. Test that SSH key login works.
3. Edit /etc/ssh/sshd_config.
4. Check /etc/ssh/sshd_config.d/ for overrides.
5. Validate the SSH config with sudo sshd -t.
6. Restart SSH.
7. Test key login.
8. Confirm password login is blocked.


Final Recommended SSH Settings

Use these settings on each VM:

```text
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
ChallengeResponseAuthentication no
PermitRootLogin no
PermitEmptyPasswords no
UsePAM yes
X11Forwarding no
MaxAuthTries 3
LoginGraceTime 30
```


Best Practice Notes

Keep Proxmox console access available in case you accidentally lock yourself out of a VM.

Do not delete the Linux user password entirely. The goal is to disable SSH password login, not remove the local user password.

Keep at least one backup admin user on each important VM.

For a homelab, a clean setup is:

Main computer or management VM
        |
        | SSH key only
        v
All Linux VMs

Password SSH login disabled.
Root SSH login disabled.
Public key SSH login enabled.
