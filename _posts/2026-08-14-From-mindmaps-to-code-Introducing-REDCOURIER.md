---
title: From mindmaps to code || Introducing REDCOURIER
author: eMVee
date: 2026-08-14 00:00:00 +0000
categories: [RedTeam, Tutorial ]
tags: [RedTeam, file-transfer, OSCP, OSEP, eCPPT, PNPT, REDCOURIER]
render_with_liquid: false
---

![REDCOURIER dashboard](/assets/img/Tutorial/RedCourier/redcourier.png){: .right }{: w="500"}


If you have stumbled across my [GitHub repository (eMVee-NL/MindMap)](https://github.com/eMVee-NL/MindMap#mindmap-transfer-files-from-victim-to-attacker) or used my Obsidian cheatsheets before, you probably know how much I love structuring complex data.

# From mindmaps to code: Introducing REDCOURIER
 For a long time, I have been maintaining my dedicated file transfer mindmaps, specifically designed to visualize moving files from an attacker to a victim and vice versa. It was my personal cheat sheet, mapping out every niche technique, fallback protocol, and living off the land bypass I could find to keep things smooth.But let us be honest. When you are deep in a shell, your exam clock is ticking down, or you are facing a hardened target during an assessment, scrolling through a massive mindmap file can slow you down. A single typo, a forgotten parameter, or an unescaped character while typing a multi-line bash heredoc or a native PowerShell script, and your shell hangs. Your file gets corrupted on disk, and you lose precious time.
 I realized that my original documentation needed to evolve into something more active. It needed to become a living, breathing utility. That is why I built [REDCOURIER](https://emvee-nl.github.io/RedCourier/).

## Turning my file transfer mindmaps into an interactive tool
[REDCOURIER](https://emvee-nl.github.io/RedCourier/) is a lightweight, interactive web application that acts as the software evolution of my original file transfer mindmap structures. Instead of forcing you to manually copy generic placeholder commands out of an Obsidian canvas and replace the variables yourself, I built a responsive frontend interface that does the heavy lifting for you.

With [REDCOURIER](https://emvee-nl.github.io/RedCourier/), you just toggle between ingestion (downloading to the target) and exfiltration (uploading from the target), plug in your infrastructure IP addresses, choose your target filename, and select your protocol. The app instantly generates a sanitized, step-by-step command matrix that you can copy with a single click.

Whether you are streaming raw binaries natively over an encrypted SSH pipeline without falling back on restricted SFTP macros, establishing pure Netcat socket listeners, or mapping native administrative SMB shares, the app handles the formatting behind the scenes so you do not have to worry about layout defects or character corruption.
## Who I designed this for
When I translated my mindmaps into this application, I kept a few specific groups of security enthusiasts and students in mind:
- Exam takers (OSCP, PNPT, eCPPT, and beyond): During hands-on certification exams, stress is your biggest enemy. Forgetting the exact Certutil split syntax or a clean modern Invoke-WebRequest alias while the clock is running is incredibly frustrating. I wanted to create an operational co-pilot that restores your confidence instantly when you need to load an enumeration tool or a script into memory.
- CTF players on [HackTheBox](https://www.hackthebox.com/), [TryHackMe](https://tryhackme.com) or [HackMyVM](https://www.hackmyvm.eu): Capture the flag platforms love to place restrictions on your transfer paths. When standard HTTP downloads are blocked, [REDCOURIER](https://emvee-nl.github.io/RedCourier/) allows you to quickly cycle through alternative techniques like Base64 terminal streaming, Socat multiplexing, WebDAV type streams, or certutil for example.
- Penetration testers on defensive or offensive assessments: Keeping things clean and native is essential. I focused heavily on utilizing built-in operating system features and legitimate binaries to help you move files across environments without dropping unnecessary external tools onto the target systems.

## Features I have included so far
I wanted to make sure the app was highly functional right out of the box, so I built several core features:
- Dynamic pipeline maps: The interface dynamically renders a live network arrow map, visually showing you the exact data flow direction, active protocol, and port assignments based on your selections.
- Safe terminal escaping: The 'compiler' automatically sanitizes complex characters, meaning the browser will not accidentally break or eat characters when generating multi-line scripts or redirection parameters.

## Help me expand and build this out
[REDCOURIER](https://emvee-nl.github.io/RedCourier/) is heavily inspired by brilliant community platforms like [revshells.com](https://www.revshells.com), and my goal was to bring that same interactive ease to the specific world of file transport. But security is a constantly moving target. Operating systems receive patches, security boundaries shift, and new bypass techniques are discovered every single week.This is why I am opening this project up to the community, and I would love for you to get involved.If you have a custom file transfer bypass, a more efficient download wrapper, or want to help me improve the JavaScript compiler engine or the responsive design layout, please join me on [GitHub](https://github.com/emvee-nl/redcourier). Read through my contributing guide, drop your techniques into the schema, and let us work together to make [REDCOURIER](https://emvee-nl.github.io/RedCourier/) the ultimate unified file transfer registry for the entire community.