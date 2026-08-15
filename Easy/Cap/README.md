# Cap — HTB Easy

**Difficulty:** Easy
**OS:** Linux
**Path:** Easy

## Summary

Cap involves an IDOR-style vulnerability in a "Security Dashboard" web app that exposes previous packet captures by simply changing a numeric ID in the URL. An earlier capture contained plaintext FTP credentials, which were reused for SSH login as the `nathan` user. Privilege escalation to root was achieved by abusing a Linux capability (`cap_setuid`) set on the `python3.8` binary.

## Recon

```
nmap -sC -sV -T4 -p- <TARGET_IP>
```

**Results:**

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Gunicorn
|_http-server-header: gunicorn
|_http-title: Security Dashboard
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

Three services open: FTP, SSH, and an HTTP app titled "Security Dashboard" running on Gunicorn (Python backend).

## Enumeration

- Tried anonymous FTP login (`ftp -a <TARGET_IP>`) — failed, no anonymous access allowed.
- Tried a guessed SSH credential pair — failed.
- Browsed to the web app on port 80. The dashboard auto-authenticated as a user named `nathan` with no login step, worth noting as unusual app behavior.
- Explored the sidebar navigation, specifically the "Security Snapshot" feature, which linked to `/data/1`.
- Manually changed the URL parameter to `/data/0` and found an earlier packet capture was accessible — no access control preventing enumeration of other capture IDs (IDOR).

## Exploitation

Downloaded the `.pcap` file returned from `/data/0` and opened it in Wireshark. Filtered for FTP traffic and found a plaintext FTP login in the capture, including a cleartext password.

Since the web app was already showing a logged-in user `nathan`, tried the same username with the password recovered from the packet capture against SSH:

```
ssh nathan@<TARGET_IP>
```

Login succeeded. Captured the user flag from the home directory.

**Root cause:** The dashboard's packet capture feature referenced captures by a predictable, unauthenticated numeric ID with no access control, allowing any user to view historical captures not belonging to them (IDOR). Compounding this, FTP transmits credentials in plaintext, meaning any captured FTP session leaks reusable credentials.

## Privilege Escalation

Searched for binaries with Linux capabilities set:

```
getcap -r /usr/bin 2>/dev/null
```

Found `python3.8` had a capability set enabling UID manipulation.

Launched the Python interpreter and confirmed current privilege level:

```python
python3.8
>>> import os
>>> os.getuid()
1001
```

Used the capability to escalate:

```python
>>> os.setuid(0)
>>> os.system("/bin/sh")
```

Confirmed root:

```
whoami
root
```

Read the root flag:

```
ls /root
cat flag.txt
```

**Root cause:** The `python3.8` binary had `cap_setuid` set, granting it the ability to change a process's UID without requiring full root/SUID. Since Python can execute arbitrary code, this capability allowed direct escalation to UID 0 from within the interpreter.

## Flag

```
cat flag.txt
```

User and root flags captured (redacted for this writeup).

## Lessons Learned

- IDOR vulnerabilities are often as simple as incrementing/decrementing a number in a URL — always test adjacent IDs on any endpoint that references a resource by number, even ones that look like internal-only debug/admin features.
- Plaintext protocols (FTP, HTTP, Telnet) captured in a pcap will leak credentials in cleartext, worth checking any recovered capture for auth traffic on unencrypted services first.
- Password reuse across services (FTP credential reused for SSH) is a realistic and common finding, always try recovered credentials against every open service, not just the one they were captured from.
- Linux capabilities are a SUID-adjacent privesc vector that's easy to miss if you only check for SUID binaries (`find / -perm -4000`) — always run `getcap -r /` as a separate enumeration step.
- GTFOBins has a dedicated "Capabilities" section (separate from SUID/Sudo) for binaries like Python that documents this exact technique.

## Remediation (real-world)

- Implement proper access control on the packet capture endpoint, verify the requesting user owns/is authorized to view the referenced capture rather than trusting a raw numeric ID.
- Avoid transmitting credentials over plaintext protocols; use FTPS/SFTP instead of FTP.
- Enforce unique credentials per service rather than reusing passwords across FTP, SSH, and other services.
- Remove unnecessary Linux capabilities from binaries; `cap_setuid` should never be set on a general-purpose interpreter like Python unless absolutely required and tightly scoped.
