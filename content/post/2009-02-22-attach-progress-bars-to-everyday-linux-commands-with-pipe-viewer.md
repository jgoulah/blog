---
title: Attach Progress Bars to Everyday Linux Commands With Pipe Viewer
author: jgoulah
type: post
date: 2009-02-23T02:45:31+00:00
url: /2009/02/attach-progress-bars-to-everyday-linux-commands-with-pipe-viewer/
categories:
  - Systems
tags:
  - linux
  - pipe viewer

---
# Introduction

A lot of Linux command line utilities don&#8217;t give any indication of progress in terms of when the job is going to finish. I&#8217;m going to take a couple of commands that I use on a near daily basis and show how to output some progress using the <a href="http://www.ivarch.com/programs/pv.shtml" target="_blank">pipe viewer</a> utility. This is a handy tool that you can insert between pipes in your shell commands that will give you a visual indication of how long its going to take for the command to finish.

# Getting Started

There isn&#8217;t much to installing this tool with your favorite package manager, so just make sure you do that first using yum or apt.

The easiest way to show what pv does is by creating a simple example. We can simply gzip a file and show how much data is being processed and how long it will take to finish. 

<pre>$ pv somefile | gzip > somefile.log.gz
64.1MB 0:00:03 [10.1MB/s] [==========>                        ] 32% ETA 0:00:06</pre>

So here can see we have processed 64.1MB in 3 seconds at a rate of about 10.1MB/s and should take about 6 seconds to finish.

# Chaining PV to Display Different Inputs and Outputs

Another really cool thing is you can chain these pv commands to see the progress of data being read off the disk and then how fast gzip is compressing it. 

<pre>$ pv -cN source_stream somefile | gzip | pv -cN gzip_stream > somefile.log.gz
source_stream: 89.6MB 0:00:06 [10.1MB/s] [============>       ] 45% ETA 0:00:07
gzip_stream:   14.8MB 0:00:06 [ 3.2MB/s] [       &lt;=>          ]
</pre>

In this example we created named streams with -N which you can call whatever you want. In this example they are called _source_stream_ and _gzip_stream_ since that&#8217;s what they are measuring. The -c parameter is recommended when running multiple pv processes in the same pipeline and prevents the processes from mangling each others output.

Its important also to point out here that we get a percentage on the source_stream because pv knows how much data exists in that file, but the second stream doesn&#8217;t know how much data will pass through gzip and so it can only display how much data has passed through so far.

# Using PV on Other Commands

Another example is using tar to compress a folder, but if we do something like this:

<pre>$ tar cjf - some_directory | pv > out.tar.bz2
3.55MB 0:00:02 [ 1.2MB/s] [   &lt;=>          ]</pre>

the output isn&#8217;t very interesting because pv doesn&#8217;t know the total amount of data its compressing. We can provide the size with the -s option to pv, and a little bash interpretation to grab the size

<pre>tar -cf - some_directory | \
     pv -s $(du -sb some_directory | awk '{print $1}') | \
     bzip2 > somedir.tar.bz2
1.39MB 0:00:03 [ 929kB/s] [==>     ]  6% ETA 0:00:45</pre>

So we are telling tar to create an archive of _some_directory_, print the output to stdout, next tell pv the size of that directory using the du command on it, and send that to bzip2 for compression.

### Create a Shell Script For Easy Reuse

At this point the command above gets to be way too much to remember. We&#8217;ve also had to type the directory name twice which is silly. I always put my ~/bin directory in my PATH so that I can throw easy scripts in there. Here&#8217;s one that will take a directory argument and compress it with progress

<pre>#!/bin/sh

if [ -z $1 ]; then
    echo "Usage: $0 dir"
    exit 1
fi

dir="${1%/*}"

if [ ! -e $dir ]; then
  echo "$dir doesn't exist"
  exit 1
fi

echo "compressing $dir"
tar -cf - $dir | pv -s $(du -sb $dir | awk '{print $1}') | bzip2 > $dir.tar.bz2
</pre>

So we&#8217;ve just taken what we&#8217;ve learned above and created a script that takes an argument, which needs to be a directory (or file) that exists, and the output is that directory name with a tar.bz2 extension.

### Seeing Progress In a MySQL Import

We can use the same exact concept to chain our tools to get a progress bar on a MySQL data import. Here it is

<pre>pv myschema.sql | \
      pv -s $(du -sb myschema.sql | awk '{print $1}') | \
      mysql -u db_user my_database</pre>

As shown above we can easily turn this into a script that can be reused. 

# Conclusion

We&#8217;ve seen how to take a simple tool called pipe viewer which gives information on the data fed to it and attached it to some commands we currently use. Since the commands can be tricky to remember we&#8217;ve seen how to wrap them in a simple shell script. Now you can experiment with using it on other commands that you frequently use and want to see progress output, which can be a really useful thing when dealing with large amounts of data.