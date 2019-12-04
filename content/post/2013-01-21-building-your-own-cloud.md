---
title: Building Your Own Cloud From Scratch
author: jgoulah
type: post
date: 2013-01-22T01:50:23+00:00
url: /2013/01/building-your-own-cloud/
categories:
  - Cloud Computing
  - Systems
tags:
  - cloud
  - debian
  - kvm
  - libvirt
  - qemu
  - virt-install

---
## Intro

There are a lot of private <a href="http://www.openstack.org/" title="openstack" target="_blank">cloud</a> <a href="http://www.eucalyptus.com/" title="eucalyptus" target="_blank">solutions</a> out there with great things built into them already to complete a full cloud stack &#8211; networking, dashboards, storage, and a framework that puts all the pieces together, amongst other things. But there is also a decent amount of overhead to getting these frameworks setup, and maybe you want more flexibility over some of the components, or even just something a little more homegrown. What might a lightweight cloud machine bootstrapping process look like if it where implemented from scratch?

## Getting Started

We can use <a href="http://libvirt.org/" title="libvirt" target="_blank">libvirt</a> and <a href="http://wiki.qemu.org/KVM" title="kvm-qemu" target="_blank">KVM/QEMU</a> to put something reasonably robust together, start by installing those packages:

<pre>apt-get install qemu-kvm libvirt libvirt-bin virtinst virt-viewer</pre>

The next important thing is to setup a <a href="http://wiki.debian.org/BridgeNetworkConnections#Libvirt_and_bridging" title="libvirt and bridging" target="_blank">bridge</a> for proper networking on this host. This will allow the guests to use the bridge to communicate on the same network. There should be a few articles <a href="http://wiki.libvirt.org/page/Networking#Altering_the_interface_config" target="_blank">out there</a> that can help you set this up, but the basics are that you want your bridge assigned the IP that your eth0 interface previously had, and then add the eth0 interface to the bridge. In this example _192.168.1.101_ is the IP of the host machine:

<pre># cat /etc/network/interfaces
auto lo
iface lo inet loopback

iface eth0 inet manual

auto br0
iface br0 inet static
  address 192.168.1.101
  netmask 255.255.255.0
  gateway 192.168.1.1
  network 192.168.1.0
  bridge_ports eth0
  
ifup br0
</pre>

## Building the Image

The first step is setting up a base template that you create your instances from. So grab an iso to start from, we&#8217;ll use debian, but this process works with any distro:

<pre>% wget http://cdimage.debian.org/debian-cd/6.0.6/amd64/iso-cd/debian-6.0.6-amd64-netinst.iso</pre>

And allocate a file on disk to the size you&#8217;d like your template to be. I created one here at 8GB, it can always be expanded later, so this should only need to be big enough to hold the initial base image that all instances will start from. Generally smaller is better because of the copy step when instances get created later. 

<pre>% dd if=/dev/zero of=/var/lib/libvirt/images/debbase.img bs=1M count=8192</pre>

Now you can start the linux installation, noting the _&#8211;graphics_ args for the ability to connect with VNC. Our installation target disk is the one we created above, _debbase.img_, and we are giving it 512M RAM and 1 CPU. 

<pre>% virt-install --name=virt-base-deb --ram=512 --graphics vnc,listen=0.0.0.0  --network=bridge=br0 \
--accelerate --virt-type=kvm --vcpus=1 --cpuset=auto --cpu=host --disk /var/lib/libvirt/images/debbase.img \
--cdrom debian-6.0.6-amd64-netinst.iso</pre>

Once thats started up you can use VNC on your client machine to connect to this instance graphically and run through the normal install setup. There are <a href="http://en.wikipedia.org/wiki/Virtual_Network_Computing#See_also" title="vnc clients" target="_blank">plenty of clients</a> out there but a decent one is <a href="http://sourceforge.net/projects/cotvnc/" title="chicken of the vnc" target="_blank">Chicken of the VNC</a>. Its also possible at this step that you&#8217;d create the image off a [PXE boot][1] or similar bootstrapping mechanism.

