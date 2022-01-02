---
layout: post
title: Silver Ticket Blog
status: done
type: post
published: false
comments: true
date: 2022-01-02
---
# Silver Ticket
## Summary
This is an attack that uses the password hash of a service account to create a Ticket- Granting Service (TGS) ticket that gives the possessor access to the service allowing persistence and lateral movement. To work, the attacker needs to have the windows service account's password hash.

--- 
### classifications
#lateral_movement 
#post-exploit 
#active_directory 
#windows 
#kerberos 
#stealthy
#persitance 

---

# Notes
The process for the attack is to get the password hash of the windows service account using methods like Notes/Active Directory/kerberoasting or gaining local admin privilege's on the server running the service and using Mimicatz to extract the hashes from memory. Once the hashes are found, Ticket- Granting Service (TGS)'s can be forged easily for users that do not exist and for large amounts of time (10 years).

This works because the service you are connecting to does not contact the Domain Controller to double check the authenticity of the ticket. This also means that all logs that are created from this attack stay on the server hosting the service and the attackers computer making this very stealthy.

## Uses

This is an very useful attack in several ways. 

### Lateral movement
When Notes/Active Directory/kerberoasting is used to get the password hashes, this provides a stealthy way to move laterally within a windows Active Directory Overview network. It is possible to create tickets for users who do and don't exist, so you may be able to log into a remote server as an admin in a very hard to detect way.

### Persistence
Once a server is compromised, it is possible to pull the hash out of memory. Because the Ticket- Granting Service (TGS) is created using only the hash, this allows you to create backup tickets that will allow you to re access the service if you get kicked out assuming the password is not changed.


## How TO

This is possible simply using Mimicatz:

Gives the password hash if admin on the machine
`.\mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" exit`

To mint the ticket and insert it into the cmd session:
` .\mimikatz.exe "kerberos::golden /user:NonExistentUser /domain:domain.com /sid:S-1-5-21-5840559-2756745051-1363507867 /rc4:8fbe632c51039f92c21bcef456b31f2b /target:FileServer1.domain.com /service:cifs /ptt" "misc::cmd" exit`
Note: kerberos::golden is not a mistake

At this point you can simply log into the remote service
`Find-InterestingFile -Path \\FileServer1.domain.com\S$\shares\`




## Defense


### Detection
 - Only possible at endpoints because the attack doesn't interact with the Domain Controller
 - Find the weird tickets:
	 - Long Expiration dates (10 min is Microsoft default & 10 years is Mimicatz default)
	 - Non existent users or users that shouldn't actually have access
	 - non default encryption for the ticket
	 - Modified groups for users
	 - Id and username mismatch
### Mitigation
 - Complex windows service account passwords prevent Notes/Active Directory/kerberoasting from working well
 - Rotating windows service account passwords prevents un discovered attackers from being able to use this to persist

### Response
 - Standard breach procedures
 - Change the passwords of the service account

### References
[source](https://www.varonis.com/blog/kerberos-attack-silver-ticket/)
[walkthrough](https://attack.stealthbits.com/silver-ticket-attack-forged-service-tickets)


