---
title: Using ddclient To Keep Your Dynamic Home IP Mapped to a Hostname
author: jgoulah
type: post
date: -001-11-30T00:00:00+00:00
draft: true
url: /?p=378
categories:
  - Uncategorized

---
http://sourceforge.net/projects/ddclient/
  
http://www.dyndns.com/support/clients/unix.html

sudo apt-get install libio-socket-ssl-perl

cp ddclient /usr/sbin/
  
mkdir /etc/ddclient

sudo mkdir /var/cache/ddclient

cp sample-etc_ddclient.conf /etc/ddclient/ddclient.conf