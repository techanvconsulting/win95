+++
author = "Anubhav Gain"
title = "Linux Hardening: The Complete Checklist for Production Servers"
date = "2026-04-06"
description = "Default Linux installs are not production-ready. This is the complete hardening checklist — from SSH config to kernel parameters to audit logging."
tags = ["linux", "hardening", "security", "sysadmin", "server"]
categories = ["security", "sysadmin"]
+++

A default Linux installation is optimized for compatibility and convenience, not for production security. Root login is enabled. Password authentication is on. The kernel ships with conservative defaults that favor interoperability over hardening. Package managers pull from unverified mirrors unless you configure them otherwise. Out of the box, a fresh server is a reasonably sized attack surface waiting for exposure. This checklist works through the layers — SSH, user accounts, filesystem, kernel, firewall, packages, auditing, and MAC enforcement — and gives you the concrete configuration to close them.

<!--more-->

## 1. SSH Hardening

SSH is the primary attack surface on any internet-facing server. Default configuration leaves too much open.

Edit `/etc/ssh/sshd_config`:

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
Port 2222
AllowUsers deploy admin
X11Forwarding no
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
```

Restart the service after changes:

```bash
systemctl restart sshd
```

Install and configure `fail2ban` to block repeated authentication failures:

```bash
apt install fail2ban
```

Create `/etc/fail2ban/jail.local`:

```ini
[sshd]
enabled = true
port = 2222
maxretry = 3
bantime = 3600
findtime = 600
```

```bash
systemctl enable --now fail2ban
```

## 2. User and Sudo Configuration

Principle of least privilege applies at the account level. No service account should carry more permissions than its function requires.

```bash
# Create a dedicated deploy user with no shell for services
useradd -r -s /usr/sbin/nologin appuser

# Lock accounts that are not in active use
passwd -l unused-account

# Audit accounts with UID 0 (should only be root)
awk -F: '($3 == 0) { print $1 }' /etc/passwd
```

When granting sudo, avoid `NOPASSWD`. Every privilege escalation should require authentication:

```bash
# Good — requires password
deploy ALL=(ALL) /usr/bin/systemctl restart myapp

# Avoid this in production
deploy ALL=(ALL) NOPASSWD: ALL
```

Tighten PAM to enforce password quality and account lockout:

```bash
apt install libpam-pwquality
```

In `/etc/pam.d/common-password`:

```
password requisite pam_pwquality.so retry=3 minlen=14 dcredit=-1 ucredit=-1 ocredit=-1 lcredit=-1
```

In `/etc/pam.d/common-auth`, add account lockout:

```
auth required pam_faillock.so preauth silent deny=5 unlock_time=900
auth required pam_faillock.so authfail deny=5 unlock_time=900
```

## 3. Filesystem Hardening

Mount `/tmp` with `noexec`, `nosuid`, and `nodev` to prevent execution of uploaded or temporary files:

```bash
# In /etc/fstab
tmpfs /tmp tmpfs defaults,noexec,nosuid,nodev 0 0
```

For production systems, `/var` and `/home` should live on separate partitions. This limits the blast radius of a full-disk condition and makes mount option enforcement cleaner.

Audit SUID and SGID binaries — every one of these is a potential privilege escalation path:

```bash
find / -perm /4000 -o -perm /2000 2>/dev/null | sort
```

Review the output and remove the SUID bit from anything that does not strictly require it:

```bash
chmod u-s /path/to/binary
```

Restrict world-writable files and directories:

```bash
find / -xdev -type f -perm -0002 2>/dev/null
find / -xdev -type d -perm -0002 ! -perm -1000 2>/dev/null
```

## 4. Kernel Hardening with sysctl

The kernel exposes dozens of network and memory parameters that are security-relevant. Apply these in `/etc/sysctl.d/99-hardening.conf`:

```ini
# Disable IP forwarding (unless this is a router)
net.ipv4.ip_forward = 0
net.ipv6.conf.all.forwarding = 0

# Disable ICMP redirects
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0

# Enable SYN cookies (SYN flood mitigation)
net.ipv4.tcp_syncookies = 1

# Ignore ICMP broadcast requests
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Disable source routing
net.ipv4.conf.all.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0

# Enable reverse path filtering
net.ipv4.conf.all.rp_filter = 1

# Restrict dmesg access to root
kernel.dmesg_restrict = 1

# Restrict kernel pointer exposure
kernel.kptr_restrict = 2

