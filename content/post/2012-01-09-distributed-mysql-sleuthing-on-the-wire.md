---
title: Distributed MySQL Sleuthing on the Wire
author: jgoulah
type: post
date: 2012-01-09T13:52:41+00:00
url: /2012/01/distributed-mysql-sleuthing-on-the-wire/
categories:
  - Databases
  - Real-time Web
  - SSH
  - Systems
tags:
  - distributed
  - dsh
  - mysql
  - percona
  - pt-query-digest
  - tcpdump

---
## Intro

Oftentimes you need to know what MySQL is doing _right now_ and furthermore if you are handling heavy traffic you probably have multiple instances of it running across many nodes. I&#8217;m going to start by showing how to take a <a href="http://www.tcpdump.org/" target="_blank">tcpdump</a> capture on one node, a few ways to analyze that, and then go into how to take a distributed capture across many nodes for aggregate analysis.

## Taking the Capture

The first thing you need to do is to take a capture of the interesting packets. You can either do this on the MySQL server or on the hosts talking to it. According to this <a href="http://www.mysqlperformanceblog.com/2011/04/18/how-to-use-tcpdump-on-very-busy-hosts/" target="_blank">percona post</a> this command is the best way to capture mysql traffic on the eth0 interface and write it into mycapture.cap for later analysis:

<pre>% tcpdump -i eth0 -w mycapture.cap -s 0 "port 3306 and tcp[1] & 7 == 2 and tcp[3] & 7 == 2"
tcpdump: listening on eth0, link-type EN10MB (Ethernet), capture size 65535 bytes
47542 packets captured
47703 packets received by filter
60 packets dropped by kernel
</pre>

## Analyzing the Capture

The next step is to take a look at your captured data. One way to do this is with tshark, which is the command line part of <a href="http://www.wireshark.org/" target="_blank">wireshark</a>. You can do _yum install wireshark_ or similar to install it. Usually you want to do this on a different host than the one taking traffic since it can be memory and CPU intensive.

You can then use it to reconstruct the mysql packets like so:

<pre>% tshark -d tcp.port==3306,mysql -T fields -R mysql.query -e frame.time -e ip.src -e ip.dst -e mysql.query -r mycapture.cap
</pre>

This will give you the time, source IP, destination IP, and query but this is still really raw output. Its a nice start but we can do better. Percona has released the <a href="http://www.percona.com/software/percona-toolkit/" target="_blank">Percona Toolkit</a> which includes some really nice command line tools (including what used to be in Maatkit). 

The one we&#8217;re interested in here is [pt-query-digest][1]

It has tons of options and you should read the documentation, but here&#8217;s a few I&#8217;ve used recently.

Lets say you want to get the top tables queried from your tcpdump

<pre>% tcpdump -r mycapture.cap -n -x -q -tttt | pt-query-digest --type tcpdump --group-by tables --order-by Query_time:cnt \
 --report-format profile --limit 5
reading from file mycapture.cap, link-type EN10MB (Ethernet)

# Profile
# Rank Query ID Response time Calls R/Call Apdx V/M   Item
# ==== ======== ============= ===== ====== ==== ===== ====================
#    1 0x        0.3140  6.1%   674 0.0005 1.00  0.00 shard.images
#    2 0x        0.8840 17.1%   499 0.0018 1.00  0.03 shard.activity
#    3 0x        0.1575  3.1%   266 0.0006 1.00  0.00 shard.listing_images
#    4 0x        0.1680  3.3%   265 0.0006 1.00  0.00 shard.connection_edges_reverse
#    5 0x        0.0598  1.2%   254 0.0002 1.00  0.00 shard.listing_translations
# MISC 0xMISC    3.5771 69.3%  3534 0.0010   NS   0.0 &lt;86 ITEMS>
</pre>

Note the tcpdump options I used this time, which the tool requires to work properly when passing _&#8211;type tcpdump_. I also grouped by tables (as opposed to full queries) and ordered by the count (the Calls column). It will stop at your _&#8211;limit_ and group the rest into MISC so be aware of that.

You can remove the _&#8211;order-by_ to sort by response time, which is the default sort order, or provide other attributes to sort on. We can also change the _&#8211;report-format_, for example to _header_:

<pre>% tcpdump -r mycapture.cap -n -x -q -tttt | pt-query-digest --type tcpdump --group-by tables --report-format header 
reading from file mycapture.cap, link-type EN10MB (Ethernet)

# Overall: 5.49k total, 91 unique, 321.13 QPS, 0.30x concurrency _________
# Time range: 2012-01-08 15:52:05.814608 to 15:52:22.916873
# Attribute          total     min     max     avg     95%  stddev  median
# ============     ======= ======= ======= ======= ======= ======= =======
# Exec time             5s     3us   114ms   939us     2ms     3ms   348us
# Rows affecte         316       0      13    0.06    0.99    0.29       0
# Query size         3.64M      18   5.65k  694.98   1.09k  386.68  592.07
# Warning coun           0       0       0       0       0       0       0
# Boolean:
# No index use   0% yes,  99% no
</pre>

If you set the _&#8211;report-format_ to _query_report_ you will get gobs of verbose information that you can dive into and you can use the _&#8211;filter_ option to do things like getting slow queries:

<pre>% tcpdump -r mycapture.cap -n -x -q -tttt | \
  pt-query-digest --type tcpdump --filter '($event->{No_index_used} eq "Yes" || $event->{No_good_index_used} eq "Yes")'
</pre>

## Distributed Capture

Now that we&#8217;ve taken a look at capturing and analyzing packets from one host, its time to dive into looking at our results across the cluster. The main trick is that tcpdump provides no option to stop capturing &#8211; you have to explicitly kill it. Otherwise we&#8217;ll just use [dsh][2] to send our commands out. We&#8217;ll assume you have a user that can hop around in a password-less fashion using ssh keys &#8211; setting that up is well outside the scope of this article but there&#8217;s plenty of info out there on how to do that. 

