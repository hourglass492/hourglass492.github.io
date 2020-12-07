---
layout: post
title: DNS Tunneling
status: done
type: post
published: true
comments: true
date: '20/12/1 17:00 '
---

DNS Tunneling is a method to send a receive data through the Domain Name Service protocol on the internet by replacing the subdomains and IP response with information. The practical purposes of this form of tunneling included extracting data or controlling malware from within a secured and monitored environment.

DNS Tunneling was developed as firewalls and intrusion detection systems become more prevalent and made covert communication more difficult. One solution to this problem was to use the DNS protocol because it is rarely blocked or monitored. Of coarse, DNS is rarely blocked or monitored because it is not designed to be used for communication and is therefor assumed not to be a threat.

# The DNS Protocol

Before going into tunneling, it is important to understand the basics of DNS. 

The beginning of the internet used IP addresses in order to connect to remote servers. However, as the internet became more popular the general public was not willing to remember four eight bit numbers for every website or email address they wanted to use. The solution was to create a hierarchical series of servers which kept track of a mapping of human readable domain, sub domain, and machine names and their corresponding IP addresses.

This allows a user to type in www.google.com into their browser and, after asking a series of name servers, the browser knows to go to 172.217.6.4. This mapping functionality has given DNS the nickname of the phone book of the internet.

How this is done is demonstrated in figure 1. Your computer will iteratively ask its local name server, the .com name server, and then the google.com name server before it gets the IP address of the www.google.com server and can connect with it.


![Figure 1 Iterative DNS From Jacobson]()

# Client To Server Communication

## Implementations

The client to server communication uses the DNS request packet, seen in Figure 2, by taking advantage of the sub domain feature of DNS. 

![Figure 2 DNS Quire Packet From Jacobson]()

To communicate, the remote server must be the name server for some sub domain. For example, mySite.com. The client then simply makes requests to fake servers or sub sub domains. For Example, all the following requests will go through the mySite.com Name Server:

1. ReallyCoolInformation.mySite.com
2. SuperSecretInformation.socialSecurityNumbers.mySite.com
3. UmVhbGx5Q29vbEluZm9ybWF0aW9u.mySite.com

On the server side, all requests could be logged which  would look something like this:

1: Request: ReallyCoolInformation
2: Request: SuperSecretInformation.socialSecurityNumbers
3: Request: UmVhbGx5Q29vbEluZm9ybWF0aW9u

These logs can then be parsed and all the data extracted. 

## Limitations

1: Each Sub Domain can only be 63 characters long.

2: A max of 256 characters can be sent at a time which limits data throughput. 

3: No encryption, but that can be added later.

# Server To Client Communication

## Implementation

Server to Client communication works in a similar way to Client to Server communication. Rather then sending the data in the sub domains, it would be placed in the IP address field, or the answers field in figure 3. This is a limited data size though because it is limited to the 32 or 64 bits for IPv4 and IPv6 addresses, but it is more then enough to send more then 4 billon unique instructions using only IPv4 addresses.

![Figure 3 DNS Response Packet From Jacobson]()

For example, the client could could send a DNS request for whatShouldIDo.mySite.com every hour. A response to the client of 182.231.10.4 for example could mean do nothing while 182.231.10.3 could mean encrypt everything in a ransomware attack. Every other IP address could have a unique command as well.

# Limitations

1: Extremely small data rate, 32 or 64 bits at a time without restructuring the packet.

2: DNS caching means every request must be unique or risk repeat of previous message.

# Detectable Characteristics & Uses

Because DNS tunneling is such a unusual use of the DNS protocol, it produces some very unique traffc characteristics that can be divided into payload anomalies and traffic anomalies. How pronounced these characteristics will be depends on the tunnelings use and the amount of traffic being sent over it.

## Characteristics

## Traffic Anomalies

* Very Heavy Traffic
* Repeating Patterns
* Odd Request Patterns

![Figure 3 Tunneling detection From Varonis]()

# Payload Anomalies

* Large subdomain names
* Full Packets
* Odd Names

The following paragraphs are supposed to corospond to the images. I have not yet figured out how to do that yet.

![Figure 4 Exfiltration From Varonis]() ![Figure 5 C2 From Varonis]() ![Figure 6 IP-Over-DNS From Varonis]()

Attackers can remove large amounts of information before being stopped and causes large amounts of traffic in a short amount of time.

Advanced malware can use tunneling to receive commands and updates and even give backdoor access which creates a repeating pattern of “calling home.”

In extreme cases, all internet traffic can be routed through the tunnel giving attacker complete remote access which can cause large amounts of sustained traffic









