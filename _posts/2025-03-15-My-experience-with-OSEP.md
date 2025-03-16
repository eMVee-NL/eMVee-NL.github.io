---
title: My experience with OSEP, insights, challenges, and lessons learned
author: eMVee
date: 2025-03-15 00:00:00 +0000
categories: [Certification, OSEP]
tags: [OSEP, Offsec, PEN300, SysReptor]
render_with_liquid: false
---

PEN300 also known as OSEP is a course not just about learning the technical skills of penetration testing; it’s a deep dive into the complexities of cybersecurity that pushes you to think critically and adapt quickly. In this blog post, I’ll share my personal experiences what worked, what didn’t, and the lessons I took away from this intense program. From the hands-on labs that tested my skills to the moments of frustration that taught me resilience, I hope to provide a candid look at what it’s really like to engage with OSEP. Whether you’re considering the course or simply curious about the world of penetration testing, I invite you to read and get excited about OSEP as well.

## The course
The OSEP course focuses on sophisticated methods for breaching defenses, making it essential for those looking to tackle modern cybersecurity challenges. Yes the course is several years old and some techniques are dated, but still relevant. During my training I noticed that subjects where updated and new ones added to the course. In this course they will learn you about process hollowing and process injection, which allow attackers to manipulate legitimate processes for malicious purposes. The curriculum includes exploiting malicious files in Word documents and MS Teams calendar invites, as well as using HTA and JavaScript files as droppers to deliver payloads effectively.

Additionally, the course covers critical topics such as Active Directory attacks, emphasizing trust relationships and lateral movement within networks even via linked MSSQL servers. It's not only Windows and Active Directory, there are still parts in the course that will teach how to use Linux systems and tools for DevOps in an enterprise environment.

## Keep practicing while you can
Each chapter has different assignments and extra mile assignments. It is wise to do both. The extra mile assignments are assignments that are a bit more challenging and go deeper into the material. There is also no help offered by the student mentors with these assignments. If you need hylp, you are on your own or on the help of your fellow students. During the course, content was updated and assignments were slightly adjusted so that you have to get a flag to prove that you have successfully completed it.
There are multiple topics that are provided to learn. In addition to breaking through security measures, Active Directory is an important topic and many attacks and assignments teach you the most important points for attacking Active Directory environments in the OSEP exam.

### So you think you know Active Directory?
Active Directory is huge and can become complex in large enterprise environments due to the setup. This complexity can also lead to misconfigurations that hackers can exploit. The OSEP course teaches you a small part of the possible AD vulnerabilities, such as kerberos delegation, trusts and linked MSSQL servers. Unfortunately, a number of vulnerabilities are not covered, such as how you can abuse Active Directory Certificate Service (ADCS). 

