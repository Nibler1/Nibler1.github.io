# Archetype - HTB Write-Up

## Introduction
Archetype is an Easy Windows machine from HTB Starting Point that focuses on SMB enumeration, MSSQL exploitation, and Windows privilege escalation through credential hunting.

---

## Enumeration
Found key ports with nmap:
- Port 445: SMB
- Port 1433: MSSQL (SQL Server)

---

## SMB Enumeration
Discovered SMB shares with anonymous access.
Found a directory accessible without credentials.
Downloaded files containing configuration data.

---

## MSSQL Exploitation
Found MSSQL credentials in the downloaded files.

Connected to SQL Server using mssqlclient:
```bash
mssqlclient.py USERNAME@10.129.x.x -windows-auth
```

---

## Enabling xp_cmdshell
The xp_cmdshell feature was disabled by default.
Enabled it through SQL commands to allow OS command execution.

---

## Getting Shell
Used xp_cmdshell to execute PowerShell commands.
Gained remote code execution on the Windows system.
Obtained reverse shell.

---

## Privilege Escalation
Uploaded and ran WinPEAS for enumeration.

### Finding Credentials
WinPEAS discovered PowerShell history file:
```
ConsoleHost_history.txt
```

This file contained administrator credentials.

---

## Getting Administrator Access
Used the discovered credentials to authenticate as Administrator.
Retrieved the root flag.

---

## Flags
- User flag: Retrieved from user directory
- Root flag: Retrieved from Administrator directory

---

## Tools Used
- nmap - Port scanning
- smbclient - SMB enumeration
- mssqlclient.py - SQL Server connection
- WinPEAS - Windows privilege escalation enumeration
- PowerShell - Command execution

---

## What I Learned
- SMB anonymous access can expose sensitive configuration files
- MSSQL xp_cmdshell allows OS command execution
- PowerShell history files can contain credentials
- Always check command history files for sensitive data
- Windows enumeration requires different tools than Linux

---

## Conclusion
Archetype demonstrates how misconfigurations in Windows services (SMB anonymous access, MSSQL settings) combined with poor credential hygiene (storing passwords in command history) can lead to full system compromise.

Key lesson: Never store credentials in command history, and restrict SMB access appropriately.

---

**Disclaimer:** This write-up is for educational purposes only. Always obtain proper authorization before testing security on systems you don't own. Unauthorized access to computer systems is illegal.
