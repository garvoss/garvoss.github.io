---
layout: post
title:  "Natas 11: Cracking Weak Encryption"
date:   2017-07-19 14:15:50 -0400
categories: master
---

Natas is a series of wargames at overthewire.org that demonstrates server-side web vulnerabilities. Natas 11 in particular uses weak encryption comprising of base64 and XOR-encrpytion with a weak key, which can easily be decoded.

The source code:

```
$defaultdata = array( "showpassword"=>"no", "bgcolor"=>"#ffffff");

function xor_encrypt($in) {
    $key = '<censored>';
    $text = $in;
    $outText = '';

    // Iterate through each character
    for($i=0;$i<strlen($text);$i++) {
    $outText .= $text[$i] ^ $key[$i % strlen($key)];
    }

    return $outText;
}

function loadData($def) {
    global $_COOKIE;
    $mydata = $def;
    if(array_key_exists("data", $_COOKIE)) {
    $tempdata = json_decode(xor_encrypt(base64_decode($_COOKIE["data"])), true);
    if(is_array($tempdata) && array_key_exists("showpassword", $tempdata) && array_key_exists("bgcolor", $tempdata)) {
        if (preg_match('/^#(?:[a-f\d]{6})$/i', $tempdata['bgcolor'])) {
        $mydata['showpassword'] = $tempdata['showpassword'];
        $mydata['bgcolor'] = $tempdata['bgcolor'];
        }
    }
    }
    return $mydata;
}

function saveData($d) {
    setcookie("data", base64_encode(xor_encrypt(json_encode($d))));
}

$data = loadData($defaultdata);

if(array_key_exists("bgcolor",$_REQUEST)) {
    if (preg_match('/^#(?:[a-f\d]{6})$/i', $_REQUEST['bgcolor'])) {
        $data['bgcolor'] = $_REQUEST['bgcolor'];
    }
}

saveData($data);



?>

<h1>natas11</h1>
<div id="content">
<body style="background: <?=$data['bgcolor']?>;">
Cookies are protected with XOR encryption<br/><br/>

<?
if($data["showpassword"] == "yes") {
    print "The password for natas12 is <censored><br>";
}

?>

<form>
Background color: <input name=bgcolor value="<?=$data['bgcolor']?>">
<input type=submit value="Set color">
</form>

<div id="viewsource"><a href="index-source.html">View sourcecode</a></div>
</div>
</body>
</html>
```

On the surface, the code changes the color of the background page given appropriate user input. In the background, it creates an ecrypted cookie called `data`, which contains an array with the variables `showpassword` and `bgcolor`. 

The user can modify `bgcolor` via the form on the webpage, but there is no direct way to modify the `showpassword` variable. To do that, we'll need to modify the encrypted cookie itself, which in return involves decrypting the cookie first.

The following shows how the cookie is encrypted:
```
function saveData($d) {
    setcookie("data", base64_encode(xor_encrypt(json_encode($d))));
}
```

There are plenty of base64 decoders on the web, so that will not be a problem. XOR encryption, however requires a key that has been censored from the source code, and will need to be cracked. 

As a trial, let's set the background color to #000000 and grab the cookie. The Cookies Manager+ for firefox makes this trivial.

The `data` cookie contains `ClVLIh4ASCsCBE8lAxMacFMZV2hdVVotEhhUJQNVAmhSRwh6QUcIaAw%3D`

After replacing the `%3D` at the end with `=`, we can decode this. Any online base64 decoder should do the trick. I've chosen to use https://www.base64decode.org/

Decoding the string will give us a binary string that we are prompted to download. We can view a hexdump of the file using the program `xxd`:
```
root@kali:~/Downloads# cat decoded.bin | xxd
00000000: 0a55 4b22 1e00 482b 0204 4f25 0313 1a70  .UK"..H+..O%...p
00000010: 5319 5768 5d55 5a2d 1218 5425 0355 0268  S.Wh]UZ-..T%.U.h
00000020: 5247 087a 4147 0868 0c                   RG.zAG.h.
```
We can't decrypt this any further as we don't have the key used in the XOR-encryption. Let's instead change the background color to `#111111`, grab that cookie, and analyze the differences.

The `data` cookie now contains `ClVLIh4ASCsCBE8lAxMacFMZV2hdVVotEhhUJQNVAmhSRgl7QEYJaAw%3D`

