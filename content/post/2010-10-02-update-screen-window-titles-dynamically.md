---
title: 'Quick Tip: Dynamically Updating Screen Window Titles With The Current Server Hostname'
type: post
date: 2010-10-02T17:13:05+00:00
url: /2010/10/update-screen-window-titles-dynamically/
categories:
  - Organization
  - SSH
  - Systems
tags:
  - hostname
  - screen
  - SSH
  - window titles

---
I haven&#8217;t had a ton of time for blogging lately but figured this tip was good enough to throw out there for all the <a href="http://en.wikipedia.org/wiki/GNU_Screen" target="_blank">screen</a> users. One way I like to organize servers that I&#8217;m ssh&#8217;d into is using screen windows. As you hopefully know you can use _Ctrl-A c_ to create sub windows within screen. Then you can switch between them in several ways such as using _Ctrl-A X_ where X is the window number, _Ctrl-A n_ or _Ctrl-A p_ for next and previous, and _Ctrl-A &#8220;_ to get a list of the windows for selection. You can move windows around with _Ctrl-A :_ then type _number X_ where X is the window you want to swap with. Finally, you can also name the windows with _Ctrl-A A_. So usually I ssh into a server, and manually change the window title to the server name I&#8217;m ssh&#8217;d into so that its easy to organize and remember where my windows are.

Turns out thats a lot of repetitive work that can easily be scripted. You can create a really simple script called _ssh-to_ and place in your _~/bin_ or somewhere in your path

<pre>#!/bin/bash

hostname=`basename $0`

# set screen title to short hostname
echo -n -e "\033k${hostname%%.*}\033\134"

# ssh to the server with any additional args
ssh -X $hostname $*

# set hostname back when session is done (-s on osx)
echo -n -e "\033k`hostname -s`\033\134"
</pre>

Now, create symbolic links to the script with each server hostname that you use. This can be a little tedious but you only have to do it once and the benefit is that you can now tab complete ssh&#8217;ing into servers. 

For example you would do something like
  
`<br />
 ln -s ssh-to myserver.com<br />
` 

Then anytime you type &#8220;myserver.com&#8221; the tab completion can fill it out, you&#8217;re ssh&#8217;d into the server (assuming you have keys setup you don&#8217;t have to type a password) and your screen window title is updated with the domain info trimmed off, in this case _myserver_. Now your screen windows update their own titles anytime you ssh into a new server!