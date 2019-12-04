---
title: MySQL Snapshots using LVM
author: jgoulah
type: post
date: 2009-01-22T19:50:50+00:00
url: /2009/01/mysql-snapshots-using-lvm/
categories:
  - Databases
tags:
  - MySQL backup LVM snapshot

---
# Introduction

LVM snapshots are a quick and easy way to take a backup of your MySQL server&#8217;s data files. The idea is to acquire a read lock on all tables, flush all server caches to disk, and create a snapshot of the volume containing the MySQL data directory, then unlock the tables again. This gives an atomic snapshot that takes only milliseconds. A mysql instance can then be launched to point at this data directory and you can perform a mysqldump to get data for setting up slaves, or you can take a completely destroyed DB and use the data directory to restore it to this point in time.

# Setup the LVM

Figure out the partition you want to put the LVM on, in our case we&#8217;ll use /data2

<pre>$ df -h | grep data2
/dev/sda2             371G  195M  352G   1% /data2</pre>

We&#8217;ll need to run fdisk on it.  If you have data on here, save it to somewhere else!

<pre>$ sudo fdisk /dev/sda2
Device contains neither a valid DOS partition table, nor Sun, SGI or OSF disklabel
Building a new DOS disklabel. Changes will remain in memory only,
until you decide to write them. After that, of course, the previous
content won't be recoverable.

The number of cylinders for this disk is set to 49932.
There is nothing wrong with that, but this is larger than 1024,
and could in certain setups cause problems with:
1) software that runs at boot time (e.g., old versions of LILO)
2) booting and partitioning software from other OSs
   (e.g., DOS FDISK, OS/2 FDISK)
Warning: invalid flag 0x0000 of partition table 4 will be corrected by w(rite)

Command (m for help): n
Command action
   e   extended
   p   primary partition (1-4)
p
Partition number (1-4): 1
First cylinder (1-49932, default 1):
Using default value 1
Last cylinder or +size or +sizeM or +sizeK (1-49932, default 49932):
Using default value 49932

Command (m for help): t
Selected partition 1
Hex code (type L to list codes): 8e
Changed system type of partition 1 to 8e (Linux LVM)</pre>

You can review the change with the &#8216;p&#8217; command:

<pre>Command (m for help): p

Disk /dev/sda2: 410.7 GB, 410704680960 bytes
255 heads, 63 sectors/track, 49932 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes

     Device Boot      Start         End      Blocks   Id  System
/dev/sda2p1               1       49932   401078758+  8e  Linux LVM</pre>

You might need to reboot here, so go ahead and run sudo shutdown -r now

Now unmount the disk we want to create the lvm on and create a physical volume

<pre>$ sudo umount /dev/sda2
$ sudo pvcreate /dev/sda2
Physical volume "/dev/sda2" successfully created</pre>

You can see our new physical volume with pvdisplay

<pre>$ sudo pvdisplay
  "/dev/sda2" is a new physical volume of "382.50 GB"
  --- NEW Physical volume ---
  PV Name               /dev/sda2
  VG Name
  PV Size               382.50 GB
  Allocatable           NO
  PE Size (KByte)       0
  Total PE              0
  Free PE               0
  Allocated PE          0
  PV UUID               myxfgQ-alfA-y8sX-af4a-ifoA-3S29-YD3tuK</pre>

Now create a volume group on the physical volume, since we&#8217;re using this for mysql we&#8217;ll call it mysql

<pre>$ sudo vgcreate mysql /dev/sda2
  Volume group "mysql" successfully created</pre>

And you can view it

<pre>$ sudo vgdisplay
  --- Volume group ---
  VG Name               mysql
  System ID
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  1
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                0
  Open LV               0
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               382.50 GB
  PE Size               4.00 MB
  Total PE              97919
  Alloc PE / Size       0 / 0
  Free  PE / Size       97919 / 382.50 GB
  VG UUID               0VZxkR-873N-E6Vj-H6SV-FCWO-3jHF-ykHRiX</pre>

