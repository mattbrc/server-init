# Server Security & Setup Documentation

## Summary

This Hetzner VPS Ubuntu server has been secured with multiple layers of defense-in-depth security measures and essential development tools.

## Prerequisites

Before starting:

- A fresh Ubuntu 22.04+ VPS (this was written for Hetzner Cloud, but should work on any Ubuntu VPS).
- The provider's web console is accessible (e.g., Hetzner Cloud Console). **Keep this tab open the whole time** — it's your only recovery path if SSH gets misconfigured.
- You are logged in as `root` over SSH (the default on a fresh Hetzner VPS).
- You know the public IPv4 address of the server.
- Optional but recommended: an SSH key pair already on your local machine. If not, run `ssh-keygen -t ed25519` locally first.

**Safety tip:** Take a snapshot of the VPS in the Hetzner console before starting, so you can roll back if anything goes wrong.

### Security Implementations
- **Firewall (UFW)**: Configured with strict rules allowing only SSH (22), HTTP (80), and HTTPS (443)
- **Fail2ban**: Intrusion prevention system monitoring and blocking suspicious login attempts
- **Unattended Upgrades**: Automatic security updates configured
- **Non-root User**: Created 'deploy' user with sudo privileges for safer administration
- **SSH Hardening**: Disabled root login, limited authentication attempts, configured idle timeout
- **Kernel Security**: Applied sysctl parameters for network hardening and attack mitigation
- **Rootkit Detection**: Installed rkhunter for malware and rootkit scanning
- **Log Monitoring**: Configured logwatch for daily system activity reports
- **Swap File**: Added 2GB swap space for memory management

### Development Tools Installed
- **Node.js 20.x LTS**: JavaScript runtime environment
- **npm**: Node package manager
- **Claude Code**: Anthropic's CLI tool for AI-assisted development
- **Git**: Version control system
- **Build essentials**: Compilers and development libraries
- **System utilities**: curl, wget, htop, nano

---

## Detailed Setup Steps

### Phase 1: Base install

#### Step 1: System Update and Essential Packages
```bash
# Update package lists and upgrade existing packages
sudo apt update && sudo apt upgrade -y

# Install essential packages
sudo apt install -y curl wget git build-essential software-properties-common ufw fail2ban unattended-upgrades apt-listchanges
```

#### Step 2: Configure Automatic Security Updates
```bash
# Configure unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
# Selected "Yes" to automatically download and install stable updates
```

#### Step 3: Firewall Setup (UFW)
```bash
# Set up basic firewall rules
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp  # SSH
sudo ufw allow 80/tcp  # HTTP
sudo ufw allow 443/tcp # HTTPS
sudo ufw --force enable
```

#### Step 4: Install Node.js and npm
```bash
# Install Node.js 20.x (LTS)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# Verify installation
node --version  # v20.x.x
npm --version   # 10.x.x
```

#### Step 5: Install Claude Code
```bash
# Install Claude Code globally
sudo npm install -g @anthropic-ai/claude-code

# Verify installation
claude --version
```

#### Step 6: Configure fail2ban
```bash
# Create a local jail configuration
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

#### Additional: Fix Interrupted Package Installation
```bash
# Fixed partially installed packages
dpkg --configure -a

# Completed installation of additional utilities
apt install -y curl wget git nano htop
```

---

### Phase 2: Hardening

#### 1. Create Non-Root User with Sudo Privileges
```bash
# Create new user 'deploy' (pick any username you like — but use it consistently below)
sudo useradd -m -s /bin/bash deploy

# Generate a strong random password and set it on the new user.
# IMPORTANT: copy the printed password somewhere safe — you may need it for sudo later.
DEPLOY_PW=$(openssl rand -base64 24)
echo "deploy:$DEPLOY_PW" | sudo chpasswd
echo "deploy password (save this!): $DEPLOY_PW"

# Add to sudo group
sudo usermod -aG sudo deploy
```
**Purpose**: Reduces risk by avoiding direct root access for daily operations.

> ⚠ **Do the SSH key + login test below BEFORE Step 2.** Step 2 disables root SSH login. If you can't log in as `deploy` first, you'll be locked out of SSH entirely and will need the provider's web console to recover.

##### 1a. Set up SSH key login for `deploy` (do this before SSH hardening)

On your **local machine** (not the server):

```bash
# Skip ssh-keygen if you already have a key at ~/.ssh/id_ed25519
ssh-keygen -t ed25519 -C "your-email@example.com"

