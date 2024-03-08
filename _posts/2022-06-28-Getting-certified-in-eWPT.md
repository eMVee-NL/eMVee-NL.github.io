---
title: Getting certified in eWPT
author: eMVee
date: 2022-06-28 20:00:00 +0800
categories: [Certification, eWPT]
tags: [eWPT, eLearnSecurity, INE]
render_with_liquid: false
---

 After completing the eJPT course, it was time to follow the next course. I was not sure which course I would like to start with. I had to courses in mind, the eCPPT and the eWPT course.

## Getting certified as eLearnSecurity Web Penetration Tester (eWPT)
![Image](/assets/img/Education/eWPT.png){: .right : w="200"} When I looked at the duration of the course I decided to follow the shortes course in duration, which is the [Web Application Penetration Testing ](https://my.ine.com/CyberSecurity/courses/38316560/web-application-penetration-testing) course from INE. According INE it will take around 20 hours to finish this course. The training is divided in some sections with different modules and is explained by Dimitrios Bougioukas. The Web Application Penetration Tester (WAPT) training prepares you for the exam to get the eWPT certificate. The course consists of 15 modules and a separate module in preparation for the eWPT exam.
The following modules are covered during the training:

eWPT training exist out of 15 modules:
* Module 1: Penetration Testing Process
* Module 2: Introduction to Web Applications
* Module 3: Information Gathering
* Module 4: Cross-Site Scripting
* Module 5: SQL Injection
* Module 6: Authentication and Authorization
* Module 7: Session Security
* Module 8: Flash Security
* Module 9: HTML5
* Module 10: File and Resource Attacks
* Module 11: Other Attacks
* Module 12: Web Services
* Module 13: XPath
* Module 14: Penetration Testing Content Management Systems
* Module 15: Penetration Testing NoSQL Databases

eWPT Exam Preparation exist out of the following subjects:
* How to connect to the lab environment
* LAB - Introduction labs
* LAB - Information Gathering
* LAB - Cross Site Scripting
* LAB - SQL Injection
* LAB - Authentication and Authorization

What immediately struck me is that a Flash module is offered. A technology that is outdated and no longer used. And I hope that this technique is **not** used in any part of an organization or somewhere deeply hidden in a dark corner.

## Why I want the eWPT certification
You will receive the eWPT certification after successfully passing the exam. Roughly speaking, the exam consists of two parts. The first part consists of 7 days of pen testing a web environment, in which multiple web applications must be tested via subdomains.
You will be instructed to perform a real pen test and to find and report all vulnerabilities. Within this environment what I understood is an admin panel that you need to access.
The second part consists of writing a pen-test report in which a management summary, vulnerabilities are mentioned and the measures that can be taken. You will have an additional 7 days to write the pentest report. So you have a total of 14 days to complete the assignment (exam). The exam is therefore not like a capture the flag (CTF), but a simulated pen test for an organization where a solid pen test report is therefore expected. Within the exam you may use any application or script you wish to use. These two points made me excited to take the exam.

## Study Material
Different study materials are offered for each module. There are slides, videos and labs that explain each module well. Personally, I found some topics to be very long-winded and then I could hardly move through the slides. I must admit that the topics are well explained. Sometimes they could explain the subject a little further so that the context of a vulnerability as a whole becomes clearer or when you can best apply it. I felt a bit ripped off by the labs, because they are mainly from the well-known open source practice environments. In these environments you can attack many vulnerabilities and practice the associated techniques. Unfortunately, it was not indicated for each module which assignment you have to do in the lab in question and when you passed the lab. By this I mean, if you have found user Bob's password or if you have found a credit card number.
In order to make the right labs, you have to quickly peek into the solutions of the labs. From old forum posts I understand that these labs were different before. The videos supported these labs that would also contain challenges. Unfortunately, you will no longer see this in the course.

### Slides, videos and labs
The training material consists of slides, videos and a lab environment. The slides can be overwhelming in the number of pages. Sometimes it felt like the slides never ended until we got to a video or lab environment. During the slides and the video I made notes for myself that I would eventually put in some kind of cheat sheet for the exam. Fortunately, lab environments were also offered. In the lab environment you can practice a specific subject or technique. You can think of scanning your network, specific attacks on websites such as SQL injection, XSS attacks, but also system attacks where you work with Metasploit. In each module, sufficient space is given to practice and to prepare for the exam. The labs have their solutions listed there, but you could also write and use your own tools and/or scripts to achieve the goal. When all modules are completed, black boxes are available to practice your new knowledge in a lab environment.

### Cheatsheet and mindmap
During the training I wrote down useful techniques, commands and tools in my cheat sheet for the exam. I was going to use the cheat sheet during my exam, so I posted it on my [Github](https://github.com/mvdvaart/eWPT) so that I can easily retrieve it on any machine. The advantage of a cheat sheet is that during the exam I could quickly search for certain topics and for each topic I have worked out the commands with a small piece of explanation. Since my cheat sheet was on Github, I could occasionally read my cheat sheet on my way to the office by public transport. In retrospect this turned out to be an advantage because I could still remember many things during the exam and otherwise I could scroll to them very quickly. I've created a mindmap for web penetration testing which can be found [here](https://github.com/eMVee-NL/MindMap#web-penetration-testing-mindmap).

### eWPT Exam Preparation
After I had gone through all the modules within the Web Application Penetration Tester course, I saw a module "eWPT exam preparation". A module in which the old lab environment of the course is still offered in preparation for the exam. This made up for part of my dissatisfaction that I could still practice the old labs as they are often called in the forum. Unfortunately, after practicing a few hours in a lab, I noticed that INE (or eLearnSecurity) had taken these labs offline for maintenance was the message. This was not announced or communicated at all. The message you see on screen gives you the impression that it would be solved in 5 minutes and the lab will be available again.

On the second day it was still not available, so I called INE and asked if they could tell me when it would be available again. Unfortunately they could not indicate this.
As an alternative, I turned to PortSwigger Academy. They offer free practice materials to practice all kinds of web vulnerabilities. I must confess that I consider the quality of these free labs very high and can recommend them to anyone. Besides the PortSwigger Academy I started to practice on the JuiceShop from OWASP as well. In the last week of June I scheduled my eWPT exam and even the day before the exam the "old" VPN lab environments are still not available. So I'm glad that I could practice some techniques at other resources the last 3 weeks before the exam. 


## The exam
*Day 1*

Monday morning at 08:15 I started with my exam. Although this exam has multiple choice questions, you need to hack into the environment to answer them. That's why I first started by going through what was being asked, so that I knew what to look for during the exam, among other things such as vulnerabilities, configurations items and other useful information.

After this I downloaded the data that includes the letter of engagement. The data I have been given I study carefully, when I have gone through it I see that I spent about half an hour going through the information I was given at the beginning of the exam.While scanning I was already able to answer a number of questions. Of course not yet with complete certainty, I noted this in my notes so that I could come back to this later. While running an exploit, I noticed that the machine I was attacking was no longer functioning properly. At the time I had no idea if I could reset the machine to a previous state or if the rest of the machines would be reset and even worse what impact this would have on my exam questions. Would my questions change and would I have to work on new questions? I decide to ask an INE if they can tell me what happens if I want to reset the machine, so you actually reset the entire lab environment. When I called in the evening they couldn't tell me if my questions would change and so I would have to make different questions. I was going to get an answer to this later, so I decided to go to bed. Tomorrow is a new day and with 2 days left, this is a good time to rest.

*Day 2*

In the morning I still had no answer from INE unfortunately. I decided to continue with inspecting the machines I had found. Although I could already answer most questions, I had to wait until the afternoon for the redeeming answer from INE. When the lab environment is reset, the questions would not change. So after this message I decided to reset the lab environment and run another scan. Indeed, the machine that previously failed was now back in the condition I started the exam. Because I had made a mistake with the exploit, I now double-checked the exploit's settings before running it. Now that the exploit could be successfully executed, I was able to continue collecting information and answering questions that I couldn't answer before. At this point I could already see that I would have answered enough questions correctly to pass. I decided it was time to rest and try a few more things on day 3. I made an agreement with myself that I would only take a few hours on day 3 because I had passed the exam and I had other private appointments that would need my attention at that time.

*Day 3*

The day started pretty relaxed together with the family, and after breakfast, it was time to turn on the computer and connect to the lab environment. I noticed that the machines responded to my actions faster than before. After 2 hours of working on the exam, I decided that enough was enough, because I could only answer a few questions not fully convinced. The most exciting part of an exam will always remain the moment of submission, even though you know that you will pass the exam because of the correct answers on the questions. When I submitted the exam, a message was shown to me that I had passed and based on the score I was convinced that the questions about which I had doubts were indeed wrong. When I look back on the exam, I can say it was a fun experience, where you can't just guessing the correct answer on multiple choice questions. Other people may finish the exam faster, just remember while taking this exam, it should be fun and even during the exam you could learn new things. That's why I did not want to rush my exam.

*Day 4 - Day 7*

On day 4 I started very relaxed with  some breakfast and a coffee before starting the pentest report. Writing the report is an extremely important part of a penetration test. And it is as well very important to pass for this exam. Writing a good report takes time and I have enough time at the moment. If needed I can make screenshots if any are missing in the lab environment or double-check other things before I include it in my pentest report. Fortunately, I already documented many findings during the penetration test to my notes. You can think of screenshots for proof, commands that I have executed but also Proof of Concept (PoC) codes that I have made to demonstrate certain vulnerabilities. A few days before the exam, I decided to use [the Mayor pentest report template](https://www.notion.so/signed/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fa2a79f69-b06c-4e41-a3a2-2182d5c86f11%2FTheMayor_-_Sample_Pentest_Report.docx?table=block&id=ae8313ed-211b-42bc-b960-a56c50ead72e&spaceId=536218c5-c19e-4418-bf8e-94b8d09cf776&name=TheMayor%20-%20Sample%20Pentest%20Report.docx&cache=v2) in combination with the[pentest report sample of Heath Adams (TCM)](https://github.com/hmaverickadams/TCM-Security-Sample-Pentest-Report/raw/master/TCMS%20-%20Demo%20Corp%20-%20Findings%20Report%20-%20Example%202.docx). I've combined them so I could use the template for my own usage.

During writing the report I noticed that I could use some better screenshots and I would like to double check some vulnerabilties before completing my pentest report and uploading it to eLearnSecurity. On day 7, I was completely done. My report looked great, that's what I thought of it. But would eLearnSecurity think the same? I hoped so on that moment. It was 08:00 o'clock in the morning and time to upload the report. The file may not exceed 10MB and should be a ZIP file or a PDF file. My PDF file was 4 MB so that would be small enough to upload to eLearnsecurity. Guess what, I could not upload the report, even the file was small enough. My next step was to check I i could upload the pentestreport within a ZIP archive, but this didn't work either. I made a screenshot of the error and sent an email to support. In the afternoon I did not received any feedback yet about this issue and I really want to finish the exam and upload the report to eLearnsecurity. So I called them in the afternoon, 9 hours later then I tried to upload my pentest report. I left a message in the voicemail and they called me back when I was driving to the swimming pool. I had spoken to a helpful lady on the phone who told me I could sent the pentest reprt as PDF to the support desk and that they would sent it to the examiners. 

When I came home again I sent the pentest report as attachement to the support desk of eLearnSecurity. I decided to call the support team directly to check if they received it correctly.
Coincidentally, I spoke to the same helpful lady again. She confirmed that the email with attachment was received and that it was OK for now. So it was 8:00 PM in the evening and I felt like my exam was finished and I could take a break. Just before I wanted to go to bed at 10:30 PM I checked my email and to my surprise I saw that I had to upload the pentest report to eLearnSecuerity. Fortunately, this doesn't take a lot of time and I logged into eLearnSecurity right away and uploaded my pentest report to eLearnSecurity. It worked, eLearnSecurity confirmed that the file had been received. Time to close and go to bed.

*Day 8*

It was 5:30am when I got up again and got ready to go to work. I thought to check my email to see if I had heard from eLearnSecurity. There was a good chance I wouldn't have received anything yet. But at 12:01 am I got an email saying I passed the exam. Apparently I was wrong and I find it impressive that they reviewed the report so quickly. After all, I found the exam very fun and instructive and can certainly recommend it.

### Conclusion
The exam is designed to feel like a real penetration test and not a Capture The Flag (CTF). There are no flags to be found during the exam, just vulnerabilities that you need to document for the pentest report. There were moments that I even learned during the exam. After completing the exam I had a good feeling about the whole exam experience, except the part where I experencied some connection issues via VPN. I do believe that the course material is on some parts outdated, but for the most of it, it is still very good. The outdated parts are for example Flash and web services that are set up on SOAP and not on REST web services. A number of techniques were still missing during the training, such as: Command Injection, REST web services and Server Site Template Injection (SSTI). Hopefully these topics will be covered in a future release.