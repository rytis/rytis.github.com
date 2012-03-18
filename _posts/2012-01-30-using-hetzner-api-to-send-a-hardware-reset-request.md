---
layout: post
title: "Using Hetzner API to send a hardware reset request"
category: System administration
tags: [api, hetzner, AppleScript]
---
{% include JB/setup %}

[Hetzner](http://www.hetzner.de/) is a great hosting company. It's hard to find a deal that would beat their hosting prices, and the service is superb. In 7 years I've experienced only one brief network outage, and one power outage (when they changed power outlets or something like that), which was clearly communicated well in advance. So I can only highly recommend them - my experience was only positive one.

They also have a nice administration interface, where you can manage your servers, request IP ranges, control bandwidth monitoring, and request server restarts.

Although it shouldn't happen very often, you may find that every now and then your server becomes unresponsive - due to misconfiguration, or an application becomes too greedy, and clings on too many resources, thus rendering the whole system unusable.

Resetting a server from Hetzner's management console couldn't be easier, but they made this task even more simple by exposing some of the management functionality via a [simple to use API](http://wiki.hetzner.de/index.php/Robot_Webservice/en). There are already [libraries available](https://github.com/rmoriz/hetzner-api), but it is simple enough so you can use curl to perform most of the tasks.

Although this blog is mostly about Python, but I'll throw in some variety and share a simple AppleScript snippet, which you can use to reset your servers hosted with Hetzner:

{% highlight applescript %}
set dd to display dialog "Reset server?"
 
set result to do shell script "curl -u [username]:[password] https://robot-ws.your-server.de/reset/[server IP] -d type=hw"
 
display dialog "Query result: " & result buttons "OK" default button "OK"
{% endhighlight %}
