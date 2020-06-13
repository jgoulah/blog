---
title: 'Quick Tip: Find the QEMU VM to Virtual Interface Mapping'
author: jgoulah
type: post
date: 2012-12-27T21:13:26+00:00
url: /2012/12/find-the-qemu-to-virtual-interface-mapping/
categories:
  - Debugging
  - Systems
tags:
  - kvm
  - libvirt
  - qemu
  - virsh
  - virtual interface
  - vnet

---
The other day we were getting some messages on our network switch that a host was flapping between ports. It turns out we had two virtual hosts on different machines using the same MAC address (not good). We had the interface and MAC information, and wanted to find what vm domains mapped to these. Its easy if you know the domain name of the vm node, you can get back the associated information like so:

{{< highlight bash >}}% sudo virsh domiflist goulah  
Interface  Type       Source     Model       MAC
-------------------------------------------------------
vnet2      bridge     br0        -           52:54:00:19:40:18
{{< /highlight >}}

But if you only have the virtual interface, how would you get the domain? I remembered virsh dumpxml will show all of the dynamic properties about the VM, including those which are not available in the static XML definition of the VM in _/etc/libvirt/qemu/foo.xml_. Which vnetX interface is attached to which VM is one of these additional dynamic properties. So we can grep for this! I concocted up a simple (not very elegant) function which given the vnet interface will return the domain associated to it:

{{< highlight bash >}}function find-vnet() { for vm in $(sudo virsh list | grep running | awk '{print $2}'); do sudo virsh dumpxml $vm|grep -q "$1" && echo $vm; done }
{{< /highlight >}}

It just looks through all the running domains and prints the domain name if it finds the interface you&#8217;re looking for. So now you can do:

{{< highlight bash >}}% find-vnet vnet2
goulah
{{< /highlight >}}

I wonder if others out there have a clever way of doing this or if this is really the best way? If you know of a better way leave a comment. Perhaps the problem is not common enough that a libvirt built-in command exists.
