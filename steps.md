# Linux User Management & Security Hardening – Ubuntu 24.04

This guide documents the full hardening process performed on an Ubuntu 24.04 server.

---

## What Was Implemented
- SSH key-only access  
- Disabled root SSH  
- UFW firewall  
- Fail2Ban  
- PAM password policy (14-day warning / 90-day max)  
- auditd logging  
- Automated security updates  

---

## Step01: System update + install tools
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ufw fail2ban auditd audispd-plugins unattended-upgrades logwatch libpam-pwquality

```
## Step02: Create non-root admin user(s) with SSH key auth
  ** create user (interactive - set a password) **
   
```bash
sudo adduser opsuser
```
  ** add to sudo group **
 
  ```bash
  sudo usermod -aG sudo opsuser
```
  ** create .ssh and authorized_keys (paste your public key) **
  ```bash
  sudo mkdir -p /home/opsuser/.ssh
  sudo tee /home/opsuser/.ssh/authorized_keys > /dev/null <<'KEY'
  ssh-rsa AAAA...your_public_key_here... user@host
  KEY
```
 
  
  ** set correct permissions **
   ```bash
    sudo chmod 700 /home/opsuser/.ssh
    sudo chmod 600 /home/opsuser/.ssh/authorized_keys
    sudo chown -R opsuser:opsuser /home/opsuser/.ssh
   ```
 
  
  
  Note: Do not remove your original key from ubuntu user until opsuser SSH test succeeds.


## Step03: Backup and harden SSH configuration
  ```bash
       sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
```
  

 ** SSH changes — open in editor OR apply safely with sed (I prefer manual edit to avoid mistakes): **
   ```bash
      sudo vi /etc/ssh/sshd_config
```
  
  

  ensure these lines (uncomment/modify accordingly):
  ```bash
  Port 22                # keep 22 unless you plan to change — if change, update AWS SG
  PermitRootLogin no
  PasswordAuthentication no    # ONLY after confirming key login works
  ChallengeResponseAuthentication no
  UsePAM yes
  PermitEmptyPasswords no
  # Optionally restrict login users:
  # AllowUsers opsuser ubuntu
```

  
  ** Then test config and reload: **
  ```bash
     sudo sshd -t    # syntax check
     sudo systemctl reload sshd
  ```
  
  
  Important: Keep your current session open; from another terminal try ssh -i key opsuser@<PUBLIC_IP> to confirm.
  
  ** If you change Port to custom (e.g., 2222), update AWS Security Group inbound rule to allow that port and run: **
  ```bash
       sudo ufw allow 2222/tcp
  ```



## Step04: Enable and configure UFW (firewall)
 ** Allow OpenSSH first (to avoid lockout), then HTTP/HTTPS as needed, then enable: **
  ```bash
    sudo ufw allow OpenSSH
    sudo ufw allow 80/tcp     # if web server
    sudo ufw allow 443/tcp    # if planning HTTPS
    sudo ufw enable
    sudo ufw status verbose
  ```
  

 
## Step05:  Install & enable Fail2Ban (protect SSH)
```bash
       sudo apt install -y fail2ban
  ```
  
# Create a local jail (safe to edit)
```bash
  sudo tee /etc/fail2ban/jail.d/local.conf > /dev/null <<'EOF'
  [sshd]
  enabled = true
  port    = ssh
  maxretry = 5
  bantime = 3600
  findtime = 600
  EOF
  ```
  ```bash
   sudo systemctl restart fail2ban
  ```

 

# check status
```bash
 sudo fail2ban-client status sshd
  ```
 
 
## Step06: Enforce password policy (PAM / pwquality)
  ** Backup then edit: **
  ```bash
    sudo cp /etc/pam.d/common-password /etc/pam.d/common-password.bak
    sudo sed -i 's/^password\s\+requisite\s\+pam_pwquality.so.*/password requisite pam_pwquality.so retry=3 minlen=12 difok=3/' /etc/pam.d/common-password || true
  ```
    


  ** Set password expiration policy: **
  ** set max 90 days, min 7 days, warn 14 days **
  
  ```bash
    sudo chage -M 90 -m 7 -W 14 opsuser
  ```
    
 
## Step07: File & webroot permissions (for hosting web)
  ** Set secure permissions for /var/www: **
  ```bash
  sudo chown -R www-data:www-data /var/www
  sudo find /var/www -type d -exec chmod 755 {} \;
  sudo find /var/www -type f -exec chmod 644 {} \;
  ```
  

## Step08: Disable unused services
  ** List enabled services: **
  ```bash
  systemctl list-unit-files --type=service | grep enabled
  Disable anything unnecessary :
  sudo systemctl disable --now avahi-daemon || true
  ```
  
  Caution: Don’t disable cloud-init or sshd. Only stop services you understand.

## Step09: Install & configure auditd (important)
    ```bash
     sudo apt install -y auditd audispd-plugins
    ```
 

  ** Add basic watch rules (create a file) **
  ```bash
    sudo tee /etc/audit/rules.d/hardening.rules > /dev/null <<'AUDP'
    -w /etc/passwd -p wa -k identity
    -w /etc/shadow -p wa -k identity
    -w /etc/group -p wa -k identity
    -w /etc/sudoers -p wa -k scope
    -w /var/log/auth.log -p wa -k authlog
    AUDP
  
    sudo augenrules --load
    sudo systemctl restart auditd
  ```
 


  ** check status **
  ```bash
  sudo auditctl -l
  ```
 
  
  
## Step10: Enable automatic security updates
```bash
  sudo apt install -y unattended-upgrades
  sudo dpkg-reconfigure -plow unattended-upgrades
```
  

  ** quick verify file: **
```bash
   cat /etc/apt/apt.conf.d/20auto-upgrades
```
  
  
  
## Step11: AppArmor hardening (Ubuntu)
 ** Check status & enable profile for services (Apache): **
  ```bash
   sudo aa-status
```
  
  
  ** Enforce apache profile (example) **
  ```bash
   sudo aa-enforce /etc/apparmor.d/usr.sbin.apache2 || true
```
  

## Step12: Centralized logging / rotate config
  Ensure logrotate and logwatch installed (we installed logwatch earlier). Configure mail alerts or export logs to external host/S3 (advanced).

## Step13: Enforce UMASK and secure cron
  Set default umask in /etc/profile or systemd service overrides. Audit cron jobs and restrict by permissions.
  
Step14: SSH banner and session timeout
  Add to /etc/ssh/sshd_config:
  ClientAliveInterval 300
  ClientAliveCountMax 2
  And add Banner ```bash /etc/issue.net``` with custom message as needed.

