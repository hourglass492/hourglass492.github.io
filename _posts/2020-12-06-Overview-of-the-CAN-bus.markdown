---
layout: post
title: Overview of the CAN bus and its vulnerabilities 
status: done
type: post
published: true
comments: true
date: '20/12/1 17:00 '
---
This is a paper that I wrote for my CPR E 530 class at Iowa state. We were supposed to look into a network and try and find vulnerbilities that the network creates. I hope you enjoy it. I'm still trying to figure out how to use this web hosting system so the pictures may not be up for a while.

# Introduction

In this paper, we will go over the vulnerabilities that the vast majority of consumer vehicles possess. These vulnerabilities are due to the internal communication protocol used by vehicles called CAN which allow the many sensors and microcontrollers to communicate with the central control unit and each other inside of the vehicle. We will first cover how CAN is implemented and then go over how this implementation opens the protocol up to vulnerabilities. Then, we will cover the steps necessary to exploit these vulnerabilities.

## The CAN Bus

### The purpose of the CAN bus

The CAN bus was originally developed in 1986 in order to allow vehicle manufacturers a more efficient method to control remote sensor, motor and actuator  without the costs of a full electrical harness[3]. It was then revised into the CAN 2.0A and CAN2.0B standards by Borsch in 1991 primarily in order to fix the limited address space in the original system and has become the predominant internal vehicle communication protocol \[3] \[4]. The protocol offers “efficiently …. real time control with a very high level of security” with “bitrates up to 1 Mbit/s” for low power systems \[3]. 

## Implementation

### The Physical implementation of the CAN bus

![Figure 1. \[5\]](/_assets/2020-12-06-Overview-of-the-CAN-bus/fig1.png)


Because research and attacks on the CAN bus may require a person to directly tap into it, we will briefly cover its physical implementation. The CAN bus transmits bits via differential voltage modulation on a pair of wrapped wires. This allows the protocol to filter out interference by discarding all bits which aren’t sent on both wires. This works by setting a middle voltage for both cables to be at rest at. When sending a dominant bit, one wire is raised X volts while the other wire is lowered by X volts, where X depends on the vehicle \[5] which is shown above with X being equal to 140 mV and the middle voltage being 240 mV  [fig 1 [5]]. 
There are several more important things to know when working with the CAN Bus. First is that the dominant bit, or raised/lowered voltage, is a 0 and recessive, or a middle voltage, is a 1 bit[3]. Next, every node on the network must be able to read and write at the same time[3]. Finally, every node is synced to start packets at the same time in a process not covered here [3].

### Packet Format

![Figure 2. \[5\]](/_assets/2020-12-06-Overview-of-the-CAN-bus/fig2.png)


As this paper is focused on the vulnerabilities associated with cars and the CAN Protocol rather than the CAN protocol in and of itself, we will have to gloss over some of the finer points of its packet format and message system. We will focus on the CAN 2.0A and 2.0B packets as described in the BOSCH standard and how Arbitration ID’s and the Data Field work as those are the current implementations of the standard and those fields allow the majority of the exploits covered below. However included below is a brief description of each field in the 2.0A packet \[3] as shown above \[fig 2 \[5]]

1.  Start Bit: 1 bit, must be dominate
1.  Arbitration ID: 11 bits, give the priority of the packet. The lower the number the higher the priority
1.  Request Remote: 1 bit, set recessive to indicated a remote request
1.  Reserver: 2 bits, must be sent dominate
1.  Data Length: 4 bits, indicates length of Data field in 8 bit bytes. Range 0-8
1.  Data:  0 - 8 bytes, length indicated in previous field. Holds arbitrary data
1.  CRC Field: 15 bits, integrity check
1.  CRC Delimiter: 1 bit, must be recessive
1.  Ack Bit: 1 bit, receiver of message sets to dominate to acknowledge message
1.  ACK Delimiter: 1 bit, must be recessive
1.  End of Frame: 7 bits, must be dominate

	
![Figure 3. \[3\]](/_assets/2020-12-06-Overview-of-the-CAN-bus/fig3.png)


