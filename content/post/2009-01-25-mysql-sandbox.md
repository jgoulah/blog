---
title: Quick DB Setups with MySQL Sandbox
author: jgoulah
type: post
date: 2009-01-25T19:57:20+00:00
url: /2009/01/mysql-sandbox/
categories:
  - Databases
tags:
  - MySQL replication sandbox

---
# Introduction

There are various reasons to setup quick &#8220;sandbox&#8221; instances of MySQL. You can use them to test different types of replication (such as master-master or various slave topologies), to test your code against different versions of MySQL, or to setup instances of MySQL on a per developer basis where each person has their own database running on a different port so they can breakdown/setup the DB easily or make schema changes without affecting other team members. A perfect tool to do all of these things easily is <a href=" https://launchpad.net/mysql-sandbox" target="_blank">MySQL Sandbox</a>.

# Download the Prerequisites

To use MySQL sandbox effectively you need two things, the MySQL sandbox tool itself, and a MySQL tarball that the sandbox script can use to setup instances.

You can <a href=" https://launchpad.net/mysql-sandbox/+download" target="_blank">download</a> the latest MySQL sandbox version like so (2.0.12 as of this writing):

`wget http://launchpad.net/mysql-sandbox/mysql-sandbox-2.0/2.0/+download/mysql_sandbox_2.0.12.tar.gz`

