---
title: Five ways to discover hosts
author: eMVee
date: 2024-02-12 00:00:00 +0800
categories: [Tutorial, host discovery]
tags: [host discover, ping sweep]
---

Recently I created a number of vulnerable machines for HackMyVM. If you download and install these machines on your own network, you may not be able to determine the IP address assigned to them. It is possible to show the IP address once the vulnerable machine is booted on the logon screen. Most vulnerable machines I have created recently do not show an IP address when the machine is started. While this is a nice feature to show the IP address of the vulnerable machine, it is not necessary to see the IP address on the machine. There are several options to discover a host on your (virtual) network. Below I describe five options.

1. netdiscover
2. nmap ping sweep
3. fping
4. arp-scan
5. ping sweep with bash script


## netdiscover
Netdiscover is a reconnaissance tool, mainly developed for wireless networks without DHCP servers in wardriving scenarios. It can also be used on switched networks. The tool can passively detect online hosts or search for them by sending ARP requests. Netdiscover can inspect your network's ARP traffic, find network addresses using auto scan mode, or scan a given range or list of ranges. It uses the OUI table to show the vendor of each MAC address discovered. The output format is suitable for parsing by another program. This tool can be used to scan a subnet in your own lab environment. The command below can be used to scan for a range.
```bash
sudo netdiscover -r <IP>/<CIDR>
```

To use a specific interface you can use the following example
```bash
sudo netdiscover -i <interface-name>
```

It should be possible to use passive mode to detect devices on your network. This can be done with the following example.
```bash
sudo netdiscover -p
```

## nmap ping sweep
One of the most used tools is nmap. Most commonly it is used to scan for open ports and running services on a host, but you can use it to run a ping sweep as well. The nmap ping sweep is a technique used to discover hosts on a network that are currently active or "up." It is performed using the nmap command with the -sn flag (previously -sP in older versions) followed by the target IP address or range in CIDR notation. The example below can be used to run a ping sweep on a subnet.
```bash
nmap -sn <IP>/<CIDR>
```
The -sn flag tells nmap to skip the port scan and only perform host discovery. By default, nmap uses a variety of techniques to determine if a host is up, including ICMP echo requests (ping), TCP SYN packets to common ports, and ARP requests.

It's important to note that some hosts may not respond to ping requests due to firewall rules or other security measures, so a ping sweep may not discover all hosts on a network. Additionally, using nmap in this way may be interpreted as a potential security threat by some network administrators, so it's important to use this technique responsibly and with permission

## fping
Fping is a command-line tool used for network troubleshooting and diagnostic purposes. It is an enhanced version of the traditional ping command, with the main difference being its ability to scan a list of hosts and determine which ones are alive. 
Fping can also discover hosts within a subnet or IP range using the "-g" option. In the example below it will create a list with IP address found on a subnet.
```bash
fping <IP>/<CIDR> -ag 2> /dev/null
```

Fping has many other options, which can be viewed by running "fping --help". It is a powerful tool for network administrators and engineers, and it can be used to quickly identify network issues and troubleshoot connectivity problems.

## arp-scan
One of the tools I use in my own lab is arp-scan. This is a command-line tool used for discovering and fingerprinting IP hosts on a local network. 
It utilizes the Address Resolution Protocol (ARP) to send requests to the target hosts and displays their responses.
The output includes the IP address, MAC address, and vendor details of the responding hosts. It can be used to scan a local network, a subnet, or a specific list of hosts. 
The output format can be customized using various options, and the tool can also resolve IP addresses to hostnames. It requires privileges on some systems to use raw sockets.
To run arp-scan we can use any interface we would to scan.
```bash
sudo arp-scan -i eth0
```

## ping sweep 
If none of the previous tools mentioned are available on your machine, you could always try to create your own script to run a ping sweep.
The script below is an example of a simple script that will run a ping sweep in a range.

```bash
#!/bin/bash

if [ "$1" == "" ]
then
echo "You forgot an IP address!"
echo "Syntax: ./ipsweep.sh 192.168.4"

else
for ip in `seq 1 254`; do
ping -c 1 $1.$ip | grep "64 bytes" | cut -d " " -f 4 | tr -d ":" &
done
fi
```

The script takes one argument, which is an IP address without the last octet. For example, 192.168.4. If the argument is missing, the script will print an error message and exit.
If the argument is given ,the for loop iterates from 1 to 254 to scan all possible last octets of an IP address in the given range. The ping command is used to send one ICMP echo request to each IP address in the range. The -c 1 option tells ping to send only one packet.
The output of the ping command is piped to grep to filter the line containing the "64 bytes" string, which indicates that the ICMP request was successful. Then the filtered line is piped to cut to extract the IP address of the responding host.
The tr command is used to remove any colons from the IP address. The & symbol is used to run each ping command in the background, allowing the script to continue to the next IP address without waiting for the previous one to complete.

Below is an example to scan a range and add all identified IP addresses to a file.

```bash
./ipsweep.sh 192.168.4 > ips.txt
```

# A little disclaimer
This articale is written for educational purposes and you are not allowed to use it any means to harm someone. You can use this knowledge to practice your skills in your own environment.