# Disable magic SysRq key
kernel.sysrq = 0
```

Apply immediately:

```bash
sysctl --system
```

## 5. Firewall: Default Deny

Start from a default-deny posture and open only what is explicitly required.

Using `ufw`:

```bash
ufw default deny incoming
ufw default deny outgoing
ufw allow out 53
ufw allow out 80
ufw allow out 443
ufw allow in 2222/tcp
ufw allow in 443/tcp
ufw enable
```

For production systems with more complex requirements, use `iptables` or `nftables` directly and version-control the ruleset. Save and restore on boot:

```bash
iptables-save > /etc/iptables/rules.v4
# Restore via iptables-persistent or a systemd service
```

## 6. Package Management Hygiene

Enable unattended security upgrades so critical CVEs do not linger:

```bash
apt install unattended-upgrades
dpkg-reconfigure --priority=low unattended-upgrades
```

Configure `/etc/apt/apt.conf.d/50unattended-upgrades` to restrict to security updates only and enable automatic reboot with a defined window if kernel updates require it.

Remove packages that are not required — every installed package is potential attack surface:

```bash
apt autoremove --purge
apt purge telnet rsh-client rsh-redone-client
```

Verify package signatures and integrity:

```bash
# Verify installed package against its expected checksums
debsums -c package-name

# Check for packages not from official repos
apt-key list
```

## 7. Auditd: Logging What Matters

`auditd` is the kernel-level audit framework. Install it and apply rules that cover the changes attackers make most frequently:

```bash
apt install auditd audispd-plugins
```

Add to `/etc/audit/rules.d/hardening.rules`:

```
# Watch critical identity files
-w /etc/passwd -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/group -p wa -k identity
-w /etc/sudoers -p wa -k sudoers
-w /etc/sudoers.d/ -p wa -k sudoers

# Privilege escalation attempts
-a always,exit -F arch=b64 -S setuid -S setgid -k privilege_escalation
-a always,exit -F arch=b64 -S setreuid -S setregid -k privilege_escalation

# Execution of binaries (high volume — scope carefully)
-a always,exit -F arch=b64 -S execve -k exec

# SSH key changes
-w /root/.ssh -p wa -k ssh_keys
-w /home -p wa -k home_ssh

# Unauthorized access attempts
-a always,exit -F arch=b64 -S open -F exit=-EACCES -k access
-a always,exit -F arch=b64 -S open -F exit=-EPERM -k access
```

```bash
augenrules --load
systemctl enable --now auditd
```

Query audit logs with:

```bash
ausearch -k sudoers --start recent
aureport --auth --summary
```

## 8. AppArmor / SELinux: Mandatory Access Control

MAC enforcement provides a second line of defense when process compromise occurs. On Ubuntu/Debian, AppArmor ships by default. Verify it is enforcing:

```bash
aa-status
```

Enable enforce mode for any profile currently in complain mode:

```bash
aa-enforce /etc/apparmor.d/*
```

For services without existing profiles, generate one in complain mode, run the application under normal load, then convert to enforce:

```bash
aa-genprof /usr/sbin/myservice
# Exercise the application, then:
aa-enforce /etc/apparmor.d/usr.sbin.myservice
```

On RHEL/CentOS/Rocky, use SELinux. Verify enforcing mode:

```bash
getenforce
# Should return: Enforcing
setenforce 1
# Make permanent in /etc/selinux/config: SELINUX=enforcing
```

Never set `SELINUX=disabled` in production. If a service breaks under SELinux, fix the policy — do not disable enforcement.

## 9. File Integrity and Real-Time Monitoring

Static hardening is a point-in-time action. You need detection when that posture drifts.

**AIDE** (Advanced Intrusion Detection Environment) builds a baseline of filesystem state and alerts on changes:

```bash
apt install aide
aideinit
mv /var/lib/aide/aide.db.new /var/lib/aide/aide.db

# Schedule daily checks
echo "0 3 * * * root /usr/bin/aide --check" >> /etc/crontab
```

For real-time detection beyond filesystem integrity — active exploit attempts, lateral movement, privilege escalation chains, and anomalous process behavior — deploy the **OpenArmor** agent. OpenArmor runs as a lightweight host agent and correlates kernel-level events against behavioral detection rules, with alerting and response actions that integrate with your existing SIEM or ticketing stack. It is open-source, auditable, and built to complement rather than replace tools like auditd and AIDE.

```bash
# Install the OpenArmor agent (see techanv.com/docs for current install instructions)
curl -s https://install.openarmor.io | bash
```

The combination of AIDE for integrity baselining, auditd for kernel-level event logging, and OpenArmor for behavioral detection gives you the visibility layer a hardened server needs — hardening reduces the attack surface, monitoring tells you when something gets through anyway.

---

A hardened server is not a one-time configuration. It is a baseline that needs to be version-controlled, periodically re-audited, and kept current as the threat landscape and your software stack evolve. Start with this checklist, test each change in staging before production, and treat the hardening configuration as infrastructure code — reviewed, committed, and deployed the same way as everything else.

**The OpenArmor project, documentation, and detection rule library are available at [techanv.com](https://techanv.com). It is open-source — run it, read it, and contribute.**
