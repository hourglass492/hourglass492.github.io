---
layout: post
title: NSA Codebreaker 2021
status: done
type: post
published: true
comments: true
date: 2022-01-30

---


### What is the NSA Codebreaker Challenge

The NSA CodeBreaker challenge is a year competition between schools where "The NSA ... provides students with a hands-on opportunity to develop their reverse-engineering / low-level code analysis skills while working on a realistic problem set centered around the NSA's mission." The competition is divided into 2 parts, solo and team, each consisting of 6 and 4 challenges respectively. 

### This Year's Story

The Internet is home to many different cyber actors. To better prepare for and defend against these actors, NSA routinely investigates foreign cyber actors and their activities. During one such investigation, a new IP address was identified to be part of an unknown actor's infrastructure. NSA believes it is a listening post (LP). The following Tasks will use network security and forensic tools to investigate this unknown actor's threat against the US Defense Industrial Base (DIB).

DISCLAIMER - The following is a FICTITIOUS story meant for providing realistic context for the Codebreaker Challenge and is not tied in any way to actual events.

  
### Attacks Seen and Overview
  - Supply Chain
  - Phishing
  - Malware

### Overview of Skills Involved
  - Log analysis and forensics
  - Powershell Malware analysis
  - Windows Registry Forensics
  - Docker file Forensics and Reverse engineering
  - Object Oriented ELF binary reverse engineering
  - Cryptography attacks

### Overview of each challenge

#### Disclaimer
Each competitor got slightly different files to insure that commands ran by one competitor couldn't simply be copy pasted by another competitor. As not all of the screen shots came from the same competitor, there will be slight discrepancies between specific information. This should have no affect on the underlying methiod to solve each challenge though.

#### Task 1
[Full Write up](task01/writeup.md)

The challenge description is as follows:

> The NSA Cybersecurity Collaboration Center has a mission to prevent and eradicate threats to the US Defense Industrial Base (DIB). Based on information sharing agreements with several DIB companies, we need to determine if any of those companies are communicating with the actor's infrastructure.

> You have been provided a capture of data en route to the listening post as well as a list of DIB company IP ranges. Identify any IPs associated with the DIB that have communicated with the LP.

#### And you are given:
    
    Network traffic heading to the LP (capture.pcap)
    DIB IP address ranges (ip_ranges.txt)

#### Overview of how challenge was completed:
Simply by matching the IP subnets with any IP addresses in the pcap file.

#### Task 2
[Full Write up](task02/writeup.md)


The challenge description is as follows:
 > NSA notified FBI, which notified the potentially-compromised DIB Companies. The companies reported the compromise to the Defense Cyber Crime Center (DC3). One of them, Online Operations and Production Services (OOPS) requested FBI assistance. At the request of the FBI, we've agreed to partner with them in order to continue the investigation and understand the compromise.

 > OOPS is a cloud containerization provider that acts as a one-stop shop for hosting and launching all sorts of containers -- rkt, Docker, Hyper-V, and more. They have provided us with logs from their network proxy and domain controller that coincide with the time that their traffic to the cyber actor's listening post was captured.

 > Identify the logon ID of the user session that communicated with the malicious LP (i.e.: on the machine that sent the beacon *and* active at the time the beacon was sent).


#### And you are given

    Subnet associated with OOPS (oops_subnet.txt)
    Network proxy logs from Bluecoat server (proxy.log)
    Login data from domain controller (logins.json)

#### Overview of how challenge was completed:
Using the subnet given, we are able to quickly see the time of the attack, and utilize that time in the proxy file. The proxy file gives us an IP for the machine used and with that and checking the times while users were logged in, we can narrow down the logonID of the culprit.


#### Task 3
[Full Write up](task03/writeup.md)


The challenge description is as follows:
 > With the provided information, OOPS was quickly able to identify the employee associated with the account. During the incident response interview, the user mentioned that they would have been checking email around the time that the communication occurred. They don't remember anything particularly weird from earlier, but it was a few weeks back, so they're not sure. OOPS has provided a subset of the user's inbox from the day of the communication.
 > Identify the message ID of the malicious email and the targeted server.


And you are given:

    User's emails (emails.zip)

Overview of how challenge was completed:

The emails given were examined by decoding the attached files. One of the emails contained a malicious powershell script, the message ID gives the first flag. The powershell script then had to be decoded to reveal a GET request, downloading data from the LP. The script had to be modified to take the data from the .pcap file included in Task 1 and derive the code being constructed by the script. Once constructed, the script is seen to be POSTing scraped data to a domain and that is the second flag.


