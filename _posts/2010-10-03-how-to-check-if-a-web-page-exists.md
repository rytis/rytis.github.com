---
layout: post
title: "How to check if a web page exists"
category: programming
tags: [httplib, python, web]
---
{% include JB/setup %}

I've come across a question on [Python-Forum.org](http://www.python-forum.org/pythonforum/viewtopic.php?f=5&p=94135) where someone was asking how to check whether a web page exists from a Python script. Something similar to what the `wget --spider` option does. Interestingly enough, the question stood there unanswered for nearly two months.

The solution is very simple and trivial however. Let’s have a look at what exactly the --spider option is supposed to do. This is what the wget manual page says:


> When invoked with this option, Wget will behave as a Web spider, which means that it will not download the pages, just check that they are there. For example, you can use Wget to check your bookmarks:
>  
> `wget --spider --force-html -i bookmarks.html`
>  
> This feature needs much more work for Wget to get close to the functionality of real web spiders.


So, all we have to do is use the HTTP `HEAD` method when we request the page. This method behaves just like the `GET` method, but instructs the remote server not to send the contents of the requested web page. The remaining information, such as the web page headers and the web server status code, is returned intact.

This is quite easy to do with a simple httplib call:

{% highlight python %}

>>> import httplib
>>> conn = httplib.HTTPConnection('www.w3.org')
>>> conn.request('HEAD', '/Protocols/rfc2616/rfc2616-sec9.html')
>>> resp = conn.getresponse()
>>> data = resp.read()
>>>
>>> # as you can see, there is no data, this is because we've used HEAD method
>>> data
''
>>>
>>> # you can get the info data about the document from the HTTP headers:
>>> for h in resp.getheaders():
...  print h
...
('content-length', '16599')
('accept-ranges', 'bytes')
('expires', 'Sun, 19 Sep 2010 15:32:32 GMT')
('server', 'Apache/2')
('last-modified', 'Wed, 01 Sep 2004 13:24:52 GMT')
('connection', 'close')
('etag', '"40d7-3e3073913b100"')
('cache-control', 'max-age=21600')
('date', 'Sun, 19 Sep 2010 09:32:32 GMT')
('p3p', 'policyref="http://www.w3.org/2001/05/P3P/p3p.xml"')
('content-type', 'text/html; charset=iso-8859-1')
>>>
>>> # and finally, check the response code, that'll give you an idea
>>> # whether the document actually exists
>>> resp.status
200
>>>

{% endhighlight %}


As you can see there is a fair amount of useful information returned. Apart from the obvious "the page exists" / "the page doesn't exist" (or HTTP status `200` and `404` respectively), you can figure out the size of the requested web page, its last modification date, etc.

This method is useful if you need to check whether the page has changed since you last checked. So you don't have to download the whole thing every time, just to discover that there were no changes.