During practicing I sometimes was looking with the loot I retrieved and what I could do next. In my cheatsheet in Obsidian I got this [mindmap for Active Direcory](https://github.com/eMVee-NL/MindMap?tab=readme-ov-file#ad-penetration-testing-mindmap) based on the mindmap or [mayfly](https://mayfly277.github.io/posts/Upgrade-Active-Directory-mindmap-v2022_11/). It helped me during the course to see what the next step might be to move lateral in the network.

### Useful resources
One of the most exciting aspects of the OSEP journey is the emphasis on crafting your own scripts, payloads, and programs designed to outsmart security measures. This hands-on experience is not just crucial for tackling the challenges presented in the OSEP exam; it’s a vital skill set that will empower you throughout several pentesting jobs.

While there are countless repositories brimming with code available online, the real magic happens when you start creating your own unique solutions. Throughout my OSEP experience, I’ve developed a variety of scripts, some of which I’m keeping under wraps for now, while others have been shared and developed together with fellow students or made available online, like the UpdateHostsFile script.

I’ve borrowed, adapted, and enhanced code from others, turning it into someting that was working during the course. Below, I’ll share some of the invaluable resources and snippets that have helped me along the way.
- [https://github.com/chvancooten/OSEP-Code-Snippets](https://github.com/chvancooten/OSEP-Code-Snippets)
- [https://github.com/beauknowstech/OSEP-Everything](https://github.com/beauknowstech/OSEP-Everything)
- [https://github.com/Extravenger/OSEPlayground](https://github.com/Extravenger/OSEPlayground)
- [https://github.com/eMVee-NL/UpdateHostsFile](https://github.com/eMVee-NL/UpdateHostsFile)
- [https://github.com/ubghacking/pen-300-scripts](https://github.com/ubghacking/pen-300-scripts)

Make smart use of those and if you have awesome scripts share them with the community, like I did for example with the update hosts file script.

### Train as You Fight
"Train as you fight" is a mantra that emphasizes the importance of practicing under conditions that closely resemble the real environment and thus your exam. When I first began the OSEP course, there were six challenges available. However, as the course content is regularly updated last months, a seventh challenge, "Cowmotors," was added. This particular challenge is a retired exam, making it an intriguing opportunity to test your skills against real-world scenarios.

Before my initial attempt, I managed to complete each challenge at least once. However, I encountered numerous stability issues during my practice, sessions and connections dropped, web servers became unreachable, and these frustrations were echoed by many of my fellow students. Despite these challenges, the exercises were engaging, and each presented various vulnerabilities.

After my first attempt did not go as planned, I decided to revisit the challenges, experimenting with different techniques and approaches across both Windows and Linux platforms to effectively exploit the vulnerabilities. I strongly encourage all students to tackle the challenges multiple times and to explore various methods of exploitation. This not only enhances your skills but also builds the confidence needed to succeed in the exam. Embrace the learning process, and remember that each attempt brings you one step closer to mastery! During practicing I created a checklist and a kind of mindmap to use if I would get stuck during the second attempt.

## Exam
The OSEP exam is a hands-on assessment that demands not only your technical skills but also your well-being. With a total of 47 hours and 45 minutes to navigate and compromise a simulated enterprise environment, it's crucial to prioritize self-care throughout the process. After completing the practical portion, you'll have 24 hours to write and submit a comprehensive penetration testing report.

During the exam, you'll be monitored via webcam by a proctor, but you have the flexibility to schedule your own breaks. Whether you need to grab a bite to eat, take your dog for a walk, or catch a quick nap, make sure to listen to your body and recharge as needed. While I can't divulge specific details about the exam, I can  say that I found the experience to be both enjoyable and rewarding. Remember, taking care of yourself is key to performing at your best!

### Getting ready for the exam
During the OSWA exam I used Sysreptor to write the report, this time I decided to use Sysreptor again for OSEP. Since I was using a new Kali version Sysreptor was not installed yet and I had some issues by the previouws installation manual for [SysReptor](https://emvee-nl.github.io/posts/SysReptor/) that I shared online. Due to this I had to figure out how I had to install it on the new Kali version. This did consider me to update that post as well after the exam. Next I imported the OffSec templates created by SysReptor for Offsec.

Of course you don't have to use Sysreptor to write a report for OSEP. You can use the template from OffSec as well
- [Microsoft Word](https://www.offensive-security.com/osep-online/OSEP-Exam-Report.docx)
- [OpenOffice/LibreOffice](https://www.offensive-security.com/osep-online/OSEP-Exam-Report.odt)

Read the [exam guide](https://help.offsec.com/hc/en-us/articles/360050293792-OSEP-Exam-Guide) and be familiar with it. For example they tell you that a screenshot via RDP is not allowed and it will not count. But there are some other useful information as well such as how you should capture the flags with what command. Remember take screenshots of the flags and add it to your notes or even immediatly into the report.


### First attempt
My first attempt at the OSEP exam took place on Monday, February 17th, at 11 AM. The week prior, I had successfully tackled the final challenge and updated my notes, which gave me a boost of confidence as I began the exam. I quickly achieved a few small victories, reinforcing my belief that I was on the right track. However, after seven hours, I had only managed to capture two flags.

During this time, I attempted to install a tool on my virtual machine, convinced it was essential for exploiting a particular vulnerability. Unfortunately, this decision backfired, causing my VM to freeze whenever I selected text in the terminal, forcing me to restart it multiple times. After 13 hours of intense focus, I decided to call it a night, but sleep eluded me. Just three hours later, I was back at it, and by morning, I had successfully penetrated another machine and captured a third flag.

By 8:30 AM, fatigue set in, signaling it was time for a quick nap. An hour later, I returned to my computer, energized and ready to tackle the next challenge. I hacked into another machine and discovered promising hints for my next attack. However, I soon found myself stuck, spending 12 hours crafting a solid payload but still only managing to secure three flags. By 10:30 PM, after nearly 36 hours in the exam, I resigned myself to the fact that passing on my first attempt was unlikely. I decided to rest and absorb as much knowledge as I could in the remaining hours.

The next morning, I was back at my desk by 6:30 AM, and to my surprise, the payload I had struggled with for over 12 hours the previous day worked flawlessly after just an hour of testing. Suddenly, everything clicked, and I found myself enjoying the exam once more. However, time flew by, and after 47 hours and 45 minutes, I finished with six flags and a wealth of experience. While I was disappointed that I didn’t achieve the results I had hoped for, I embraced the opportunity for a second attempt, determined to approach it with a fresh perspective next time.

### Second attempt
On Tuesday, March 4th, at noon, I embarked on my second attempt at the OSEP exam. This time, I had taken the precaution of creating two clones of my virtual machine, ensuring I could switch to another VM if I encountered the same issues as during my first attempt. Right from the start, the exam felt more manageable, likely due to my increased familiarity with the challenges and my diverse hacking strategies.

After two and a half hours, I took my first break, having already secured four flags. Following a refreshing smallbreak with a cookie and a soda, I continued my progress and, by 5:30 PM, I had collected six flags. I decided to take a dinner break with my family, and after that, I returned to my exam. By 8 PM, I had gathered enough flags to pass! I quickly informed my family and then settled back in with some snacks and drinks to continue.

Two hours later, I had gathered more than enough flags, but I was enjoying the experience so much that I kept going until midnight. Eventually, I decided it was time to get some rest. The next morning, I woke up at 5:30 AM and took my dog for a refreshing walk before returning to my computer by 6:30 AM. With less than 24 hours left in the exam and plenty of flags secured, I focused on writing my report, meticulously double-checking every step I had taken.

By 10 PM that evening, I was ready to call it a night, planning to finalize the report the next day. On Thursday morning, I didn’t sit down at my computer until around 9 AM, where I added the finishing touches to my report. With some time left before the exam environment would close, I took the opportunity to hack into even more machines within the fictional company network, further expanding my knowledge and skills. It was a rewarding conclusion to an intense and fulfilling exam experience!


## Lessons learned
During my first exam, I encountered a series of challenges that made the experience quite difficult. I spent hours meticulously trying all kind of payloads for a specific vulnerability, convinced it was the key to open the door, only to realize later that I had overlooked other possibilities. To complicate matters further, I had installed and updated some software during the exam, which caused certain applications to malfunction. On top of that, my virtual machine became unstable and frequently crashed whenever I tried to select text in the terminal for my notes. It was certainly a learning experience by failing the exam for the first time!

Here are some lessons learned
- Have a backup plan
    - Backup VM
    - Location to store your data
- Make a snapshot of the VM before installing new software or libraries
- Don't get a tunnel vision if some vulnerability is not working as expected even if you are convinced it should work
- Try several methods to exploit vulnerabilities or misconfigurations

Wishing you a stress-free exam experience… just kidding, that’s as likely as finding a unicorn in the library! But remember, all those late-night study sessions will pay off. You've got this!