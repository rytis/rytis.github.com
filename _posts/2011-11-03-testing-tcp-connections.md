---
layout: post
title: "Testing TCP connections"
category: sysadmin
tags: [python, networking]
---
{% include JB/setup %}

Here's a quick snippet of python code that attempts to establish a TCP connection and reports whether it was successful or not. Netcat is a lot better suited for this role, but sometimes you may not be able to install it. In these situations this little script can come in handy:

{% highlight python %}

#!/usr/bin/env python
 
import socket
import time
 
HOST=""
PORT=""
TIMEOUT=1.0
 
while True:
    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.settimeout(TIMEOUT)
        s.connect((HOST, PORT))
        print "[%s] Connection established" % time.strftime("%H:%M:%S")
        time.sleep(1)
        s.close()
    except:
        print "[%s] Cannot connect" % time.strftime("%H:%M:%S")

{% endhighlight%}
