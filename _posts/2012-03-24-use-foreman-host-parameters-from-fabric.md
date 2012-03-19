---
layout: post
title: "Use Foreman host parameters from Fabric"
category: sysadmin
tags: [foreman, fabric]
---
{% include JB/setup %}

[Fabric](http://www.fabfile.org) is great for managing remote systems. You can script whatever you want to do on a remote system, and run the tasks at any time, on any set of systems. This is an excellent automation tool, as it does not require any agent processes running on the remote systems - it uses SSH for its communication with the remote hosts.

If you are using [Foreman](http://www.theforeman.org) to manage your systems, you already maintain a list of servers. What's even better - Foreman allows you to assign custom parameters to the hosts, using host parameter settings. These parameters can be used by Puppet, if you're using Foreman as External Node Classifier.

Perhaps you've already guessed where I'm going with this... Wouldn't it be nice if you could tell Fabric to fetch a list of nodes based on some parameter stored in Foreman? Of course it would!

For example, if you're managing a Hadoop cluster, you might define a host parameter called `hadoop_node_role`, which defines what each node does. Hadoop cluster typically is made up of a pair of master nodes and a large number of slave nodes.

So let's assume that for each node in your Hadoop cluster you set `hadoop_node_role` to either `slave` or `master`.

Let's create a `fabfile.py` that will query Foreman to get a list of all nodes, and then filter out only those that meet a specified selector rule:

{% highlight python %}
from fabric.api import run, env
import requests
import json
import yaml
import pprint

# couple of node management tasks

def puppet(command='status'):
    """Manage puppet service.

    Available parameters: `command', default: `status'

    Example: fab <nodelist> puppet
             fab <nodelist> puppet:restart"""

    run("service puppet %s" % command)

def hadoop(command='status'):
    """Manage Hadoop services. 

    Available parameters: `command', default: `status'

    Example: fab <nodelist> hadoop
             fab <nodelist> hadoop:restart"""

    if env.host in master_list:
        services = ['hadoop-0.20-namenode', 'hadoop-0.20-jobtracker']
    if env.host in slave_list:
        services = ['hadoop-0.20-datanode', 'hadoop-0.20-tasktracker']
    if command == 'stop':
        services.reverse()

    for srv in services:
        run("service %s %s" % (srv, command))


# retrieve and filter a list of nodes from Foreman

def _get_host_list(param_name, param_value):
    host_list = []
    headers = { 'Accept': 'application/json' }
    r = requests.get('http://localhost/hosts', headers = headers)
    hosts = json.loads(r.text)
    for host in hosts:
        r = requests.get("http://localhost/hosts/%(host)s?format=yaml" % {'host': host}, headers = headers)
        host_data = yaml.load(r.text)
        if host_data['parameters'].get(param_name, '') == param_value:
            host_list.append(host)
    return host_list

if __name__ != '__main__':
    if 'selector' in env:
        key, val = env.selector.split(':')
        env.hosts += _get_host_list(key, val)
{% endhiglight %}

Now you can specify any host parameter name / value pair, and run any Fabric command on the resulting list of nodes:

{% highlight bash %}
# fab --set selector='hadoop_node_role:slave' puppet
[hadoop-slave-1.domain] Executing task 'puppet'
[hadoop-slave-1.domain] run: service puppet status
[hadoop-slave-1.domain] out: puppetd (pid  14160) is running...

[hadoop-slave-2.domain] Executing task 'puppet'
[hadoop-slave-2.domain] run: service puppet status
[hadoop-slave-2.domain] out: puppetd (pid  30375) is running...

[hadoop-slave-3.domain] Executing task 'puppet'
[hadoop-slave-3.domain] run: service puppet status
[hadoop-slave-3.domain] out: puppetd (pid  22611) is running...

[hadoop-slave-4.domain] Executing task 'puppet'
[hadoop-slave-4.domain] run: service puppet status
[hadoop-slave-4.domain] out: puppetd (pid  23080) is running...


Done.
Disconnecting from hadoop-slave-1.domain... done.
Disconnecting from hadoop-slave-2.domain... done.
Disconnecting from hadoop-slave-3.domain... done.
Disconnecting from hadoop-slave-4.domain... done.
{% endhighlight %}

