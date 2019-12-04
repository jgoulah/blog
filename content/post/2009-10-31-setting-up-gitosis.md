---
title: Setting up Gitosis
author: jgoulah
type: post
date: 2009-10-31T21:33:16+00:00
url: /2009/10/setting-up-gitosis/
categories:
  - Version Control
tags:
  - distributed
  - Git
  - gitosis
  - how to
  - repository
  - setup gitosis
  - Version Control

---
## Overview

This article is part one of a two part series that covers setting up a hosting server using gitosis for your central repository, and in the next article, taking an existing SVN repository and running the appropriate scripts and commands necessary to migrate it into something git can work with. 

So this article is how to setup and manage a git repository.There are some great [services][1] out there than can do this for you, but why pay money for something you can easily do for free? This article shows how to setup and manage a secure and private git repository that people can use as a central sharing point.

## Setting Up Gitosis

[Gitosis][2] is a tool for hosting git repositories. Its common usage is for a central repository that other developers can push changes to for sharing. 

First clone the gitosis repository and run the basic python install. You just need the python setuptools package

<pre>sudo apt-get install python-setuptools
</pre>

And then you can easily install it

<pre>git clone git://eagain.net/gitosis.git
cd gitosis
sudo python setup.py install</pre>

Next you need to create a user that will own the repositories you want to manage. You can put its home directory wherever you want, but in this example we&#8217;ll put it in the standard _/home_ location.

<pre>sudo adduser \
    --system \
    --shell /bin/sh \
    --gecos 'git version control' \
    --group \
    --disabled-password \
    --home /home/git \
    git</pre>

Then you must create an ssh public key (or use your existing one) for your first repository user. We&#8217;ll use an init command to copy it to server and load it. If you don&#8217;t have a public key you can create one with _ssh-keygen_ like so

<pre>ssh-keygen -t dsa</pre>

Then _gitosis-init_ is for the first time only, loads up your users key, and goes like this

<pre>sudo -H -u git gitosis-init &lt; ~/.ssh/id_dsa.pub</pre>

Here it doesn't hurt to make sure your post-update hook has execute permissions. 

<pre>sudo chmod 755 /home/git/repositories/gitosis-admin.git/hooks/post-update</pre>

Now you can clone the gitosis-admin repository, which is used to manage our repository permissions. 

<pre>git clone git@YOUR_SERVER_HOSTNAME:gitosis-admin.git
cd gitosis-admin</pre>

Now you can see you have a _gitosis.conf_ file and a _keydir_ directory

<pre>$ ls -l
total 8
-rw-r--r-- 1 jgoulah mygroup   83 2009-10-31 20:44 gitosis.conf
drwxr-xr-x 2 jgoulah mygroup 4096 2009-10-31 20:44 keydir
</pre>

The _gitosis.conf_ file holds group and permission information for your repositories, and the _keydir_ folder holds your public keys. 

If I look in there I see my public key was imported from our earlier _gitosis-init_ command

<pre>$ ls -l keydir/
total 4
-rw-r--r-- 1 jgoulah mygroup 603 2009-10-31 20:44 jgoulah.pub
</pre>

So open up _gitosis.conf_ and you should already see you have an entry for the gitosis-admin repository that we just cloned. The gitosis-init command above setup the access for us. From now on we can just crack open gitosis.conf and edit the permissions, commit and push back to our central repository.

If I wanted to create a new project for a repository called pizza_maker it would look something like this. 

<pre>[group myteam]
members = jgoulah
writable = pizza_maker
</pre>

Don't forget the members section is the name of your public key file without the .pub at the end. If your key was named XYZ.pub then your member line would have XYC here.

<pre>git commit -a -m "Create new repo permissions for pizza_maker project"
git push</pre>

As a reminder the second part of this series will show an svn to git import. For now lets assume we are starting from scratch. We'd create our project like this

<pre>cd && mkdir pizza_maker
cd pizza_maker
git init
git remote add origin git@YOUR_SERVER_HOSTNAME:pizza_maker.git
git add *
git commit -m "some stuff"
git push origin master:refs/heads/master</pre>

The only other thing to know is if you want to grant another user access to your repository. All you have to do is add their public key to the _keydir_ folder, and then give the user permissions by modifying _gitosis.conf_

<pre>cd gitosis-admin
cp ~/otherdude.pub keydir/</pre>

<pre>[group myteam]
- members = jgoulah
+ members = jgoulah otherdude
  writable = pizza_maker</pre>

If you need to, you can also grant public access over the git:// protocol like so

<pre>sudo -u git git-daemon --base-path=/home/git/repositories/ --export-all
</pre>

Then someone can clone like

<pre>git clone git://YOUR_SERVER_HOSTNAME/pizza_maker.git
</pre>

## Conclusion

This article showed how to setup gitosis, how to initialize your gitosis-admin repository, which is a unique concept in itself to use a repository to manage repositories, and it works rather well. We also went over how to create our own new git repository, and how to manage the access permissions through _gitosis.conf_. Part two of this series will explain how to port from your current SVN setup to a Git setup. This article was a prerequisite if you want to host your own private repository when you're converting from SVN to Git, and thats what we'll look at next time.

 [1]: http://github.com/
 [2]: http://eagain.net/gitweb/?p=gitosis.git