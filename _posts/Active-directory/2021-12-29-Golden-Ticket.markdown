---
layout: post
title: Golden Ticket
status: done
type: post
published: true
comments: true
date: 2021-12-29
---
# Golden Ticket
## Summary
Very powerful persistence and lateral movement technique. The idea is to create a Ticket Granting Ticket (TGT) using the KRBTGT account password hash. This allows users to access any service in the directory as admin and can persist for years. However, you basically need to have domain admin to do this

--- 
### Classifications

#persitance 
#post-exploit
#DomainAdminNeeded
#windows 
#active_directory 
#kerberos

[Source](https://www.qomplx.com/qomplx-knowledge-golden-ticket-attacks-explained/)

 To understand why this attack works as well as it does, it is important to understand how/what tickets are in Kerberos. In Kerberos the permission to do anything over the network (access a computer, remotely log in, or access a service) is granted by the ?? Key server and the  KRBTGT account?? in the form of a ticket which is given when the user first authenticates.

 The idea of this attack is to create one of these Ticket Granting Ticket (TGT) for a fake administrator which can last up to 10 years. The attacker can then use this Ticket Granting Ticket (TGT) to request other tickets which are Ticket- Granting Service (TGS) that give them access to anything and everything on the network.

[source](https://blog.quest.com/golden-ticket-attacks-how-they-work-and-how-to-defend-against-them/)

The information needed to create the ticket is:
-   The FQDN (Fully Qualified Domain Name) of the domain
-   The SID (Security Identifier) of the domain
-   The username of the account they want to impersonate
-   The KRBTGT account password hash


The first 3 are relatively easy to find as any user of the service. However getting the KRBTGT account password hash is the difficult part as it *should* only be held  server that hosts the Kerberos Key Distribution Center which is held on the Domain Controller. The KRBTGT account is the service account that runs the Kerberos Key Distribution Center service and it uses it's password hash to sign & encrypt the Ticket Granting Ticket (TGT) given when a user authenticates to the domain for the first time. 

The only thing that you really need to perform the attack is the password hash and there are several ways that you are able to get it.


The easiest is to already have local admin privilege on a Domain Controller. At this point you can use Mimicatz to pull the hash out of memory and then crack them offline


You can also steal the NTDS.DIT file from ` C:\Windows\NTDS\NTDS.DIT` which is a database that stores Active Directory data present on all of the Domain Controllers, including the password hashes for all users in the domain. 


 
??When there are multiple domains connected or Domain Controllers?? the systems need to have a way to stay connected and up to date with each others. This can be exploited in a DCSync attack which requires replication privilege's and the attacker pretends to be another Domain Controller asking to sync up.
