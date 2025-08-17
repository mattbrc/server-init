# Server Security & Setup Documentation

## Summary

This Hetzner VPS Ubuntu server has been secured with multiple layers of defense-in-depth security measures and essential development tools.

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

### Phase 1: Initial Server Setup (User Completed)

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

### Phase 2: Enhanced Security Measures (Claude Completed)

#### 1. Create Non-Root User with Sudo Privileges
```bash
# Created new user 'deploy'
sudo useradd -m -s /bin/bash deploy
echo "deploy:<random-password>" | sudo chpasswd

# Added to sudo group
sudo usermod -aG sudo deploy
```
**Purpose**: Reduces risk by avoiding direct root access for daily operations

#### 2. SSH Hardening
Created `/etc/ssh/sshd_config.d/99-hardening.conf`:
```
PermitRootLogin no              # Disable root SSH login
PasswordAuthentication yes      # Allow password auth (consider disabling after setting up keys)
PermitEmptyPasswords no        # No empty passwords
PubkeyAuthentication yes        # Enable public key authentication
MaxAuthTries 3                  # Limit authentication attempts
MaxSessions 10                  # Limit concurrent sessions
ClientAliveInterval 300         # Disconnect idle clients after 5 minutes
ClientAliveCountMax 2           # Max keep-alive messages
X11Forwarding no               # Disable X11 forwarding
AllowUsers deploy              # Only allow specific user
Protocol 2                     # Use SSH protocol 2 only
```
**Purpose**: Prevents brute force attacks and unauthorized access

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

# Disable IPv6 (if not needed)
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1

# Restrict core dumps
fs.suid_dumpable = 0

# Hide kernel pointers
kernel.kptr_restrict = 2

# Enable ASLR (Address Space Layout Randomization)
kernel.randomize_va_space = 2
```
**Purpose**: Hardens the kernel against various network attacks and information disclosure

#### 4. Rootkit Detection (rkhunter)
```bash
# Installed rkhunter
sudo apt install -y rkhunter

# Fixed configuration
sudo sed -i 's|WEB_CMD="/bin/false"|WEB_CMD=""|' /etc/rkhunter.conf

# Updated database
sudo rkhunter --propupd
```
**Purpose**: Scans for rootkits, backdoors, and local exploits

#### 5. Log Monitoring (logwatch)
```bash
# Installed logwatch
sudo apt install -y logwatch

# Created configuration
sudo mkdir -p /etc/logwatch/conf
cat << 'EOF' | sudo tee /etc/logwatch/conf/logwatch.conf
MailTo = root
MailFrom = Logwatch <logwatch@localhost>
Detail = Med
Service = All
Range = yesterday
EOF
```
**Purpose**: Daily system activity reports for security monitoring

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