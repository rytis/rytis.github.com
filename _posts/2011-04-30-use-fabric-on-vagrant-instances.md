---
layout: post
title: "Use Fabric on Vagrant instances"
category: sysadmin
tags: [vagrant, fabric]
---
{% include JB/setup %}

[Vagrant](http://vagrantup.com/) is a great tool that lets you create and destroy [VirtualBox](https://www.virtualbox.org/) VMs with only few commands. It also supports [Puppet](http://www.puppetlabs.com/) and [Chef](http://wiki.opscode.com/display/chef/Home) out of the box. This means that you can have just a single OS image, which you then can customise to your needs on a per project basis.

Puppet (or Chef for that matter, however I'm not a big fan of it) is a nice tool, but I find that writing manifests for quick and one-off tasks can be bit time consuming, especially if the task boils down to a simple shell one liner.

You might have guessed that I'm going to introduce yet another tool to the deployment stack. I tend to do all the fine grained post-install and post-configure tasks with [Fabric](http://www.fabfile.org/). As an alternative to it is another great tool called [Func](https://fedorahosted.org/func/). I think Func is more heavyweight and better suited to manage larger estates, but I'm willing to be convinced otherwise.

Let's go back to the Vagrant VM management.

So you created an OS image (more about this in other post, which I'll write up shortly) or used one created by someone else. You also written a manifest (or recipe) that installs additional packages required by your project (httpd, radis, etc, etc) and you ready to deploy your application.

As I said, you can do that with Puppet (or Chef) as well, but while it's still in development phase a lot is going to change in the way you structure and deploy the project. So I prefer to use Fabric here.

Here's how you do it.

Firstly, create a fabfile.py and place it in the same directory where you have your Vagrantfile. Vagrant uses its own user (called vagrant) to access the VM, so you have to tell Fabric that you wish to use different username. You also need to find out Vagrant SSH key, so that it will be used by Fabric.

All that happens in the vagrant function in the fabfile.py below:

{% highlight python %}

from fabric.api import env, local, run
 
def vagrant():
    # change from the default user to 'vagrant'
    env.user = 'vagrant'
    # connect to the port-forwarded ssh
    env.hosts = ['127.0.0.1:2222']
 
    # use vagrant ssh key
    result = local('vagrant ssh_config | grep IdentityFile', capture=True)
    env.key_filename = result.split()[1]
 
def uname():
    run('uname -a')

{% endhighlight %}

We set up three configuration parameters in the vagrant function, which then can be chained with any other function in the fabfile. Note that I'm using a forwarded SSH port. You can setup host only networking for your VM instance and then use its own IP, but for the sake of simplicity I'm not doing that here.

So now you can manage your Vagrant VM with Fabric:

{% highlight bash %}

$ fab vagrant uname
[localhost] local: vagrant ssh_config | grep IdentityFile
[127.0.0.1:2222] Executing task 'uname'
[127.0.0.1:2222] run: uname -a
[127.0.0.1:2222] out: Linux vagrant-fedora-14 2.6.35.6-45.fc14.i686 #1 SMP Mon Oct 18 23:56:17 UTC 2010 i686 i686 i386 G[127.0.0.1:2222] out: 
[127.0.0.1:2222] out: 
 
Done.
Disconnecting from 127.0.0.1:2222... done.

{% endhighlight %}
