---
title: Compile the Linux Kernel and Create Distributable Debian Packages
author: jgoulah
type: post
date: 2009-01-30T23:04:55+00:00
url: /2009/01/compile-the-linux-kernel-and-create-distributable-debian-packages/
categories:
  - Deployment
  - Kernel
tags:
  - 2.6.28
  - compile
  - debian
  - ext4
  - Kernel
  - linux
  - minstrel
  - source

---
# Introduction

Compiling a [kernel][1] is actually a fairly easy thing to do these days. I&#8217;m going to go over how to do this on a [Debian][2] box since that happens to be my distro of choice. This will work just as well on Ubuntu. You can always wait for the packaged version, but you&#8217;ll always be a little behind some of the cutting edge features. This method allows you to get the latest upgrades that are incorporated into the kernel, or even to apply cutting edge kernel patches against the kernel source. 

# Getting the Source

You can always find the kernel at kernel.ftp.org. Login as _anonymous_ and with your email address as the password:

<pre>$ ftp ftp.kernel.org
Connected to pub.us.kernel.org.
220 Welcome to ftp.kernel.org.
Name (ftp.kernel.org:jgoulah): anonymous
331 Please specify the password. {email address}
Password: 
</pre>

Change directories into the 2.6.x series

<pre>ftp> cd pub/linux/kernel/v2.6</pre>

We want linux-2.6.28.tar.bz2, which is the newest at the time of this article

<pre>ftp> binary
200 Switching to Binary mode.
ftp> get linux-2.6.28.tar.bz2
ftp> exit</pre>

Now you have the kernel.

You may also need these tools depending what you&#8217;ve installed so far

<pre>apt-get install kernel-package libncurses5-dev fakeroot wget bzip2 build-essential</pre>

# Extract and Configure the Source

We&#8217;ll put the tarball into _/usr/src_

<pre>$ sudo mv linux-2.6.28.tar.bz2 /usr/src/</pre>

Extract it

<pre>$ cd /usr/src
$ sudo tar xjf linux-2.6.28.tar.bz2</pre>

Its good measure to point a symlink to your current kernel

<pre>$ sudo ln -s linux-2.6.28 linux</pre>

And change into the directory

<pre>cd /usr/src/linux</pre>

If you have any patches, now is the time to install them

<pre>bzip2 -dc /usr/src/patch.bz2 | patch -p1</pre>

Clean things up

<pre>make clean && make mrproper</pre>

Now we can finally configure the kernel. Its a really smart idea to copy your existing configuration into the current kernel as a starting point. You certainly don&#8217;t want to lose any of your current modules. 

<pre>$ sudo cp /boot/config-`uname -r` .config</pre>

There is one more step to load in your old settings

$ sudo make menuconfig

Now select _Load an Alternate Configuration File_
  
Enter your config file _.config_ when it prompts you

When you exit out make sure to save and then you can do a diff against your old config and see the new kernel options:

<pre>$ diff /boot/config-`uname -r` .config</pre>

You can go back into menuconfig to make any changes necessarily, which is typically some new module you&#8217;d like to try out. For this kernel version I&#8217;m at least enabling [ext4][3] and [minstrel][4].

# Compiling the Kernel

<pre>$ sudo make-kpkg clean</pre>

On this command you will want to set the string that gets appended to the version in the new kernel name. I usually just do something like -custom-buildX where X is the number of times I&#8217;ve changed configurations on this kernel version and rebuilt it. You can name it whatever you like as long as it begins with a minus (-) and doesn&#8217;t contain spaces

<pre>$ sudo fakeroot make-kpkg --initrd \ 
--append-to-version=-custom-build1 kernel_image kernel_headers</pre>

Go get a sandwich or something, depending on your computer this can take a while.

# Installing the Kernel

The cool part about this is we&#8217;ve just created two .deb files that can be installed on other Debian servers, no re-compilation necessary. The files will look something like this, given the above parameter to the _append-to-verson_ option from above

<pre>linux-headers-2.6.28-custom-build1_2.6.28-custom-build1-10.00.Custom_i386.deb
linux-image-2.6.28-custom-build1_2.6.28-custom-build1-10.00.Custom_i386.deb</pre>

So install them like a regular Debian package

<pre>$ sudo dpkg -i linux-image-2.6.28-custom-build1_2.6.28-custom-build1-10.00.Custom_i386.deb
Selecting previously deselected package linux-image-2.6.28-custom-build1.
(Reading database ... 418301 files and directories currently installed.)
Unpacking linux-image-2.6.28-custom-build1 (from linux-image-2.6.28-custom-build1_2.6.28-custom-build1-10.00.Custom_i386.deb) ...
Done.
Setting up linux-image-2.6.28-custom-build1 (2.6.28-custom-build1-10.00.Custom) ...
Running depmod.
Finding valid ramdisk creators.
Using mkinitramfs-kpkg to build the ramdisk.
Other valid candidates: mkinitramfs-kpkg mkinitrd.yaird
Running postinst hook script /sbin/update-grub.
You shouldn't call /sbin/update-grub. Please call /usr/sbin/update-grub instead!

