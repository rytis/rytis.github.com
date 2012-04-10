---
layout: post
title: "Setting up Foreman on CentOS/RHEL 5.7 (part 1)"
category: sysadmin
tags: [foreman, puppet]
---
{% include JB/setup %}

In this set of articles we are going to go through the steps required to get a fully functional setup of Puppet 2.7.12 and Foreman 0.5 (git development branch) on a system running RedHat Enterprise Linux (or CentOS) version 5.7.

Before we dive in, let's discuss why such and old version of OS is used in this example.

---
* Table of contents
{:toc}
---

# Why use CentOS/RHEL version 5.7 ?

RHEL5 is an old product. Released on 2007-03-15 it will be going through the following life cycle dates:

* End of Production 1 in **Q4 2012**
* End of Production 2 in **Q1 2014**
* End of Production and End of Regular Life Cycle on **2017-03-21**
* End of Extended Life Cycle on **2020-03-31**

What does it mean? 

It means two things - the release in ancient, and a lot of companies are still using it. Firstly, it is really old, most packages included in the release are dated 2006 (Python is 2.4, Ruby is 1.8.3). Secondly, most enterprise companies are rather slow in upgrading to the later versions, given that RedHat will support it for another 5 years.

The development community does not stand still though, and most of the recent projects require newer versions of standard packages such as Python or Ruby. Unfortunately, updating is not always a straightforward activity - updating one package typically cascades into a lot of updates of various other packages.

So you end up making a choice - upgrading the OS or upgrading the individual packages. Each has its advantages and really depends on your organisation and its OS upgrade strategies.

For this set of articles we will assume that you have chosen to stick with the older OS, but want to get the latest and the greatest tools installed on it - Puppet 2.7.12 and Foreman 0.5. Puppet works just fine on RHEL5.7, but the latest version of Foreman (still in development at the moment of writing) requires Ruby 1.8.7, which is not avialable on RHEL5.7.

# Upgrade system packages

We will start by adding additional repositories to our system and upgrading some base packages. We will be building RPM packages ourselves where required, but once they are built, you can obviously skip this step the next time, and use them instead.

I will assume the minimal system install, so if some of the packages are already installed on your system, just ignore the messages saying that the package is already installed.

Some packages are only available via EPEL, and we also be installing the latest version of PostgreSQL - 9.1. And while we are at it, let's add Puppet repository here as well.

{% highlight bash %}

# rpm -ihv http://dl.fedoraproject.org/pub/epel/5/i386/epel-release-5-4.noarch.rpm
# rpm -ihv http://yum.postgresql.org/9.1/redhat/rhel-5-x86_64/pgdg-redhat-9.1-4.noarch.rpm
# rpm -ihv http://yum.puppetlabs.com/el/5/products/x86_64/puppetlabs-release-5-1.noarch.rpm

{% endhighlight %}

We will definitely need these packages:

{% highlight bash %}

# yum install -y httpd httpd-devel mod_ssl git gcc make wget

{% endhighlight %}

But I would also recommend installing the following packages too, it will make your life easier if you need to debug or troubleshoot the errors that might occur during the installation process:

{% highlight bash %}

# yum install -y file man tcpdump strace vim-common vim-enhanced bind-utils telnet

{% endhighlight %}

## Setting up RPM build environment

As I have mentioned before, this step is only required once. When you have built the RPMs you can just install them on any system that is running RHEL5.7.


{% highlight bash %}

# yum install -y rpm-build rpmdevtools
# rpmdev-setuptree

{% endhighlight %}

These steps will prepare your RPM build environment, you should now have `rpmbuild` directory created in your home directory.

## Building Ruby, RubyGems and Phusion Passenger RPMs

The following steps follow the same pattern for all three packages (or set of packages) - get the source RPM, install it, build your own RPM using the provided `.spec` file.

They all worked fine for me, and did not require any patching.

### Build Ruby 1.8.7-352 RPM

{% highlight bash %}

