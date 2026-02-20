# Eedspeak Ansible Playbook

## Description

This Ansible playbook automates the deployment of a complete TeamSpeak 3 server on Ubuntu Linux with enterprise-grade security and configuration management. The playbook handles the following:

**System Administration:**
- Updates system packages and installs essential utilities (curl, wget, git, vim, neovim, bzip2, tar)
- Configures kernel-level iptables firewall with explicit ACCEPT rules for SSH and TeamSpeak services
- Sets up fail2ban for SSH brute-force protection with configurable ban policies

**TeamSpeak Server Deployment:**
- Creates a dedicated system user (TeamSpeak) for service isolation
- Downloads and installs TeamSpeak 3 server (v3.13.7)
- Cleans up any existing TeamSpeak database before initial setup
- Generates initial server tokens during setup
- Creates a systemd service file for automatic startup and lifecycle management
- Enables the service to auto-start on system boot

**Security Features:**
- Runs TeamSpeak under a non-login system user account
- Proper file ownership and permissions throughout
- Kernel-level firewall rules (iptables) with explicit ACCEPT chains for required ports:
  - SSH (22/tcp)
  - TeamSpeak voice (9987/udp)
  - File transfer (30033/tcp)
  - ServerQuery (10011/tcp)
- Rules are loaded from `iptables-rules.v4` file during playbook execution
- Rules are persisted across reboots via iptables-persistent
- All ACCEPT rules appear before REJECT rules in the chain

**System Hardening:**
- SSH hardening:
  - Root login disabled
  - Password authentication disabled (key-based only)
  - Empty passwords disabled
  - Challenge-response authentication disabled
  - MaxAuthTries limited to 3 attempts
  - AuthenticationMethods set to publickey only
- fail2ban provides SSH brute-force protection with automatic IP blocking
- iptables rules positioned at the top of INPUT chain before any REJECT/DROP rules
- Automatic security updates via unattended-upgrades
- Log rotation configured for TeamSpeak logs (7-day retention)

## Setup Instructions

### 1. Create Vault Password File
Create a `.vault-password` file in the root directory (this is gitignored for security):
```bash
echo "your_secure_vault_password" > .vault-password
chmod 600 .vault-password
```

### 2. Encrypt the Secrets File
Encrypt the vault/secrets.yml file:
```bash
ansible-vault encrypt vault/secrets.yml
```
You'll be prompted to enter your vault password.

### 3. Edit Vault File
To edit encrypted secrets:
```bash
ansible-vault edit vault/secrets.yml
```

### 4. Run the Playbook

Execute the playbook with sudo password prompt:
```bash
ansible-playbook playbook.yml -K -u $(whoami)
```

Or check the changes first (still requires sudo password):
```bash
ansible-playbook playbook.yml --check -K -u $(whoami)
```

## Verify Installation

After the playbook completes successfully, verify the installation with these commands:

### 1. Check TeamSpeak Service Status
```bash
sudo systemctl status teamspeak
```
Expected: Service should be active and running.

### 2. View Generated Server Token
```bash
sudo cat /opt/TeamSpeak/token.log
```
Expected: Contains the initial admin token and server information needed for first login.

### 3. Verify iptables Firewall Rules
```bash
sudo iptables -L INPUT -n | grep ACCEPT
```
Expected: Shows ACCEPT rules for all required ports in the INPUT chain:
- Port 22/tcp (SSH)
- Port 9987/udp (TeamSpeak voice)
- Port 30033/tcp (TeamSpeak file transfer)
- Port 10011/tcp (TeamSpeak ServerQuery)

### 4. Check fail2ban SSH Protection
```bash
sudo fail2ban-client status ssh
```
Expected: Shows fail2ban is active with SSH jail configured.

### 5. Verify TeamSpeak Directory Structure
```bash
ls -la /opt/TeamSpeak/
```
Expected: Shows TeamSpeak server files with TeamSpeak user ownership.