There&#8217;s a few ways you can let a process run on a &#8220;timeout&#8221; but I&#8217;m assuming we don&#8217;t have any script written or tools like <a href="http://www.bashcookbook.com/bashinfo/source/bash-4.0/examples/scripts/timeout3" target="_blank">bash timeout</a> or the one distributed in coreutils available.

So we&#8217;re going off the premise that you will background the process and kill it after a sleep by grabbing its pid:

<pre>( /path/to/command with options ) & sleep 5 ; kill $!
</pre>

Simple enough, except we&#8217;ll want to capture the output on each host, so we need to ssh the output back over to the target using a pipe to grab the stdout. This means that $! will return the pid of our ssh command instead of our tcpdump command. We end up having to do a little trick to kill the right process, since the capture won&#8217;t be readable if we kill ssh command that is writing the output. We&#8217;ll need to kill tcpdump and to do that we can look at the parent pid of the ssh process, ask pkill (similar to pgrep) for all of the processes that have this parent, and finally kill the oldest one, which ends up being our tcpdump process.

Then end result looks like this if I were to run it across two machines:

<pre>% dsh -c -m web1000,web1001 \
   'sudo /usr/sbin/tcpdump -i eth0 -w - -s 0 -x -n -q -tttt "port 3306 and tcp[1] & 7 == 2 and tcp[3] & 7 == 2" | \
   ssh dshhost "cat - &gt; ~/captures/$(hostname -a).cap" & sleep 10 ; \
   sudo pkill -o -P $(ps -ef | awk "\$2 ~ /\&lt;$!\&gt;/ { print \$3; }")'
</pre>

So this issues a dsh to two of our hosts (you can make a dsh group with 100 or 1000 hosts though) and runs the command concurrently on each (-c). We issue our tcpdump on each target machine and send the output to stdout for ssh to then cat back to a directory on the source machine that issued the dsh. This way we have all of our captures in one directory with each file named with the target name of each host the tcpdump was run. The sleep is how long the dump is going to run for before we then kill off the tcpdump.

The last piece of the puzzle is to get these all into one file and we can use the mergecap tool for this, which is also part of wireshark:

<pre>% /usr/sbin/mergecap -F libpcap -w output.cap *.cap
</pre>

And then we can analyze it like we did above.

## Further Reading

### References

<a href="http://www.mysqlperformanceblog.com/2011/04/18/how-to-use-tcpdump-on-very-busy-hosts" target="_blank">http://www.mysqlperformanceblog.com/2011/04/18/how-to-use-tcpdump-on-very-busy-hosts</a>

<a href="http://stackoverflow.com/questions/687948/timeout-a-command-in-bash-without-unnecessary-delay" target="_blank">http://stackoverflow.com/questions/687948/timeout-a-command-in-bash-without-unnecessary-delay</a>

<a href="http://www.xaprb.com/blog/2009/08/18/how-to-find-un-indexed-queries-in-mysql-without-using-the-log/" target="_blank">http://www.xaprb.com/blog/2009/08/18/how-to-find-un-indexed-queries-in-mysql-without-using-the-log/</a>

<https://cdn.comparitech.com/wp-content/uploads/2019/06/tcpdump-cheat-sheet.jpg>

### Breaking the distributed command down further

Just to clarify this command a bit more, particularly how the kill part works since that was the trickiest part for me to figure out.

When we run this

<pre>$ dsh -c -m web1000,web1001 \
   'sudo /usr/sbin/tcpdump -i eth0 -w - -s 0 -x -n -q -tttt "port 3306 and tcp[1] & 7 == 2 and tcp[3] & 7 == 2" | \
   ssh dshhost "cat - &gt; ~/captures/$(hostname -a).cap" & sleep 10 ; \
   sudo pkill -o -P $(ps -ef | awk "\$2 ~ /\&lt;$!\&gt;/ { print \$3; }")'
</pre>

on the server the process list looks something like

<pre>user     12505 12504  0 03:12 ?        00:00:00 bash -c sudo /usr/sbin/tcpdump -i eth0 -w - -s 0 -x -n -q -tttt "port 3306 and tcp[1] & 7 == 2 and tcp[3] & 7 == 2" | ssh myhost.myserver.com "cat - > /home/etsy/captures/$(hostname -a).cap" & sleep 5 ; sudo pkill -o -P $(ps -ef | awk "\$2 ~ /\&lt;$!\>/ { print \$3; }")
pcap     12506 12505  1 03:12 ?        00:00:00 /usr/sbin/tcpdump -i eth0 -w - -s 0 -x -n -q -tttt port 3306 and tcp[1] & 7 == 2 and tcp[3] & 7 == 2
user     12507 12505  0 03:12 ?        00:00:00 ssh myhost.myserver.com cat - > ~/captures/web1001.cap
</pre>

So $! is going to return the pid of the ssh process, 12507. We use awk to find the process matching that, and then print the parent pid out, which is then passed to the -P arg of pkill. If you use pgrep to look at this without the -o you&#8217;d get a list of the children of 12505, which are 12506 and 12507. The oldest child is the tcpdump command and so adding -o kills that guy off.

So if we were only running the command on one host we could use something much simpler

<pre>ssh dbhost01 '(sudo /usr/sbin/tcpdump -i eth0 -w - -s 0 port 3306) & sleep 10; sudo kill $!' | cat - &gt; output.cap
</pre>

 [1]: http://www.percona.com/doc/percona-toolkit/2.0/pt-query-digest.html
 [2]: http://www.netfort.gr.jp/~dancer/software/dsh.html.en