# wget http://rbel.frameos.org/stable/el5/SRPMS/ruby-1.8.7.352-5.el5.src.rpm
# rpm -ihv ruby-1.8.7.352-5.el5.src.rpm
# yum install readline-devel db4-devel gdbm-devel ncurses-devel openssl-devel autoconf bison byacc
# QA_RPATHS=$[ 0x0001|0x0010 ] rpmbuild -ba ~/rpmbuild/SPECS/ruby.spec

{% endhighlight %}

The packages will be built and placed in `~/rpmbuild/RPMS/<your architecture>/` directory. You will see the names of the packages and their location in the `rpmbuild` command output.

You can now install them right away with the `rpm -Uhv <package file>` command.

Once installed, test the installation:

{% highlight bash %}

# ruby -v
ruby 1.8.7 (2011-06-30 patchlevel 352) [x86_64-linux]

{% endhighlight %}

### Build RubyGems 1.8.10-1 RPM

Similarly we will build RubyGems packages. Once they are built - just install them.

{% highlight bash %}

# wget http://rbel.frameos.org/stable/el5/SRPMS/rubygems-1.8.10-1.el5.src.rpm
# rpm -ihv rubygems-1.8.10-1.el5.src.rpm
# rpmbuild -ba ~/rpmbuild/SPECS/rubygems.spec

{% endhighlight %}

Once again, test it:

{% highlight bash %}

# gem -v
1.8.10

{% endhighlight %}

### Build Phussion Passenger 3.0.11 RPM

Building Phusion Passenger will require additional package - `rubygem-daemon_controller`, which is not available in the standard RHEL5 repo, so we will have to build and install it too.

Also, make sure you do not forget to update GEM packages, as some of them (`rake` and `rack`) need to be of a newer version for the build to succeed.

{% highlight bash %}

# yum install -y rubygem-rake rubygem-rack rubygem-fastthread pcre-devel curl-devel doxygen asciidoc graphviz rubygem-daemons selinux-policy-devel libxslt-devel GeoIP-devel gd-devel
# wget http://passenger.stealthymonkeys.com/SRPMS/rubygem-daemon_controller-0.2.5-1.src.rpm
# rpm -ihv rubygem-daemon_controller-0.2.5-1.src.rpm
# rpmbuild -ba ~/rpmbuild/SPECS/daemon_controller.spec
# rpm -ihv /root/rpmbuild/RPMS/noarch/rubygem-daemon_controller-0.2.5-1.noarch.rpm
# gem update
# wget http://passenger.stealthymonkeys.com/SRPMS/rubygem-passenger-3.0.11-9.src.rpm
# rpm -ihv --nosignature rubygem-passenger-3.0.11-9.src.rpm
# QA_RPATHS=$[ 0x0001|0x0010 ] rpmbuild -ba ~/rpmbuild/SPECS/passenger.spec

{% endhighlight %}

And the test:

{% highlight bash %}

# passenger-config --version
3.0.11

{% endhighlight %}

# Installing and configuring Puppet

Now with the minor nuisances out of the way, let's get back to the main task - installing Puppet. For the sake of simplicity I will skip some of the tasks, such as setting up correct certificates (will just use the standard ones instead).

It is also worth mentioning, that this setup will use Apache with Phusion Passenger (aka `mod_rails`). This will give a better performance and is assumed to be more reliable than using a built-in Puppet web server.

## Puppet installation and configuration (no DB)

Let's start with the package installation. We will install both, Puppet and PostgreSQL, but will deal with PGSQL configuration a bit later. First we need to get Puppet up and running with Phusion Passenger.

{% highlight bash %}

# yum install -y postgresql91 postgresql91-server postgresql91-devel
# yum install -y puppet puppet-server

{% endhighlight %}

Shuffle some files around, create some extra directories, and set permissions right:

{% highlight bash %}

# mkdir -p /var/lib/puppet/ssl
# chown puppet.puppet /var/lib/puppet/ssl
# mkdir -p /etc/puppet/rack/{public,tmp}
# chown -R puppet /etc/puppet/rack
# chmod -R 755 /etc/puppet/rack
# cp /usr/share/puppet/ext/rack/files/config.ru /etc/puppet/rack
# chown puppet /etc/puppet/rack/config.ru
# ln -s /var/lib/puppet/ssl /etc/puppet/ssl

