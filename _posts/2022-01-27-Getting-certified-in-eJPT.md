---
title: Getting certified in eJPT
author: eMVee
date: 2022-01-27 20:00:00 +0800
categories: [Certification, eJPT]
tags: [eJPT, eLearnSecurity, INE]
render_with_liquid: false
---



## Getting certified as eJPT
![Image](/assets/img/Education/eJPT.png){: .right : w="200"} Since the end of 2019 I'm working as a security professional and before I started in security I did not get any security certificate. In 2020 I started to following different security trainings but none of them were really with a practical exam and the ultimate goal was to follow a live class for OSCP in a classroom. Due to COVID-19, the OSCP training has already been postponed a number of times and therefore I decided to look around and get some experience in practical exams. In 2021 I saw the eLearnSecurity Junior Penetration Tester (eJPT) certificate whic can be completed by passing a 100% practical exam. This could be accomplished by following the [Penetration Testing Student](https://my.ine.com/CyberSecurity/learning-paths/a223968e-3a74-45ed-884d-2d16760b8bbd/penetration-testing-student) course from INE. According ine the duration of the penetration testing student training is lead by Lukasz Mikula and it will take arount 77 hours and 41 minutes. The training is divided in some sections with different modules.
1. Section 1: Penetration Testing Prerequisites
    * Module 1: Introduction
    * Module 2: Networking
    * Module 3: Web Applications
    * Module 4: Penetration Testing
2. Section 2: Penetration Testing: Preliminary Skills & Programming 
    * Module 1: Introduction to Programming
    * Module 2: C++
    * Module 3: Python
    * Module 4: Command Line Scripting
3. Section 3: Penetration Testing Basics 
    * Module 1: Information Gathering
    * Module 2: Footprinting and Scanning
    * Module 3: Vulnerability Assessment
    * Module 4: Web Attacks
    * Module 5: System Attacks
    * Module 6: Network Attacks
    * Module 7: Next Steps
4. Section 4: eJPT Exam Preparation
    * Module 1: Prerequisites
    * Module 2: Penetration Testing

### Slides, videos and labs
The training material consists of slides, videos and a lab environment. The slides can be overwhelming in the number of pages. Sometimes it felt like the slides never ended until we got to a video or lab environment. During the slides and the video I made notes for myself that I would eventually put in some kind of cheat sheet for the exam. Fortunately, lab environments were also offered. In the lab environment you can practice a specific subject or technique. You can think of scanning your network, specific attacks on websites such as SQL injection, XSS attacks, but also system attacks where you work with Metasploit. In each module, sufficient space is given to practice and to prepare for the exam. The labs have their solutions listed there, but you could also write and use your own tools and/or scripts to achieve the goal. When all modules are completed, black boxes are available to practice your new knowledge in a lab environment.

### Black boxes
There are several black boxes (lab environments) in which you have to hack multiple machines. These black boxes are challenging enough to prepare you for the exam. The machines are all set up in such a way that they are vulnerable and can be taken over by you as an attacker. Again, there are several ways to Rome to achieve a goal. To be successful, you need to master the process and know which tools/commands you need for which. I don't want to give away a lot of details here, but I can say that the new (beta) lab environments are challenging. These were added during my preparation, which gives me the suspicion that the old black boxes will probably disappear at some point.

### Cheatsheet 
During the training I wrote down useful commands and tools in my cheat sheet for the exam. I was going to use the cheat sheet during my exam, so I posted it on my [Github](https://github.com/mvdvaart/eJPT) so that I can easily retrieve it on any machine. The advantage of a cheat sheet is that during the exam I could quickly search for certain topics and for each topic I have worked out the commands with a small piece of explanation. Since my cheat sheet was on Github, I could occasionally read my cheat sheet on my way to the office by public transport. In retrospect this turned out to be an advantage because I could still remember many things during the exam and otherwise I could scroll to them very quickly. 

## The exam
Monday morning at 08:15 I started with my exam. Although this exam has multiple choice questions, you need to hack into the environment to answer them. That's why I first started by going through what was being asked, so that I knew what to look for during the exam, among other things such as vulnerabilities, configurations items and other useful information.

After this I downloaded the data that includes the letter of engagement. The data I have been given I study carefully, when I have gone through it I see that I spent about half an hour going through the information I was given at the beginning of the exam.While scanning I was already able to answer a number of questions. Of course not yet with complete certainty, I noted this in my notes so that I could come back to this later. While running an exploit, I noticed that the machine I was attacking was no longer functioning properly. At the time I had no idea if I could reset the machine to a previous state or if the rest of the machines would be reset and even worse what impact this would have on my exam questions. Would my questions change and would I have to work on new questions? I decide to ask an INE if they can tell me what happens if I want to reset the machine, so you actually reset the entire lab environment. When I called in the evening they couldn't tell me if my questions would change and so I would have to make different questions. I was going to get an answer to this later, so I decided to go to bed. Tomorrow is a new day and with 2 days left, this is a good time to rest.

Day 2

In the morning I still had no answer from INE unfortunately. I decided to continue with inspecting the machines I had found. Although I could already answer most questions, I had to wait until the afternoon for the redeeming answer from INE. When the lab environment is reset, the questions would not change. So after this message I decided to reset the lab environment and run another scan. Indeed, the machine that previously failed was now back in the condition I started the exam. Because I had made a mistake with the exploit, I now double-checked the exploit's settings before running it. Now that the exploit could be successfully executed, I was able to continue collecting information and answering questions that I couldn't answer before. At this point I could already see that I would have answered enough questions correctly to pass. I decided it was time to rest and try a few more things on day 3. I made an agreement with myself that I would only take a few hours on day 3 because I had passed the exam and I had other private appointments that would need my attention at that time.

Day 3

The day started pretty relaxed together with the family, and after breakfast, it was time to turn on the computer and connect to the lab environment. I noticed that the machines responded to my actions faster than before. After 2 hours of working on the exam, I decided that enough was enough, because I could only answer a few questions not fully convinced. The most exciting part of an exam will always remain the moment of submission, even though you know that you will pass the exam because of the correct answers on the questions. When I submitted the exam, a message was shown to me that I had passed and based on the score I was convinced that the questions about which I had doubts were indeed wrong. When I look back on the exam, I can say it was a fun experience, where you can't just guessing the correct answer on multiple choice questions. Other people may finish the exam faster, just remember while taking this exam, it should be fun and even during the exam you could learn new things. That's why I did not want to rush my exam.