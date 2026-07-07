
This report details the solutions to the tasks in the SEEDLabs ARP Cache Poisoning Lab.

## Tools
- This lab has been tested on the SEEDUbuntu20.04 VM

The lab environment is described below:

| Host       | IP Address | MAC Address       |
| ---------- | ---------- | ----------------- |
| Host A     | 10.9.0.5   | 02:42:0a:09:00:05 |
| Attacker-M | 10.9.0.105 | 02:42:0a:09:00:69 |
| Host B     | 10.9.0.6   | 02:42:0a:09:00:06 |
## Task 1: ARP Cache Poisoning

In this task, we poison Host A's ARP cache by mapping Host B's IP address to the Attacker's MAC address. We accomplish this using three distinct methods, detailed below.

## Task 1.A: Using ARP Request:

![[Screenshot 2026-02-02 222604.png]]
This script sends a spoofed ARP request to poison Host A's cache, mapping Host B's IP to Attacker's MAC:
- Ethernet frame targets Host A from Attacker MAC.
- ARP packet with source address as B's IP and M's MAC and destination as A's IP and MAC address
- The `op` field's default value is used i.e. `1` indicating it's an ARP Request.

After running the code we see the packet sent out as:
![[Screenshot 2026-02-02 224611.png]]

The following is the ARP Cache entries in A and B, respectively, before and after executing the script:
![[task1.a_arp_A.png]]![[task1.a_arp_B.png]]
We can see that in the ARP Cache of A, the IP of B is associated to the MAC Address of the Attacker M.

## Task 1.B: Using ARP Reply:

![[task1.b_code.png]]
This script sends a spoofed ARP reply to poison Host A's cache, mapping Host B's IP to Attacker's MAC:
- To construct a spoofed reply, the `op` field of the ARP layer is modified to `2`, leaving the rest of the code intact. 

Once executed, the program transmits the following network packet:
![[task1.b_M.png]]

The "is-at" string indicates the execution of an ARP reply. 

The following is the ARP Cache entries in A and B, respectively, before and after executing the script:
![[task1.b_arp_A.png]]![[task1.b_arp_B.png]]
We can see, once again, that in the ARP Cache of A, the IP of B is associated to the MAC Address of the Attacker M.

## Task 1.C: Using ARP gratuitous message:

A spoofed ARP gratuitous message mapped to Host B's IP address is constructed and transmitted utilizing the following script:
![[task1.c_code.png]]

Once executed, the program transmits the following network packet:
![[task1.c_M.png]]

The following is the ARP Cache entries in A and B, respectively, before and after executing the script:
![[task1.c_arp_A.png]]![[task1.c_arp_B.png]]
Only A's cache got updated. B received the broadcast but dropped it because the sender IP matched its own. An ARP cache only stores mappings for other hosts, so B completely ignored the packet.


## Task 2: MITM Attack on Telnet using ARP Cache Poisoning

### Step 1 (Launch the ARP cache poisoning attack):

Here is the code to poison both A and B's ARP caches, using the ARP Request method, mapping each other's IP addresses to Attacker M's MAC address:
![[task2.a_code.png]]

The following is the ARP Cache entries in A and B, respectively, before and after executing the script:
![[task2.a_arp_A.png]]![[task2.a_arp_B.png]]

### Step 2 (Testing):

Once the ARP cache was poisoned, pinging from A to B gave these results:
![[task2.a_ping_A.png]]We sent 15 packets but only received 6. Here’s the capture from Wireshark:
![[task2.a_ping_wireshark.png]]
Initially, the ping failed because no echo replies came back. After some unsuccessful ping requests, A started broadcasting ARP requests to find B's MAC address at packet #77. B finally replied at packet #80, allowing the subsequent pings to go through.

This happened because A’s poisoned cache sent B's traffic to M’s MAC address. M’s network card accepted the packets, but its kernel dropped them because the destination IP didn’t match. Since B never actually received the packets, no replies were sent. Eventually, A broadcasted a new ARP request, got B’s real MAC, and overrode our spoofed entry, which fixed the ping.

### Step 3 (Turn on IP forwarding):

After enabling IP forwarding, we tried the attack again:
![[task2.a_M.png]]

We run a ping from A to B again, and this time it works perfectly:
![[task2.a_ping2_A.png]]

Here is the Wireshark capture of the ping traffic:
![[task2.a_ping2_wireshark.png]]

The above shows that Host A pings Host B but the poisoned ARP cache sends the packet straight to Host M. Host M catches it, realizes the destination IP isn't its own, and prepares to forward it to B. Before doing so, M's system sends an ICMP redirect back to Host A telling it that it has redirected the packet because it was destined for B and not M. Host B receives the ping and responds back with an echo reply. Because Host B's cache is also corrupted, that reply routes right back through Host M. Host M runs the exact same play, sending to Host B an ICMP redirect and passing the packet to Host A. The IP forwarding option enables M to forward the packet instead of dropping the packet.

### Step 4 (Launch the MITM attack):

Here is the script used to execute the MitM attack on the Telnet session once the ARP cache is poisoned:
![[task2.4_code1 1.png]]
![[task2.4_code2.png]]

After poisoning the ARP cache just like in Task 2.A, we keep IP forwarding on at first so the Telnet session between A and B can successfully connect. Once the session is active, we set IP forwarding to 0 so we can manually intercept and alter the packets using our sniff-and-spoof script. For any traffic heading from A to B, the script swaps out every single letter with a 'Z' before sending the spoofed packet along. Return traffic from B to A is left completely untouched, letting the Telnet responses pass through exactly like the originals.

This shows the output on Machine A while telnetting into Machine B:
![[task2.4_A.png]]
Typing any letters on Machine A turns them into 'Z', while numbers pass through completely unchanged.

Here is the attacker’s terminal:
![[task2.4_M.png]]

## Task 3: MITM Attack on Netcat using ARP Cache Poisoning

This task follows the exact same steps as Task 2.4, only we are targeting a Netcat session instead of Telnet. Here is the script used to execute the MitM attack on the Netcat traffic:
![[netcat_code1.png]]
![[netcat_code2.png]]
This script sniffs TCP traffic and targets packets traveling from A to B, swapping out the string "abdo" for "AAAA" inside the payload. If "abdo" isn't found, the packet is forwarded completely unchanged to its destination. Meanwhile, any return traffic from B to A is ignored by the script and left entirely alone.

Once the ARP cache is poisoned, we executed these commands on the attacker's terminal to open the Netcat connection and start the spoofing script:
```bash
$ Task2.A.py
$ sysctl net.ipv4.ip_forward=1  # Temporarily enable forwarding to connect Netcat
$ sysctl net.ipv4.ip_forward=0  # Disable forwarding to catch and manipulate the packets
$ Task2.4.2.py
```

The following is the output on Terminal of Machine A and B, respectively:
![[netcat_A.png]]
![[netcat_B.png]]
