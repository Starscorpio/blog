![](http://vedanttare.com/wp-content/uploads/2022/04/Screenshot-2022-09-25-at-12.08.09-PM.png)

IP: 10.10.10.105

Enumeration

We start off with a simple nmap scan: sudo nmap -sC -sV -O -p- -oA nmap/jerry 10.10.10.105, we get,

Ports open:

- 8080

After navigating to https://10.10.10.105:8080, we can see that Apache tomcat server is running. The credentials to Apache tomcat manager app is leaked in an error message.

`tomcat:s3cret`

![](http://vedanttare.com/wp-content/uploads/2022/04/creds-leaked.png)

Now we can easily login and upload a war reverse shell file and get a reverse shell!

![](http://vedanttare.com/wp-content/uploads/2022/04/war-file-upload.png)

Generating a war reverse shell payload:

![](http://vedanttare.com/wp-content/uploads/2022/04/generated-war-payload.png)

As we can see that our reverse shell war file has been uploaded:

![](http://vedanttare.com/wp-content/uploads/2022/04/war-file-uploaded.png)

Reverse shell was a high privileged one direct root!!!

![](http://vedanttare.com/wp-content/uploads/2022/04/got-root-shell.png)

No privesc needed!!!!

![](http://vedanttare.com/wp-content/uploads/2022/04/root-and-user-flags.png)

Check out my previous post:

<figure class="wp-block-embed is-type-wp-embed is-provider-vedant-tare wp-block-embed-vedant-tare"><div class="wp-block-embed__wrapper">> [Hackthebox – Granny](https://vedanttare.com/hackthebox-granny/)

<iframe class="wp-embedded-content" data-secret="ftU1R4j9eK" frameborder="0" height="338" marginheight="0" marginwidth="0" sandbox="allow-scripts" scrolling="no" security="restricted" src="https://vedanttare.com/hackthebox-granny/embed/#?secret=j3pE8Y98Mx#?secret=ftU1R4j9eK" style="position: absolute; clip: rect(1px, 1px, 1px, 1px);" title="“Hackthebox – Granny” — VEDANT TARE" width="600"></iframe></div></figure>
