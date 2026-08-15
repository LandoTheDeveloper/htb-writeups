# Meow — HTB Starting Point

**Difficulty:** Very Easy
**OS:** Linux
**Path:** Starting Point — Tier 0

## Summary

Meow is an introductory box demonstrating the risk of exposing Telnet with default/no authentication configured. Access was gained by connecting via Telnet and logging in as `root` with no password required, landing directly in a root shell.

## Recon

Full TCP port scan:

```
nmap -sC -sV -p- -T4 <TARGET_IP>
```

**Results:**

```
PORT   STATE SERVICE VERSION
23/tcp open  telnet  Linux telnetd
```

Only one port open: Telnet (23). No web, no SSH, no SMB — a strong signal the intended path runs entirely through Telnet, likely via a misconfiguration rather than a complex exploit chain.

## Enumeration

An open Telnet port with no other services present is a classic sign of weak or default authentication on beginner-tier boxes. Before researching any CVEs, the logical first move is to test for blank, default, or commonly-used credentials.

## Exploitation

Connected via Telnet:

```
telnet <TARGET_IP>
```

At the login prompt, entered:

```
Username: root
```

No password was requested/required — this dropped directly into a root shell.

**Root cause:** Telnet service configured to allow root login with no authentication — a default/misconfiguration issue rather than a specific software vulnerability. (Initially considered a telnetd CVE involving a `USER='-f root'` bypass, but this box's behavior matched a simpler no-auth misconfiguration rather than that specific bug — worth confirming which is actually in play before citing a CVE.)

## Privilege Escalation

Not required — initial access landed directly as root.

## Flag

```
cat flag.txt
```

Flag captured (redacted for this writeup).

## Lessons Learned

- Always check for anonymous/blank/default credentials before jumping to exploit research, especially when a box exposes an unusual number of legacy/plaintext services (Telnet, FTP, etc.).
- A single open port is itself a clue — it narrows the intended attack path rather than being something to "scan past."
- Don't attach a specific CVE to a finding without confirming the mechanism actually matches — misconfigurations and CVEs can produce similar symptoms but have different root causes worth distinguishing in a professional writeup.

## Remediation (real-world)

- Disable Telnet entirely; use SSH with key-based authentication instead.
- If Telnet must be used (e.g., legacy hardware), restrict access via firewall/VPN and never allow root login without authentication.
