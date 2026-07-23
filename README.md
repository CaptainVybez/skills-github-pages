# 🛡️ Penetration Testing Report: Mr. Robot 1 (VulnHub / TryHackMe)

![Target OS](https://img.shields.io/badge/OS-Linux%20Ubuntu-orange)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-yellow)
![Status](https://img.shields.io/badge/Status-Completed%20%2F%20Rooted-brightgreen)

## 📌 Executive Summary
This document details the complete penetration testing methodology executed against the **Mr. Robot: 1** vulnerable virtual machine. The test covered the full offensive kill chain, progressing from initial reconnaissance and web directory enumeration to credential brute-forcing, arbitrary code execution, lateral movement, and host-level privilege escalation to full `root` access.

---

## 🎯 Target Information
- **Target IP:** `192.168.56.102`
- **Attacker IP (Kali):** `192.168.56.101`
- **Environment:** Isolated Host-Only Lab Environment

---

## ⚔️ Attack Kill Chain Breakdown

### Phase 1: Reconnaissance & Enumeration
- **Network Scanning:** Identified HTTP (Port 80/443) open on target host.
- **Directory Discovery:** Inspected `http://192.168.56.102/robots.txt` and discovered two key endpoints:
  - `key-1-of-3.txt` (First Flag Captured)
  - `fsocity.dic` (Raw dictionary wordlist containing 800,000+ entries)

- **Data Processing:** Cleaned and deduplicated `fsocity.dic` locally using Unix pipeline tools to optimize dictionary attack efficiency:
  ```bash
  sort fsocity.dic | uniq > cleaned_wordlist.txt
  # Reduced wordlist from ~800,000 lines down to 11,452 unique passwords
  ```

---

### Phase 2: Vulnerability Analysis & Initial Access
- **Vector:** WordPress Login Portal (`/wp-login.php`)
- **Brute Force Attack:** Executed multi-threaded HTTP POST form brute-forcing via **Hydra**:
  ```bash
  hydra -l elliot -P cleaned_wordlist.txt 192.168.56.102 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:F=incorrect" -t 16
  ```
- **Discovered Credentials:** 
  - **Username:** `elliot`
  - **Password:** `ER28-0652`

---

### Phase 3: Exploitation & Remote Code Execution (RCE)
- **Arbitrary File Modification:** Authenticated into WordPress Admin Panel (`Appearance > Editor`).
- **Webshell Injection:** Modified active theme file `404.php` with a PHP reverse shell payload set to dial back to Kali (`192.168.56.101:4444`).
- **Shell Capture:** Initialized Netcat listener (`nc -lvnp 4444`) and triggered execution via HTTP GET request:
  ```text
  [http://192.168.56.102/index.php/does-not-exist](http://192.168.56.102/index.php/does-not-exist)
  ```
- **Access Level Obtained:** `daemon` (uid=1, gid=1)

---

### Phase 4: Post-Exploitation & Lateral Movement
- **Terminal Stabilization:** Upgraded dumb netcat connection to interactive PTY shell:
  ```bash
  python -c 'import pty; pty.spawn("/bin/bash")'
  ```
- **Credential Harvesting:** Discovered world-readable hash in `/home/robot/password.raw-md5`:
  - Hash: `robot:c3fcd3d76192e4007dfb496cca67e13b`
  - Decoded MD5 String: `abcdefghijklmnopqrstuvwxyz`
- **User Escalation:** Switched user execution context to target account:
  ```bash
  su robot
  cat /home/robot/key-2-of-3.txt
  ```
- **Captured Flag 2:** `822c73956184f694993bede3eb39f959`

---

### Phase 5: Root Privilege Escalation
- **Audit Vector:** Searched filesystem for executable files with the SUID bit set (`-perm -u=s`):
  ```bash
  find / -perm -u=s -type f 2>/dev/null
  ```
- **Vulnerability Identified:** Identified legacy `nmap` binary (`/usr/local/bin/nmap`) with improper SUID permissions owned by `root`.
- **SUID Exploitation:** Abused Nmap interactive mode (available in v2.02–v5.21) to spawn an elevated root shell context:
  ```bash
  nmap --interactive
  nmap> !sh
  whoami  # Returns root
  ```
- **Captured Flag 3:** `04787ddef27c3dee1ee161b21670b4e4` (`/root/key-3-of-3.txt`)

---

## 🛠️ Tools & Technologies Used
| Tool | Category | Purpose |
| :--- | :--- | :--- |
| `sort` / `uniq` | Data Science / CLI | Wordlist deduplication and dictionary optimization |
| **Hydra** | Web Attack | Multi-threaded HTTP POST authentication brute-forcing |
| **PentestMonkey PHP** | Payload | Reverse TCP shell payload execution |
| **Netcat (`nc`)** | Networking | Listener for inbound TCP reverse connection |
| **Python (`pty`)** | Post-Exploit | Spawning interactive pseudo-terminal environment |
| **Linux `find`** | Enumeration | SUID binary privilege escalation search |
| **Nmap** | PrivEsc | Interactive mode shell escape execution |

---

## 🔒 Security Remediation Guidelines
1. **Disable SUID on Non-Essential Binaries:** Remove SUID permissions from binary tools like `nmap` using `chmod u-s /usr/local/bin/nmap`.
2. **Account Lockout Policy:** Enforce login rate-limiting or CAPTCHA on `/wp-login.php` to neutralize automated Hydra/WPScan dictionary attacks.
3. **Restrict File Editing:** Disable line-item PHP code edits directly from the WordPress dashboard by adding `define('DISALLOW_FILE_EDIT', true);` to `wp-config.php`.
4. **Strong Password Hashing:** Re-hash stored passwords using modern salt-and-hash functions (Bcrypt/Argon2) rather than legacy un-salted MD5.
