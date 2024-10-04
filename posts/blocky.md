![](http://vedanttare.com/wp-content/uploads/2022/04/Screenshot-2022-09-25-at-11.57.51-AM.png)

IP: 10.10.10.37

## Enumeration

As we start off with a basic nmap scan: `sudo nmap -sC -sV -O -p- -oA nmap/block 10.10.10.37`, we get,

Ports open:

- 21/ftp
- 22/ssh
- 80/http

To access ftp, credentials were needed, so didn’t poke at that. Moved on to port 80 and the nmap scan showed that wordpress was running on it.

Default page when visiting the IP:

![](http://vedanttare.com/wp-content/uploads/2022/04/default.png)

Running dirb I found out that phpmyadmin was also installed as well as wordpress. Then, wordpress need creds as well as phpmyadmin. So, the only way was to somehow find creds and login it to either of the 2 softwares.

So, used wpscan also (wpscan –url –enumerate vp,p,tt,t), but didn’t find anything interesting.

After googline, the ftp version had a rce, and after trying the exploit it wasn’t working. There was a 10.10.10.37/plugins directory which dirb had found.

![](http://vedanttare.com/wp-content/uploads/2022/04/plugins-dir.png)

It had 2 .jar files which are basically compiled java files. After downloading them, and unzipping them. .jar files are basically zip files. Then unzipping a .jar file and then using jad a java decompiler to decompile .class file which is basically compiled java file. Then, checking it out, it had creds!!!!!

![](http://vedanttare.com/wp-content/uploads/2022/04/creds-found.png)

We tried these in phpmyadmin and go logged in!!!! After getting logged in we saw that there was a wordpress database which had wp\_users table. Then, checking that out we found a hashed password and a username.

![](http://vedanttare.com/wp-content/uploads/2022/04/phpmyadmin.png)

`notch@blockcraftfake.com`

We had got a username as: notch. And a hashed password: `$P$BiVoTj899ItS1EZnMhqeqVbrZI4Oq0/`

We just simply tried to ssh using this hash but we were unsuccessful. Then, we tried to use the previous creds from the .class file with the obtained username as: `notch`

AND BOOM!!!! We got a low privilege shell!!!!!

![](http://vedanttare.com/wp-content/uploads/2022/04/got-user-shell.png)

## Privilege Escalation

Privilege escalation was a piece of cake!!!!  
We used sudo -l to list the privileges and it asked us for the sudo password for the user notch. Then we entered the password obtained from the .class file and we got a list:

`notch@Blocky:~$ sudo -l<br></br>[sudo] password for notch:<br></br>Matching Defaults entries for notch on Blocky:<br></br>env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin`

`User notch may run the following commands on Blocky:<br></br>(ALL : ALL) ALL`

![](http://vedanttare.com/wp-content/uploads/2022/04/sudo-l.png)

As we could see that we could run any commands as user notch. Hence, we just used sudo su to switch to the root user and BOOM!!!! We got a root shell!!!!!

`notch@Blocky:~$ sudo su<br></br>root@Blocky:/home/notch# id<br></br>uid=0(root) gid=0(root) groups=0(root)<br></br>root@Blocky:/home/notch# ls<br></br>minecraft user.txt<br></br>root@Blocky:/home/notch# pwd<br></br>/home/notch<br></br>root@Blocky:/home/notch# cd /root<br></br>root@Blocky:~# ls<br></br>root.txt<br></br>root@Blocky:~# ls -la<br></br>total 28<br></br>drwx------ 3 root root 4096 Jul 14 2017 .<br></br>drwxr-xr-x 23 root root 4096 Jul 2 2017 ..<br></br>-rw------- 1 root root 1 Dec 24 2017 .bash_history<br></br>-rw-r--r-- 1 root root 3106 Oct 22 2015 .bashrc<br></br>drwxr-xr-x 2 root root 4096 Jul 2 2017 .nano<br></br>-rw-r--r-- 1 root root 148 Aug 17 2015 .profile<br></br>-r-------- 1 root root 32 Jul 2 2017 root.txt<br></br>root@Blocky:~# cat root.txt<br></br>0a9694a5b4d272c694679f7860f1cd5froot@Blocky:~#`

Now, we grabbed the root flag:

![](http://vedanttare.com/wp-content/uploads/2022/04/flag-root.png)

Important links or resources:

Check out Ippsec’s video on youtube about it -&gt; https://www.youtube.com/watch?v=C2O-rilXA6I

Check out my previous post:

<figure class="wp-block-embed is-type-wp-embed is-provider-vedant-tare wp-block-embed-vedant-tare"><div class="wp-block-embed__wrapper">> [Hackthebox – Tabby](https://vedanttare.com/hackthebox-tabby/)

<iframe class="wp-embedded-content" data-secret="rMH5cmx0P2" frameborder="0" height="338" marginheight="0" marginwidth="0" sandbox="allow-scripts" scrolling="no" security="restricted" src="https://vedanttare.com/hackthebox-tabby/embed/#?secret=UsRGnup0p2#?secret=rMH5cmx0P2" style="position: absolute; clip: rect(1px, 1px, 1px, 1px);" title="“Hackthebox – Tabby” — VEDANT TARE" width="600"></iframe></div></figure>
