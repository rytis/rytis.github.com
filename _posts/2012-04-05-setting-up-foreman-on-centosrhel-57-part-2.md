---
layout: post
title: "Setting up Foreman on CentOS/RHEL 5.7 (part 2)"
category: sysadmin
tags: [foreman, puppet]
---
{% include JB/setup %}

In the previous article we installed and configured Puppet. We have also set up puppet to write host configuration data to a PostgreSQL database using storedconfig.

Today, I'm going to show you how to install and configure:

- Foreman
- Foreman's smart-proxy
- Integrate smart-proxy with DHCP/TFTP/DNS/Puppet

---
* Table of contents
{:toc}
---

# Install Foreman from the latest development branch

Installing Foreman from RPM has its own advantages, but currently the situation is such that the packaged (Stable) version lags quite far behind the development branch. At the time of writing, the development branch is quite stable and comes with a lot of new features, that the stable release (0.4) is lacking.

Another advantage of installing directly from the Github repository is that you'll get better understanding of how the system is put together and how it works.

## Foreman dependencies

Foreman 0.5 uses Rails 3 and uses `bundler` to manage the packages, so let's install this Gem:

{% highlight bash %}

# gem install bundle

{% endhighlight %}

We also need to install `libvirt` packages and their dependencies. If you are using RHEL distribution and it is subscribed to RHN, you can use official RHN **RHEL Virtualisation Server 5** channel to install these packages.

Alternatively, if you're using CentOS or do not have access to RHN for some reason, you can install CentOS RPMs directly, which works fine on RHEL distribution too:

{% highlight bash %}

# rpm -ihv http://mirror.centos.org/centos-5/5/os/x86_64/CentOS/xen-libs-3.0.3-135.el5.x86_64.rpm
# rpm -ihv http://mirror.centos.org/centos-5/5/os/x86_64/CentOS/xen-devel-3.0.3-135.el5.x86_64.rpm
# rpm -ihv http://mirror.centos.org/centos-5/5/os/x86_64/CentOS/libvirt-0.8.2-25.el5.x86_64.rpm
# rpm -ihv http://mirror.centos.org/centos-5/5/os/x86_64/CentOS/libvirt-devel-0.8.2-25.el5.x86_64.rpm

{% endhighlight %}

## Getting the latest Foreman code

Now with all prerequisites sorted, let's get the latest Foreman build from Github:

{% highlight bash %}

# export GIT_SSL_NO_VERIFY=true
# export PATH=/usr/pgsql-9.1/bin/:$PATH
# cd /opt
# git clone https://github.com/theforeman/foreman.git -b develop

{% endhighlight %}

The next code snippet shows how to build the Foreman using bundler. You can see that I'm excluding some options, such as MySQL and SQLite support, as well as Development and Test components.

If you are using MySQL instead of PostgreSQL, you will have to modify the `--without` option to exclude `postgresql` instead of `mysql`.

For testing purposes you might want to leave SQLite, and remove the other two databases.

{% highlight bash %}

# cd foreman
# bundle install --without mysql sqlite test development --path vendor
# cp config/settings.yaml.example config/settings.yaml

{% endhighlight %}

Now we need to initialise the database by creating all required tables and updating them with initial data. In Rails world this is achieved by running DB migration scripts and then starting up the Foreman service, as per below:

{% highlight bash %}

# RAILS_ENV=production bundle exec rake db:migrate
# RAILS_ENV=production ./script/rails s

{% endhighlight %}

