# Metasploitable-2-Penetration-Testing-Security-Assessment

## Project Overview

The objective of this project is to practice the methodology used during a penetration test, including:

- Network reconnaissance
- Port and service enumeration
- Vulnerability identification
- Service analysis
- Exploitation
- Privilege assessment
- Post-exploitation analysis
- Security risk assessment
- Defensive recommendations

The project also incorporates a Blue Team perspective by documenting the security implications of discovered vulnerabilities and identifying appropriate remediation and detection opportunities.

---

## Lab Environment

| Component | Details |
|-----------|---------|
| Attacker Machine | Kali Linux |
| Target Machine | Metasploitable 2 |
| Target IP | 172.30.1.21 |
| Target Hostname | metasploitable.localdomain |
| Virtualization | Oracle VirtualBox |
| Network | Isolated Virtual Lab |
| Target OS | Linux 2.6.x |

---

## Authorization

All testing documented in this repository was performed against an intentionally vulnerable virtual machine within an authorized personal cybersecurity lab environment.

**No unauthorized systems were targeted.**

---

## 1. Reconnaissance

### Initial Nmap Scan

The initial scan was performed using Nmap to identify open ports, services, operating system information, and potential attack surfaces.

```bash
sudo nmap -A 172.30.1.21 -oN nmap-aggressiveScan.txt
```

### OS Detection

Nmap identified the target as:

- **Running:** Linux 2.6.X
- **OS details:** Linux 2.6.9 - 2.6.33
- **Hostname:** metasploitable.localdomain
- **Virtual NIC:** Oracle VirtualBox

---

## 2. Discovered Services

The initial scan identified a large number of exposed services.

| Port | Service | Version / Information |
|------|---------|----------------------|
| 111 | RPCBind | RPC #100000 |
| 139 | SMB | Samba 3.X–4.X |
| 445 | SMB | Samba 3.0.20-Debian |
| 512 | rexec | Netkit-rsh |
| 513 | rlogin | OpenBSD/Solaris rlogind |
| 514 | rsh | Netkit rshd |
| 1099 | Java RMI | GNU Classpath grmiregistry |
| 1524 | Bindshell | Metasploitable root shell |
| 2049 | NFS | NFS v2–4 |
| 2121 | FTP | ProFTPD 1.3.1 |
| 3306 | MySQL | MySQL 5.0.51a |
| 5432 | PostgreSQL | PostgreSQL 8.3 |
| 5900 | VNC | VNC protocol 3.3 |
| 6000 | X11 | Access denied |
| 6667 | IRC | UnrealIRCd |
| 8009 | AJP13 | Apache JServ |
| 8180 | HTTP | Apache Tomcat 5.5 |

**The number of exposed services demonstrates that the target has a very large attack surface.**

---

## 3. SMB Enumeration

Nmap identified:

```
445/tcp open netbios-ssn Samba smbd 3.0.20-Debian
```

### SMB Information

- **OS:** Unix
- **Computer name:** metasploitable
- **Domain name:** localdomain
- **FQDN:** metasploitable.localdomain

### SMB Security Configuration

Nmap reported:

- **account_used:** guest
- **authentication_level:** user
- **challenge_response:** supported
- **message_signing:** disabled

### Security Significance

Several security concerns were identified:

- The server is running an obsolete Samba version
- Guest authentication information was exposed during enumeration
- SMB message signing is disabled
- Legacy SMB services increase the available attack surface

**The Samba version is particularly significant because Samba 3.0.20 is associated with known historical vulnerabilities.**

### Defensive Recommendations

- Upgrade Samba to a supported release
- Disable unnecessary guest access
- Require SMB signing where appropriate
- Disable SMBv1 and other obsolete protocols where applicable
- Restrict SMB access using host-based and network-level firewall rules
- Monitor authentication and SMB activity through centralized logging

---

## 4. NFS Enumeration

RPC enumeration identified NFS:

```
111/tcp  open rpcbind
2049/tcp open nfs
```

### Export List

