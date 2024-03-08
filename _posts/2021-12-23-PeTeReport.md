---
title: PeTeReport an open-source vulnerability reporting tool
author: eMVee
date: 2021-12-23 20:00:00 +0800
categories: [Tools, Reporting]
tags: [Tools, PeTeReport, Reporting]
render_with_liquid: false
---

A few weeks ago I had read an article on [hakin9](https://hakin9.org/petereport-open-source-application-vulnerability-reporting-tool/) about an open source vulnerability reporting tool called PeTeReport. Lately I've had my priorities on other things than looking into PeTeReport. And now that I've passed the CSSLP exam, I have time to experiment a bit with this tool. 

## PeTeReport is an open-source vulnerability reporting tool

![Image](/assets/img/Tools/petereport/dashboard.png){: .right : w="400"}

PeTeReport is an open source application vulnerability reporting tool developed by Miguel Morillo (1modm) and designed to assist pentesting efforts, by simplifying the task of writing and generation of reports. The application can be found on the [Github page](https://github.com/1modm/petereport) of 1modm. The [documentation](https://1modm.github.io/petereport/) of PeTeReport is well explained. The web application has already a few features which are really nice such as finding templates, CVSS 3.1 score and customizale reports. Perhaps, this web application could assist you during an exam such as OSCP, eWPT, ECPPT and in your daily work.


## Features
Although the web application was not developed long ago, there are already quite a few nice functionalities present in the web application. At the time of writing, the following functionalities are supported according to the documentation. 
- [x] Customizable reports output
- [x] Customizable reports templates
- [x] Findings template database
- [x] Possibility to add appendix to findings
- [x] Possibility to add attack trees [Deciduous](https://www.deciduous.app/) to findings
- [x] HTML Output format
- [x] CSV Output format
- [x] PDF Output format
- [x] Jupyter Notebook Output format
- [x] Markdown Output format
- [x] CVSS 3.1 Score
- [x] Docker installation
- [x] DefectDojo integration
- [x] User management

Enough to make me decide to install this web application locally to try it out.

## Installation
If you would like to try PeTeReport, you could get it from the [Github page](https://github.com/1modm/petereport) of 1modm. The documentation on how to perform the installation is self-explanatory, I have described a short summary below.

If docker has not been installed yet, you have to install it
```bash
~$ sudo apt install docker.io docker-compose
```

Clone the repository into the ```/opt``` directory and build the docker. **Please, pay attention to the security of the application by chancing the default username and password for example. Or by using your own certificate.** Most of this is documented in the [documentation](https://1modm.github.io/petereport/). There are more things which you should do for hardening you installation. But that's another story, which will not be described in this post. So as mentioned, clone the repository and run docker-compose to build the container.
```bash
~$ cd /opt
/opt$ sudo git clone https://github.com/1modm/petereport
/opt$ cd petereport
/opt/petereport$ sudo docker-compose up --build
```

If all succeed, you we be able to open PeTeReport in your browser. Login with the username and password and you are ready to go!

## Conclusion
Hopefully PeTeReport will remain open source, because so far I am very excited about the web application. This makes my and many other lives so much easier when writing reports. There are still tings that could be added, developed or tested to improve this applaction, therefor I will keep a close eye on this project and test the upcoming versions. I will share my findings with Miguel on his Github page. And I encourage you to also help as much as possible to realize this product into a mature product. My experience is that any suggestions are welcomed and for for that I would like to say thank you to Miguel.
![Image](/assets/img/Tools/petereport/demo.gif){: width="700" height="400" }

