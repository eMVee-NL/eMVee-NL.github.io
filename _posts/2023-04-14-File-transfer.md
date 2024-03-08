---
title: Transfer files during a pentest
author: eMVee
date: 2023-04-14 20:00:00 +0800
categories: [Tutorial, file transfer]
tags: [OSCP,Mindmap, file transfer ]
render_with_liquid: false
---

While performing a pentest, it is often necessary to have file transfers between an attacker and the victim.

The victim may differ in terms of operating system. This means that other techniques must also be used to copy files. I noticed that some fellow students found it difficult to determine which technique to use to move a file from one machine to another. That's why I made a kind of mind map for this. With the help of this mind map you can identify techniques that can be used and which commands are needed.

Two mind maps have been created. This was done because there is a difference between sending data to the victim or from the victim to the attacker.

Of course, the mind maps are made in Obsidian so that copying commands is easy and at least something needs to be adjusted to a command so that it can be executed successfully. If you know of additions or other techniques, please let me know!

**Transfer file FROM attacker**
![Mindmap](https://github.com/eMVee-NL/MindMap/raw/main/image/Mindmap%20transfer%20files%20to%20VICTIM.png){: width="700" height="400" }
_Transfer files from attacker to victim_
The canvas file for Obisdian can be downloaded [here](https://raw.githubusercontent.com/eMVee-NL/MindMap/main/File-Transfer/Mindmap%20transfer%20files%20to%20VICTIM.canvas)


**Transfer file TO attacker**
![Mindmap](https://github.com/eMVee-NL/MindMap/raw/main/image/Mindmap%20transfer%20files%20to%20ATTACKER.png){: width="700" height="400" }
_Transfer files from victim to attacker_
The canvas file for Obisdian can be downloaded [here](https://raw.githubusercontent.com/eMVee-NL/MindMap/main/File-Transfer/Mindmap%20transfer%20files%20to%20ATTACKER.canvas)