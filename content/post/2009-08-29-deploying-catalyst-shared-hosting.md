---
title: Deploying a Catalyst App on Shared Hosting
author: jgoulah
type: post
date: 2009-08-29T18:48:48+00:00
url: /2009/08/deploying-catalyst-shared-hosting/
categories:
  - Deployment
tags:
  - catalyst
  - Deployment
  - fastcgi
  - perl

---
## Intro

People have long complained that one of the tricky things about perl is the deployment phase, much because of the intricacies of mod_perl and its unfriendliness towards shared environments. In fact, I would highly recommend <a href="http://www.fastcgi.com" target="new">FastCGI</a> over mod_perl since it is quite easy to understand and configure. This post is going to focus on smaller shared hosting environments, and how easy it can be to quickly deploy a Catalyst web app.

## Getting Started

### Assumptions

There are some basic assumptions here. First we need a Linux webserver that has Apache installed and is loading up
  
<a href="http://fastcgi.coremail.cn/" target="_blank">fcgid</a>. I believe you can also use the favored <a href="http://www.fastcgi.com/mod_fastcgi/docs/mod_fastcgi.html" target="_blank">mod_fastcgi</a> which I just pointed to above, but I have yet to test this on a shared host. These are binary compatible modules so in theory both work. But again I&#8217;ve only used mod_fastcgi for large _non_-shared hosted deployments. You&#8217;ll also need <a href="http://httpd.apache.org/docs/1.3/mod/mod_rewrite.html" target="_blank">mod_rewrite</a> which is now fairly common.

### Installing Your Modules

The best way to install the modules your application depends on is using <a href="http://search.cpan.org/~apeiron/local-lib-1.004006/lib/local/lib.pm" target="_blank">local::lib</a>. I&#8217;ve talked about this <a href="http://www.catalystframework.org/calendar/2007/8" target="_blank">before</a> so there isn&#8217;t a lot of need to go over the process in detail again, but in a nutshell you can do 

<pre>$ wget http://search.cpan.org/CPAN/authors/id/A/AP/APEIRON/local-lib-1.004006.tar.gz
$ tar xzvf local-lib-1.004006.tar.gz
$ cd local-lib-1.004006
$ perl Makefile.PL --bootstrap
$ make test && make install
$ echo 'eval $(perl -I$HOME/perl5/lib/perl5 -Mlocal::lib)' >>~/.bashrc
$ source ~/.bashrc
</pre>

Now you have an environment that you can install your modules into. By default this is localized to ~/perl5. The next step is to install your modules that the application requires. It is good practice to put these into your Makefile.PL so that you can easily install them in one shot. A very basic one would follow this template

<pre>use inc::Module::Install;

name 'MyApp';
all_from 'lib/MyApp.pm';

requires 'Catalyst::Runtime' => '5.80011';
requires 'Config::General';
# require other modules here

install_script glob('script/*.pl');
auto_install;
WriteAll;
</pre>

Now its easy to do

<pre>$ export PERL_MM_USE_DEFAULT=1
$ perl Makefile.PL
$ make installdeps
</pre>

The PERL\_MM\_USE_DEFAULT will configure things such that you don&#8217;t have to press enter at every question about a dependency. The make installdeps will install any missing modules, which in this case is going to be everything. You can upgrade the version numbers in the Makefile.PL &#8220;requires&#8221; lines if you want installdeps to grab the newer distributions as they are released to CPAN.

### Configuring Your App

First thing we have to do is a minor edit to our fastcgi script, which is to tell it to use our local::lib. Since its not part of the environment we setup earlier in .bashrc we have to tell the fastcgi perl script where to find things. Below the &#8220;use warnings;&#8221; line add this

<pre>use lib "/home/myuser/perl5/lib/perl5";
use local::lib;
</pre>

Make sure to change the path to the correct location of your perl5 modules directory.

The last thing is to make sure your app is located in the public directory root for your host. In my case I created a symbolic link from the public_html folder to my app. 

<pre>$ cd && mv public_html public_html.old  # get rid of current root folder
$ ln -s ~/myapp ~/public_html
</pre>

Then create a _.htaccess_ file in that folder, which should reside beside all of your code

<pre>Options ExecCGI FollowSymLinks
Order allow,deny
Allow from all
AddHandler fcgid-script .pl

RewriteEngine On
RewriteCond %{REQUEST_URI} !^/?script/myapp_fastcgi.pl
RewriteRule ^(.*)$ script/myapp_fastcgi.pl/$1 [PT,L]
RewriteRule ^static/(.*)$ static/$1 [PT,L]
</pre>

We&#8217;re just telling apache to turn CGI on, and make sure to execute our fastcgi perl script. Be sure to change the rewrite lines to point to your script (hint: change myapp to your app name).

Now, you should be able to hit your domain. Simple!

# Conclusion

So we&#8217;ve seen that deploying perl can actually be fairly easy. There are of course some assumptions here, for example, to get the rewrite rules working you&#8217;ll need mod_rewrite.so, but this is fairly standard these days. Now you can deploy a perl app with the same ease as languages such as PHP, where it is pretty much plug and play. This should enable people to more easily compete with all the badly written blog, forum, and other generically useful software in the open source world.