#### Extract the Partition

Here we take advantage of QEMU ability to load Linux kernels and init ramdisks directly, thereby circumventing bootloaders such as GRUB. It then can be launched with the physical partition containing the root filesystem as the virtual disk.

There are two steps to make this work. First you&#8217;ll need the vmlinuz and initrd files, and the easiest way to get those is to copy them from the base image we setup above:

<pre>% scp BASEIP:/boot/vmlinuz-2.6.32-5-amd64 /var/lib/libvirt/kernels/
% scp BASEIP:/boot/initrd.img-2.6.32-5-amd64 /var/lib/libvirt/kernels/
</pre>

The next step is to extract the root partition from that same base image. We want to take a look at how those partitions are laid out so that we can get the right numbers to pass to the <a href="http://en.wikipedia.org/wiki/Dd_(Unix)" title="dd" target="_blank">dd</a> command. 

<pre>% sfdisk -l -uS /var/lib/libvirt/images/debbase.img

Disk /var/lib/libvirt/images/debbase.img: 1044 cylinders, 255 heads, 63 sectors/track
Warning: extended partition does not start at a cylinder boundary.
DOS and Linux will interpret the contents differently.
Units = sectors of 512 bytes, counting from 0

   Device Boot    Start       End   #sectors  Id  System
/var/lib/libvirt/images/debbase.img1   *      2048  15988735   15986688  83  Linux
/var/lib/libvirt/images/debbase.img2      15990782  16775167     784386   5  Extended
/var/lib/libvirt/images/debbase.img3             0         -          0   0  Empty
/var/lib/libvirt/images/debbase.img4             0         -          0   0  Empty
/var/lib/libvirt/images/debbase.img5      15990784  16775167     784384  82  Linux swap / Solaris
</pre>

We are going to pull the first partition out, note how the numbers line up to the first line corresponding to _debbase.img1_ line. We start at sector 2048 and get 15986688 sectors of 512 bytes each:

<pre>% dd if=/var/lib/libvirt/images/debbase.img of=/var/lib/libvirt/debian-tmpl skip=2048 count=15986688 bs=512</pre>

#### Templatize the Image

Now we have a disk file that serves as our image template. There&#8217;s a few things we want to change directly on this template. Note that we are using a few all caps placeholders ending in _-TMPL_ that we&#8217;ll replace later with sed. We can edit the templates files by mounting the disk:

<pre>% mkdir -p /tmp/newtmpl
% mount -t ext3 -o loop /var/lib/libvirt/debian-tmpl /tmp/newtmpl
% chroot /tmp/newtmpl
</pre>

Note at this point we are <a href="http://en.wikipedia.org/wiki/Chroot" title="chroot" target="_blank">chrooted</a> and these commands are acting against our template disk file. 

Clear out the old IPs tied to our NIC when the base image networking was setup:

<pre>% echo "" > /etc/udev/rules.d/70-persistent-net.rules
</pre>

We&#8217;re going to put a placeholder for our hostname in _/etc/hostname_:

<pre>% echo "HOSTNAME-TMPL" > /etc/hostname
</pre>

Set a nameserver template in _/etc/resolv.conf_:

<pre>% echo "nameserver NAMESERVER-TMPL" > /etc/resolv.conf 
</pre>

In the file _/etc/network/interfaces_:

<pre># The loopback network interface
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
address ADDRESS-TMPL
netmask NETMASK-TMPL
gateway GATEWAY-TMPL
</pre>

This will give us console access when we boot it. Make sure _/etc/inittab_ has this line (usually just uncomment it):

<pre>T0:23:respawn:/sbin/getty -L ttyS0 9600 vt100
</pre>

## Creating an Instance

Now we have all the pieces together to launch an instance from our image. This script will create the instance given the IP and hostname. It does no error checking for readability reasons, and is well commented so that you know whats going on:

<pre>#!/bin/bash

# read in '&lt;ip> &lt;host>' from command line
virt_ip=$1
virt_host=$2