#### Task 4
[Full Write up](task04/writeup.md)


The challenge description is as follows:
 > A number of OOPS employees fell victim to the same attack, and we need to figure out what's been compromised! Examine the malware more closely to understand what it's doing. Then, use these artifacts to determine which account on the OOPS network has been compromised.




And you are given:

    OOPS forensic artifacts (artifacts.zip)
    
And the flags are

    The name of the machine the attackers can now access
    The username the attackers can use to access that machine  
   
   
Overview of how challenge was completed:

Using the malicious script found in Task 4, the functionality of the script had to be determined. It was found to dig for PuTTY and WinSCP session data in the Windows Registry. From the given NTUSER file, you had to determine which PuTTY and WinSCP keys were persisted in the registry. Using the list of previous session, we had to determine which key was not encrypted. Once the non-encrypted key was found we had to go back to the registry and find the Hostname that the malware sent to the LP and now had access to.

#### Task 5
[Full Write up](task05/writeup.md)

The challenge description is as follows:

 > A forensic analysis of the server you identified reveals suspicious logons shortly after the malicious emails were sent. Looks like the actor moved deeper into OOPS' network. Yikes.

 > The server in question maintains OOPS' Docker image registry, which is populated with images created by OOPS clients. The images are all still there (phew!), but one of them has a recent modification date: an image created by the Prevention of Adversarial Network Intrusions Conglomerate (PANIC).

 > Due to the nature of PANIC's work, they have a close partnership with the FBI. They've also long been a target of both government and corporate espionage, and they invest heavily in security measures to prevent access to their proprietary information and source code.

 > The FBI, having previously worked with PANIC, have taken the lead in contacting them. The FBI notified PANIC of the potential compromise and reminded them to make a report to DC3. During conversations with PANIC, the FBI learned that the image in question is part of their nightly build and test pipeline. PANIC reported that nightly build and regression tests had been taking longer than usual, but they assumed it was due to resourcing constraints on OOPS' end. PANIC consented to OOPS providing FBI with a copy of the Docker image in question.

 > Analyze the provided Docker image and identify the actor's techniques.

##### And you are given:

    PANIC Nightly Build + Test Docker Image (image.tar)

##### And the flags are:

  Enter the email of the PANIC employee who maintains the image:
  Enter the URL of the repository cloned when this image runs:
  Enter the full path to the malicious file present in the image:

##### Overview of how challenge was completed:
The tar image given is a complete docker image. Opening up the tar image and looking through the configuration json files gives the first flag. Then looking at the commands ran at startup gives a script that downloads the GitHub repo from the link and the name of the malicious file. At this point, unpacking all of the layer 2 tar files and using find gives the full path.

#### Task 6
[Full Write up](task06/writeup.md)

##### The challenge description is as follows:

 > Now that we've found a malicious artifact, the next step is to understand what it's doing. Identify some characteristics of the communications between the malicious artifact and the LP.


And you are given no new files

##### And the flags are:

  Enter the IP of the LP that the malicious artifact sends data to
  Enter the public key of the LP (hex encoded)
  Enter the version number reported by the malware

##### Overview of how challenge was completed:

Drop the binary file into a reverse engineering program. After analyzing source code, it becomes clear which functions are called to give the flags. At this point run the program in GDB (which needs to be installed on the docker image) and simply call the functions that give the flags.


### Detection and defense of attacks
It is important to note that the scenerio given assumes that the NSA had already compromised the malicious listening post and captured the packets to discover the compromise. This is unrealistic for the average company as it is illegal to attack suspected malicious servers in the US. However, it is perfectly reasonable for this whole thing to be kicked off by suspicious communication to an unknown server by several computers within a controlled corporate network. 