### Arbitration ID

Because all nodes on the network write to the same wire, some sort of collision avoidance is needed in order to allow different nodes to not distort each other's messages. This is done with the Arbitration ID of each node, which is the closest thing that the CAN protocol has to addresses and is the primary limiter on the number of nodes in a network. 
The system works by assigning every node on the network their own arbitration ID. If multiple nodes begin to broadcast at the same time, all but the node with the lowest Arbitration ID will realize they are broadcasting at the same time as another node and stop broadcasting. This allows the highest priority node to always broadcast on it’s first attempt without causing collisions. 
To understand how the lower priority nodes realize they are about to cause a collision it is important to remember that the lower the priority the higher the Arbitration ID, all nodes are always reading the wire, and a one bit is transmitted by having no voltage difference between the two wires. If two nodes begin transmitting at the same time they will create a voltage difference on the wire. This will continue to happen until the node with the lower Arbitration ID transmits a 1. It will expect to read a 1 back from the wire, however it will read that there is still a 0 being sent which means another node with a lower Arbitration ID is transmitting and that it needs to stop transmitting in order to prevent a collision. This makes the arbitration ID the limittier on the number of nodes on the network.
The final thing of note on the CAN bus is that it comes in the form of CAN 2.0A and CAN 2.0B. CAN 2.0A is basically the original CAN protocol and was shown above while the main difference between A and B is an extended address field \[3]. This is done by taking advantage of the reserve bit as demonstrated below [Fig 4 [3]].

![Figure 4. \[3\]](/_assets/2020-12-06-Overview-of-the-CAN-bus/fig4.png)



# Attack Opportunities

## Impact of Attacks

Before delving into the types of attacks possible, it is worth looking at the possible impacts of attacks on vehicles. Probably the most well known example was demonstrated by security researchers in several Wired articles over car hacking\[6] \7]. In these articles, they showed how an attack on the CAN bus can have severe implications to the driver and other users of the road. Once researchers were able to gain access to the CAN network, they were  able to control every aspect of the vehicle that was controlled by commands from the network\[2]\[6]\[7]. This included shutting the car down while on a busy interstate, activating windshield wipers and fluid and even violently turning the steering wheel using the power steering apparatus \[6] \[7]\[2]. The impacts of these attacks can only get worse as self driving cars become more prevalent and more aspects of the car are operated by a computer without the chance for a human to intervene. It takes little imagination to see how any of these attacks could cause serious damage and threat to life.

## Attack Types

### Flooding

As with most networks, flooding attacks are a distinct possibility on the CAN bus. All that would be needed is a constant broadcasting of the lowest possible arbitration ID and no other unit would be able to send a packet. This would make this attack trivially easy to write and while the impacts of flooding the network would vary vehicle to vehicle, it would likely just result in the vehicle freezing up \[5]. This makes a flooding style attack a perfect method to make vehicle ransomware as it would be easy to write and work on a wide variety of vehicles.

### Spoofing

The more insidious attacks that could take place involve spoofing of packets on the network. This is easy to do because every node on the network has the ability to broadcast and claim to be any other device. Because there is no authentication of the sender, an attacker is able to do this freely. This allows the attacker to issue commands to any electronic control unit that relies on a remote controller. Spoofing was the method used by the security researchers who spoke to Wired for their attacks. The one upside is that these attacks would have to be tailor made for each vehicle, which we get into below.

## Problems to Overcome

While these attacks are certainly a problem, it is clear that they aren’t being constantly exploited in the wild today. While this may be due to a lack of knowledge, it is primarily because vehicles do have some security that is built into the system.

### Prevent Access

