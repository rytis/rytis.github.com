---
layout: post
title: "Quick way to check for URL redirects"
category: sysadmin
tags: [web, tips]
---
{% include JB/setup %}

Recently I had to go through a bunch of URLs and identify whether or not they redirect to another URL. Since the list was rather long (several hundred entries), I thought I should rather write a script to do this for me.

I used the [Requests](http://docs.python-requests.org/en/latest/) library to make the initial request for a given URL. If the response code was `3XX`, then I assumed it is being redirected. There are a couple of other edge cases (timeouts and generic connection errors, either due to non-existent domain, or network error) that the script attempts to cover for.

The script accepts a single parameter, which should be a filename containing URL strings. If the filename is not provided, the script will look for `domains.txt` in the current directory.

{% highlight python %}
#!/usr/bin/env python

import sys
import requests


def check_for_redirects(url):
    try:
        r = requests.get(url, allow_redirects=False, timeout=0.5)
        if 300 <= r.status_code < 400:
            return r.headers['location']
        else:
            return '[no redirect]'
    except requests.exceptions.Timeout:
        return '[timeout]'
    except requests.exceptions.ConnectionError:
        return '[connection error]'


def check_domains(urls):
    for url in urls:
        url_to_check = url if url.startswith('http') else "http://%s" % url
        redirect_url = check_for_redirects(url_to_check)
        print("%s => %s" % (url_to_check, redirect_url))


if __name__ == '__main__':
    fname = 'domains.txt'
    try:
        fname = sys.argv[1]
    except IndexError:
        pass
    urls = (l.strip() for l in open(fname).readlines())
    check_domains(urls)
{% endhighlight %}

Below is an example of URLs. If protocol is not defined, `http` will be assumed.

{% highlight python %}
google.com
http://gmail.com
www.ibm.com
redhat.com
example.com
i-believe-this-domain-does-not-exist-123abc.com
{% endhighlight %}

And the results:

{% highlight python %}
http://google.com => http://www.google.com/
http://gmail.com => http://mail.google.com/mail/
http://www.ibm.com => http://www.ibm.com/us/en/
http://redhat.com => http://www.redhat.com/
http://example.com => http://www.iana.org/domains/example/
http://i-believe-this-domain-does-not-exist-123abc.com => [connection error]
{% endhighlight %}

