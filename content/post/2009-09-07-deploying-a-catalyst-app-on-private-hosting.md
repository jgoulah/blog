---
title: Deploying A Catalyst App on Private Hosting
type: post
date: 2009-09-07T18:26:06+00:00
url: /2009/09/deploying-a-catalyst-app-on-private-hosting/
categories:
  - Deployment
tags:
  - catalyst
  - Deployment
  - fastcgi
  - perl
  - private hosting
  - VPS

---
## Intro

In the last post I <a title="deploying catalyst on shared hosting" href="http://blog.johngoulah.com/2009/08/deploying-catalyst-shared-hosting/" target="_blank">wrote about</a> deploying Catalyst on shared hosting.  While shared hosting may seem attractive pricewise, you&#8217;ll quickly grow out of it, and it makes sense to move to hosting with some more control if you are serious about your website.   There are lots of choices when it comes to where you may want to host,  and if you are looking for virtual options, <a title="slicehost" href="http://www.slicehost.com/" target="_blank">Slicehost</a>, <a title="linode" href="http://www.linode.com/" target="_blank">Linode</a>, <a title="prgmr" href="http://prgmr.com/xen/" target="_blank">prgmr</a>, or even <a title="ec2" href="http://aws.amazon.com/ec2/" target="_blank">Amazon EC2</a> are great choices.  Picking and setting these up are well beyond the scope of this article,  so I&#8217;m assuming you have a server with root privileges ready to go.

## Setting Up Your Modules

As I probably sound like a broken record if you&#8217;ve read any of my other perl-related entries,  its a good idea to setup your modules using local::lib.  Check out the <a title="catalyst-shared-hosting-deployment" href="http://blog.johngoulah.com/2009/08/deploying-catalyst-shared-hosting/" target="_blank">last article</a> on how to set it up,  or the <a title="local::lib docs" href="http://search.cpan.org/~apeiron/local-lib-1.004007/lib/local/lib.pm" target="_blank">module docs</a> do a fine job if you follow the bootstrap instructions.  I always create a new user to set this up as,  such as &#8216;perluser&#8217;  or similar,  this way your modules aren&#8217;t affected by upgrades as your own user.  We&#8217;ll show how to point to these modules shortly.

For now, you&#8217;ll want to also make sure  you have <a title="FCGI" href="http://search.cpan.org/~skimo/FCGI-0.67/FCGI.PL" target="_blank">FCGI</a> and <a title="fcgi::procmanager" href="http://search.cpan.org/~gbjk/FCGI-ProcManager-0.19/ProcManager.pm" target="_blank">FCGI::ProcManager</a> installed as that user that you just setup along with the rest of your apps modules.

{{< highlight bash >}}$ cpan FCGI
$ cpan FCGI::ProcManager
{{< /highlight >}}

You may want to checkout your app under this user, so that you can just do 

{{< highlight bash >}}$ perl Makefile.PL
$ make installdeps
{{< /highlight >}}

assuming you&#8217;ve kept your Makefile.PL updated, this should be all of the perl modules you need to run your application.

## Setting Up Apache

You can setup <a title="apache" href="http://httpd.apache.org/" target="_blank">apache</a> using your package manager of choice.  For ease of this article I&#8217;ll assume you&#8217;re on Ubuntu, or Debian based system, and you can do something like

{{< highlight bash >}}$ sudo apt-get install apache2.2-common
$ sudo apt-get install apache2-mpm-worker
{{< /highlight >}}

And you also need the <a title="mod_fastcgi" href="http://www.fastcgi.com/mod_fastcgi/docs/mod_fastcgi.html" target="_blank">mod_fastcgi</a> module.  You could <a title="download fastcgi" href="http://www.fastcgi.com/dist/" target="_blank">download</a> and compile it.  Or you could grab it from apt.   You&#8217;ll probably have to add the multiverse repository,  so update your _/etc/apt/sources.list_so that each line will end with

{{< highlight bash >}}main restricted universe multiverse{{< /highlight >}}

Now you can

{{< highlight bash >}}$ sudo apt-get update
$ sudo apt-get install libapache2-mod-fastcgi
{{< /highlight >}}

## Setup the VirtualHost and App Directory Structure

You need to put a VirtualHost so that Apache knows how to handle the requests to your domain

{{< highlight bash >}}FastCgiExternalServer /opt/mysite.com/fake/prod -socket /opt/mysite.com/run/prod.socket

&lt;VirtualHost *:80&gt;
ServerName www.mysite.com

DocumentRoot /opt/mysite.com/app
Alias /static /opt/mysite.com/app/root/static/
Alias / /opt/mysite.com/fake/prod/
&lt;/VirtualHost&gt;
{{< /highlight >}}

This &#8220;/&#8221; alias ties your document root to the listening socket defined in the FastCgiExternalServer line. The &#8220;/static&#8221; alias make sure your static files are served by apache, instead of fastcgi. 

This config also assumes some directory structure is setup,  which is really entirely up to you.  But here we&#8217;ll assume you have a directory located at _/opt/mysite.com_ with a few directories under that called _fake_, _run_, and _app_.

{{< highlight bash >}}$ sudo mkdir -p /opt/mysite.com/{fake,run,app}
{{< /highlight >}}

The only directory you have to put anything in is _app_, which should contain your code.

Note, this is a very simplified layout.  In the real world I&#8217;d put the _fake_, _run_, and _app_ dirs under a versioned directory,  which my active virtualhost would then point to.  I&#8217;ve talked briefly about this kind of deployment technique <a title="deployment techniques" href="http://blog.johngoulah.com/2009/03/code-deployment-techniques/" target="_blank">before</a> at a high level and there is a great writeup <a href="http://use.perl.org/firehose.pl?op=view&id=3641" target="_blank">here</a> on the technical details on how to use multiple fastcgi servers to host your staging and production apps with seamless deployment.  Part of the beauty of using FastCGI is that you can run two copies of the app against the same socket so its easy to bring up instances pointing to different versions of your code, and deploy with zero downtime.

## Launching FastCGI

The last piece of the puzzle is to have a launch script,  which makes sure that your app is listening on the socket.  So to keep it simple you would have a script called _launch_mysite.sh_ that looks like this

{{< highlight bash >}}#!/bin/sh

export PERL5OPT='-Mlib=/home/perl_module_user/perl5/lib/perl5'

/opt/mysite.com/app/script/mysite_fastcgi.pl \
-l /opt/mysite.com/run/prod.socket -e \
-p  /opt/mysite.com/run/prod.pid -n 15
{{< /highlight >}}

The first line is telling the script to use the modules from the user we setup the local::lib to hold the modules, so make sure you change this to the correct location.  Then it starts up fastcgi to listen on your socket, and create a process pid,  and to spawn in this case 15 processes to handle requests. Go ahead and hit your domain, and it should show you your website. 

## Conclusion

We&#8217;ve gone over the basics on how to setup FastCGI using FastCgiExternalServer. You now have a lot of flexibility in how many processes are handling requests, the ability to run different copies of your app and flipping the switch, and pointing to which modules are run with your app. There are a lot of improvements you can now make to setup a very sane deployment process so that each version of code deployed can be its own standalone build and ensuring your production app has 100% uptime, but at this point its up to your imagination.