Now that we have a running Foreman instance, try connecting to it (http://your-foreman-server:8140/) and see if you get a response from a working Rails application. You should see a Foreman dashboard, similar to the below:

<img src="/assets/images/foreman-dashboard.png" width="400" />

## Configure Passenger

It is OK to use default WEBrick web server for quick tests, but you don't really want to do that in the production environment. So just as we did with Puppet, let's run Foreman from withing Passenger.

The configuration is very similar to Puppet configuration. Assuming that you are using Apache, create `/etc/httpd/conf.d/foreman.conf` with the following contents:

{% highlight apache %}

<VirtualHost *:80>
    DocumentRoot /opt/foreman/public
    RailsAutodetect On
    AddDefaultCharset UTF-8
</VirtualHost>

<VirtualHost *:443>
    RailsAutoDetect On
    RailsEnv production
    DocumentRoot /opt/foreman/public

    # Use puppet certificates for SSL
    SSLEngine On
    SSLCertificateFile /var/lib/puppet/ssl/certs/chidatapp666.bskyb.com.pem
    SSLCertificateKeyFile /var/lib/puppet/ssl/private_keys/chidatapp666.bskyb.com.pem
    SSLCertificateChainFile /var/lib/puppet/ssl/ca/ca_crt.pem
    SSLCACertificateFile /var/lib/puppet/ssl/ca/ca_crt.pem
    SSLVerifyClient optional
    SSLOptions +StdEnvVars
    SSLVerifyDepth 3
</VirtualHost>

{% endhighlight %}

## Integrate Puppet and Foreman

There are couple of steps needed to integrate Puppet and Foreman, so that each could see and use others information.

### Sending Puppet reports to Foreman

Once a Puppet client finishes its run, it will report back to Puppet master with the status. Puppet allows you to define hooks, so that every time when a report comes back, a custom script is fired. We are going to create and register a script that'll put the information into the Foreman's DB.

The source of this script can be found on [Foreman Github repository](https://raw.github.com/theforeman/puppet-foreman/master/templates/foreman-report.rb.erb), so check for an updated version.

Add the following code the a file called `/usr/lib/ruby/site_ruby/1.8/puppet/reports/foreman.rb`. You will have to create this file, as it wouldn't exist:

{% highlight ruby %}

$foreman_url='http://__foreman_host__'

require 'puppet'
require 'net/http'
require 'net/https'
require 'uri'

Puppet::Reports.register_report(:foreman) do
    Puppet.settings.use(:reporting)
    desc "Sends reports directly to Foreman"

    def process
      begin
        uri = URI.parse($foreman_url)
        http = Net::HTTP.new(uri.host, uri.port)
        if uri.scheme == 'https' then
          http.use_ssl = true
          http.verify_mode = OpenSSL::SSL::VERIFY_NONE
        end
        req = Net::HTTP::Post.new("#{uri.path}/reports/create?format=yml")
        req.set_form_data({'report' => to_yaml})
        response = http.request(req)
      rescue Exception => e
        raise Puppet::Error, "Could not send report to Foreman at #{$foreman_url}/reports/create?format=yml: #{e}"
      end
    end
end

{% endhighlight %}

Now tell Puppet to run this script by modifying `/etc/puppet/puppet.conf`:

{% highlight ini %}

[main]
    reports=log, foreman

{% endhighlight %}

### Telling Puppet to use Foreman as ENC

ENC means External Node Classifier, and it is a way to provide information about a node to Puppet. The information provided includes Puppet classes that need to be applied to a node, and Puppet configuration parameters.

One way to manage node configuration is by using `nodes.pp` Puppet configuration file, but Foreman is already holding all that information in its data structures, so we'll make use of it.

To do so, we need to create a script that Puppet would call to get the node information. It can be any script as long as it returns valid configuration in YAML format. The script should accept a hostname as parameter.

Create `/etc/puppet/node.rb` with this code in it:

{% highlight ruby %}

#!/usr/bin/env ruby
# a simple script which fetches external nodes from Foreman
# you can basically use anything that knows how to get http data, e.g. wget/curl
 etc.

# Foreman url
foreman_url="http://__foreman_host__" 

require 'net/http'

foreman_url += "/node/#{ARGV[0]}?format=yml" 
url = URI.parse(foreman_url)
req = Net::HTTP::Get.new(foreman_url)
res = Net::HTTP.start(url.host, url.port) { |http|
  http.request(req)
}

case res
when Net::HTTPOK
  puts res.body
else
  $stderr.puts "Error retrieving node %s: %s" % [ARGV[0], res.class]
end

{% endhighlight %}

Next step is to tell Puppet what script it needs to call when it needs to get information about a node. We can do this by adding the following configuration to `/etc/puppet/puppet.conf`:

{% highlight ini %}

[master]
  external_nodes = /etc/puppet/node.rb
  node_terminus  = exec

{% endhighlight %}

# Install Foreman Proxy (smart-proxy)

So we have a running and configured instance of Foreman, but the system is not ready yet for host provisioning. Most of the operation that Foreman performs, like adding hosts to DHCP, creating PXEboot files and managing DNS, are done using **smart-proxy** components, also refered to as **foreman-proxy**. I'll use these terms interchangeably.

Installing the latest Foreman-Proxy code is very simple, just follow the steps below:

{% highlight bash %}

# cd /opt
# git clone git://github.com/theforeman/smart-proxy.git
# useradd -r foreman-proxy
# usermod -G puppet foreman-proxy
# usermod -G foreman-proxy puppet
# chown -R foreman-proxy smart-proxy
# chown -R foreman-proxy.root /var/log/foreman-proxy
# gem install sinatra -v 1.0
# mkdir /var/run/foreman-proxy/
# su - foreman-proxy
$ /opt/smart-proxy/bin/smart-proxy

{% endhighlight %}

You should have a working smart-proxy, albeit without any capabilities. You can test it with your browser by going to http://__your_host__:8443/features. You should see an empty 'Suported features' list.

In this example I'm starting smart proxy directly running the script. Although it does 'daemonise' itself properly, you might want to create a proper `init.d` script.

## Configure Smart-Proxy to control Puppet

Foreman Proxy needs to be able to run Puppet certificate management commands. While we're at it, we also enable remote Puppet run for the Foreman user.

Run `visudo` as `root` user and add the following configuration:

{% highlight bash %}

Defaults:foreman !requiretty
Defaults:foreman-proxy !requiretty
foreman ALL=(ALL) NOPASSWD:/usr/sbin/puppetrun
foreman-proxy ALL=(ALL) NOPASSWD:/usr/sbin/puppetca*

{% endhighlight %}

You also need to tell smart proxy that it is supposed to manage Puppet by making these changes to `/opt/smart-proxy/config/settings.yml`:

{% highlight bash %}

:puppetca: true
:puppet: true
:puppet_conf: /etc/puppet/puppet.conf

{% endhighlight %}

## Adding support for TFTP and DHCP management

You aobviously can separate TFTP and DHCP services, but they typically are not resource hungry and because they are both involved in PXE boot process it makes sense to run these two services on the same host. So I will deal with their configuration in one section.

### TFTP configuration

Install TFTP server and SysLinux package, which will provide PXELinux boot image:

{% highlight bash %}

# yum install -y tftp-server syslinux
# cp /usr/lib/syslinux/pxelinux.0 /tftpboot

{% endhighlight %}

Make sure you enable TFTP server in `xinetd` configuration by modifying `/etc/xinetd.d/tftp` and restarting `xinetd` service:

{% highlight bash %}

server_args             = -s /tftpboot
disable                 = no

{% endhighlight %}

Finally, tell Foreman Proxy that it is now in charge for TFTP management by changing `/opt/smart-proxy/config/settings.yml`:

{% highlight bash %}

:tftp: true
:tftproot: /tftpboot

{% endhighlight %}

### Add DHCP support

Blah
 

# Troubleshooting

# Further reading

Visit these websites for more information:

- [Foreman project website][1]
- [Cloud Provisioning blog][2]
- [Puppet documentation][3]

[1]: http://theforeman.org/
[2]: http://cloudprovisioning.wordpress.com/
[3]: http://docs.puppetlabs.com/