# Copy your public key to the server's deploy user
ssh-copy-id deploy@<server-ip>
```

Then **open a second terminal** and confirm you can log in:

```bash
ssh deploy@<server-ip>
sudo -v   # confirm sudo works using the password from Step 1
```

Only continue to Step 2 once both succeed. Leave this second terminal open until the very end as your safety net.

#### 2. SSH Hardening

> ⚠ **Replace `deploy` in `AllowUsers` below with whatever username you picked in Step 1.** A typo here will lock everyone out.

Write `/etc/ssh/sshd_config.d/99-hardening.conf`:

```bash
sudo tee /etc/ssh/sshd_config.d/99-hardening.conf > /dev/null <<'EOF'
PermitRootLogin no              # Disable root SSH login
PasswordAuthentication yes      # Allow password auth (flip to "no" once you're sure key login works)
PermitEmptyPasswords no         # No empty passwords
PubkeyAuthentication yes        # Enable public key authentication
MaxAuthTries 3                  # Limit authentication attempts
MaxSessions 10                  # Limit concurrent sessions
ClientAliveInterval 300         # Disconnect idle clients after 5 minutes
ClientAliveCountMax 2           # Max keep-alive messages
X11Forwarding no                # Disable X11 forwarding
AllowUsers deploy               # Only allow specific user — CHANGE if your user isn't named "deploy"
Protocol 2                      # Use SSH protocol 2 only
EOF
```

**Validate the config before restarting** — a bad config will brick SSH on next restart:

```bash
sudo sshd -t   # silent = good. Any error means: do not restart yet, fix the file first.
```

Apply it:

```bash
sudo systemctl restart ssh
```

**Verify from your second terminal (the one already logged in as `deploy`):**

```bash
# Open a THIRD terminal and try a fresh SSH login as deploy:
ssh deploy@<server-ip>
```

If the new login works, you're safe. If not, go back to your already-logged-in terminal and revert with:

```bash
sudo rm /etc/ssh/sshd_config.d/99-hardening.conf
sudo systemctl restart ssh
```

**Purpose**: Prevents brute-force attacks and unauthorized access.

Once you're confident key login works, tighten further by setting `PasswordAuthentication no` in the file above and restarting SSH.

#### 3. Kernel Security Parameters
Created `/etc/sysctl.d/99-security.conf`:
```
# IP Spoofing protection
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# Ignore ICMP redirects
net.ipv4.conf.all.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0

# Disable source packet routing
net.ipv4.conf.all.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0

# Log Martians (packets with impossible addresses)
net.ipv4.conf.all.log_martians = 1

# Ignore ICMP ping requests to broadcast
net.ipv4.icmp_echo_ignore_broadcasts = 1

# SYN flood protection
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_syn_retries = 2
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_max_syn_backlog = 4096

# Disable IPv6 (OPTIONAL — only if you're not using IPv6).
# Modern guidance is to configure IPv6 properly rather than disable it; some
# services (e.g. GitHub) increasingly prefer IPv6. Leave these commented unless
# you have a specific reason to disable.
# net.ipv6.conf.all.disable_ipv6 = 1
# net.ipv6.conf.default.disable_ipv6 = 1
# net.ipv6.conf.lo.disable_ipv6 = 1

# Restrict core dumps
fs.suid_dumpable = 0

# Hide kernel pointers
kernel.kptr_restrict = 2

# Enable ASLR (Address Space Layout Randomization)
kernel.randomize_va_space = 2
```

**Apply the new settings without rebooting:**

```bash
sudo sysctl --system
```

**Purpose**: Hardens the kernel against various network attacks and information disclosure.

#### 4. Rootkit Detection (rkhunter)
```bash
# Install rkhunter
sudo apt install -y rkhunter

# Clear WEB_CMD so rkhunter's update step doesn't fail on Ubuntu's
# locked-down /bin/false default. This is a well-known workaround.
sudo sed -i 's|WEB_CMD="/bin/false"|WEB_CMD=""|' /etc/rkhunter.conf

# Build the baseline of "known good" file properties
sudo rkhunter --propupd
```
**Purpose**: Scans for rootkits, backdoors, and local exploits.

#### 5. Log Monitoring (logwatch)
```bash
# Install logwatch
sudo apt install -y logwatch

