---
layout: page
title: About
permalink: /NSA-Codebreaker-2021/task1.md
---

Author: AwesomArcher8093 on Github

### First Insights

For Task 1 of the 2021 NSA Codebreaker Challenge, our primary objective is to prevent threats to the US Defense Industrial Base. 
Based on the agreements with other DIBs we need to determine if these companies are directly accessing the infrastructure. 

We have been given capture data to the listening post. This is represented with a .pcap extention. We are also given a list of company IPs 
that are assicuated with any DIBs that have been communicating with the listening post as signified by the .txt file

### Starting the Task

To begin with we are given a text file with a list of IP Ranges in the *ip_ranges.txt* file:

```
10.255.96.0/23
10.243.131.48/28
172.21.169.80/29
192.168.154.0/23
198.18.136.128/29
10.128.0.0/15
10.200.104.0/21
198.18.135.136/29
172.21.103.128/26
172.26.243.96/27
198.19.147.48/29
198.18.136.0/27
10.160.128.0/17
172.19.8.0/21
198.19.216.192/26
172.21.169.216/29
10.228.20.0/23
172.27.68.0/23
10.203.220.0/22
198.18.217.0/28
10.243.130.0/27
10.243.153.0/24
198.18.142.8/29
198.19.218.144/28
172.18.198.0/23
10.255.30.0/24
198.19.176.0/21
```
By itself these ranges have no value to us so we will need to set it aside. Next we come across a .pcap file more specifically *capture.pcap*.

To analyze the various different sources and the network traffic, we will have to open up the file in WireShark. The file within Wireshark
shows source and destination IP addresses, Protocol, and any additional info. 

### Identify IP Addresses

Now based on the ip ranges that were found in *ip_ranges.txt*, we will have to search for which specific IPs that have communicated with the LP. We can use the folllowing command to make this easier:

```
ip_addr== 'ip address'
```
Now we will have to manually enter each ip range. After entering approximately 27 ip ranges, 4-5 ip addresses should've appeared under the filter:

```
198.18.136.129
10.228.20.40
172.27.69.32
172.18.198.11
```
With this information in hand, task 1 is complete!
Onto task 2!
