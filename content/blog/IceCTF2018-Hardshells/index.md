---
title:
  "IceCTF2018 - Hard shells"
date: 2018-08-26T01:00:27+02:00
cover:
  src: cover.png
draft: false
comments: true
socialShare: true
tags:
  - IceCTF2018
  - hardshells

aliases:
  - IceCTF2018-Hardshells

---

# IceCTF2018 - Hard shells

## Challenge Overview
* Name: `Hard shells`
* Type:`forensics`
* Difficulty: `easy`
* Points:`200`


### Description

After a recent hack, a laptop was seized and subsequently analyzed. The victim of the hack? An innocent mexican restaurant. During the investigation they found this suspicous file. Can you find any evidence that the owner of this laptop is the culprit?

## Solution Walkthrough


```
pwn@pwn:~/Labs/forensics/IceCTF2018/hardshells$ file hardshells
hardshells: Zip archive data, at least v1.0 to extract, compression method=store
```

changed the extension

```
mv hardshells hardshells.zip
```

We try to extract this Zip archive , but it ask us a password .

![Pasted image 20250412131501](https://github.com/user-attachments/assets/cf6509f9-97bb-427d-93e2-42b5df98ddaf)

let's use `fcrackzip` to bruteforce the password.

```
fcrackzip -u -D -p ~/Tools/rockyou.txt hardshells.zip

#-D use bruteforce dictionnary
#-p dictionnary file 
#-u try unzip
```

![Pasted image 20250412131541](https://github.com/user-attachments/assets/3783cec5-bea7-4da4-ad49-7315d55eaf68)

there's file called `d`, let's try to find the file type

```
pwn@pwn:~/Labs/forensics/IceCTF2018/hardshells/hardshells$ file d
d: Minix filesystem, V1, 30 char names, 20 zones
```

![Pasted image 20250412131641](https://github.com/user-attachments/assets/2b38aeee-f84c-4abf-a704-338d328f8f28)

We have to mount the minix file system .


```
pwn@pwn:~/Labs/forensics/IceCTF2018/hardsShells/hardshells$ mkdir mountpoint         
pwn@pwn:~/Labs/forensics/IceCTF2018/hardsShells/hardshells$ sudo mount d mountpoint/ 
```

We can see a file called `dat` inside the mount point . The `file` command on `dat` says us data.

![Pasted image 20250412132147](https://github.com/user-attachments/assets/8f95a9a1-d184-45be-b099-1b97542ddb0a)


```
pwn@pwn:~/Labs/forensics/IceCTF2018/hardsShells/hardshells/mountpoint$ file dat
dat: data 
```

Try to `hexedit` on this file in order to have more information

```
hexedit dat
```

```
00000000  89 50 55 47  0D 0A 1A 0A   00 00 00 0D  49 48 44 52   .PUG........IHDR
00000010  00 00 07 80  00 00 04 38   08 06 00 00  00 E8 D3 C1   .......8........
00000020  43 00 00 20  00 49 44 41   54 78 9C EC  DD 79 78 54   C.. .IDATx...yxT
...
...
0004D9C0  00 00 00 00  49 45 4E 44   AE 42 60 82  0A            ....IEND.B`..
```

It's possible that's dat is an PNG file. you can do a quick goole search the header values `IHDR` . 

![Pasted image 20250412123224](https://github.com/user-attachments/assets/91b4fbef-e8ea-4bc8-b776-4c1d40e6c1d3)

It wasn't detected by `file` because it doesn't have the proper magic byte. PNG files must begin with the bytes `89 50 4e 47` in order to be valid.

In other words, the "U" in "PUG" needs to be changed to an "N" to make "PNG." 

```
00000000  89 50 55 47  0D 0A 1A 0A   00 00 00 0D  49 48 44 52   .PNG........IHDR
00000010  00 00 07 80  00 00 04 38   08 06 00 00  00 E8 D3 C1   .......8........
00000020  43 00 00 20  00 49 44 41   54 78 9C EC  DD 79 78 54   C.. .IDATx...yxT
...
...
0004D9C0  00 00 00 00  49 45 4E 44   AE 42 60 82  0A            ....IEND.B`..
```

Let's try that let's make PUG to PNG.

We can use `Tab` command to move between hex and ASCII. `CTRL+o` to save `CTRL+x` to exit 

now let's use file command again and see:


```
pwn@pwn:~/Labs/forensics/IceCTF2018/hardsShells/hardshells/mountpoint$ file dat         │
dat: PNG image data, 1920 x 1080, 8-bit/color RGBA, non-interlaced 
```

let's open the image and see.

```
eog dat
```

![Pasted image 20250412121257](https://github.com/user-attachments/assets/b42d7866-c21f-4bad-91e2-811bc41d551d)


## Flag
`IceCTF{look_away_i_am_hacking}`
