---
id: 259
title: 'Hackthebox &#8211; Haircut'
date: '2020-02-25T05:01:15+00:00'
author: vedanttare
excerpt: 'In this post, we are going to be pwning "Haircut" from Hackthebox.'
layout: post
guid: 'https://codesupply.co/vici-consequat-justo-enim-adipiscing-luctus-nullam-fringilla-pretium/'
permalink: /hackthebox-haircut/
csco_singular_sidebar:
    - right
csco_page_header_type:
    - default
csco_page_load_nextpost:
    - default
csco_layout_in_loop:
    - full
csco_post_video_location:
    - 'a:0:{}'
csco_post_video_url:
    - ''
csco_post_video_bg_start_time:
    - '0'
csco_post_video_bg_end_time:
    - '0'
csco_post_text_color:
    - '#ffffff'
csco_post_video_location_hash:
    - ''
powerkit_share_buttons_transient_facebook:
    - '1692134029'
powerkit_share_buttons_transient_pinterest:
    - '1727892097'
image: 'http://vedanttare.com/wp-content/uploads/2020/02/Screenshot-2022-09-25-at-2.33.11-PM-496x166.png'
categories:
    - Writeups
---

![](http://vedanttare.com/wp-content/uploads/2020/02/Screenshot-2022-09-25-at-2.32.24-PM.png)

Three phases:

- Enumeration
- Exploitation
- Privilege Escalation

### Enumeration

Doing a simple nmap scan using command:

`nmap -sC -sV -O -oA nmap/haircut 10.10.10.24`

Ports open:

- 22/ssh
- 80/http

Checking for the SSH version to see if it was vulnerable, we couldn’t get anything.

Enumerating the default page on port 80, and checking the source code for something interesting, we didn’t find anything there either.

![](http://vedanttare.com/wp-content/uploads/2020/02/defaulthaircut.png)

Using dirb to enumerate and fuzz for directories: dirb -u http://10.10.10.24/ -w /usr/share/wordlists/dirb/common.txt -x .txt,.php

### Exploitation

We found an interesting page **/exposed.php**. This page has a functionality which lets us call any machine given its IP address and any other page on hosted on the machine. We can use this to our advantage maybe!

![](http://vedanttare.com/wp-content/uploads/2020/02/haircutexposed.png)

It can be seen that there was a curl request being made. Hence, we could use curl’s -o option to output a php reverse shell on the victim machine!

After setting up a listener, we get a user shell!

`listening on [any] 9001 …<br></br>connect to [10.10.14.10] from (UNKNOWN) [10.10.10.24] 53864<br></br>bash: cannot set terminal process group (1249): Inappropriate ioctl for device<br></br>bash: no job control in this shell<br></br>www-data@haircut:~/html/uploads$ id<br></br>id<br></br>uid=33(www-data) gid=33(www-data) groups=33(www-data)`

### Privilege Escalation

After running LinEnum.sh, we could see that SUID bit was enabled for a unusual binary which was /usr/bin/screen-4.5.0:

`--- -rwsr-xr-x 1 root root 1588648 May 19 2017 /usr/bin/screen-4.5.0`

Googling the binary, we find that there is a local privesc exploit available. Hence, we followed the instructions and made libhax.c libhax.so rootshell and rootshell.c and setup a web server and transferred the file over to the victim machine.

Running the commands as given in the exploit, we get a ROOT shell!!!

`cd /root`

ls  
`root.txt`

`cat root.txt<br></br>4c...........................`

Check out my previous post:

<figure class="wp-block-embed is-type-wp-embed is-provider-vedant-tare wp-block-embed-vedant-tare"><div class="wp-block-embed__wrapper">> [Hackthebox – Lame](https://vedanttare.com/hackthebox-lame/)

<iframe class="wp-embedded-content" data-secret="gzJC039BOt" frameborder="0" height="338" marginheight="0" marginwidth="0" sandbox="allow-scripts" scrolling="no" security="restricted" src="https://vedanttare.com/hackthebox-lame/embed/#?secret=5ELYX8u3JC#?secret=gzJC039BOt" style="position: absolute; clip: rect(1px, 1px, 1px, 1px);" title="“Hackthebox – Lame” — VEDANT TARE" width="600"></iframe></div></figure>