# Fawn — HTB Starting Point

**Difficulty:** Very Easy
**OS:** Unix
**Path:** Starting Point — Tier 0

## Summary

Fawn demonstrates the risk of leaving anonymous FTP access enabled. The target's only open port was FTP (vsftpd 3.0.3), which allowed anonymous login and directly exposed a readable flag file — no exploit or credential guessing required.

## Recon

```
nmap -sC -sV -T4 <TARGET_IP>
```

**Results:**

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:10.10.14.95
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 4
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0              32 Jun 04  2021 flag.txt
Service Info: OS: Unix
```

Only one port open: FTP (21). Nmap's `ftp-anon` NSE script already confirmed anonymous login was allowed *and* listed `flag.txt` in the directory listing, before any manual connection was made — a good reminder that script output deserves as much attention as the open-port table itself.

## Enumeration

Given the nmap script output directly confirmed anonymous access and revealed a file of interest, no further enumeration was needed before moving to exploitation. This was also a clear enough signal that researching a vsftpd 3.0.3 CVE wasn't the intended path for this box.

## Exploitation

Connected via FTP with anonymous login:

```
ftp -a <TARGET_IP>
```

Listed directory contents:

```
ls
```

Confirmed `flag.txt` present, then downloaded it:

```
get flag.txt
```

Exited FTP and read the file locally:

```
quit
cat flag.txt
```

**Root cause:** FTP server configured to allow anonymous read access to a directory containing sensitive data — a misconfiguration, not a software vulnerability in vsftpd itself.

## Privilege Escalation

Not required — flag was directly accessible via anonymous FTP.

## Flag

```
cat flag.txt
```

Flag captured (redacted for this writeup).

## Lessons Learned

- Read nmap's NSE script output in full, not just the port/service table — `ftp-anon` here answered the entire box before a manual connection was even made.
- Anonymous FTP access is a common and high-impact misconfiguration; always test it first when FTP is present, before jumping to version-specific exploit research.
- Not every box requires a CVE — many rely purely on misconfigurations, which is often the more realistic finding in real-world assessments.

## Remediation (real-world)

- Disable anonymous FTP login unless explicitly required for public file distribution.
- If anonymous access is necessary, restrict it to a dedicated directory with no sensitive files and read-only, tightly scoped permissions.
- Prefer SFTP/FTPS over plaintext FTP to protect credentials and data in transit.