{% endhighlight %}

Now we need to generate Puppet own certificates. This happens when you run puppet server for the first time. This is the only time we will use this service.

{% highlight bash %}

# service puppetmasterd start
# service puppetmasterd stop
# chmod g+w /var/lib/puppet/ssl/ca/private/*
# chmod g+w /var/lib/puppet/ssl/ca/ca*pem

{% endhighlight %}

Puppet comes with a sample configuration for Apache, so we will just reuse it for our purposes. You might want to review various Passenger settings and tune it, but for now we will stick with the default values.

{% highlight bash %}

# cp /usr/share/puppet/ext/rack/files/apache2.conf /etc/httpd/conf.d/puppet.conf

{% endhighlight %}

Make sure you replace all appearences of `squigley.namespace.at` with a FQDN of your Puppet master node (the one you are building right now!).

Final steps before we run the test is to add some settings to the `/etc/puppet/puppet.conf` (do not replace the contents of the file with the code below, just add settings to appropriate sections):

{% highlight ini %}

[main]
  server = "<FQDN of the Puppet master>"

[master]
  ssl_client_header = SSL_CLIENT_S_DN
  ssl_client_verify_header = SSL_CLIENT_VERIFY

{% endhighlight %}

Now we need to restart the Apache web server, so that it will pick up the new settings, and, fingers crossed, puppet agent works the first time:

{% highlight bash %}

# service httpd restart
# puppet agent --test
info: Caching catalog for puppet-master.domain
info: Applying configuration version '1332260314'
notice: Finished catalog run in 0.02 seconds

{% endhighlight %}

## Adding StoredConfigs with PostgreSQL backend

Getting Puppet talking to PostgreSQL is relatively straightforward, but you need to watch out for Ruby GEM versions, as some newer versions of `activerecord` don't seem to play well with Puppet.

We need to initialise a PostgreSQL and create a new database for Puppet. Feel free to use a more secure password...

{% highlight bash %}

# service postgresql-9.1 initdb
# service postgresql-9.1 start
# chkconfig postgresql-9.1 on
# su - postgres
$ psql template1

template1=# create database puppet;
CREATE DATABASE
template1=# create user puppet with unencrypted password 'puppet';
CREATE ROLE
template1=# grant create on database puppet to puppet;
GRANT
template1=# \q

{% endhighlight %}

Now let's tell Puppet that it should use storedconfigs and how to connect to the database. Add the following configuration to `/etc/puppet/puppet.conf`, obviously use correct database and user names, and password. 

{% highlight ini %}

[master]
  storeconfigs = true
  dbadapter = postgresql
  dbuser = puppet
  dbpassword = puppet
  dbserver = localhost
  dbname = puppet

{% endhighlight %}

Also check that the authentication method is set to `password` (default configuration is `ident`) in `/var/lib/pgsql/9.1/data/pg_hba.conf`.

We are nearly there, just need to install a couple of Ruby GEMs that would allow Puppet to talk to PostgreSQL database server. Make sure you install the correct versions of the GEMs:

{% highlight bash %}

# PATH=/usr/pgsql-9.1/bin/:$PATH gem install pg
# gem install activerecord --version 3.0.11

{% endhighlight %}

With this, the configuration is complete, and hopefully Puppet should be able to talk to the database and create storedconfigs records. Let's restart the application and database and test it:

{% highlight bash %}

# service postgresql-9.1 restart
# service httpd restart
# puppet agent --test
info: Caching catalog for puppet-master.domain
info: Applying configuration version '1332336455'
notice: Finished catalog run in 0.02 seconds

{% endhighlight %}

Also don't forget to test that the data is actually recorded in the database:

{% highlight bash %}

# psql -U puppet puppet
Password for user puppet:
psql (9.1.3)
Type "help" for help.

