---
title: Ditching Vino for X11vnc
author: jgoulah
type: post
date: 2013-01-01T19:07:30+00:00
url: /2013/01/ditching-vino-for-x11vnc/
categories:
  - Systems
tags:
  - chef
  - cookbook
  - gnome
  - opscode
  - vino
  - x11vnc

---
I&#8217;d been using <a href="http://en.wikipedia.org/wiki/Vino_(VNC_server)" title="vino" target="_blank">gnome vino</a> as a <a href="http://en.wikipedia.org/wiki/Virtual_Network_Computing" title="VNC" target="_blank">VNC server</a> for years on my media computer. This way I can use <a href="https://itunes.apple.com/us/app/touchpad/id297623931?mt=8" title="touchpad" target="_blank">touchpad</a> to control it from my iPad. It works fine, but a little bit clunky and badly documented, plus its tied directly to gnome. The other day I woke up to a filled drive (~2TB) of vino errors. I killed it off and cleaned up the error log file and tried to start it up again. No go this time due to some startup errors. This isn&#8217;t the first time I&#8217;ve fought with it, surely something better exists.

This led me to <a href="http://en.wikipedia.org/wiki/X11vnc" title="X11vnc" target="_blank">X11vnc</a>. A bit of fresh air its fairly easy to setup and get going very quickly. I&#8217;ll first show how to do it manually and then present my open sourced chef cookbook.

## Manual Install

First thing is to install it:

{{< highlight bash >}}sudo apt-get install x11vnc{{< /highlight >}}

Setup a password file:

{{< highlight bash >}}sudo x11vnc -storepasswd YOUR_PASS_HERE /etc/x11vnc.pass{{< /highlight >}}

Create an upstart config:

{{< highlight bash >}}sudo touch /etc/init/x11vnc.conf{{< /highlight >}}

Open it and put this into it:

{{< highlight bash >}}start on login-session-start
script
x11vnc -xkb -noxrecord -noxfixes -noxdamage -display :0 -rfbauth /etc/x11vnc.pass \
 -auth /var/run/lightdm/root/:0 -forever -bg -o /var/log/x11vnc.log
end script
{{< /highlight >}}

and start it up:

{{< highlight bash >}}sudo service x11vnc start{{< /highlight >}}

Simple, easy, and just works. 

## Chef Cookbook

Naturally, I ported this to a <a href="http://community.opscode.com/cookbooks/x11vnc" title="x11vnc" target="_blank">chef cookbook which you can find here</a>. I have to admit I&#8217;m not totally happy with it yet, mainly because it doesn&#8217;t restart x11vnc on config changes. In some cases such as our production servers, we actually prefer this so that we don&#8217;t accidentally roll out a broken config change and have an auto-restart bring everything to its knees (there are some ways to avoid this but thats another post). In any case on my home computer I prefer it to restart if I make changes, but I&#8217;m struggling to get upstart to stop the process correctly due to what seems to be some disassociation with its pid file. The other thing is the recipe currently assumes you are using Ubuntu or something similar, but can easily be extended. Hope this helps someone else out there!
