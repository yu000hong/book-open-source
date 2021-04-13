# DoS & DDoS

A denial-of-service (DoS) attack floods a server with traffic, making a website or resource unavailable. A distributed denial-of-service (DDoS) attack is a DoS attack that uses multiple computers or machines to flood a targeted resource. Both types of attacks overload a server or web application with the goal of interrupting services. 

As the server is flooded with more Transmission Control Protocol/User Datagram Protocol (TCP/UDP) packets than it can process, it may crash, the data may become corrupted, and resources may be misdirected or even exhausted to the point of paralyzing the system.

From a high level, a DDoS attack is like an unexpected traffic jam clogging up the highway, preventing regular traffic from arriving at its destination.

![](https://www.cloudflare.com/img/learning/ddos/what-is-a-ddos-attack/ddos-attack-traffic-metaphor.png)

### Difference Between DoS and DDoS Attacks?

The principal difference between a DoS and a DDoS is that the former is a system-on-system attack, while the latter involves several systems attacking a single system. There are other differences, however, involving either their nature or detection, including:

- **Ease of detection/mitigation**: Since a DoS comes from a single location, it is easier to detect its origin and sever the connection. In fact, a proficient firewall can do this. On the other hand, a DDoS attack comes from multiple remote locations, disguising its origin.

- **Speed of attack**: Because a DDoS attack comes from multiple locations, it can be deployed much faster than a DoS attack that originates from a single location. The increased speed of attack makes detecting it more difficult, meaning increased damage or even a catastrophic outcome. 

- **Traffic volume**: A DDoS attack employs multiple remote machines (zombies or bots), which means that it can send much larger amounts of traffic from various locations simultaneously, overloading a server rapidly in a manner that eludes detection.

- **Manner of execution**: A DDoS attack coordinates multiple hosts infected with malware (bots), creating a botnet managed by a command-and-control (C&C) server. In contrast, a DoS attack typically uses a script or a tool to carry out the attack from a single machine.

- **Tracing of source(s)**: The use of a botnet in a DDoS attack means that tracing the actual origin is much more complicated than tracing the origin of a DoS attack.

### Identify a DDoS attack

The most obvious symptom of a DDoS attack is a site or service suddenly becoming slow or unavailable. But since a number of causes — such a legitimate spike in traffic — can create similar performance issues, further investigation is usually required. Traffic analytics tools can help you spot some of these telltale signs of a DDoS attack:

- Suspicious amounts of traffic originating from a single IP address or IP range
- A flood of traffic from users who share a single behavioral profile, such as device type, geolocation, or web browser version
- An unexplained surge in requests to a single page or endpoint
- Odd traffic patterns such as spikes at odd hours of the day or patterns that appear to be unnatural (e.g. a spike every 10 minutes)

There are other, more specific signs of DDoS attack that can vary depending on the type of attack.

### Common types of DDoS attacks

Different types of DDoS attacks target varying components of a network connection. In order to understand how different DDoS attacks work, it is necessary to know how a network connection is made.

![](https://cloudflare.com/img/learning/ddos/what-is-a-ddos-attack/osi-model-7-layers.svg)

While nearly all DDoS attacks involve overwhelming a target device or network with traffic, attacks can be divided into three categories. An attacker may use one or more different attack vectors, or cycle attack vectors in response to counter measures taken by the target. 

Three categories:

- Application layer attacks
- Protocol attacks
- Volumetric attacks

#### Application layer attacks

Sometimes referred to as a layer 7 DDoS attack (in reference to the 7th layer of the OSI model), the goal of these attacks is to exhaust the target’s resources to create a denial-of-service.

The attacks target the layer where web pages are generated on the server and delivered in response to HTTP requests. A single HTTP request is computationally cheap to execute on the client side, but it can be expensive for the target server to respond to, as the server often loads multiple files and runs database queries in order to create a web page.

Layer 7 attacks are difficult to defend against, since it can be hard to differentiate malicious traffic from legitimate traffic.

![](https://www.cloudflare.com/img/learning/ddos/what-is-a-ddos-attack/http-flood-ddos-attack.png)

**HTTP flood**

This attack is similar to pressing refresh in a web browser over and over on many different computers at once – large numbers of HTTP requests flood the server, resulting in denial-of-service.

This type of attack ranges from simple to complex.

Simpler implementations may access one URL with the same range of attacking IP addresses, referrers and user agents. Complex versions may use a large number of attacking IP addresses, and target random urls using random referrers and user agents.

**DNS flood**

DNS flood attacks constitute a relatively new type of DNS-based attack that has proliferated with the rise of high bandwidth Internet of Things (IoT) botnets like Mirai. DNS flood attacks use the high bandwidth connections of IP cameras, DVR boxes and other IoT devices to directly overwhelm the DNS servers of major providers. The volume of requests from IoT devices overwhelms the DNS provider’s services and prevents legitimate users from accessing the provider's DNS servers.

**Memcached DDoS Attack**

A memcached distributed denial-of-service (DDoS) attack is a type of cyber attack in which an attacker attempts to overload a targeted victim with internet traffic. The attacker spoofs requests to a vulnerable UDP memcached* server, which then floods a targeted victim with internet traffic, potentially overwhelming the victim’s resources. While the target’s internet infrastructure is overloaded, new requests cannot be processed and regular traffic is unable to access the internet resource, resulting in denial-of-service.

#### Protocol attacks

Protocol attacks, also known as a **state-exhaustion attacks**, cause a service disruption by over-consuming server resources and/or the resources of network equipment like firewalls and load balancers.

Protocol attacks utilize weaknesses in layer 3 and layer 4 of the protocol stack to render the target inaccessible.

![](https://cloudflare.com/img/learning/ddos/what-is-a-ddos-attack/syn-flood-ddos-attack.png)

**SYN flood**

A SYN Flood is analogous to a worker in a supply room receiving requests from the front of the store.

The worker receives a request, goes and gets the package, and waits for confirmation before bringing the package out front. The worker then gets many more package requests without confirmation until they can’t carry any more packages, become overwhelmed, and requests start going unanswered.

This attack exploits the TCP handshake — the sequence of communications by which two computers initiate a network connection — by sending a target a large number of TCP “Initial Connection Request” SYN packets with spoofed source IP addresses.

The target machine responds to each connection request and then waits for the final step in the handshake, which never occurs, exhausting the target’s resources in the process.

**IP Fragmentation Attack**

An IP fragmentation attack is a type of DoS attack that delivers altered network packets that the receiving network cannot reassemble. The network becomes bogged down with bulky unassembled packets, using up all its resources.

**Teardrop Attack**

A teardrop attack is a DoS attack that sends countless Internet Protocol (IP) data fragments to a network. When the network tries to recompile the fragments into their original packets, it is unable to. 

For example, the attacker may take very large data packets and break them down into multiple fragments for the targeted system to reassemble. However, the attacker changes how the packet is disassembled to confuse the targeted system, which is then unable to reassemble the fragments into the original packets.

#### Volumetric attacks

This category of attacks attempts to create congestion by consuming all available bandwidth between the target and the larger Internet. Large amounts of data are sent to a target by using a form of amplification or another means of creating massive traffic, such as requests from a botnet.

![](https://cloudflare.com/img/learning/ddos/what-is-a-ddos-attack/ntp-amplification-botnet-ddos-attack.png)

**DNS Amplification Atack**

A DNS amplification attack is like if someone were to call a restaurant and say “I’ll have one of everything, please call me back and repeat my whole order,” where the callback number actually belongs to the victim. With very little effort, a long response is generated and sent to the victim.

By making a request to an open DNS server with a spoofed IP address (the IP address of the victim), the target IP address then receives a response from the server.

**NTP Amplification Attack**

An NTP amplification attack is a reflection-based volumetric distributed denial-of-service (DDoS) attack in which an attacker exploits a Network Time Protocol (NTP) server functionality in order to overwhelm a targeted network or server with an amplified amount of UDP traffic, rendering the target and its surrounding infrastructure inaccessible to regular traffic.

### How To Improve DoS and DDoS Attack Protection

- **Monitor your network continually**: This is beneficial to identifying normal traffic patterns and critical to early detection and mitigation.

- **Run tests to simulate DoS attacks**: This will help assess risk, expose vulnerabilities, and train employees in cybersecurity.

- **Create a protection plan**: Create checklists, form a response team, define response parameters, and deploy protection.

- **Identify critical systems and normal traffic patterns**: The former helps in planning protection, and the latter helps in the early detection of threats.

- **Provision extra bandwidth**: It may not stop the attack, but it will help the network deal with spikes in traffic and lessen the impact of any attack.

DDoS attacks are evolving, becoming more sophisticated and powerful, so organizations require solutions that use comprehensive strategies to monitor countless threat parameters simultaneously. To protect an organization from known attacks and prepare for potential zero-day attacks, multilayered DDoS protection, such as FortiDDoS, is necessary. 

FortiDDoS includes the Fortinet DDoS attack mitigation appliance, which provides continuous threat evaluation and security protection for Layers 3, 4, and 7.

### How To Mitigate a DDoS attack

The key concern in mitigating a DDoS attack is differentiating between attack traffic and normal traffic. The difficulty lies in telling the real customers apart from the attack traffic.

In the modern Internet, DDoS traffic comes in many forms. The traffic can vary in design from **un-spoofed single source attacks** to **complex and adaptive multi-vector attacks**.

A multi-vector DDoS attack uses multiple attack pathways in order to overwhelm a target in different ways, potentially distracting mitigation efforts on any one trajectory. An attack that targets multiple layers of the protocol stack at the same time, such as a DNS amplification (targeting layers 3/4) coupled with an HTTP flood (targeting layer 7) is an example of multi-vector DDoS.

Generally speaking, the more complex the attack, the more likely it is that the attack traffic will be difficult to separate from normal traffic - the goal of the attacker is to blend in as much as possible, making mitigation efforts as inefficient as possible.

**Blackhole routing**

One solution available to virtually all network admins is to create a blackhole route and funnel traffic into that route. In its simplest form, when blackhole filtering is implemented without specific restriction criteria, both legitimate and malicious network traffic is routed to a null route, or blackhole, and dropped from the network.

**Rate limiting**

Limiting the number of requests a server will accept over a certain time window is also a way of mitigating denial-of-service attacks. While rate limiting is useful in slowing web scrapers from stealing content and for mitigating brute force login attempts, it alone will likely be insufficient to handle a complex DDoS attack effectively.

**Web application firewall**

A Web Application Firewall (WAF) is a tool that can assist in mitigating a layer 7 DDoS attack. By putting a WAF between the Internet and an origin server, the WAF may act as a reverse proxy, protecting the targeted server from certain types of malicious traffic.

**Anycast network diffusion**

This mitigation approach uses an Anycast network to scatter the attack traffic across a network of distributed servers to the point where the traffic is absorbed by the network.

Like channeling a rushing river down separate smaller channels, this approach spreads the impact of the distributed attack traffic to the point where it becomes manageable, diffusing any disruptive capability.