puppet=> \x on
Expanded display is on.
puppet=> select * from hosts;
-[ RECORD 1 ]---+---------------------------
id              | 1
name            | puppet-master.domain
ip              | 192.168.0.10
environment     | production
last_compile    | 2012-03-21 13:27:35.802142
last_freshcheck |
last_report     |
updated_at      | 2012-03-21 13:27:35.803318
source_file_id  |
created_at      | 2012-03-21 09:22:07.916274

puppet=> \q

{% endhighlight %}

You can see that the record shows when the node last reported in and the time when the entry was created for the first time. So now you should have fully functioning Puppet setup, backed up by PostgreSQL database and running with Apache+Passenger. If not, then proceed to the Troubleshooting section, which might help you to resolve some of the typical problems.

In the next article we will proceed with the installation of Foreman version 0.5.

# Troubleshooting

As we all know, life is not always a bed of roses. Although the installation steps are pretty simple, there's always a chance that something will go wrong. I have done few installations and they usually tend to go smooth, the only problems I've come across were cause by the wrong permissions. The typical problem is that puppet user is not able to access configuration files or Ruby GEMs.

Also, check that:

* SELinux is disabled or set up properly, if you really need to use it
* iptables rules are setup correctly, if you are testing with a remote client

## Rails application permissions

Check that you have the following ownership and permissions combination in `/etc/puppet/rack`:

{% highlight bash %}

# ls -l /etc/puppet/rack/
total 24
-rw-r--r-- 1 puppet root  431 Mar 20 14:27 config.ru
drwxr-xr-x 2 puppet root 4096 Mar 20 14:27 public
drwxr-xr-x 2 puppet root 4096 Mar 20 14:27 tmp

{% endhighlight %}

## Ruby GEMs permissions

On some "hardened" systems, with very restrictive `umask` settings you might end up installing Ruby GEMs that are only usable by the `root` user, if you use `gem install` command.

Here's a quick way to fix these problems:

{% highlight bash %}

# find /usr/lib/ruby/gems/1.8/gems/ -type d -exec chmod 755 {} \;
# find /usr/lib/ruby/gems/1.8/gems/ -type f -exec chmod 644 {} \;

{% endhighlight %}

After you do this, you need to restore the executable bit on the Passenger agents:

{% highlight bash %}

# chmod +x /usr/lib/ruby/gems/1.8/gems/passenger-3.0.11/agents/PassengerWatchdog
# chmod +x /usr/lib/ruby/gems/1.8/gems/passenger-3.0.11/agents/apache2/PassengerHelperAgent
# chmod +x /usr/lib/ruby/gems/1.8/gems/passenger-3.0.11/agents/PassengerLoggingAgent

{% endhighlight %}

## Ruby GEM versions

If you followed the instructions to the letter, you should have the correct versions installed. If something, somewhere drifted apart, you might end up with a slightly different versions of the Ruby GEMs. Wrong Ruby GEM versions can cause a lot of grief, so double check that you have the following GEMs installed:

{% highlight bash %}

# gem list

*** LOCAL GEMS ***

activemodel (3.0.11)
activerecord (3.0.11)
activesupport (3.0.11)
arel (3.0.2, 2.0.10)
builder (3.0.0, 2.1.2)
daemon_controller (1.0.0, 0.2.5)
daemons (1.1.8, 1.0.10)
fastthread (1.0.7)
i18n (0.6.0, 0.5.0)
multi_json (1.1.0)
passenger (3.0.11)
pg (0.13.2)
rack (1.4.1, 1.1.0)
rake (0.9.2.2, 0.8.7)
tzinfo (0.3.32)

{% endhighlight %}

If any of the versions are different, uninstall them, and install the correct ones, as displayed in the list above. Two commands to install and uninstall Ruby GEMs:

{% highlight bash %}

# gem uninstall <gem name> --version <version>
# gem install <gem name> --version <version>

{% endhighlight %}

Once you're done, please check the permissions, as described in the previous Troubleshooting sections.

---
[Part 2 of this series is already available](../../../04/05/setting-up-foreman-on-centosrhel-57-part-2/)

