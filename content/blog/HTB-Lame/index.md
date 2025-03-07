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