#### task1
##### Attacker's Avoidance
Task 1 is only identifying the IPs that communicated with the listening post. Currently from the attcaker's side, there isn't much worth noting yet.
##### Improve Defense
The current origin of the attack is unknown at this time in the challenge, so the defense correlates with the defense of Task 3.
#### task2
##### Attacker's Avoidance
It was fairly easy for the attacker so far, the only reason the attack has been easy to narrow down in Task 2, is due to the low amount of activity from the matching IP address. With this the defense is able to see a suspicious HTTP request, but currently doesn't know the true nature of the attack. That will be discussed more in Task 3.
##### Improve Defense
Like Task 1, the current origin of the attack is unknown at this time in the challenge, so the defense correlates with the defense of Task 3.
#### task3
##### Attacker's Avoidance
The attacker does a great job at making their attack complicated and hidden. We are able to realize it is an attack through email, given that the user claims to have only used email during their time logged in. If the user was doing more on the machine, the attack would have been much harder to find. Given the nature of email threats, we are able to narrow down the email with links/downloads in them a quickly find one that is flagged as a virus (or suspicious file). Encrypting the data in Base64 helps the nature of the attack to be more hidden, but after getting cracked by a defender, the attacker put another way of encryption in via PowerShell code. Having several layer of encryption ensures a more complex virus and making its contents harder to figure out.
##### Improve Defense
In Task 3, we discover that the attack was performed through a malicious attachment in an email. The best way to improve defense would be either: have phishing email training for employees, improve the companies email filter, or to have a better anti-virus installed to immediately take action when said attachemnt is clicked (downloaded). Improving employees ability to spot phishing emails seems like the best option in this case as it appeared that he employee was not phased at all about having been a target of a spear phishing attack. Overall the ability to spot these attacks will be the best line of defense.
#### task4
##### Attacker's Avoidance
The attacker was smart in this task to avoid being spotted. First, the executed powershell script was ran in a hidden window, so the user of the machine would not know that the script was executing. Second, the script did not directly contain malicious code, to allow it to pass an anti-virus scan. Instead the script downloaded seemingly useless data thorugh an HTTP GET call and constructed the code itself. Lastly, the script had a delay between the GET request and POSTing the information it had scraped from the user's machine, making the two events appear to be unrelated, once again avoiding an IPS system.
##### Improve Defense
The best line of defense against this script would be to not allow users to execute powershell script on their machine unless as admin. This would have forced the script to prompt "Run Powershell as Administrator", alerting the user of it. Another method of avoidance would be to have a better IPS system in place. This might have alerted the firewall of malicious traffic and blocked the request from getting to the user's machine or sending data to the LP. Though it is hard to detect malware once it is on the machine, a better Firewall table might have detected and stopped this event.
#### task5
##### Attacker's Avoidance
In this case the malware was detected on the machine due in part of guessing based on the nature of CTF's. If this wasn't the case it would have been possible to hide the malware better. This obviously would mean stripping the binary so that it does not stand out. However, editing a known binary such as make leaves the attacker vulnerable to programs such as debsums which compares the hash of all of the binaries on the computer to a known good one on a remote system. This would be possible to get around however by infecting some underlying code needed in checking the hash. The idea behind this was first stated in an paper by Ken Thompson [on trusting trust](https://www.cs.cmu.edu/~rdriley/487/papers/Thompson_1984_ReflectionsonTrustingTrust.pdf). By infecting the debsums equivalent it would be possible to hard code it to always return that debsums is correct. If there is not a debsums type application installed trojonizing make would work just fine
##### Improve Defense
The main defenses against trojanized binaries that we looked at in this challenge is cryptographic signatures and certificates. The idea would be using known good hashes for each binary to check against. If any of the binaries were edited to make them malicious then the hashes would change. It would also be possible to sign the docker image itself with a gpg key. Assuming that the attackers did not find this key on the server, they would be unable to change anything in the docker image without breaking the signature attached to it.
#### task6
##### Attacker's Avoidance
From the attacker side, the same advantages from task 5 apply, however with a larger focus on obfuscating the code. To a certain extent, you have to assume that your malware will eventually be found by the defenders. At this point obfuscation and packing are incredibly important and many different methods could be used. Static analyzing could be made significantly more difficult by using layers of encryption and encoding that slowly undoes itself each round. This would even be more effective if the entire binary is never un encrypted at any one time to hinder dynamic analisis. Another ideal would be requiring the malware to retrieve a remote key. This could mean that if you suspect your malware is about to be or already has been captured you can stop serving up the key. If the defenders haven't captured the key yet the sample may be useless.
##### Improve Defense
Malware analyzing in this case is an incredibly interesting a detail driven job. A couple of the important things that would help the defender in this case are detailed log files and a large amount of time and extra resources to work on this project. There is a direct correlation between how much time an analyzst is able to put into a problem and how much information they are able to get out of it. In this case, it may make sense for the company to stop looking at the malware after task 6 and they know what it is doing. In fact there may even be enough to create an AV rule that detects similar implants and that is good enough for many companies. However, deeper analisis (as done in the solo tasks) can help give an much deeper understand of the ATP that we are dealing with and gives a chance to bring them to justice. The additional log files mentioned as will would be important to both detect the attack and to store a key sent over the wire if the malware used encryption as  mentioned above.

### Sources

Sources and references are included in each Task's write-up.

https://nsa-codebreaker.org/challenge

https://nsa-codebreaker.org/resources