You can download the MySQL tarball from any <a href=" http://dev.mysql.com/downloads/mirrors.html" target="_blank">MySQL mirror</a>. Its important to get the non RPM, Intel C/C++ compiled, glibc-2.3 version (look for a tar.gz file with &#8220;icc&#8221; in the filename) for example if you want version 5.1:

`wget ftp://mirror.anl.gov/pub/mysql/Downloads/MySQL-5.1/mysql-5.1.30-linux-i686-icc-glibc23.tar.gz`

# Installing an Instance with MySQL Sandbox

First you need to extract the MySQL Sandbox tool and change directory to it:

<pre>$ tar xzvf mysql_sandbox_2.0.12.tar.gz
$ cd mysql_sandbox_2.0.12</pre>

The easiest and quickest way to create an instance is:

<pre>$ ./make_sandbox /path/to/mysql-X.X.XX-osinfo.tar.gz</pre>

where _mysql-X.X.XX-osinfo.tar.gz_ is the MySQL tarball we just downloaded. And you&#8217;re done.

However, this will put the sandbox in a directory under your home directory ($HOME/sandboxes/msb\_X\_X_XX), which may or may not suit your purposes. It sets it up with default users, passwords, ports, and directory name. Lets fine tune things a bit.

# Setting up a Custom Tuned Instance

I want to put my instance into a partition I created called _/mnt/mysql_sandboxes_. I&#8217;ve created a subdirectory in there called _tarballs_, which holds the MySQL tarball that we downloaded above which MySQL Sandbox will extract for setup. Since I&#8217;m installing version 5.1.30 I want to call the directory that houses the MySQL data files _5.1.30_single_, but you can call it anything you like. I&#8217;ll create a default user named _jgoulah_ and a password _goulah_. By default it sets the port to the version number without the dots (5130 in this case) so we&#8217;ll give it a custom port so that it listens on 10000 instead.

<pre>mysql_sandbox_2.0.12 $  ./make_sandbox \
/mnt/mysql_sandboxes/tarballs/mysql-5.1.30-linux-i686-icc-glibc23.tar.gz \
--upper_directory=/mnt/mysql_sandboxes/ --sandbox_directory=5.1.30_single \
--db_user=jgoulah --db_password=goulah --sandbox_port=10000</pre>

Here&#8217;s the output:

<pre>unpacking /mnt/mysql_sandboxes/tarballs/mysql-5.1.30-linux-i686-icc-glibc23.tar.gz
Executing ./low_level_make_sandbox \
        --basedir=/mnt/mysql_sandboxes/tarballs/5.1.30 \
        --sandbox_directory=msb_5_1_30 \
        --install_version=5.1 \
        --sandbox_port=5130 \
        --no_ver_after_name \
        --upper_directory=/mnt/mysql_sandboxes/ \
        --sandbox_directory=5.1.30_single \
        --db_user=jgoulah \
        --db_password=goulah \
        --basedir=/mnt/mysql_sandboxes/tarballs/5.1.30 \
        --sandbox_port=10000 \
        --my_clause=log-error=msandbox.err
    The MySQL Sandbox,  version 2.0.12 16-Oct-2008
    (C) 2006,2007,2008 Giuseppe Maxia, Sun Microsystems, Database Group
installing with the following parameters:
upper_directory                = /mnt/mysql_sandboxes/
sandbox_directory              = 5.1.30_single
sandbox_port                   = 10000
datadir_from                   = script
install_version                = 5.1
basedir                        = /mnt/mysql_sandboxes/tarballs/5.1.30
my_file                        =
operating_system_user          = jgoulah
db_user                        = jgoulah
db_password                    = goulah
my_clause                      = log-error=msandbox.err
prompt_prefix                  = mysql
prompt_body                    =  [\h] {\u} (\d) > '
force                          = 0
no_ver_after_name              = 1
verbose                        = 0
load_grants                    = 1
no_load_grants                 = 0
do you agree? ([Y],n) y
loading grants
. sandbox server started
installation options saved to current_options.conf.
To repeat this installation with the same options,
use ./low_level_make_sandbox --conf_file=current_options.conf
----------------------------------------
Your sandbox server was installed in /mnt/mysql_sandboxes//5.1.30_single
</pre>

Its now installed and started up, we can see that the process is running with the correct options:

<pre>$  ps -ef | grep mysql | grep jgoulah
jgoulah  11128     1  0 13:48 pts/3    00:00:00 /bin/sh /mnt/mysql_sandboxes/tarballs/5.1.30/bin/mysqld_safe --defaults-file=/mnt/mysql_sandboxes//5.1.30_single/my.sandbox.cnf
jgoulah  11203 11128  0 13:48 pts/3    00:00:00 /mnt/mysql_sandboxes/tarballs/5.1.30/bin/mysqld --defaults-file=/mnt/mysql_sandboxes//5.1.30_single/my.sandbox.cnf --basedir=/mnt/mysql_sandboxes/tarballs/5.1.30 --datadir=/mnt/mysql_sandboxes//5.1.30_single/data --user=jgoulah --log-error=/mnt/mysql_sandboxes//5.1.30_single/data/msandbox.err --pid-file=/mnt/mysql_sandboxes//5.1.30_single/data/mysql_sandbox10000.pid --socket=/tmp/mysql_sandbox10000.sock --port=10000</pre>

And we can connect to it on the port 10000:

<pre>$  mysql -u jgoulah --protocol=TCP  -P 10000 -pgoulah</pre>

You can also go into the directory where we&#8217;ve installed this and there are some convenience scripts:

<pre>$ cd /mnt/mysql_sandboxes/5.1.30_single/</pre>

You can run the _use_ script to connect into mysql (same thing we just did above except we don&#8217;t have to remember our port, user, or pass):

<pre>$ ./use
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 5
Server version: 5.1.30 MySQL Community Server (GPL)

Type 'help;' or '\h' for help. Type '\c' to clear the buffer.

mysql [localhost] {jgoulah} ((none)) ></pre>

Stop the instance:

<pre>$  ./stop</pre>

Or start it back up again:

<pre>$  ./start
. sandbox server started</pre>

There are a few other scripts which you can experiment with in this directory which are <a href="http://forge.mysql.com/wiki/MySQL_Sandbox#USING_A_SANDBOX" target="_blank">documented here</a>

# Setting up a Replicated Instance

The nice thing about this tool is it will also setup replicated instances of MySQL with a single command. This allows you to test your application under a replicated environment, or even test different replication topologies including multiple slaves or multi-master replication.

We&#8217;ll use similar options as the single instance above, except we&#8217;ll use port 11000 this time (the slaves get port + 1, &#8230;, port + n where n is number of slaves). We&#8217;ll put the install into _/mnt/mysql\_sandboxes/5.1.30\_replicated_. Note this time we use the _make\_replication\_sandbox_ script:

<pre>mysql_sandbox_2.0.12 $  ./make_replication_sandbox \
/mnt/mysql_sandboxes/tarballs/mysql-5.1.30-linux-i686-icc-glibc23.tar.gz \
--upper_directory=/mnt/mysql_sandboxes/ \
--replication_directory=5.1.30_replicated --sandbox_base_port=11000
installing and starting master
installing slave 1
installing slave 2
starting slave 1
. sandbox server started
starting slave 2
. sandbox server started
initializing slave 1
initializing slave 2
replication directory installed on /mnt/mysql_sandboxes//5.1.30_replicated</pre>

Now we have a master and two slaves going. Note that the command to setup replicated sandboxes will not let us specify the user as we did with the single instance, but two users are created by default:

User: root@localhost Password: msandbox
  
User: msandbox@% Password: msandbox

You can run the _use_ script as shown above, or connect directly to the master:

<pre>$  mysql -u msandbox --protocol=TCP  -P 11000 -pmsandbox</pre>

Create a database:

<pre>mysql> create database jg_repl_test;
mysql> exit;</pre>

Connect to one of the slaves:

<pre>$  mysql -u msandbox --protocol=TCP  -P 11001 -pmsandbox
mysql> show databases like '%jg_%';
+------------------+
| Database (%jg_%) |
+------------------+
| jg_repl_test     |
+------------------+
</pre>

And we can see that it has replicated across.

There are various options you can give the _make\_replication\_sandbox_ command, for example you can give it the &#8211;master_master option to setup a multi master instance.

# Conclusion

We&#8217;ve seen how to create a single instance of MySQL and also a replicated master with two slaves. Each of these takes only a few seconds to setup once you have the directory layout decided. There really isn&#8217;t a much easier way to setup scratch instances of MySQL for all kinds of different purposes.