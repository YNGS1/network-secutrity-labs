# MITM – ARP Spoofing - Lab 01
## Scenario 
ACME corp employee logs into an internal portal over HTTP.
ASSUMPTION: attacker already has a foothold on the LAN 

## Topology 


|  Host   |   Role    |       IP        |        MAC        |
|---------|-----------|-----------------|-------------------|
| ALPINE  | Server    |  192.168.100.10 | 00:0c:29:d3:d6:72 |
| Lubuntu | Victim    |  192.168.100.20 | 00:0c:29:96:46:11 |
| Kali    | Attacker  | 192.168.100.30  | 00:0c:29:bc:e2:2b |

## Baseline 
<img width="945" height="649" alt="image" src="https://github.com/user-attachments/assets/5787d477-d25b-4c21-8705-a64ca69b491a" /> 
- The victim can see the mac address of the server before the attack. The other mac address is that of the attacker on the same LAN

## Attack 
1. Enable IP forwarding - without it the attack becomes a DOS instead of a MITM: echo 1 > /proc/sys/net/ipv4/ip_forward
  <img width="751" height="516" alt="image" src="https://github.com/user-attachments/assets/1bcbcb5c-2d0c-4992-a9f9-056d24ba848f" />
  
2. arpspoof -i eth0 -t 192.168.100.20 -r 192.168.100.10 - Target stands for the victim machine, whose traffic we will be intercepting. Reverse tells the server - xxx.xxx.100.10, that xxx.xxx.100.20 is me.
<img width="527" height="57" alt="image" src="https://github.com/user-attachments/assets/cdb506d2-c2c6-4e8a-9635-2431b51f37c3" />
<img width="945" height="1306" alt="image" src="https://github.com/user-attachments/assets/7fc4587a-9502-4489-b0e8-9190cb14dd1e" />

## Result 
<img width="945" height="649" alt="image" src="https://github.com/user-attachments/assets/cbbf6b8a-a96f-480b-a294-62cee0493c4d" /> 
- xxx.xxx.100.10 now maps to attacker MAC address.
<img width="675" height="659" alt="image" src="https://github.com/user-attachments/assets/83b201c9-aaa3-4a5a-bc55-11184374d84c" />
- the victim enters credentials in the company portal.
<img width="945" height="434" alt="image" src="https://github.com/user-attachments/assets/afe1a6bd-9106-49fb-b896-50b0ad0f0c94" />
- packet capture from the attacker machine
<img width="228" height="315" alt="image" src="https://github.com/user-attachments/assets/480a7842-e883-442a-ad59-43dc1387d538" />
- the credentials transmitted over plain text 

## Detection 
1. Detect duplicate MAC addresses in the ARP table.
2. Flood of gratuitous ARP replies
3. ICMP host-redirected storm from one host

## Mitigation
1. Dynamic ARP Inspection / static ARP
2. HTTPS for encrypted traffix

## MITRE ATT&CK
T1557.002 — ARP Cache Poisoning