Command used:
```bash
showmount -e 172.30.1.21
```

**Result:**
```
Export list for 172.30.1.21:
/ *
```

Nmap confirmed the same result:
```bash
sudo nmap -p 111,2049 --script nfs-showmount 172.30.1.21
```

### Security Significance

**The root filesystem `/` is exported to `*`.**

This represents a severe NFS configuration issue because the export is not restricted to a specific trusted host or network. Depending on the NFS configuration and filesystem permissions, this could expose sensitive files and system information to unauthorized clients.

### Planned Validation

The next step is to safely determine whether the exported filesystem can be mounted:

```bash
sudo mkdir -p /mnt/metasploitable-nfs
sudo mount -t nfs -o ro,nolock 172.30.1.21:/ /mnt/metasploitable-nfs
```

The filesystem will initially be mounted read-only to minimize changes to the target.

---

## 5. FTP Enumeration

The initial aggressive scan identified:

```
2121/tcp open ftp ProFTPD 1.3.1
```

A targeted scan was subsequently performed:

```bash
sudo nmap -p 2121 --script ftp-anon,ftp-syst 172.30.1.21
```

### Service Identification

Nmap reported: `2121/tcp open ccproxy-ftp`

**The difference between the service labels demonstrates why service enumeration should be validated rather than relying solely on a single Nmap result.**

### Planned Validation

The FTP banner can be manually retrieved using:

```bash
nc -nv 172.30.1.21 2121
```

The service version can also be reassessed with:

```bash
sudo nmap -sV -p 2121 172.30.1.21
```

### Security Considerations

**ProFTPD 1.3.1 is an extremely old release and should be investigated for known vulnerabilities and insecure configuration.**

---

## 6. Apache Tomcat Enumeration

The target exposed:

```
8009/tcp open ajp13
8180/tcp open http
```

Nmap identified the HTTP service as:

- Apache Tomcat/Coyote JSP engine 1.1
- Apache Tomcat/5.5

### HTTP Enumeration Scan

```bash
sudo nmap -p 8180 --script http-enum,http-title,http-methods 172.30.1.21
```

### Results

- **8180/tcp:** open
- **http-title:** Apache Tomcat/5.5
- **Supported Methods:** GET, HEAD, POST, OPTIONS

### Security Significance

- Apache Tomcat 5.5 is an obsolete application server release and represents a significant attack surface
- The presence of AJP on port 8009 warrants investigation because AJP is an internal application-server protocol that should generally not be unnecessarily exposed

### Planned Enumeration

The HTTP response headers can be reviewed with:

```bash
curl -I http://172.30.1.21:8180
```

The web application can also be accessed from the Kali system:

```
http://172.30.1.21:8180
```

---

## 7. Root Bindshell

One of the most significant findings from the initial scan was:

```
1524/tcp open bindshell Metasploitable root shell
```

Nmap specifically identified this service as the Metasploitable root shell.

### Security Significance

**A remotely accessible root shell represents a critical security issue** because successful access can provide the highest level of privilege on the operating system.

This service will be treated as a known vulnerability within the lab and documented separately from vulnerabilities discovered through independent enumeration.

---

## 8. Other Identified Attack Surface

### RSH / Rlogin

```
512/tcp exec
513/tcp login
514/tcp shell
```

Legacy remote-access protocols that can provide significant security risk because they lack the security properties expected from modern encrypted administration protocols.

### MySQL

```
3306/tcp open mysql
MySQL 5.0.51a-3ubuntu5
```

### PostgreSQL

```
5432/tcp open postgresql
PostgreSQL 8.3.0 - 8.3.7
```

Both database versions are extremely old and warrant vulnerability and authentication enumeration.

### VNC

```
5900/tcp open vnc
VNC protocol 3.3
```

Remote graphical access represents another potential attack surface.

### IRC

```
6667/tcp open irc
UnrealIRCd
```

UnrealIRCd is known to have historical security vulnerabilities and is therefore another service requiring investigation.

---

## 9. Preliminary Risk Assessment

