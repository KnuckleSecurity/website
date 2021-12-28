---
layout: post
title: Subnetting
date: 2021-11-18 12:00
image: '/assets/img/posts/subnetting/subnetting1.jpg'
tags: [subnetting,switch,router]
featured: false
---
# WHAT IS SUBNETTING ?

As the name suggest, subnetting means dividing a network into sub sub branches. For example, you have a class with 50 students,
if you divide them into two groups, you will have two subclasses, 25 students per class. Subetting is just the same however,
instead of class, this time it is the network which is going to be divided into **smaller chunks**.

# WHY USING SUBNETTING ?

Subnets provide efficiency for networks. Imagine you want to send a message to your dear friend Jack, who lives in the flat
next to you. Would you deliver the message to some international package delivery service, or to your local post office ?
If you were to hand out your message to international package delivery service, it would take way too long to deliver it from
you to your dear friend Jack, due to bigger scale regulations, procedures etc. Probably your message will be carried out to some
far located central warehouse to wait for dispersion, and the message you want to send to Jack will need to travel hundreds of more miles
. As you can see it is painful and costly.
<br><br>
Just like in the example, when we are talking about network, we are actually sending messages to our friends or someone else. It is 
much more efficient, time and cost-saving way to communicate without roaming through all the routers in companies'
network just to reach some node sitting next to us, a co-worker from IT department for example.
<br><br>
Instead of having one big network for a company, we can divide the departments into smaller chunks. For instance, one subnet for 
IT department and one for Human Resources department prevents interference between those two departments. It is not needed
both for them to see each other's traffic as they are completely different departments.

## Benefits of using subnets

|:-----------------------|:-------------------------------------------------------|
|Network Security and<br>Management Easiness | Subnetting facilitates network management for network administrators,<br>thus, more detailed traffic control can be implemented. It also reduces <br>the risk of spread by quarantining compromised network sections.     |
|Reduces Traffic Congestion | Subnet Mask guarantees that traffic remains within the assigned subnet.<br>This reduces extensive congestion and diminishes the load on the network.<br> In subnets, less distance needs to be travelled for any data, less likely to lose <br>data.
|Improve Speed and Performance | After sending a broadcast message, every device on the network will <br>receive and reply to that message even though the message is not relevant <br>for most of the machines.By doing that, network capacity and bandwidth <br>getting drained.However, with subnets only the relevant machines are in<br> communication.This means performance !

**Note:** Before moving any further, I highly recommend you to read [**IPv4 Protocol**](https://krygennn.github.io/posts/ipv4-addresses) blogpost first.

<br>
To be continued
