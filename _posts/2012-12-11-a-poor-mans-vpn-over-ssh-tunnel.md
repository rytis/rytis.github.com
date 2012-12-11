---
layout: post
title: "A poor man's VPN over SSH tunnel"
category: sysadmin
tags: [ssh, tips]
---
{% include JB/setup %}

Couple of scenarios:
- You are connected to a publicly available WiFi network that you do not trust, but you have a trusted server somewhere where you can connect to
- You are connected to a network that has limited access (or no access at all as it happened in my case), but you can connect to another host that can

So, what are your options?

Obviously you can set up a VPN server on the remote machine and connect to it, but it can be a bit fiddly.

An alternative is to set up a virtual private network over an ssh connection.

The instructions below explain how to do this:

On a server (any linux node that has access to the internet):

{% highlight bash %}
# enable remote root, port forwarding and tunnelling
# vi /etc/ssh/sshd_config
# service sshd restart
{% endhighlight %}

On a client (server that has no internet access, or is connected to an insecure network):

{% highlight bash %}
# open an ssh tunnel to the server, this will create tun interface
# ssh -f -w any <server ip> true

# assign an ip to it (192.168.0.1)
# ifconfig tun0 192.168.0.2 192.168.0.1 netmask 255.255.255.252
{% endhighlight %}

On the server:

{% highlight bash %}
# assign an ip (192.168.0.1)
# ifconfig tun0 192.168.0.1 192.168.0.2 netmask 255.255.255.252

# enable packet forwarding
# echo 1 > /proc/sys/net/ipv4/ip_forward

# enable masquerading
# iptables -t nat -A POSTROUTING -s 192.168.0.0/24 -o eth0 -j MASQUERADE
{% endhighlight %}

On the client (10.1.1.254 is the default gateway on the network you are currently connected to):

{% highlight bash %}
# replace default route (that has no access to the internet) with the new remote tunnel ip (that has access)
#  ip route add 10.0.0.0/8 via 10.1.1.254
#  ip route del default via 10.1.1.254 dev eth0
#  ip route add default via 192.168.0.2

# now you should have access to the internet
# ping 8.8.8.8
{% endhighlight %}