### 6. Monitor TeamSpeak Log
```bash
sudo tail -f /opt/TeamSpeak/logs/ts3server_*.log
```
Expected: Real-time TeamSpeak server log output showing server operations.

## Verify Server Hardening

After deployment, verify that security hardening measures are in place:

### SSH Hardening Verification
```bash
# Check SSH configuration for hardening settings
sudo grep -E "PermitRootLogin|PasswordAuthentication|PermitEmptyPasswords|MaxAuthTries|AuthenticationMethods" /etc/ssh/ssh_config
```
Expected:
```
PermitRootLogin no
PasswordAuthentication no
PermitEmptyPasswords no
ChallengeResponseAuthentication no
AuthenticationMethods publickey
MaxAuthTries 3
```

### Automatic Updates Verification
```bash
# Check unattended-upgrades is configured
sudo cat /etc/apt/apt.conf.d/50unattended-upgrades | grep -E "APT::Periodic|Unattended-Upgrade" | head -10
```
Expected: Shows automatic update configuration is active.

### iptables Persistence Verification
```bash
sudo cat /etc/iptables/rules.v4 | grep -E "dpt:22|dpt:9987|dpt:30033|dpt:10011"
```
Expected: Shows all four ACCEPT rules are saved and will persist across reboots.

### Verify netfilter-persistent Service
```bash
sudo systemctl status netfilter-persistent
```
Expected: Service should be active and enabled, indicating rules will load on boot.

### Manual Save of Current iptables State
If you manually modify iptables rules with iptables commands, persist them with:
```bash
sudo netfilter-persistent save
```
This saves the current kernel iptables state to `/etc/iptables/rules.v4`.

### TeamSpeak Log Rotation Verification
```bash
# Check logrotate configuration
sudo cat /etc/logrotate.d/teamspeak
```
Expected: Shows daily rotation with 7-day retention and proper permissions.

## File Structure
```
.
├── playbook.yml              # Main playbook
├── iptables-rules.v4         # iptables rules in iptables-save format
├── ansible.cfg               # Ansible configuration
├── hosts                     # Inventory file
├── vault/
│   └── secrets.yml          # Encrypted secrets (ansible-vault)
├── .vault-password          # Vault password file (gitignored)
└── .gitignore
```

## iptables Rules Configuration

The playbook uses the `iptables-rules.v4` file which contains firewall rules in standard iptables-save format. The rules define:

- **Connection tracking**: Allows established and related connections
- **SSH access**: Allows port 22/tcp from any source
- **TeamSpeak ports**:
  - 9987/udp - Voice communication
  - 30033/tcp - File transfers
  - 10011/tcp - ServerQuery interface
- **Loopback**: Allows local traffic
- **Default deny**: Rejects all other inbound traffic

To modify firewall rules:

1. Edit `iptables-rules.v4` directly
2. Run the playbook to apply changes:
   ```bash
   ansible-playbook playbook.yml -K -u $(whoami)
   ```
3. Changes will be saved to `/etc/iptables/rules.v4` and persist across reboots

## iptables Persistence on Boot

The playbook ensures iptables rules persist across system reboots through:

1. **iptables-persistent Package**: Installed via apt (already in your playbook)
2. **netfilter-persistent Service**: Enabled and started automatically
   - Automatically loads rules from `/etc/iptables/rules.v4` on boot
   - Restores the firewall configuration before network services start
3. **Rules File**: `/etc/iptables/rules.v4` contains the persistent rules

**Boot sequence:**
- System boots → netfilter-persistent service starts → loads rules from `/etc/iptables/rules.v4` → iptables firewall is active

If you need to manually update iptables and persist changes:
```bash
sudo iptables [your rule modifications]
sudo netfilter-persistent save
```

Both IPv4 and IPv6 rules can be persisted (IPv6 rules go to `/etc/iptables/rules.v6`).

## Next Steps
Tell me what features or tasks you'd like to add to the playbook!
