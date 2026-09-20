---
title: Practice your web pentesting skills on OWASP Juice Shop
author: eMVee
date: 2026-09-05 00:00:00 +0800
categories: [CTF, OWASP]
tags: [OWASP, OWASP Juice Shop, Juice Shop, OSWA]
render_with_liquid: false
---


![Image](/assets/img/WriteUp/OWASP-Juice-Shop/JuiceShopCTF_Logo.png){: .right}When you think of learning cybersecurity, you might picture dry textbooks, endless lines of confusing code, or boring compliance videos. But what if you could learn how to defend web applications by doing the exact opposite by aggressively trying to break them?

That is where [OWASP Juice Shop](https://owasp.github.io/www-project-juice-shop/) comes in. As one of the most sophisticated and engaging "intentionally broken" applications ever built, it flips traditional training on its head. Whether you are a developer aiming to write secure code, an aspiring penetration tester, or just a tech enthusiast curious about the hacker mindset, this platform is your perfect starting line.

## What is OWASP Juice Shop?
Created and maintained by the Open Worldwide Application Security Project (OWASP), Juice Shop is a fully functional web application written entirely in JavaScript (Node.js, Express, and Angular). 

On the surface, it looks like a modern, sleek online store where you can buy juice, fruit-themed merchandise, and even leave product reviews. But beneath the surface, it is riddled with security flaws. From simple design oversights to complex cryptographic vulnerabilities, Juice Shop mimics the real world mistakes made by modern development teams.

## Why it shines: The power of gamification
Learning by doing is effective, but learning by *playing* is a superpower. Juice Shop leverages gamification to keep you hooked through an interactive Score Board.

When you first launch the application, the Score Board is actually hidden. Finding it is your very first challenge! Once uncovered, the board tracks your progress across dozens of challenges rated from 1 star (very easy) to 6 stars (highly complex). 

Here is why it stands out from other hacking labs:
* **The OWASP Top 10 Coverage:** It provides hands on practice for the most critical web application security risks, including SQL Injection, Broken Authentication, and Cross Site Scripting (XSS).
* **Realistic Modern Tech Stack:** Unlike older vulnerable apps that use outdated PHP, Juice Shop uses a modern Single Page Application (SPA) architecture. The hacking techniques you learn here apply directly to modern web apps.
* **No "Reset" Frustration:** The application automatically tracks your solved challenges in the background. If your server crashes or you need to restart, your progress is saved.

## Sneak peek: Your first challenge
To give you a taste of the juice, let's look at a classic 1-star challenge: DOM-based Cross-Site Scripting (XSS). 

In the search bar of the shop, a user can look for products like "Apple Juice". If you type a small piece of malicious JavaScript like `<iframe src="javascript:alert('xss')">` and the application blindly executes it in the browser, you’ve just successfully exploited an XSS vulnerability. 

Juice Shop instantly rewards you with a satisfying notification banner: *“You solved a challenge!”* It is that instant feedback loop that makes it so addictive.

## How to get started in 3 quick steps
You don't need a powerful hacking rig to run Juice Shop. You can set it up locally on your machine in just a few minutes:
1. **The Easiest Way (Docker):** If you have Docker installed, simply run:
   `docker pull bkimminich/juice-shop` followed by `docker run --rm -p 3000:3000 bkimminich/juice-shop`.
2. **The Node.js Way:** Clone the GitHub repository, run `npm install`, and then `npm start`.
3. **The Cloud Way:** You can even deploy it instantly to free tier cloud platforms or try it via pre-built environments like [Herokuapp](https://juice-shop.herokuapp.com/#/). Or you can take a look at [tryhackme](https://tryhackme.com/room/owaspjuiceshop).

*Note: Once it's running, open your browser and navigate to `http://localhost:3000` to start your hacking journey!*
![image](/assets/img/WriteUp/OWASP-Juice-Shop/1.png){: width="700" height="400" }

## Conclusion: Ready to take a sip?
Securing the web requires understanding how to break it. OWASP Juice Shop bridges the gap between theoretical knowledge and practical execution in a way that feels like playing a video game. 

So, what are you waiting for? Fire up your local instance, hunt down that hidden Score Board, and see how many challenges you can conquer. Happy hacking!