Searching for GRUB installation directory ... found: /boot/grub
Searching for default file ... found: /boot/grub/default
Testing for an existing GRUB menu.lst file ... found: /boot/grub/menu.lst
Searching for splash image ... none found, skipping ...
Found kernel: /boot/vmlinuz-2.6.28-custom-build1
Found kernel: /boot/vmlinuz-2.6.26-custom-2.6.26
Found kernel: /boot/vmlinuz-2.6.26-custom-build7
Found kernel: /boot/vmlinuz-2.6.26-custom-build6
Found kernel: /boot/vmlinuz-2.6.26-custom-build5
Found kernel: /boot/vmlinuz-2.6.26-custom-build4
Found kernel: /boot/vmlinuz-2.6.26-custom-build3
Found kernel: /boot/vmlinuz-2.6.26-custom-build2
Found kernel: /boot/vmlinuz-2.6.24.4-custom13
Found kernel: /boot/vmlinuz-2.6.24.4-custom12
Found kernel: /boot/vmlinuz-2.6.24.4-custom11
Found kernel: /boot/vmlinuz-2.6.24.4-custom10
Found kernel: /boot/vmlinuz-2.6.24.4-custom9
Found kernel: /boot/vmlinuz-2.6.24.4-custom8
Found kernel: /boot/vmlinuz-2.6.24.4-custom7
Found kernel: /boot/vmlinuz-2.6.24.4-custom6
Found kernel: /boot/vmlinuz-2.6.24.4-custom5
Found kernel: /boot/vmlinuz-2.6.24.4-custom4
Found kernel: /boot/vmlinuz-2.6.24.4-custom3
Found kernel: /boot/vmlinuz-2.6.24.4-custom2
Found kernel: /boot/vmlinuz-2.6.24.4-custom
Found kernel: /boot/vmlinuz-2.6.18-6-686
Updating /boot/grub/menu.lst ... done</pre>

And the kernel headers

<pre>$ sudo dpkg -i linux-headers-2.6.28-custom-build1_2.6.28-custom-build1-10.00.Custom_i386.deb
Selecting previously deselected package linux-headers-2.6.28-custom-build1.
(Reading database ... 418509 files and directories currently installed.)
Unpacking linux-headers-2.6.28-custom-build1 (from linux-headers-2.6.28-custom-build1_2.6.28-custom-build1-10.00.Custom_i386.deb) ...
Setting up linux-headers-2.6.28-custom-build1 (2.6.28-custom-build1-10.00.Custom) ...</pre>

That&#8217;s pretty much it. You can look at your grub config

<pre>$ vim /boot/grub/menu.lst</pre>

Scroll down and you&#8217;ll see an entry for your new kernel. The topmost entry is the default, but remember you can also choose a different kernel at boot

<pre>title       Debian GNU/Linux, kernel 2.6.28-custom-build1
root        (hd1,0)
kernel      /boot/vmlinuz-2.6.28-custom-build1 root=/dev/sdb1 ro
initrd      /boot/initrd.img-2.6.28-custom-build1
savedefault</pre>

You can see the correct files installed into _/boot_

<pre>$ ls -al /boot/*2.6.28*
-rw-r--r-- 1 root root   63928 2009-01-28 23:56 /boot/config-2.6.28-custom-build1
-rw-r--r-- 1 root root 1958212 2009-01-30 17:34 /boot/initrd.img-2.6.28-custom-build1
-rw-r--r-- 1 root root 1173217 2009-01-29 00:07 /boot/System.map-2.6.28-custom-build1
-rw-r--r-- 1 root root 2899952 2009-01-29 00:07 /boot/vmlinuz-2.6.28-custom-build1</pre>

We are done, reboot

$ sudo shutdown -r now

# Conclusion

We&#8217;ve seen in how just a few commands a new kernel can be configured and installed with some additional options while keeping the current configuration. Not only that, but we&#8217;ve produced Debian package files that can be installed onto other machines. This is one easy way to upgrade your kernel across many servers without having to wait for your vendor to release it.

 [1]: http://www.kernel.org/
 [2]: http://www.debian.org/
 [3]: http://en.wikipedia.org/wiki/Ext4
 [4]: http://linuxwireless.org/en/developers/Documentation/mac80211/RateControl/minstrel