# Linux User Management & Security Hardening – Ubuntu 24.04

## Overview
This project demonstrates a full server-hardening workflow for Ubuntu 24.04.  
It covers secure user management, SSH hardening, firewall setup, intrusion-prevention tools, auditing, and automated security updates.  
The guide and automation script can be used as a baseline for production servers or cloud instances (AWS, GCP, Azure).

## Key Features
- **SSH Key-only Access** – disables password login and root SSH.
- **Firewall (UFW)** – default-deny rules with only required ports opened.
- **Fail2Ban** – protection against brute-force attacks.
- **Password Policy (PAM)** – 12-character minimum, 14-day warning, 90-day expiry.
- **Auditd Logging** – system-wide audit trails for critical files.
- **Automated Security Updates** – unattended upgrades configured.
- **Quick Automation Script** – one-shot Bash script to apply all settings.

## Repository Contents
- `Linux-Hardening-Ubuntu24.docx` – detailed step-by-step documentation.
- `hardening.sh` – automation script (customize variables before running).
- `README.md` – project summary and usage instructions.

## Usage
1. **Review the documentation**  
   Read `Linux-Hardening-Ubuntu24.docx` or the optional Markdown guide to understand each step.
2. **Customize the script**  
   Edit `hardening.sh` and replace:
   ```bash
   USER="opsuser"
   SSH_PUBKEY="ssh-rsa AAAA...your_public_key_here"