The greatest defense CAN networks have is the difficulty in accessing them. Most internal CAN networks are essentially air gapped, which makes exploitation nearly impossible. However once on the network, it becomes trivial to carry out attacks which could have devastating effects. It is important to note that the designers seemed to rely on the difficulty in accessing the network, so once that barrier is overcome many other defenses are lacking. As there is no form of authentication built into the system or way to isolate a “evil” node, once on the network the attacker has free reign on that network.

### Segmented Networks

There are two saving graces if an attacker compromises a network. First, some vehicles have segmented networks in order to allow high priority messages to travel through an uncluttered network [3]. Although it doesn’t appear to have been implemented with security in mind, it does provide some defense in depth by preventing a single compromised node access to everything in the vehicle. However, implementing multiple networks and designing interfacing between the networks adds more expense and even this may not be foolproof from a security standpoint as networks will almost certainly have to pass messages between each other.

### Proprietary systems

The second saving grace consumers have  is the fact that the vast majority of vehicles have proprietary Arbitration ID’s and data formats. This means that for an attacker to spoof a vehicle's ECU packets, they would first need to go through the process of reverse engineering the protocol or find the documentation from elsewhere. While this certainly places a substantial burden to enter the field, it is certainly doable as will be discussed below and is horrific security practice.

## Next Steps

### Reverse engineering

The obvious next step in developing exploits for vehicles is reverse engineering the proprietary control signals for the vehicles that I wish to be working with. The Car Hackers Handbook goes into great detail on how to fully reverse engineer the protocol for each vehicle, including open source tools and equipment needed \[5]. If reverse engineering every vehicle becomes impractical, creating a bounty for documentation or using the documentation already made open source due to George Hotz’s research is also a possibility \[1]. Once this is done, it is relatively trivial to write a program that monitors the CAN Bus and injects the packets that you wish into it to cause things to happen \[5].

### Remote Access

After discovering what packets to inject into the network, some reliable method to get onto the network is needed. Two possible methods would be attaching a physical device to the CAN bus or gaining control of a device already connected to the bus. The more physical method would involve designing a device to sit on the network which would require some sort of microcomputer like a Pi Zero with a cellular signal attachment. Designing that could be its own project. The other option would be to find another system that has access to the CAN bus and the internet that can be exploited. The most direct examples would involve the driver's phone, the vehicles built in infotainment system, gps, or AAA type systems. This too is an area where extensive research can be done. 

# Conclusion

The security of cars is only likely to decline if manufacuriors do not implement new security measures. The current reliance on air gapped networks and obfuscated protocols is unlikely to remain effective for much longer. Companies like Coma AI will continue to collect the protocols and make them available to the public. This combined with the fact that reverse engineering is only going to get easier with new tools will remove what little protection obscure systems give. Cars are also becoming more and more connected with the internet. Whether it is through interfacing with a phone or directly with GPS or cell services, vehicles are slowly removing the air gap that has protected them for years. As few vehicles have effective ways to update their software, even patches will be difficult to implement. Ultimately, car hacking is a new and scarry front in cyber security. 

# References

\[1] ReasonTV, Super Hacker George Hotz: I Can Make Your Car Drive Itself for Under $1,000. United States: YouTube, 2017.

\[2] C. Valasek, steer fast 2. United States: YouTube, 2016.

\[3] BOSCH, “CAN Specification Version 2.0,” BOSCH, Gerlingen Germany, 1991.

\[4] “CAN in Automation (CiA): History of the CAN technology,” Can-cia.org. \[Online]. Available: https://www.can-cia.org/can-knowledge/can/can-history/. [Accessed: 09-Nov-2020].

\[5] C. Smith, The car hacker’s handbook: A guide for the penetration tester. San Francisco, CA: No Starch Press, 2016.

\[6] A. Greenberg, “The Jeep Hackers Are Back to Prove Car Hacking Can Get Much Worse,” Wired, 01-Aug-2016.

\[7] A. Greenberg, “A Deep Flaw in Your Car Lets Hackers Shut Down Safety Features,” Wired, 16-Aug-2017.
