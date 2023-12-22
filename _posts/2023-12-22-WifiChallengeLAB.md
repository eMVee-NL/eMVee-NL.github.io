---
title: It's not a lie. Even you can hack WiFi!
date: 2023-12-22 09:00:00 +0000
categories: [Platform, WiFiChallengeLAB]
tags: [WiFiChallengeLAB]     
---

A while ago I was able to follow Offsec's PEN 210 (WiFi) and successfully take the exam. In that time, I purchased different hardware to practice hacking WiFi networks. Buying those hardware costs money, which is of course an investment in your equipment and will help you proceed in hacking networks. If you have the hardware to hack WiFi networks, please hack your own network. Hacking other people's networks without permission is illegal and is therefore discouraged. If you don't have the hardware yet, figuring out which equipment you need for those exercises (WiFI Hacking) takes some time and research. Setting up your own WiFi network and having all the right settings and configurations to hack unlimited WiFi networks is time-consuming. Time that you would rather spend directly on hacking your WiFi network. Fortunately, you can also hack WiFi networks in a safe environment without all the equipment. How? Well, that's easy! 

## WiFiChallengeLAB
![Image](/assets/img/WriteUp/WiFiChallengeLAB/B-WifiChallengeLab-LOGOWHITE.png){: .dark .right : w="300"}
![Image](/assets/img/WriteUp/WiFiChallengeLAB/B-WifiChallengeLab-LOGO.svg){: .light .right : w="300"}
The answer? [WiFiChallengeLAB](https://wifichallengelab.com/). WiFiChallengeLAB is a virtualized WiFi pentesting laboratory without the need for physical WiFi cards. Setting up your own WiFi network environment for hacking is no longer necessary. Everything is arranged virtually on the WiFiChallengeLAB virtual machine(s). From an open and WEP WiFi network to WPA2, WPA3 and WPA2 enterprise WiFi networks. A true gold mine to practice WiFi hacking! That said, you do need to run a virtual machine on your local machine. 

### Download the Virtual Machine
There are two versions available on WiFiChallengeslabs. There are some minor differents in the environments, but for the main attacks you could use either one of them to practice.

**WiFiChallengeLABs v1**

In this version the WPS attack vector is available. To use this virtual environment you have to use VMware.
[Download the VM](https://github.com/r4ulcl/WiFiChallengeLAB#download-vmdk-for-vmware-only)

**WiFiChallengeLABs v2**

In this version WPA3 is available, some tweaks on WPA Enterprise network. To use this new version you can choose to use VMware or Oracle VirtualBox.
[Download the VM](https://github.com/r4ulcl/WiFiChallengeLAB-docker#install)


### Challenges
The challenges are made in a constructive order so that a beginner can start working on the challanges. They start with basic skills such as scanning networks, identifying network names, MAC addresses, which channel a network uses. But also identifying workstations on a network. In addition, you actually practice hacking WiFi networks, all virtual of course. While completing the challanges you have to capture flags. There are different types of flags, such as captureing a flag from a website, but also a MAC address, a channel, a WiFi name or a WiFi password. If you can't figure it out, you can ask for a hint, which costs you a number of points that you earn by successfully completing a challenge. The answers in the challenges are so far as I know only valid from version 2.

### Scoreboard
As mentioned, points can be earned by solving challenges. The number of points you receive, depends on how difficult a challenge is, but also on whether you have used a hint. In the scoreboard you can see how you rank compared to other WiFi hackers. This is not necessary if you want to learn how to hack a WiFi network, but it is a nice bonus to compare how others are doing. It could even be competetive if you do it with friends. In your profile you can also see your score and the number of challenges you have solved, which category they are and how many you got wrong. A nice extra features for people who, like me, like funny statistics in graphs.

### Walkthrough
Without any knowledge, hacking a WiFi network can be intimidating. First try to find out on the internet how you could hack it. If you still can't figure it out, you can fortunately use the walkthroughs on [WiFiChallengeLAB](https://r4ulcl.com/posts/walkthrough-wifichallenge-lab-2.0/). Many of these walkthroughs explained well and some of them are relying on automated tools, I have hacked some of these networks manually and shared them in previous posts here. If you have any issues with hacking those networks you can contact r4ulcl on discord. His help is awesome! When I was learning for PEN210 I created a [mindmap](https://github.com/eMVee-NL/MindMap#wireless-network-pentesting) with all kind of flows of commands. He did review my mindmap about hacking WiFi.

### Conclusion
A fantastic solution was created by r4ulcl for anyone who wants to learn WiFi hacking. Everything is virtual and there are many different WiFi networks available. All credits and praise to r4ulcl. Partly because of him, I have spent hours in the evening hacking on virtualized WiFi networks in recent months. This solution was a part in preparing for the OSWP exam. If you are not completely convinced yet, take a look at [WiFiChallengeLAB](https://wifichallengelab.com/) and get started.