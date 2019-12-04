---
title: Code Deployment Techniques
author: jgoulah
type: post
date: 2009-03-15T00:09:50+00:00
url: /2009/03/code-deployment-techniques/
categories:
  - Deployment
tags:
  - Deployment
  - Versioning

---
# Introduction

Getting code out to your target production machines is a bit of an art form that many seem to take for granted. A lot of shops that I talk to seem to think of the deployment process as an _svn up_ on each node. Even if this works for you, it&#8217;s really not a great way to deploy code. The main reason is that it does not scale well. You lend yourself to having different versions of code on different machines at the same time. For example, if your load balancing algorithm made the logs on one machine fill up faster than another and you run out of disk space during the svn update on a node. The correct process should be such that different releases of code can by deployed nearly instantaneously, with old copies left around for rollback. If you just svn up your code has the potential to be out of sync a lot more often than if you deploy code with an rsync and provide your webserver symbolic links to the current version of code. This article will show how this process can work in a scalable way.

Like most of my articles, this is targeted towards shops that are running something along the lines of your typical LAMP stack. I&#8217;ve setup this deployment architecture under a PHP setup with Apache, and also a slightly more complicated Perl setup running ModFastCGI instances behind Apache. In one of my most complex deployments I&#8217;ve had the database updates go out simultaneously along with the code that worked with those changes, so there is a lot of flexibility in what you can do. I&#8217;ll be talking at a very high level on how to accomplish this.

# Code Environments

Deployment is the process of getting the code from your development environment into a working production system. There are usually at least three environments: development, staging, and production. Some places may have a demo or test environment for their own specific purpose but we will focus on the main three. Your development environment is active when code is being written and tested by the developers. Typically the development environment points at a &#8216;dev&#8217; database or if you are using something like <a href="http://blog.johngoulah.com/2009/01/mysql-sandbox/" target="_blank">MySQL Sandbox</a> perhaps each developer has his own database to work against. Your staging environment should be as close of a replica to your production environment as possible.  I typically deploy staging on the exact same servers as production, but point it to a copy of the production database instead of the real thing.  Its ideal for this copy to be taken nightly with something like <a href="http://blog.johngoulah.com/2009/01/mysql-snapshots-using-lvm/" target="_blank">MySQL LVM snapshotting</a> so you&#8217;re working on similar data.  When staging code is assured to be &#8220;bug free&#8221; it is then pushed to your live production environement.

Its a good idea to have your code respect these environments. The initialization for your application should be able to infer the environment and pull the appropriate config based on that so that you are pointing to the right DB and any other specifics for that particular environment. One way to do this is to use an environment variable and define that inside of your virtual host file. Since you&#8217;ll have a separate virtual host for staging and production your application can then look at the variable and know what configuration to grab.

# The Build Process

To get your code from development to staging you have to undergo some form of a build process. Some build processes are more complicated than others. A simple build process may just be as trivial as doing an _svn export_ on your repository to the location you wish to push that code from. More complex builds may want to build and bundle external libraries that the application needs to work. For example in a Perl deployment I&#8217;ve compiled the required modules on the build step of each release so that the build consists of the same version of modules that the code was written against. This was really important since CPAN is ever changing, otherwise its possible in a rollback situation that there would be incompatibilities. For the purposes of simplicity I&#8217;ll just assume you are deploying code that has everything setup and working on the server (such as PHP usually is).

The build should also be responsible for putting the code into a versioned directory. There are a few benefits here. One is that you can correspond a build of code to the repository revision that it correlates to. Second is that it makes rollback a cinch because multiple versions of the code can exist on the server at the same time. Apache knows which version is active because the virtual host for the active version of code is pointed to by a symbolic link that gets modified as part of the deploy procedure.

### Generating a Version

I&#8217;ve explained that your build script should generate a version and it makes sense for this to come from your repository revision that is getting deployed. Here&#8217;s a short bash function that can generate a version from an SVN repository (also works with SVK which is an SVN wrapper)

<pre>function get_version {
  if [ -e .svn ]; then
    eval $1=$(svn info | grep ^Revision | perl -ne "/Revision: (\d+)/; printf('1.%06i', \$1);")
  else
    eval $1=$(svk info | grep "Mirrored From" | perl -ne "/Rev. (\d+)/; printf('1.%06i', \$1);")
  fi
}</pre>

and now the build script can do something like

<pre>cd /path/to/svn/checkout
svn up
get_version version
echo "this is the version: $version"</pre>

### Exporting the Code

Now that you&#8217;ve generated a version you should export that version of code into the directory you are going to push from. For this example we&#8217;ll say that we store versions in _/opt/test.com_ so the structure might look something like:

<pre>/opt/test.com
     {version1}
     {version2}
     {versionN}
     ...</pre>

Then the code would live under _version1_, _version2_, &#8230; _versionN_ for each version of code deployed.

One way to do this is something like:

<pre>mkdir /opt/test.com/$version
svn export -q svn://${svn_host}/trunk ${site_dir}/${version}</pre>

### Informing Apache of the Document Root

Apache needs to know where the new version of code is. Since we&#8217;ve decided to put it into an ever changing directory name (the build version) there are at least two methods for letting Apache know the document root of the code. One way is to dynamically generate a virtual host with the explicit versioned path from above (eg. /opt/test.com/0.00010). The document root here will always point to the version that its bundled with, and then on deployment, a symlink from apache&#8217;s conf.d directory is pointed to the active virtual host config file. Another way to do this is to have a more static virtual host file, in which the document root path points to a symlinked directory to the version which changes on deploy (eg. /opt/test.com/current-www such that current-www is a symlink to the appropriate version).

# The Deploy Process

In the end the point of deployment is to get your code into production. The version of code should first be rolled into staging, and then when verified as good, into production. The beauty of this is that all that it takes to move code from staging to production is an update to a symbolic link. The current production symlink is wiped out, and pointed to the new version of code, the same one that we deployed for staging.

The deployment should at a minimum

  * rsync code to each web server
  * point the symbolic links to the new version of code
  * restart apache

The great thing is that the symbolic links are not repointed until all of the code already exists on each server (the rsync to all servers should happen first).   This gives a deploy that doesn&#8217;t leave servers out of sync with each other.  The deploy is instantaneous and verified working without trying to rsync or svn up over top of running production code.

# Conclusion

I&#8217;ve gone over a robust way to do code deployments that can be fully automated with a little bit of scripting.  Its an easy way to get code out to production and also to ensure that the exact version of code that currently exists in staging is the version of code that goes out.   It gives us a very easy way to rollback if a major problem does happen to be found in production and to quickly see what version is currently deployed.  And it allows us to change versions of code simultaneously without the end user ever knowing.