# Vaccine - HTB Write-Up
## Introduction
Vaccine is an Easy Linux machine from HTB Starting Point that teaches FTP enumeration, password cracking, SQL injection, and privilege escalation.
- -
## Enumeration
Found 3 ports with nmap:
- Port 21: FTP
- Port 22: SSH
- Port 80: HTTP
- -
## FTP Access
Connected with anonymous login. Found backup.zip file.
- -
## Password Cracking
backup.zip was password-protected.
Used zip2john and john with rockyou wordlist.
Successfully cracked the password.
- -
## Credentials Found
Extracted backup files. Found login credentials in index.php.
- -
## SQL Injection
Logged into admin panel at port 80.
Found SQL injection in search field.
Used sqlmap to exploit it:
```bash
sqlmap -r request.txt - os-shell - batch
```
Got OS command execution through the database.
- -
## Getting Shell
From sqlmap os-shell, executed reverse shell command.
Got shell as postgres user.
- -
## Privilege Escalation
Checked sudo permissions:
```bash
sudo -l
```
Found I could run vi as root on a PostgreSQL configuration file.
Escaped from vi to get root shell:
```
:!/bin/bash
```
Got root access!
- -
## Flags
- User flag: Retrieved from postgres user home directory
- Root flag: Retrieved from /root directory
- -
## Tools Used
- nmap - Port scanning
- ftp - FTP client
- zip2john - Convert ZIP to crackable format
- john - Password cracking
- sqlmap - SQL injection exploitation
- netcat - Reverse shell listener
- vi - Privilege escalation
- -
## What I Learned
- FTP anonymous login can expose sensitive files
- Weak passwords are easily cracked with common wordlists
- SQL injection can lead to OS command execution
- Text editors with sudo access are dangerous
- Always check sudo -l for privilege escalation
- -
## Conclusion
Vaccine demonstrates how multiple small vulnerabilities can be chained together for full system compromise. Key lesson: proper security at every level is essential.
Disclaimer: This write-up is for educational purposes only. Always obtain proper authorization before testing security on systems you don't own. Unauthorized access to computer systems is illegal.
