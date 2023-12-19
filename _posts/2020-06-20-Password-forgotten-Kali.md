---
title: Oops, forgotten the password to logon to Kali!
author: eMVee
date: 2020-06-20 20:00:00 +0800
categories: [Tutorial, Password]
tags: [Kali, Password,]
render_with_liquid: false
---

Forgotten your password for your Kali Linux machine? It would happen to you, even though you would prefer to get started with the machine as quickly as possible. Fortunately, there is a way to recover the password. The tutorial below explains how you can change the password of your own Kali Machine during startup.


**Step 1 - Start or reboot your Kali machine**

When your Kali Machine is booting the GRUB boot menu will be loaded and a countdown timer will countdown. Before the countdown timer is finished we should press the `UP` or `DOWN` key on the keyboard to stop the countdown timer.
![Image](/assets/img/Tutorial/Kali-Password-Reset/Step1.png){: width="700" height="400" }

**Step 2 - GRUB menu**

Next we should press the `e` key in order to edit this boot menu entry. 
![Image](/assets/img/Tutorial/Kali-Password-Reset/Step2.png){: width="700" height="400" }

Once you pressed the `e` key on the keyboard the GRUB menu edit mode will be presented as shown in the screenshot above. Now we have to scroll down until we see the line starting with keyword `linux` as shown in the screenshot below. 

![Image](/assets/img/Tutorial/Kali-Password-Reset/Step3.png){: width="700" height="400" }

We should use navigational arrows to look for keyword `ro` and replace it with keyword `rw`. Next, on the same row we have to remove `quiet splash` and replate it with `init=/bin/bash` as shown in the screenshot below.

![Image](/assets/img/Tutorial/Kali-Password-Reset/Step4.png){: width="700" height="400" }


**Step 3 - Reboot your Kali machine**

Press the key combination `CTRL + X` to reboot the Kali machine.

![Image](/assets/img/Tutorial/Kali-Password-Reset/Step5.png){: width="700" height="400" }

Once it is rebooted you are root! 

**Step 4 - Reset password**

While we are root we can change the password for the root user. Type `passwd` command and enter a new password for the root user. Enter the root password again to verify. Hit the `ENTER` key again to confirm the new password. The system will confirm us that the password reset was successful.


![Image](/assets/img/Tutorial/Kali-Password-Reset/Step6.png){: width="700" height="400" }

But with the root we can change any users password. 
In the screenshot below we first checked the users listed in the `/etc/passwd` file and then changed the password for this user.

![Image](/assets/img/Tutorial/Kali-Password-Reset/Step7.png){: width="700" height="400" }

**Step 5 - Reboot your Kali machine to logon**

Now simply reboot your system or continue booting using the following linux command: `exec /sbin/init`
![Image](/assets/img/Tutorial/Kali-Password-Reset/Step8.png){: width="700" height="400" }

```bash
root@(none):/# exec /sbin/init
```

When the login prompt is shown, you can login with the new password.






