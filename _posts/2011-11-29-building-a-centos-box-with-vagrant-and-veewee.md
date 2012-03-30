---
layout: post
title: "Building a CentOS box with Vagrant and VeeWee"
category: sysadmin
tags: [vagrant]
---
{% include JB/setup %}

This is more of a quick checklist for building a CentOS box with Vagrant and VeeWee than a real blog post.

### Prerequisites

- Oracle VirtualBox installed
- Vagrant installed (`sudo gem install vagrant`)
- VeeWee installed (`sudo gem install veewee`)
- CentOS 6.0 i386 DVD ISO downloaded from one of the mirrors listed in <http://isoredirect.centos.org/centos/6/isos/i386/>

### Building a CentOS box

- Create working directories and copy the ISO image

{% highlight bash %}

$ mkdir -p ~/vagrant/centos60build/iso
$ cd ~/vagrant/centos60build
$ cp __PATH_TO_THE_ISO_IMAGE__ ./iso/

{% endhighlight %}

- Create a box definition

{% highlight bash %}

$ vagrant basebox define centos60 CentOS-6.0-i386

{% endhighlight %}

- Make any changes you need to the definition file in `definitions/centos60/definition.rb`. Make sure the name of the ISO and its MD5 are correct.
- Adjust postinstall and kickstart files if you need to
- Kickstart a Virtual Machine with CentOS 6.0 (this is going to take a while)

{% highlight bash %}

$ vagrant basebox build centos60

{% endhighlight %}

- Check that it is functional

{% highlight bash %}

$ vagrant basebox validate centos60

{% endhighlight %}

- Build a Vagrant box file

{% highlight bash %}

$ vagrant basebox export centos60

{% endhighlight %}

- Import it to Vagrant

{% highlight bash %}

$ vagrant box add centos60 centos60.box

{% endhighlight %}

### Using your new box

To spin off a new instance, run the usual Vagrant commands:

{% highlight bash %}

$ vagrant init centos60
$ vagrant up
$ vagrant ssh

{% endhighlight %}

Check out [these instructions](../../../../2011/04/30/use-fabric-on-vagrant-instances) on how to set up Fabric so that it can talk to your Vagrant VM instances.
