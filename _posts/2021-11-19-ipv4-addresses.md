---
layout: post
title: IPv4 Protocol
date: 2021-11-18 12:00
author: krygennn
date: 2021-11-18 12:00
image: '/assets/img/posts/ipv4-protocol/ip1.jpg'
tags: [network protocols,ip-addresses]
featured: false
---
---
image:
    src: /assets/img/posts/ipv4-protocol/ip1.jpg
    alt: SUBNETTING
    width: 800
    height: 450
---
## IPv4

IPv4 stands for Internet Protocol version 4. IPs are addresses for users on the internet. Messages and packages delivered via
IP addresses. IPv4 address spaces consists of four octets. Depending on the class and configuration, some portion of bits in the
octet represents the network ID, while the rest represents hosts.
<br><br>
An IP address consists of 4 octets, which
determines the class and configuration of a network, for example you can think of an IP address as Octet1.Octet2.Octet3.Octet4
<br><br>
There are 5 IPv4 classes that determines which portion of the network is usable for devices you want to connect and how much of
them are allowed. Each has its own usable address spaces for maximum range of devices.
### CLASS A

Class A type is for big scale network insfrastructures with large number of total hosts needed.By using the first octet for the
network ID, class A grants for 126 networks with 16,777,214 hosts per network. The first bit in the first octet is always zero, the rest of the
7 bits creates different network ID's when they are turned on.

|:-------|:-------|
|Network Bits: first octet|Host Bits:last three octets|

|:-------|:-------|:-------|:-------|
|01111111|00000000|00000000|00000000|

* Public IP Range: 1.0.0.0 to 127.0.0.0
    * Range for first octet value from 1 to 127
* Private IP Range: 10.0.0.0 10.255.255.255
* Number of networks: 126
* Number of hosts: 16,777,214
* Subnet Mask: 255.0.0.0 (8 bits) 

### CLASS B 
Class B type is for medium to large-sized networks. By using the first two octets for the network ID, class B grants for
16,382 networks with 65,534 hosts per network. The first two bits of the first octet are always 1 and 0, the rest of the 14 bits 
creates different network ID's when they are turned on.

|:-------|:-------|
|Network Bits: first two octets|Host Bits: last two octets|

|:-------|:-------|:-------|:-------|
|10111111|11111111|00000000|00000000|

* Public IP Range: 128.0.0.0 to 192.255.0.0
    * Range for first octet value from 128 to 191 
* Private IP Range: 172.16.0.0 to 172.31.255.255
* Subnet Mask: 255.255.0.0 (16 bits)
* Number of networks: 16,382
* Number of hosts: 65,534

### CLASS C
Cass C type is for small-sized local area networks (LANs). By using the first three octets for the network ID, class C grants for
2,097,150 networks with 254 hosts per network. The first three bits of the first are always 1 1 0, the rest of the 21 bits creates
different network ID's when they are turned on.


|:-------|:-------|
|Network Bits: first three octets|Host Bits: last octet|

|:-------|:-------|:-------|:-------|
|11011111|11111111|11111111|00000000|

* Public IP Range: 192.0.0.0 to 223.255.255.0
    * Range for first octet value from 192 to 223 
* Private IP Range: 192.168.0.0 to 192.168.255.255
* Special IP Range: 127.0.0.1 to 127.255.255.255
* Subnet Mask: 255.255.255.0 (24 bits)
* Number of networks: 2,097,150
* Number of hosts: 254

### CLASS D
Class D type is not for hosts and used for multicasting. In order to send a stream of data  to thousands of nodes across the 
Internet concurrently, multicasting is required. Mostly used for video, audio streaming and broadcast any kind of altered global
data to the institutions or organizations, for example currency rate information.

* Range 224.0.0.0 to 230.255.255.255
    * First octet value range from 224 to 239
* Number of Networks: NULL
* Number of Hosts per Network: Network Multicasting

### CLASS E

Class E type is reserved addresses for research purposes (NASA etc.), and not available for regular users.

* Range 240.0.0.0 255.255.255.255
    * First octet value range from 244 to 255
* Number of Networks: NULL
* Number of Hosts per Network: Experimental/Reserved/Research

## PRIVATE IP ADDRESSES

Within every and each network class, there are specified IP addresses that are specifically reserved for internal/private use 
only. These IP addresses cannot use internet as they are not routeable and ensure internal communication. For instance, 
FTP and web servers have to use public IP addresses instead of private. However, within a business or a home network, 
private IP addresses leased to the devices.  

* **Class A Private Range**: from 10.0.0.0 to 10.255.255.255 
Class B APIPA Private Range: from 169.254.0.0 to 169.254.255.255
    * In general, a Dynamic Host Configuration Protocol (DHCP) server is used to lease IP addresses to any woken device within 
the network. However, if there is not one, or the device is not configured properly for DHCP offer, Automatic Private IP Addressing
(APIPA) is used to assign a private IP address to the device in question **if it is a Microsoft-based device**.

* **Class B Private Range**: from 172.16.0.0 172.31.255.255
* **Class C Private Range**: from 192.168.0.0 to 192.168.255.255

## SPECIAL IP ADDRESSES

IP Range from 127.0.0.1 to 172.255.255.255 are used for network testing, which also referred as loop-back addresses. These are
virtual IP addresses and can not be assigned to a device as a logical address. 
<br><br>
The most common one is **127.0.0.1**
, which is repeatedly used to troubleshoot the network connectivity issues using the ICMP protocol. If
there is a connectivity issue, before checking for network infrastructures or the other nodes, making sure of the TCP/IP stack
is correctly working on that machine must be prior check as the ICMP packet that you sent to that special IP address tests each
layer. If ping responses with a packet loss, then the connectivity issue is on the local computer, you would need to resolve
this first. If there is no packet loss, you can start to explore other possibilities that may cause failure.
