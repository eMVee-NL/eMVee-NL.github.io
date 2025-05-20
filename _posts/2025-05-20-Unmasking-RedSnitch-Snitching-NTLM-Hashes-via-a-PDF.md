---
title: Unmasking RedSnitch || Snitching NTLM Hashes via a PDF
author: eMVee
date: 2025-05-20 00:00:00 +0000
categories: [RedTeam, Tutorial ]
tags: [RedTeam, NTLM, PDF, T1550]
render_with_liquid: false
---

![RedSnitch-PDF](/assets/img/Tutorial/RedSnitch/redteamPDF.jpg){: .right }{: w="500"}
In the world of cybersecurity, the red team plays a crucial role in simulating attacks to identify vulnerabilities within an organization. One of the most critical tasks for these ethical hackers is to gather credentials for lateral movement within a network. Imagine a scenario where a red team is on a mission to infiltrate a corporate network, and they need to obtain NTLM hashes to move undetected. This is where the tool, RedSnitch, comes into play. 

RedSnitch is a tool designed to create specially crafted PDF files that can snitch NTLM hashes, providing red teamers with an efficient and covert way to gather credentials during engagements. But what makes RedSnitch particularly intriguing is its ability to disguise these malicious PDFs as legitimate documents, complete with logos from well-known antivirus companies. This clever tactic not only enhances the likelihood of user interaction but also adds a layer of deception that can be hard to detect. 

### A closer look on how RedSnitch works
RedSnitch operates by leveraging the art of deception to snitch NTLM hashes from unsuspecting users. At its core, the tool creates seemingly legitimate PDF files that mimic the appearance of confidential documents encrypted by well-known antivirus companies. This clever disguise is essential for social engineering attacks, as users are far more likely to open a document that appears trustworthy.

Once the user interacts with the created PDF by RedSnitch by clicking a link to access the file it will send the NTLM hash to your server. It captures the NTLM hash, a critical piece of information that can facilitate lateral movement within the network. This hash can be exploited by red teams to gain access to additional systems and sensitive data.

RedSnitch should be used in conjunction with tools such as Responder. Responder listens for authentication requests on the network, allowing it to intercept the NTLM hashes from the PDFs created with RedSnitch. This combination creates a powerful duo for credential harvesting.

### The process in action
The red team(er) begins by configuring [RedSnitch](https://github.com/eMVee-NL/RedSnitch?tab=readme-ov-file#getting-started) on their machine, customizing the PDF with a chosen antivirus logo to enhance its credibility and make it more enticing for the target.
![RedSnitch1](/assets/img/Tutorial/RedSnitch/RedSnitch1.png){: width="700" height="400" }
After the PDF has been crafted it can be delivered through various methods, including phishing emails, file shares, WebDAV, or even hosted on a web server. This flexibility allows red teams to choose the most effective delivery method for their specific target. Stealing NTLM hashes works only if you are in the same network. 
After sucessful delivery, the red team(er) should wait on user interaction. When the target user opens the PDF that has been created with RedSnitch and interacts with its hyperlink, the NTLM hashes of that user can then be caught with Responder or any other similar tool. The moment the user interacts with the hyperlink is crucial, as it opens the door for further exploitation within the network.
![RedSnitch2](/assets/img/Tutorial/RedSnitch/RedSnitch2.png){: width="700" height="400" }
With the captured NTLM hash in hand, the red team can now navigate laterally within the network, accessing sensitive information and systems that were previously out of reach.

### Conclusion
In summary, RedSnitch serves as a valuable addition to the red team's toolkit, enhancing their ability to gather credentials through clever social engineering tactics. While it offers unique functionalities for creating PDFs to steal NTLM hashes, it is important to remember that no single tool can be deemed the ultimate solution for all penetration testing challenges. RedSnitch should be viewed as a complementary resource that, when used alongside other established tools and techniques, can help red teams effectively identify and exploit vulnerabilities within a network. As always, a well-rounded approach to cybersecurity assessments is essential for achieving the best results in any engagement.