Create a logical volume on the volume group, we&#8217;ll put it on our mysql volume group and call it maindb. We want it to be about half (or less) of the entire amount of space we have to make a backup, in this case thats about half of 382GB so we&#8217;ll go with 190:

<pre>$ sudo lvcreate --name maindb --size 190G mysql
  Logical volume "maindb" created</pre>

And you can view it:

<pre>$ sudo lvdisplay
  --- Logical volume ---
  LV Name                /dev/mysql/maindb
  VG Name                mysql
  LV UUID                4hlcMf-cs5a-xjYU-yxjE-Fx2f-Pngv-z0RAEH
  LV Write Access        read/write
  LV Status              available
  # open                 0
  LV Size                190.00 GB
  Current LE             48640
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:0</pre>

Now we have to put an actual filesystem on it:

<pre>$ sudo mkfs.ext3 /dev/mysql/maindb
mke2fs 1.39 (29-May-2006)
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
24903680 inodes, 49807360 blocks
2490368 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=4294967296
1520 block groups
32768 blocks per group, 32768 fragments per group
16384 inodes per group
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
        4096000, 7962624, 11239424, 20480000, 23887872

Writing inode tables: done
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done

This filesystem will be automatically checked every 26 mounts or
180 days, whichever comes first.  Use tune2fs -c or -i to override.</pre>

And now we can actually re mount it on the /data2 folder:

<pre>$ sudo mount /dev/mysql/maindb /data2</pre>

And add it to fstab, making sure to comment out the old entry

<pre>#LABEL=/data2            /data2                  ext3    defaults        1 2
/dev/mysql/maindb       /data2                  ext3       rw,noatime   0 0</pre>

Note that we do not have to create the other logical volume on the volume group that is used for the snapshot, as the lvmbackup tool will take care of it for us.

# Move MySQL to the LVM Partition

Now we want to move our mysql installation into this folder:

<pre>$ sudo mkdir -p /data2/var/lib
$ sudo mv /var/lib/mysql/ /data2/var/lib/</pre>

We need to ensure the my.cnf has settings pointing to this new directory:

<pre>[mysqld]
datadir=/data2/var/lib/mysql
socket=/data2/var/lib/mysql/mysql.sock
log-bin = /data2/var/lib/mysql/bin.log

[mysql.server]
user=mysql
basedir=/data2/var/lib

[client]
socket=/data2/var/lib/mysql/mysql.sock</pre>

We can keep the log and pid outside of the LVM if we want:

<pre>[mysqld_safe]
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid</pre>

Now you can start mysqld:

<pre>sudo /usr/bin/mysqld_safe &</pre>

# Install mylvmbackup

Grab the latest mylvmbackup utility:

<pre>$ wget http://www.lenzg.org/mylvmbackup/mylvmbackup-current.tar.gz</pre>

Extract and install, its just a script so there is no compilation necessary:

<pre>$ tar xzvf mylvmbackup-0.11.tar.gz
$ cd mylvmbackup-0.11
$ sudo make install</pre>

You&#8217;ll also need some perl modules:

*Config::IniFiles
  
*DBI
  
*DBD::mysql
  
*Sys::Syslog
  
*Date::Format
  
Now we need to setup the config file for mylvmbackup, located at /etc/mylvmbackup.conf

There are lots of options but at the very least we need to set:

<pre>[mysql]
user=root
password={your root pass here}

socket=/data2/var/lib/mysql/mysql.sock

[lvm]
vgname=mysql
lvname=maindb
lvsize=190G

[misc]
innodb_recover=1

[tools]
lvcreate=/usr/sbin/lvcreate
lvremove=/usr/sbin/lvremove
lvs=/usr/sbin/lvs</pre>

You may also want to change the backupdir parameter, as this is where the tarball of your database backup is stored,  so you&#8217;ll need the space for that.

You&#8217;ll want to make sure this backup directory exists, so if you are using the default:

<pre>$ sudo mkdir -p /var/tmp/mylvmbackup/backup</pre>