# build the fqdn based off the short host name
virt_fqdn=${virt_host}.linux.bogus

# fill in your network defaults
virt_gateway=192.168.1.1
virt_netmask=255.255.225.0
virt_nameserver=192.168.1.101

# how the disk/ram/cpu is sized
virt_disk=10G
virt_ram=512
virt_cpus=1

# random mac address
virt_mac=$(openssl rand -hex 6 | sed 's/\(..\)/\1:/g; s/.$//')

cp /var/lib/libvirt/images/debian-tmpl /var/lib/libvirt/images/${virt_host}-disk0

# optionally resize the disk
qemu-img resize /var/lib/libvirt/images/${virt_host}-disk0 ${virt_disk}
loopback=`losetup -f --show /var/lib/libvirt/images/${virt_host}-disk0`
fsck.ext3 -fy $loopback
resize2fs $loopback ${virt_disk}
losetup -d $loopback

mountbase=/tmp/${virt_host}
mkdir -p ${mountbase}
mount -o loop /var/lib/libvirt/images/${virt_host}-disk0 ${mountbase}

# replace our template vars
sed -i -e "s/ADDRESS-TMPL/$virt_ip/g" \
       -e "s/NETMASK-TMPL/$virt_netmask/g" \
       -e "s/GATEWAY-TMPL/$virt_gateway/g" \
       -e "s/HOSTNAME-TMPL/$virt_fqdn/g" \
       -e "s/NAMESERVER-TMPL/$virt_nameserver/g" \
  ${mountbase}/etc/network/interfaces \
  ${mountbase}/etc/resolv.conf \
  ${mountbase}/etc/hostname

# unmount and remove the tmp files
umount /tmp/${virt_host}
rm -rf /tmp/${virt_host}*

# run a file system check on the disk
fsck.ext3 -pv /var/lib/libvirt/images/${virt_host}-disk0

# specify the kernel and initrd (these we copied with scp earlier)
vmlinuz=/var/lib/libvirt/kernels/vmlinuz-2.6.32-5-amd64
initrd=/var/lib/libvirt/kernels/initrd.img-2.6.32-5-amd64

# install the new domain with our specified parameters for cpu/disk/memory/network
virt-install --name=$virt_host --ram=$virt_ram \
--disk=path=/var/lib/libvirt/images/${virt_host}-disk0,bus=virtio,cache=none \
--network=bridge=br0 --import --accelerate --vcpus=$virt_cpus --cpuset=auto --mac=${virt_mac} --noreboot --graphics=vnc \
--cpu=host --boot=kernel=$vmlinuz,initrd=$initrd,kernel_args="root=/dev/vda console=ttyS0 _device=eth0 \
_ip=${virt_ip} _hostname=${virt_fqdn} _gateway=${virt_gateway} _dns1=${virt_nameserver} _netmask=${virt_netmask}"

# start it up
virsh start $virt_host
</pre>

assuming we named it _buildserver_, run the above like:

<pre>% buildserver 192.168.1.197 jgoulah
</pre>

## Conclusion

This is really just the first step, but now that you can bring a templated disk up you can decide a little more about how you&#8217;d like networking to work for your cloud. You can either continue to use static IP assignment as shown here, and use <a href="http://en.wikipedia.org/wiki/Nsupdate" title="nsupdate" target="_blank">nsupdate</a> to insert dns entries when new guests come up, or you can set things up such that the base image uses dhcp, and you can <a href="http://lani78.wordpress.com/2012/07/23/make-your-dhcp-server-dynamically-update-your-dns-records-on-ubuntu-12-04-precise-pangolin/" target="_blank">configure your dhcp server</a> <a href="http://www.held.org.il/blog/2011/01/make-dhcp-auto-update-dynamic-dns/" target="_blank">to update records in dns</a> when clients come online. You may also want to bake your favorite config management system into the template so that you can bootstrap the nodes and maintain configurations on them. Have fun!

 [1]: http://en.wikipedia.org/wiki/Preboot_Execution_Environment