| Finding | Severity | Reason |
|---------|----------|--------|
| Root bindshell on 1524 | **Critical** | Direct root-level remote access |
| NFS exporting / to * | **Critical** | Excessive filesystem exposure |
| Samba 3.0.20 | **Critical/High** | Obsolete and historically vulnerable |
| Legacy RSH/Rlogin | **High** | Insecure remote administration protocols |
| Tomcat 5.5 | **High** | Obsolete application server |
| Exposed AJP | **Medium/High** | Additional application-server attack surface |
| ProFTPD 1.3.1 | **High** | Obsolete FTP software |
| MySQL 5.0 | **High** | Obsolete database software |
| PostgreSQL 8.3 | **High** | Obsolete database software |
| VNC | **Medium/High** | Remote graphical access |
| UnrealIRCd | **High** | Legacy software with historical vulnerabilities |

*Severity ratings are preliminary and will be refined as additional enumeration and validation are performed.*

---

## 10. Methodology

The assessment follows a structured penetration-testing methodology:

```
Reconnaissance
    ↓
Port Scanning
    ↓
Service Enumeration
    ↓
Vulnerability Identification
    ↓
Vulnerability Validation
    ↓
Exploitation
    ↓
Privilege Assessment
    ↓
Post-Exploitation
    ↓
Evidence Collection
    ↓
Risk Assessment
    ↓
Remediation
    ↓
Validation
```

The goal is to understand not only how a vulnerability can be exploited, but also how the vulnerability could be detected and remediated from a defensive security perspective.

---

## 11. Blue Team Perspective

For each vulnerability identified, the project will attempt to answer:

- What is the vulnerability?
- Why is it dangerous?
- How could an attacker exploit it?
- What evidence would exploitation leave behind?
- What logs or SIEM events could detect the activity?
- How should the vulnerability be remediated?
- How can remediation be validated?

This approach connects penetration testing with SOC and incident-response practices.

---

## 12. Planned Next Steps

- [x] Identify target
- [x] Perform aggressive Nmap scan
- [x] Identify operating system
- [x] Enumerate exposed services
- [x] Enumerate NFS exports
- [x] Confirm Tomcat service
- [x] Identify SMB security configuration
- [ ] Safely mount and inspect NFS export
- [ ] Enumerate SMB shares
- [ ] Validate FTP service/banner
- [ ] Enumerate Tomcat applications
- [ ] Research identified vulnerabilities
- [ ] Prioritize vulnerabilities
- [ ] Perform controlled exploitation
- [ ] Document initial access
- [ ] Perform privilege assessment
- [ ] Perform post-exploitation enumeration
- [ ] Identify potential detection mechanisms
- [ ] Develop remediation recommendations
- [ ] Validate remediation
- [ ] Complete final penetration-testing report

---

## 13. Tools Used

- Kali Linux
- Nmap
- NFS utilities
- Netcat
- cURL
- Metasploit Framework (planned)
- VirtualBox
- Linux command-line utilities

**Additional tools will be documented as they are introduced into the assessment.**

---

## 14. Key Lessons Learned

### Attack Surface Matters

A single host can expose numerous services, each representing a potential avenue for compromise.

### Enumeration Comes Before Exploitation

Version information, service configuration, authentication methods, and exposed resources can reveal significant vulnerabilities before exploitation begins.

### Misconfiguration Can Be as Dangerous as Vulnerable Software

The NFS configuration:

```
/ *
```

demonstrates how an insecure configuration can create substantial risk even without exploiting a software vulnerability.

### Defensive Analysis Is Important

Identifying a vulnerability is only part of a security assessment. Understanding detection, remediation, and validation provides a more complete picture of the security problem.

---

## Disclaimer

This repository is intended for educational cybersecurity research and authorized lab environments.

**The techniques documented here should only be performed against systems for which the tester has explicit permission to conduct security testing.**

The target used in this project is a deliberately vulnerable Metasploitable virtual machine running in an isolated personal laboratory environment.