And also the mount directory (this is where the LVM snapshot is mounted so a mysqldump can be taken:

<pre>$ sudo mkdir -p /var/tmp/mylvmbackup/mnt</pre>

# Create the Backup

Now you can simply run the mylvmbackup command as root.   Heres what the output looks like:

<pre>$ sudo mylvmbackup
20090122 13:26:47 Info: Connecting to database...
20090122 13:26:47 Info: Flushing tables with read lock...
20090122 13:26:47 Info: Taking position record...
20090122 13:26:47 Info: Running: /usr/sbin/lvcreate -s --size=190G 
--name=maindb_snapshot /dev/mysql/maindb
File descriptor 3 (socket:[101615]) leaked on lvcreate invocation.
 Parent PID 7209: /usr/bin/perl
  Logical volume "maindb_snapshot" created
20090122 13:26:47 Info: DONE: taking LVM snapshot
20090122 13:26:47 Info: Unlocking tables...
20090122 13:26:47 Info: Disconnecting from database...
20090122 13:26:47 Info: Mounting snapshot...
20090122 13:26:47 Info: Running: /bin/mount -o rw /dev/mysql/maindb_snapshot 
/var/tmp/mylvmbackup/mnt/backup
20090122 13:26:47 Info: DONE: mount snapshot
20090122 13:26:47 Info: Running: /bin/mount -o bind,ro 
/tmp/mylvmbackup-backup-20090122_132647_mysql-9zMEJA/pos 
/var/tmp/mylvmbackup/mnt/backup-pos
20090122 13:26:47 Info: DONE: bind-mount position directory
20090122 13:26:47 Info: Recovering InnoDB...
20090122 13:26:47 Info: Running: echo 'select 1;' | mysqld_safe 
--socket=/tmp/mylvmbackup.sock 
--pid-file=/var/tmp/mylvmbackup_recoverserver.pid 
--datadir=/var/tmp/mylvmbackup/mnt/backup 
--skip-networking 
--skip-grant --bootstrap --skip-ndbcluster --skip-slave-start
Starting mysqld daemon with databases from /var/tmp/mylvmbackup/mnt/backup
STOPPING server from pid file /var/tmp/mylvmbackup_recoverserver.pid
090122 13:26:47  mysqld ended
20090122 13:26:47 Info: DONE: InnoDB recovery on snapshot
20090122 13:26:47 Info: Copying my.cnf...
20090122 13:26:47 Info: Taking actual backup...
20090122 13:26:47 Info: Creating tar archive 
/var/tmp/mylvmbackup/backup/backup-20090122_132647_mysql.tar.gz
20090122 13:26:47 Info: Running: cd '/var/tmp/mylvmbackup/mnt' ;
'/bin/tar' cvf - backup/  
backup-pos/backup-20090122_132647_mysql.pos 
backup-pos/backup-20090122_132647_mysql_my.cnf
 /bin/gzip --stdout --verbose 
--best -&gt;
 /var/tmp/mylvmbackup/backup/backup-20090122_132647_mysql.tar.gz.INCOMPLETE-uqO2w7
backup/
backup/test
backup/var/
backup/var/lib/
backup/var/lib/mysql/
backup/var/lib/mysql/bin.index
backup/var/lib/mysql/test/
backup/var/lib/mysql/ibdata1
backup/var/lib/mysql/ib_logfile1
/bin/tar: backup/var/lib/mysql/mysql.sock: socket ignored
backup/var/lib/mysql/mysql/
backup/var/lib/mysql/mysql/time_zone_transition.MYI
backup/var/lib/mysql/mysql/func.frm
backup/var/lib/mysql/mysql/procs_priv.MYD
backup/var/lib/mysql/mysql/db.MYI
backup/var/lib/mysql/mysql/proc.frm
backup/var/lib/mysql/mysql/proc.MYD
backup/var/lib/mysql/mysql/proc.MYI
backup/var/lib/mysql/mysql/time_zone_leap_second.MYI
backup/var/lib/mysql/mysql/help_category.MYI
backup/var/lib/mysql/mysql/tables_priv.frm
backup/var/lib/mysql/mysql/user.MYI
backup/var/lib/mysql/mysql/help_relation.MYD
backup/var/lib/mysql/mysql/columns_priv.frm
backup/var/lib/mysql/mysql/columns_priv.MYI
backup/var/lib/mysql/mysql/time_zone_name.frm
backup/var/lib/mysql/mysql/help_keyword.MYI
backup/var/lib/mysql/mysql/help_relation.MYI
backup/var/lib/mysql/mysql/host.frm
backup/var/lib/mysql/mysql/user.frm
backup/var/lib/mysql/mysql/procs_priv.MYI
backup/var/lib/mysql/mysql/help_relation.frm
backup/var/lib/mysql/mysql/time_zone_transition_type.MYD
backup/var/lib/mysql/mysql/help_keyword.frm
backup/var/lib/mysql/mysql/time_zone_transition_type.MYI
backup/var/lib/mysql/mysql/time_zone.MYD
backup/var/lib/mysql/mysql/tables_priv.MYD
backup/var/lib/mysql/mysql/host.MYI
backup/var/lib/mysql/mysql/help_category.frm
backup/var/lib/mysql/mysql/user.MYD
backup/var/lib/mysql/mysql/time_zone_leap_second.MYD
backup/var/lib/mysql/mysql/time_zone.MYI
backup/var/lib/mysql/mysql/time_zone_name.MYD
backup/var/lib/mysql/mysql/time_zone_transition.MYD
backup/var/lib/mysql/mysql/tables_priv.MYI
backup/var/lib/mysql/mysql/columns_priv.MYD
backup/var/lib/mysql/mysql/db.MYD
backup/var/lib/mysql/mysql/time_zone_transition_type.frm
backup/var/lib/mysql/mysql/help_topic.MYI
backup/var/lib/mysql/mysql/procs_priv.frm
backup/var/lib/mysql/mysql/time_zone_name.MYI
backup/var/lib/mysql/mysql/func.MYD
backup/var/lib/mysql/mysql/time_zone.frm
backup/var/lib/mysql/mysql/help_topic.frm
backup/var/lib/mysql/mysql/time_zone_transition.frm
backup/var/lib/mysql/mysql/func.MYI
backup/var/lib/mysql/mysql/help_keyword.MYD
backup/var/lib/mysql/mysql/help_category.MYD
backup/var/lib/mysql/mysql/db.frm
backup/var/lib/mysql/mysql/time_zone_leap_second.frm
backup/var/lib/mysql/mysql/help_topic.MYD
backup/var/lib/mysql/mysql/host.MYD
backup/var/lib/mysql/jgtest/
backup/var/lib/mysql/jgtest/test2.MYD
backup/var/lib/mysql/jgtest/test2.frm
backup/var/lib/mysql/jgtest/db.opt
backup/var/lib/mysql/jgtest/test2.MYI
backup/var/lib/mysql/jgtest/test.frm
backup/var/lib/mysql/bin.000001
backup/var/lib/mysql/ib_logfile0
backup/lost+found/
backup-pos/backup-20090122_132647_mysql.pos
backup-pos/backup-20090122_132647_mysql_my.cnf
 99.2%
20090122 13:26:48 Info: DONE: create tar archive
20090122 13:26:48 Info: Cleaning up...
20090122 13:26:48 Info: LVM Usage stats:
20090122 13:26:48 Info:LV VG Attr LSize Origin Snap% Move Log Copy% Convert
20090122 13:26:48 Info:   maindb_snapshot mysql swi-a- 190.00G maindb   0.00
  Logical volume "maindb_snapshot" successfully removed</pre>

And now you should have a tarball in your backup directory, with all the files needed to restore:

<pre>$ ls /var/tmp/mylvmbackup/backup
backup-20090122_132647_mysql.tar.gz</pre>