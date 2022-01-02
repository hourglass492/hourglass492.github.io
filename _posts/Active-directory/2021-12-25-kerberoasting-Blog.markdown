---
layout: post
title: kerberoasting Blog
status: done
type: post
published: false
comments: true
date: 2021-12-25
---

### What the objective is

Kerberoasting is a method of privilege escalation that requires a local account on a domain joined system. The idea is to pull a the Ticket granting service (TGS) ticket out of memory from the system. That ticket is encrypted with the NTLM hash of the password of the account running that service. We are then able to crack the password and log into the account.



Ticket granting service ticket - This is the ticket that is encrypted with the NTLM hash of the service password. These are the service principles password that are chosen by humans and there for you can possibly crack them. With t


### What is needed

This is a privilege escalation attack, so we need to have an initial foothold on a machine that is AD joined for this to work. 


### First steps

Once authenticated to the domain, you get a ticket granting ticket (TGT) from the Kerberos key distribution center (KDC). You then use this ticket granting ticket to ask for the service you want to attack (or just all of them because why not).

You then receive a Ticket granting service (TGS) ticket for each of those services. Granted, you the computer doesn't just give you the ticket, but it is easy enough to pull the ticket out of memory using a tool like Mimicatz]].

An important note: There is no authentication check if you are allowed to use the service. This is done when you try to use the service with with the TGS. 


### Password from the ticket

The end goal of this attack is to get the service account password. We get this password by cracking the Ticket- Granting Service (TGS) ticket. The ticket is encrypted with hash of the password.

In the following idea:
encrypted_stuff = algorithm(stuff, key)

what_we_have = algorithm(ticket_stuff, hash(password))

What we want is the password.

This is where password cracking comes into play. We have a general idea of what the ticket stuff will be and we know the algorithm for encryption and hashing. At this point, it's just a series of guessing and checking that is password cracking.

Either the password is cracked or not at this point. Hopefully the admin choice something bad. 


### Exploitation
Once you have the password for the service account it is possible to simply log into that account or use the password to create a Silver Ticket in order to gain access to that account.



### Advantages 

This attack has several major advantages:
 - Can be done with a low privileged user
 - Getting TGS's is not unusual for a network, so as long as the attacker doesn't do something unusual (like ask for literally everything), it is hard to detect
 - The cracking can take place offline farther reducing chance of discovery
 - Service account processes are rarely changed, granting longer persistence
 - Large number of service accounts, increases probability that one has a bad password


### Mitigations
 - Don't have bad service account passwords
 - Rotate password, prevents persistence if compromised (may cause problems if bad passwords sneak in)
