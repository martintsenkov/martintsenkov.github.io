WPA2-PSK Wireless Network Compromise

Host: Kali Linux
Target: WPA2-PSK wireless network
Attack Type: Offline password cracking
Tools: airodump-ng, aireplay-ng, aircrack-ng
Wordlist: Custom

Overview

This write-up documents a basic but realistic attack against a WPA2-PSK protected wireless network.
The goal was to capture a valid WPA handshake and perform an offline password attack using a custom wordlist.

This scenario reflects a common real-world weakness: strong encryption rendered ineffective by weak passwords.

Reconnaissance & Preparation

The wireless interface was placed into monitor mode to allow passive monitoring of nearby wireless networks.
To reduce traceability, the MAC address of the wireless adapter was randomized before starting the attack.

Once prepared, the surrounding wireless environment was scanned to identify potential targets.

Target Identification

After identifying the target access point, traffic was filtered to monitor only the selected network.
This made it easier to observe client activity and capture a WPA handshake when a client connects or reconnects.

airodump-ng --bssid <TARGET_BSSID> --channel <CH> -w capture wlan1mon


This command was used to monitor the target network and write captured traffic to a file for later analysis.

Handshake Capture

To accelerate the process, a deauthentication attack was launched against a connected client.
This forces the client to disconnect and immediately reconnect, generating a WPA handshake in the process.

aireplay-ng --deauth 3 -a <TARGET_BSSID> -c <CLIENT_MAC> wlan1mon


Shortly after executing the deauthentication attack, a valid WPA handshake was successfully captured.

Offline Password Cracking

With a valid handshake captured, the attack moved to the offline phase.
The captured .cap file was tested against a custom wordlist using aircrack-ng.

aircrack-ng capture.cap -w custom_wordlist.txt


The correct WPA2-PSK passphrase was successfully recovered.

Result

✔️ Valid WPA handshake captured

✔️ WPA2-PSK passphrase cracked

✔️ Wireless network fully compromised

Once the pre-shared key is known, an attacker can:

Join the network legitimately

Monitor internal traffic

Launch further internal attacks