# Create configuration
sudo mkdir -p /etc/logwatch/conf
cat << 'EOF' | sudo tee /etc/logwatch/conf/logwatch.conf
MailTo = root
MailFrom = Logwatch <logwatch@localhost>
Detail = Med
Service = All
Range = yesterday
EOF
```
**Purpose**: Daily system activity reports for security monitoring.

> Without a local mail server (postfix etc.), reports sent to `root` land in `/var/mail/root`. Read them with:
> ```bash
> sudo less /var/mail/root
> ```
> If you'd rather have reports emailed to you, install `postfix` (choose "Internet Site" during setup) and change `MailTo = your-email@example.com`.

#### 6. Swap File Creation
```bash
# Created 2GB swap file
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Made permanent
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```
**Purpose**: Prevents out-of-memory issues and improves system stability

---

## Maintenance & Security Best Practices

### Regular Security Tasks
1. **Check for rootkits**: `sudo rkhunter --check`
2. **Review logs**: Check `/var/mail/root` for logwatch reports
3. **Update system**: `sudo apt update && sudo apt upgrade`
4. **Monitor failed logins**: `sudo fail2ban-client status`
5. **Check firewall status**: `sudo ufw status verbose`

### SSH Key Authentication (Recommended Next Step)
```bash
# On your local machine:
ssh-keygen -t ed25519 -C "your-email@example.com"
ssh-copy-id deploy@your-server-ip

# Then on server, disable password authentication:
# Edit /etc/ssh/sshd_config.d/99-hardening.conf
# Set: PasswordAuthentication no
```

### GitHub SSH Key Setup

Run these as the `deploy` user (not root) so the keys live under `/home/deploy/.ssh/`:

```bash
# Make sure ~/.ssh exists with correct permissions
mkdir -p ~/.ssh && chmod 700 ~/.ssh

# Generate SSH key for GitHub (no passphrase — fine for a server-only key)
ssh-keygen -t ed25519 -C "github-vps-key" -f ~/.ssh/github_ed25519 -N ""

# Display public key to add to GitHub
cat ~/.ssh/github_ed25519.pub

# Append (NOT overwrite!) the GitHub entry to ~/.ssh/config.
# If you already have an entry for github.com, edit the file by hand instead.
cat << 'EOF' >> ~/.ssh/config
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/github_ed25519
    IdentitiesOnly yes
EOF

# Set proper permissions
chmod 600 ~/.ssh/config

# Add GitHub to known hosts
ssh-keyscan -t ed25519 github.com >> ~/.ssh/known_hosts

# Configure git identity (replace with your own)
git config --global user.name "your-username"
git config --global user.email "your-email@example.com"
```

**To add SSH key to GitHub:**
1. Go to https://github.com/settings/keys
2. Click "New SSH key"
3. Title: "VPS Server" (or preferred name)
4. Key type: Authentication Key
5. Paste the public key from `cat ~/.ssh/github_ed25519.pub`
6. Click "Add SSH key"

**Test connection:**
```bash
ssh -T git@github.com
# Should see: "Hi username! You've successfully authenticated..."
```

### Important Notes
- **SSH Access**: Root login is disabled. Use the 'deploy' user for SSH access
- **Firewall**: Only ports 22 (SSH), 80 (HTTP), and 443 (HTTPS) are open
- **Updates**: Security updates are automatically installed daily
- **Monitoring**: fail2ban is actively monitoring and blocking suspicious IPs

### System Information
- **OS**: Ubuntu (Linux kernel 6.8.0-71-generic)
- **Platform**: Linux x86_64
- **Memory**: 1.9GB RAM + 2GB Swap
- **Node.js**: v20.x LTS
- **Security Tools**: UFW, fail2ban, rkhunter, logwatch

---

## Troubleshooting

### If locked out via SSH:
1. Use Hetzner's console access
2. Check fail2ban: `sudo fail2ban-client status sshd`
3. Unban IP: `sudo fail2ban-client unban <your-ip>`

### Check service status:
```bash
sudo systemctl status ssh
sudo systemctl status fail2ban
sudo systemctl status ufw
```

### View security logs:
```bash
sudo journalctl -u ssh
sudo tail -f /var/log/auth.log
sudo tail -f /var/log/fail2ban.log
```

---

*Documentation created: August 17, 2025*
*Server secured with defense-in-depth approach implementing multiple security layers*