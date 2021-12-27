---
layout: post
title: Tryhackme Blue Ctf Writeup
Description: desc1
author: krygennn
date: 2021-10-9 00:59
image:  '/assets/img/posts/tryhackme-blue-ctf-writeup/blue1.jpg'
tags: [ctf,pentesting,metasploit,remote-code-execution,buffer-overflow,cracking]
featured: false
---
[**Solve Yourself >>**](https://www.tryhackme.com/room/blue)

Blue is a machine that specifically designed to cover EternalBlue (CVE-2017-0144), 
which allows remote attackers to execute arbitrary code via crafted packets, 
aka "Windows SMB Remote Code Execution Vulnerability." 
<br>
<br>
## 1-Enumeration 

Let us run Network Mapper (**nmap**) to discover opened ports and services.   

| Parameter              | Functionality                                          | 
|:-----------------------|:-------------------------------------------------------|
|-sV                     | Probe open ports to determine service/version info     |
|-sC or --script=default | Performs a script scan using the default set of scripts|
|-O                      | Enable OS detection                                    |
|-T4                     | T{0-5} Set scan speed, higher is faster                |
|-p-                     | Scan all 65536 ports                                   |

Full command : 
{% highlight bash %}
nmap <machine IP> -sV -sC -O -T4 -p-
{% endhighlight %}
![Desktop View](/assets/img/posts/tryhackme-blue-ctf-writeup/blue2.jpg){:style="max-width: 80%" .normal} 
<br>
It seems like plenty of services are enabled in the system.However one of them catches the eye.

_**445 /tcp open microsoft-ds Windows 7 Professional 7601 Service Pack 1 microsoft-ds (workgroup : WORKGROUP)**_ 

After went through the web searching phase for that specific service, we got this:

[**CVE-2017-0144 - EternalBlue SMB Remote Code Execution (MS17-010)**](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0144)
<br>

## 2-Exploitation 

Let's use **Metasploit** framework
<br>
![Desktop View](/assets/img/posts/tryhackme-blue-ctf-writeup/blue3.jpg){:.normal}
<br>
And than use **MS17-010**
![Desktop View](/assets/img/posts/tryhackme-blue-ctf-writeup/blue4.jpg){:.normal}
<br>

Six modules are showed up, two of them are auxiliary modules and the rest are exploiting scripts.In a real-life scenario it is not that obvious that those pre-arranged scripts would work.It is highly possible that target system would crash if you ran any careless exploit to target system or you are not sure host is vulnerable to it. To check this we can run **auxiliary module**.

If you are not familiar with what an auxiliary module is, it is some kind of script that helps us to verify if the target system is vulnerable to our actual exploit or not, without crashing or corrupting the system.
More technical description would be as following; An auxiliary module does not execute a payload. It can be used to perform arbitrary actions that may not be directly related to exploitation. Examples of auxiliary modules include scanners, fuzzers, and denial of service attacks. 
 
In this case Metasploit database offers us those two auxiliary files, run:
{% highlight bash %}
use auxiliary/scanner/smb/smb_ms17_010
{% endhighlight %}
Than set RHOSTS  as IP address of  our target machine, and run it.
![Desktop View](/assets/img/posts/tryhackme-blue-ctf-writeup/blue5.jpg){:.normal}
<br>
As you can see, the output tells us the that, **Host is likely VULNERABLE to MS17-010 !!!** 
After that point we got our proof that the exploit would work.So let us move to the exploitation phase.
 
_Exploit - An exploit module executes a sequence of commands to target a specific vulnerability found in a system or application. An exploit module takes advantage of a vulnerability to provide access to the target system. Exploit modules include buffer overflow, code injection, and web application exploit._
 
Run:
{% highlight bash %}
use exploit/windows/smb/ms17_010_eternalblue
{% endhighlight %}
Like the previous auxiliary module, set RHOSTS as our target machine and run it.
![Desktop View](/assets/img/posts/tryhackme-blue-ctf-writeup/blue6.jpg){:.normal}
<br>
![Desktop View](/assets/img/posts/tryhackme-blue-ctf-writeup/blue7.jpg){:.normal}
<br>
Congrats !!! You've hacked into the machine by using MS17-010 vulnerability.Now we have got our Meterpreter shell which provides additional powerful tools. 
 
Now we need to use **getsystem** command,which is provided by Metasploit's meterpreter shell, that will use number of different techniques in attempt to gain SYSTEM level priveleges on the remote target. 
![Desktop View](/assets/img/posts/tryhackme-blue-ctf-writeup/blue8.jpg){:.normal}
<br>
Now, lets list all runnig processes with **ps** command, we will need to migrate our suspicious process into trusted process to ensure consistency.
![Desktop View](/assets/img/posts/tryhackme-blue-ctf-writeup/blue9.jpg){:.normal}
<br>
SearchIndexer.exe seems like a good candidate to migrate our process.So type migrate {PID} to migrate spoolsv.exe, which is our reverse tcp payload.

Right now we have %100 access to target system and our session's persistance is solid, lets move on for crediantels and flags.
<br>

## 3-Post Exploitation - Capturing the Flags 
Just like the getsystem command, meterpreter provides **hashdump** command, which brings the system hashes for us.
![Desktop View](/assets/img/posts/tryhackme-blue-ctf-writeup/blue10.jpg){:.normal}
<br>
We need to find out what kind of hashes they are. It is easy to tell they are NTLM hashes, because Windows is using NTLM hashes for system as default but for the best practice, lets use a hash analyzer to identify the type, you can use any hash analyzer.
![Desktop View](/assets/img/posts/tryhackme-blue-ctf-writeup/blue11.jpg){:.normal}
<br>
As you can see, it is **NTLM**.

So, as we know the hash type, we can move forward for to cracking it.I will use **John the Ripper**,you can use **Hashcat** or any other cypher cracking tool.
![Desktop View](/assets/img/posts/tryhackme-blue-ctf-writeup/blue12.jpg){:.normal}
<br>

| Parameter                    | Functionality                                 | 
|:-----------------------------|:----------------------------------------------|
|hashes                        | ASCII Text file, which contains copied hashes.|
|--wordlist                    | Location of the wordlist                      |
|-format                       | Hash format                                   |

We've cracked the first hash.

Now type shell to use windows' shell instead of meterpreter shell and start looking for flags.
#### flag1.txt
![Desktop View](/assets/img/posts/tryhackme-blue-ctf-writeup/blue13.jpg){:.normal}
<br>
#### flag2.txt
![Desktop View](/assets/img/posts/tryhackme-blue-ctf-writeup/blue14.jpg){:.normal}
<br>
#### flag3.txt
![Desktop View](/assets/img/posts/tryhackme-blue-ctf-writeup/blue15.jpg){:.normal}
<br>
Congrats! End of the machine.