Decrypting this gives us:
```
root@kali:~/Downloads# cat decoded.bin1 | xxd
00000000: 0a55 4b22 1e00 482b 0204 4f25 0313 1a70  .UK"..H+..O%...p
00000010: 5319 5768 5d55 5a2d 1218 5425 0355 0268  S.Wh]UZ-..T%.U.h
00000020: 52[46 097b 4046 09]68 0c                 RF.{@F.h.
```

We see that the cookies are identical except for 6 bytes in the last line.

`47 087a 4147 08` is stored when bgcolor is `#000000`, whereas `46 097b 4046 09` is stored when `bgcolor` is set to `#111111`.

Let's compare our original value of "000000" to it's post XOR-ecrypted value:
```
30 30 30 30 30 30
47 08 7a 41 47 08
```
30 is the hex value of the ascii value "0". We see that 47 and 08 get repeated, which means that the key is only 4 bytes long. We have all that we need to decrypt the key.

First let's convert the hex values to their binary form. As the last two bytes are the same as the first two, we'll drop the last two bytes.
```
[30]		[30]		[30]		[30]
0011 0000       0011 0000       0011 0000       0011 0000 [original value]

0100 0111       0000 1000       0111 1010       0100 0001 [result after XOR]
[47]		[08]		[7a]		[41]
```

The way XOR-encryption (XOR is short for 'exclusive or') is that if the first and second bits are the same, the result is 0. If they're different, the result is 1. 
```
0 0 1 1	[first bit]
0 1 0 1 [second bit]
0 1 1 0 [result]
```
Let's take a look at the first byte as an example. If the result ends in a 0, then the key bit is the same as the original bit. And if the result ends in a 1, then the key bit and the original bit are the same. With this knowledge, we can reconstruct the key value.   

```
[30]
0011 0000 [original input]
0111 0111 [key]

0100 0111 [result]
[47]
```

Translated to hex, one of the values of the key is 77 or 'w' in ascii. Now to decode the rest of the key:

```
[30]		[30]		[30]		[30]
0011 0000       0011 0000       0011 0000       0011 0000

0111 0111       0011 1000       0100 1010       0111 0001

0100 0111       0000 1000       0111 1010       0100 0001
[47]		[08]		[7a]		[41]

77 [w]          38 [8]          4A [J]          71 [q]
```

We have our entire key now, but we don't know which character is the first character in the sequence.

From the original binary file, we have decoded the follow:
```
0a55 4b22 1e00 482b 0204 4f25 0313 1a70
5319 5768 5d55 5a2d 1218 5425 0355 0268
52[w  8J  q]47 0868 0c
```

Each line is 16 bytes long. The key is 4  bytes long, so each line starts with the same character of the key:

```
52[w 8J q]47 0868 0c
q[w  8J q]w 8J q
```
Which means our key is `qw8J`.

We can now decrypt the XOR-layer of the encryption and get the original string. As with base64, there are various online tools for this. I've chosen http://xor.pw/

```
0a554b221e00482b02044f2503131a70
531957685d555a2d1218542503550268
5247087a414708680c
```
decrypts to:
```
7b2273686f7770617373776f7264223a
226e6f222c226267636f6c6f72223a22
23303030303030227d
```
And converting to ascii finally gives us:
```
{"showpassword":"no","bgcolor":"#000000"}
```

Now we have to change "no" to "yes", and encrypt it with the key value we decoded.

Using the site http://md5decrypt.net/en/Xor/, we can XOR-encrypt `{"showpassword":"yes","bgcolor":"#000000"}` with the key to gives us:
```
0a554b221e00482b02044f2503131a70
530e5d39535b1a28161457261e051a70
5354087a4147087a530a
```
Encoding this with a hex-to-base64 encoder at http://tomeko.net/online_tools/hex_to_base64.php?lang=en:
```
ClVLIh4ASCsCBE8lAxMacFMOXTlTWxooFhRXJh4FGnBTVAh6QUcIelMK
```

Adding %3D to the end of the string above, will give us the cookie value we need.
Using Cookies Manager+ to edit `data` cookie value to:
```
ClVLIh4ASCsCBE8lAxMacFMOXTlTWxooFhRXJh4FGnBTVAh6QUcIelMK%3D
```

Refreshing the webpage with no parameters given gives us the password to the next level.
