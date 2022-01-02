---
layout: post
description: Bruteforce a websites login with Hydra, identify and use a public exploit then escalate your privileges on this Windows machine!
title: THM Hackpark CTF Writeup
date: 2021-12-31 18:20
image: '/assets/img/posts/tryhackme-hackpark-ctf-writeup/parkbanner.jpg'
tags: [TryHackme-Machines,ctf,pentesting]
featured: false
---

# ENUMERATION

Run Network Mapper (**nmap**) on Hackpark machine to discover opened ports and services.   

Full command : 
{% highlight bash %}
nmap -sV -sS -O -T4 -p- {machine IP} -Pn -oN {outputfile}
{% endhighlight %}

| Parameter              | Functionality                                          | 
|:-----------------------|:-------------------------------------------------------|
|-sV                     | Probe open ports to determine service/version info     |
|-sS                     | SYN, half TCP scan                                     |
|-O                      | Enable OS detection                                    |
|-T4                     | T{0-5} Set scan speed, higher is faster                |
|-p-                     | Scan all 65536 ports                                   |
|-Pn                     | Skip host discovery                                    |
|-oN                     | Write output to a file                                 |

Output of the scan:
{% highlight bash %}
# Nmap 7.92 scan initiated Thu Dec 30 14:40:55 2021 as: nmap -sV -sS -O -T4 -p- -Pn -oN hackpark.nmap 10.10.123.142
Nmap scan report for 10.10.123.142
Host is up (0.033s latency).
Not shown: 65533 filtered tcp ports (no-response)
PORT     STATE SERVICE            VERSION
80/tcp   open  http               Microsoft IIS httpd 8.5
3389/tcp open  ssl/ms-wbt-server?
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2012 (89%)
OS CPE: cpe:/o:microsoft:windows_server_2012
Aggressive OS guesses: Microsoft Windows Server 2012 (89%), Microsoft Windows Server 2012 or Windows Server 2012 R2 (89%), Microsoft Windows Server 2012 R2 (89%)
No exact OS matches for host (test conditions non-ideal).
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Thu Dec 30 14:43:54 2021 -- 1 IP address (1 host up) scanned in 179.18 seconds
{% endhighlight %}
Two ports opened:

## Port 3389 ssl/ms-wbt-server?
Port 3389 is running an [RDP](https://docs.microsoft.com/en-us/troubleshoot/windows-server/remote/understanding-remote-desktop-protocol) service.
However, it is not possible to exploit it since we do not have any information about the version of the service.

## Port 80 http Microsoft IIS httpd 8.5
[**Microsoft IIS**](https://en.wikipedia.org/wiki/Internet_Information_Services) web server is running on port 80.
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/1.png)
<br><br>Let us quickly check the [robots.txt](https://www.cloudflare.com/learning/bots/what-is-robots.txt/) to see if we can find something interesting.
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/3.png)
Nothing fancy here. Let us move on.
<br><br>[Dirsearch](https://github.com/maurosoria/dirsearch) can help us to enumerate subdirectories of the webserver.
Frameworks like **GoBuster** and **DirBuster** are also an option.
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/2.png)
The **/admin** directory might be interesting. However, the request to the directory gets redirected to a **.aspx** file 
which is probably a login form.
<br><br>![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/4.png)
We will have to launch a [brute-force](https://www.fortinet.com/resources/cyberglossary/brute-force-attack) attack in order to
gain access to the backend panel.
<br><br>There is only one blog post published on the main page, and the author of it is the **ADMINISTRATOR**. 
So we can assume that the only user registered to the panel is the admin’s itself.
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/5.png)

# EXPLOITATON

## Brute-Forcing the Panel
In order to create continuous HTTP requests, we have to know how the HTTP request looks.
Launch the [**Burp Suite**](https://www.geeksforgeeks.org/what-is-burp-suite/) to examine the HTTP POST request sent 
when a login attempt occurs.
