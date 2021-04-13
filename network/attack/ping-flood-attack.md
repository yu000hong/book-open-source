# Ping Flood Attack

### What is a Ping (ICMP) flood attack?

A ping flood is a denial-of-service attack in which the attacker attempts to overwhelm a targeted device with ICMP echo-request packets, causing the target to become inaccessible to normal traffic. When the attack traffic comes from multiple devices, the attack becomes a DDoS or distributed denial-of-service attack.

### How does a Ping flood attack work?

The `Internet Control Message Protocol (ICMP)`, which is utilized in a Ping Flood attack, is an internet layer protocol used by network devices to communicate. The network diagnostic tools traceroute and ping both operate using ICMP. Commonly, ICMP echo-request and echo-reply messages are used to ping a network device for the purpose of diagnosing the health and connectivity of the device and the connection between the sender and the device.

An ICMP request requires some server resources to process each request and to send a response. The request also requires bandwidth on both the incoming message (echo-request) and outgoing response (echo-reply). **The Ping Flood attack aims to overwhelm the targeted device’s ability to respond to the high number of requests and/or overload the network connection with bogus traffic**. By having many devices in a botnet target the same internet property or infrastructure component with ICMP requests, the attack traffic is increased substantially, potentially resulting in a disruption of normal network activity. Historically, attackers would often spoof in a bogus IP address in order to mask the sending device. With modern botnet attacks, the malicious actors rarely see the need to mask the bot’s IP, and instead rely on a large network of un-spoofed bots to saturate a target’s capacity.

The DDoS form of a Ping (ICMP) Flood can be broken down into 2 repeating steps:

- The attacker sends many ICMP echo request packets to the targeted server using multiple devices.

- The targeted server then sends an ICMP echo reply packet to each requesting device’s IP address as a response.

![Ping ICMP DDoS Attack Diagram](https://www.cloudflare.com/img/learning/ddos/ping-icmp-flood-ddos-attack/ping-icmp-flood-ddos-attack-diagram.png)


The damaging effect of a Ping Flood is directly proportional to the number of requests made to the targeted server. Unlike reflection-based DDoS attacks like NTP amplification and DNS amplification, Ping Flood attack traffic is symmetrical; the amount of bandwidth the targeted device receives is simply the sum of the total traffic sent from each bot.

### How is a Ping flood attack mitigated?

Disabling a ping flood is most easily accomplished by **disabling the ICMP functionality of the targeted router, computer or other device**. A network administrator can access the administrative interface of the device and disable its ability to send and receive any requests using the ICMP, effectively eliminating both the processing of the request and the Echo Reply. The consequence of this is that all network activities that involve ICMP are disabled, making the device unresponsive to ping requests, traceroute requests, and other network activities.

### How does Cloudflare mitigate Ping Flood attacks?

Cloudflare mitigates this type of attack in part by standing between the targeted origin server and the Ping flood. When each ping request is made, Cloudflare handles the processing and response process of the ICMP echo request and reply on our network edge. This strategy takes the resource cost of both bandwidth and processing power off the targeted server and places it on Cloudflare’s Anycast network. Learn more about Cloudflare DDoS Protection.

