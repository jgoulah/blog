---
title: Prevent SSH Attacks Using DenyHosts
author: jgoulah
type: post
date: 2009-02-02T00:39:52+00:00
url: /2009/02/prevent-ssh-attacks-using-denyhosts/
categories:
  - SSH
tags:
  - brute force
  - denyhosts
  - dictionary
  - security
  - SSH

---
# Introduction

If you have any servers that are running <a href="http://en.wikipedia.org/wiki/Secure_Shell" target="_self">SSH</a> and listening on a public net connection, its a good idea to prevent against dictionary attacks, since they are the simplest way to gain entry. You can get an idea of who is connecting or attempting to connect by viewing your ssh log which is located at _/var/log/secure_ on Redhat or _/var/log/auth.log_ on Debian.

There is an easy tool that you can install to deal with this attack called <a href="http://denyhosts.sourceforge.net/" target="_blank">DenyHosts</a>. It basically monitors these log files and then if it finds a potential attacker based on your threshold it will add them to the _/etc/hosts.deny_ file to prevent them from brute forcing your system.

# Getting DenyHosts

DenyHosts can be found <a href="http://denyhosts.sourceforge.net/" target="_blank">here on Sourceforge</a>

We&#8217;ll grab the tarball

{{< highlight bash >}}$ wget \
http://internap.dl.sourceforge.net/sourceforge/denyhosts/DenyHosts-2.6.tar.gz{{< /highlight >}}

Extract and install

{{< highlight bash >}}$  tar xzvf DenyHosts-2.6.tar.gz
$  cd DenyHosts-2.6
$  sudo python setup.py install{{< /highlight >}}

This installs it into _/usr/share/denyhosts_ so we&#8217;ll change directory

{{< highlight bash >}}$ cd /usr/share/denyhosts{{< /highlight >}}

Copy the default config

{{< highlight bash >}}$  sudo cp denyhosts.cfg-dist denyhosts.cfg{{< /highlight >}}

We need to set a few values in this file. The default values are for RedHat and this is how it looks on Debian

{{< highlight bash >}}SECURE_LOG = /var/log/auth.log
LOCK_FILE = /var/run/denyhosts.pid{{< /highlight >}}

You may also want to set the _DENY\_THRESHOLD\_INVALID_ and _DENY\_THRESHOLD\_VALID_ values, which controls the threshold of failed login attempts for fake and real users respectively.

Now we need to setup the script that controls the deamon that does the work. There is a default one available so we can copy that

{{< highlight bash >}}$ sudo cp daemon-control-dist daemon-control{{< /highlight >}}

There is only one value to change here on Debian

{{< highlight bash >}}DENYHOSTS_LOCK = "/var/run/denyhosts.pid"{{< /highlight >}}

Now symlink the control script into our bootup scripts folder and add it to start on boot

{{< highlight bash >}}$  sudo ln -s /usr/share/denyhosts/daemon-control /etc/init.d/denyhosts
$  sudo update-rc.d denyhosts defaults{{< /highlight >}}

Finally start it up for the first time

{{< highlight bash >}}$  sudo /etc/init.d/denyhosts start{{< /highlight >}}

If you haven&#8217;t changed the default you can find the logs in _/var/log/denyhosts_ if you want to see what its doing.

# Conclusion

We&#8217;ve pretty quickly setup a tool to defend against brute force attacks. Its easy to configure and also supports email, smtp, and syslog notifications. This is a tool everyone should install for any servers that have SSH listening on the Internet.
