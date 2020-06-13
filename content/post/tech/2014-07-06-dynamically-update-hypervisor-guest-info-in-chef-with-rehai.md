---
title: Dynamically Update Hypervisor Guest Info in Chef With Rehai
author: jgoulah
type: post
date: 2014-07-06T16:31:09+00:00
url: /2014/07/dynamically-update-hypervisor-guest-info-in-chef-with-rehai/
categories:
  - Cloud Computing
  - Configuration Management
tags:
  - chef
  - configuration management
  - kvm
  - libvirt
  - ohai
  - rehai

---
## Introduction

It turns out this little tool is long overdue, as simple of a concept as it is, but also easy to misunderstand the use cases for, ours at Etsy however was very targeted. Several years ago we were hammering out our internal cloud infrastructure, using KVM/QEMU based solution that you can read about <a href="http://codeascraft.com/2012/03/13/making-it-virtually-easy-to-deploy-on-day-one/" target="_blank">over here</a>. We were populating our virtual machine frontend using <a href="http://docs.opscode.com/ohai.html" target="_blank">Chef Ohai data</a> as the canonical source of our system information. There is an <a href="https://github.com/albertsj1/ohai-plugins/blob/master/kvm_extensions.rb" target="_blank">ohai plugin</a> that gathers KVM virtualization data using <a href="http://libvirt.org/virshcmdref.html" target="_blank">virsh</a> commands which are part of <a href="http://libvirt.org/" target="_blank">libvirt</a>. It was a perfect way to capture information about which guests existed on a given system and other information about them. 

## Our problem

We were hitting a bottleneck in that our chef clients were setup to run about every ten minutes. But within that ten minutes it would be possible that a virtual machine would be added or deleted, and therefore it was difficult to keep our interface in sync. Imagine creating a new virtual machine but not being able to display data about it until you waited around ten minutes, and to make matters worse these clients run at a splay interval, which means they don&#8217;t run all at the same time. Therefore, we started on a simple script that would let us run ohai quickly without needing to do the full chef run. While our chef runs are relatively quick (usually < 1 minute) it would introduce problems if we try to run chef while the client is already running. 

## Going to open source

It was supposed to be released a while ago, but has taken some time for various reasons. It&#8217;s a shockingly tiny amount of code but there were some barriers to releasing it. The majority of the code was written by <a href="https://twitter.com/digitallogic" target="_blank">Mark Roddy</a> but he&#8217;d turned it over to me to open source. I went through the normal chef contribution process, which at the time required opening a <a href="https://tickets.opscode.com/browse/CHEF-4732" target="_blank">Jira ticket</a> that you can read if you&#8217;re interested in some of the details. In short there were some questions about the use case but when I explained that we weren&#8217;t trying to re-invent graphite in some horrible way, we were able to agree there could be some real world use cases. That being said, it was not accepted into the core yet because this does introduce very small race conditions since chef uses a read-modify-save model of changing the attribute data. There is <a href="https://github.com/opscode/chef-rfc/blob/attribute-consistency-fixes/rfc006-attribute-consistency.md" target="_blank">a proposal</a> to fix this, which divides attributes into different levels in which automated updates can access them without causing this issue. However in the wild this has not actually posed an issue for us even with several hundred nodes running it.

## Installation

If you&#8217;re interested in this tool, you can install it with ruby gems using the command:

{{< highlight bash >}}% gem install rehai
{{< /highlight >}}

If you&#8217;d like to see the source, head on over to <a href="https://github.com/jgoulah/rehai" target="_blank">github</a>.
