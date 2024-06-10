---
title: OSWA a different course on web attacks
author: eMVee
date: 2024-06-10 00:00:00 +0000
categories: [Certification, OSWA]
tags: [OSWA, Offsec, WEB200, SysReptor]
render_with_liquid: false
---

Last week I did pass the OffSec Web Assessor (OSWA) exam from Offsec. This is a hands-on exam and is 23 hours and 45 minutes long before the lab time ends. In the next 24 hours you have to deliver a report to OffSec with all the proof to pass. To pass the exam you will need 7 flags out of 10 flags.

## The WEB200 course
As expected, the WEB200 course is completely focused on hacking web applications. Various known vulnerabilities are discussed as can be read in the [syllabus](https://www.offsec.com/documentation/WEB-200-Syllabus.pdf) from OffSec. Then content in combination with the exercises are very good and they do explain how to attack those vulnerabilities. During the course they have updated the course with a new topic: `Web Application Enumeration Methodology`. I believe this is a good addition to the course indeed. In short these are the topics in WEB200.
- Web Application Enumeration Methodology 
- Tools for the Web Assessor
- Cross-Site Scripting (XSS) Introduction, Discovery, Exploitation and Case Study
- Cross-Site Request Forgery (CSRF)
- Exploiting CORS Misconfigurations
- Database Enumeration 
- SQL Injection (SQLi)
- Directory Traversal
- XML External Entity (XXE) Processing
- Server-Side Template Injection (SSTI)
- Server-Side Request Forgery (SSRF)
- Command Injection
- Insecure Direct Object Referencing
- Assembling the Pieces: Web Application Assessment Breakdown

### Create a learning plan
A lot of these topics can be learned in other courses like eWPT and Port Swigger Web Academy as well. Since I have some experience with eWPT and OSWA  I can tell they are not the same completely and they complement each other. Because I had already experience with the eWPT certification, I was able to easily go through a number of chapters in the WEB200 course. In total, it took me about 5 weeks to obtain my OSWA certification. This was because I could not schedule the exam at a time that I liked. Create a learning plan in which you master the material per chapter. Make sure that the learning plan is realistic and gives you the option to make adjustments if necessary. If you do not want to create a learning plan yourself, you could also choose one of OffSec's itself

- [12 weeks learning plan](https://help.offsec.com/hc/en-us/articles/15701019987732-OffSec-WEB-200-Learning-Plan-12-Week)
- [24 weeks learning plan](https://help.offsec.com/hc/en-us/articles/15712408711828-OffSec-WEB-200-Learning-Plan-24-Week)

### Study buddies
On Discord there are some channels for the WEB200 course. If you get stuck or having some issues ,you can ask here help. I've met wonderful people here they could help me and explain some techniques used as well. Yes I do have experience with eWPT, but still I encountered some issues and challenges in the WEB200 course. I am glad I had some help from other study buddies and I am gratefull for the help of Maggi, K3nn3y and the OffSec Student Mentors. They offered on some issues and challenges new insights so I could continue and learn about them. While learning here I've updated my cheat sheet with all kind of notes. I started even reordering some of my notes based on new insights. Thanks Maggi for showing how a good structure can be. 

### Takeaways
Some takeaways to get started on your OSWA journey.
1. Make some notes and create your own cheat sheet
2. Create a checklist for if you are stuck at that moment
3. Practice, practice and practice more
4. Create a template for your exam. You will need the notes and your cheat sheet
5. Create your own methodology for web pentesting based on the course and experience
6. Get used to write an OffSec pentest report 

## Creating a Comprehensive Report for OSWA: A Guide to Success
A penetration testing report is crucial in bridging the technical aspects of a pentest and its strategic implications. It is the primary way the assessment results are communicated to the client or relevant stakeholders. A pentest report highlights the vulnerabilities discovered and provides insights into the potential risks and recommended remediation actions. The templates from OffSec don't have all items I would like to address in a report. But for the exam this should be enough and that's why I did choose to use the kind of templates from OffSec. When creating a report for OSWA, it's essential to cover all the critical aspects of the course and exam. The [documentation requirements](https://help.offsec.com/hc/en-us/articles/4410105650964-WEB-200-Foundational-Web-Application-Assessments-with-Kali-Linux-OSWA-Exam-Guide#documentation-requirements) are published by OffSec and they do not tell you  you have to use their template. So you can feel free to use your own template as well. They do inform you what should be in the report. Make sure that you understand what they expect in the report before starting with your exam. Makes some notes on commands that you have used, make screenshots and capture the flags as described in the documentation requirements.

OffSec does provide a template for your documentation during your exam. Those can be downloaded in two formats andare available at the following URLs:
- [Microsoft Word](https://www.offensive-security.com/oswa-online/OSWA-Exam-Report.docx)
- [OpenOffice/LibreOffice](https://www.offsec.com/oswa-online/OSWA-Exam-Report.odt)

After my OSCP exam I tried SysReptor for reporting and I did like the ease of writing a report. This time I decided to use it during my exam. At the start of the course I've installed [SysReptor](https://emvee-nl.github.io/posts/SysReptor/) on my VM. Next I imported the OffSec templates created by SysReptor. Based on the template from Offsec I had to remove on paragraph in the template in SysReptor. To get familiar with SysReptor and writing a report in this format you can use the 8 challenge labs machines from Offsec. Make a report for these machines to get an idea how you should use their template and to fulfill the documentation requirements set by OffSec.

## Practice makes perfect
Practice, practice and practice again. It is so important to master the topics covered in the course and also to create your own methodology on how to attack a web application from A to B. You need to understand the vulnerabilities so you can successfully exploit them. There are 8 challenge machines in the course. Those will help you prepare for the exam. I recommend that you hack these machines in several ways. I'm almost certain that after hacking these 8 machines you are not completely sure whether you will pass the exam or not. In the table below I have written a number of vulnerable machines on different platforms in which you can practice the techniques learned.

| PG Play | PG Practice | HTB | HackMyVM | PortSwigger | Other |
|----------------|--------------------|-----|----------|----------|----------|
|FunboxEasyEnum  | megavolt           | Headless | Preload | XSS | OWASP Juice Shop |
| Inclusivenes | Muddy | Photobomb | Quick | CSRF | DVWA |
| Potato | Wheels | Red Panda | Quick3 | CORS | OWASP Mutillidae II| 
| Loly | Hawat | Vault | Quick4 | SQL injection | bWAPP |
| Sumo | Slort | Precious | System | SSRF | Gin & Juice Shop |
| Funbox | Snookums | PC | Luz | Path traversal |
| Shakabra | Dibble | Validation | Literal | OS command injection| 
| SoSimple | Rookie Mistake | Forge | wmessage | XXE Injection |
| Election1 | | Sau | Locker |SSTI | |
| Noname | | Topology | insomnia | |
| DC9 | | GoodGames | Medusa| |
| | | Numbchucks | Ephemeral| |
| | | SecNotes | Boxing | |
| | | TwoMillion | | |

## Exam time!
Book your exam in advance so that you can choose a time that suits you. I scheduled the exam at the start of the course and the exam was planned 5 weeks later. This was the best option for me so that I had a 'decent' time for the exam. 

Make use of your preparation for the exam, this may mean that you have already updated your notes or your report template so that you can start well. This may save some time during your exam and allow you to rest more quickly.

During the exam you should take care of yourself. Take plenty of breaks, drinks and eat healthy food. Be sure to take some time for yourself to relax. The exam can be stressful enough. Take a short walk to clear your mind and feel refreshed. Your sleep is also very important. While I was sleeping I had another inspiration about a vulnerability that woke me up. This was successful and led to my ninth flag.

Wishing you a stress-free exam experience... just kidding, that's not possible. But you'll do great anyway!