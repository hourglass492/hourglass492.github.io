---
layout: page
title: About
permalink: /NSA-Codebreaker-2021/task2.md
---

Author: kscanlon3 on Github

### First Insights

After finishing Task 1, we now have roughly 4 IP addresses (varies from person to person) in relation to the DIB companies that have communicated with the LP. One of the reporting companies *Online Operations and Production Services* (OOPS) has provided us with their network proxy logs, login data, and which subnet was associated with their company. The task is to now narrow down the logon ID of the user that communicated with the LP.

### Starting the Task

To begin we will need to look at the subnet for OOPS, which was given in the *oops_subnet.txt* file:

```
172.17.130.0/23
```

Now that we have the subnet, we will match it with one of the IPs we flagged in Task 1. In my case, the only IP that matched was *172.17.130.222* which sent out an HTTP request. Viewing the pcap file in Wireshark, we can figure out the time this request was sent: *12:13:09* in UTC +0. We now have enough information to move onto the proxy file.

### Viewing the Proxy Logs

The first part worth noting in the proxy log is the time. At the top of the file, it states:

```
#Date: 2021-04-16 15:38:08 EST (UTC-0400)
```

Which is a 4-hour time zone difference from our discovered time. Therefore, we will now be looking at *08:13:09* for our time in these logs. Upon doing a quick search (I used the *grep* command) to find the time, there is only one instance of it in the log:

```
2021-03-16 08:13:09 42 192.168.11.79 200 TCP_MISS 12734 479 GET http jhthx.invalid movement - - DIRECT 172.28.218.70 application/octet-stream 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.116 Safari/537.36' PROXIED none - 192.168.11.123 SG-HTTP-Service - none -
```

With this information obtained, for this task, we will only need the initial IP address listed in the proxy log: *192.168.11.79*. We can now take this IP and move onto the *logins.json* file that was given in Task 2.

### Narrowing Down the ID through the logins file

First step is to search through the login file for that IP we discovered. Upon doing so, several results show up, but we only care about the results with logonIDs attached. Piping to search commands together, we get roughly 20 results of users logging in. In my version, one user could be discarded immediately, as the logged on after *12:13:09* (oddly, the login file is back in UTC +0). The next step is to search up each of the logonIDs narrowed down and check their logout times. This is down by opening a new terminal and searching the file for each logonID.

After several attempts, most users had logged on roughly 25 minutes before the incident, and logged off quickly (within 5 minutes), meaning they couldn't have been the culprit. However, one ID (*0x37E2E1*) hadn't logged off for 3 whole hours after logging in (Login: *11:47:50*, Logout: *14:45:25*). This proves that they were online during the incident and matched the IP from the proxy log, making them our culprit.

Onto the next Task!

Author: kscanlon3 on Github
