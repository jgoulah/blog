---
title: Investigating Data in Memcached
author: jgoulah
type: post
date: 2009-06-12T02:58:16+00:00
url: /2009/06/investigating-memcached/
categories:
  - Databases
  - Systems
tags:
  - cache dump
  - caching
  - inspecting cache
  - memcached
  - peep

---
## Intro

Almost every company I can think of uses <a target="_blank" href="http://www.danga.com/memcached/">Memcached</a> at some layer of their stack, however I haven&#8217;t until recently found a great way to take a snapshot of the keys in memory and their associated metadata such as expire time, LRU time, the value size and whether its been expired or flushed. The tool to do this is called <a target="_blank" href="http://github.com/fauna/peep/tree/master">peep</a> 

## Installing Peep

The main thing about installing peep is that you have to compile memcached with debugging symbols. Not really a big deal though. First thing you&#8217;ll want to do is install <a target="_blank" href=" http://monkey.org/~provos/libevent/">libevent</a> 

{{< highlight bash >}}$ wget http://monkey.org/~provos/libevent-1.4.11-stable.tar.gz
$ tar xzvf libevent-1.4.11-stable.tar.gz
$ cd libevent-1.4.11-stable
$ ./configure
$ make
$ sudo make install
{{< /highlight >}}

Now grab <a target="_blank" href="http://www.danga.com/memcached/">memcached</a>

{{< highlight bash >}}$ wget http://memcached.googlecode.com/files/memcached-1.2.8.tar.gz
$ tar xzvf memcached-1.2.8.tar.gz
$ cd memcached-1.2.8
$ CFLAGS='-g' ./configure --enable-threads
$ make 
$ sudo make install
{{< /highlight >}}

Note the configure line on this one sets the -g flag which enables debug symbols. Now move this directory somewhere standard like _/usr/local/src_ because we&#8217;ll need to reference it when installing peep

{{< highlight bash >}}$ sudo mv memcached-1.2.8 /usr/local/src
{{< /highlight >}}

Ok, now for peep, which is written in ruby. If you don&#8217;t have <a target="_blank" href="http://rubygems.org/">rubygems</a> you need that and the ruby headers. Just use your package management system for this. I happened to be on a CentOS box so I ran

{{< highlight bash >}}$ sudo yum install rubygems.noarch
$ sudo yum install ruby-devel.i386
{{< /highlight >}}

Finally you can install peep

{{< highlight bash >}}$ sudo gem install peep -- --with-memcached-include=/usr/local/bin/memcached-1.2.8
{{< /highlight >}}

## Using Peep

Finally you can actually use this thing, you just give it the running memcached process pid 

{{< highlight bash >}}$ sudo peep --pretty `cat /var/run/memcached/memcached.pid`
{{< /highlight >}}

Here&#8217;s a snippet of what peep can show us in its &#8220;pretty&#8221; mode

{{< highlight bash >}}time |   exptime |  nbytes | nsuffix | it_f | clsid | nkey |                           key | exprd | flushd
       485 |       785 |   17925 |      10 | link |    25 |   64 |  "post.ordered-posts1-03afdb" |  true |  false
       537 |       721 |   16991 |      10 | link |    24 |   63 |  "post.ordered-posts1-03bd6"  |  true |  false
       240 |      3684 |     434 |       8 | link |     9 |   22 |  "channel.channel.105.v1"     | false |  false
       241 |      3687 |   27286 |      10 | link |    27 |   55 |  "post.post_count-fec35129"   | false |  false
       538 |      4022 |    3223 |       9 | link |    17 |   55 |  "post.post_count-2ff57a7"    | false |  false
       538 |      4024 |   17169 |      10 | link |    25 |   55 |  "post.post_count-2ba928998d" | false |  false
       241 |      3686 |   10763 |      10 | link |    22 |   55 |  "post.post_count-3879a24011" | false |  false
        25 |       320 |    8874 |       9 | link |    22 |   22 |  "channel.posterframes.4"     |  true |  false
{{< /highlight >}}

## Putting the Data Into MySQL

The above is not the easiest thing to analyze, especially if you have a lot of data in cache. But we can easily load it in to MySQL so that we can run queries on it.

First create the db and permissions

{{< highlight bash >}}mysql> create database peep;
mysql> grant all on peep.* to peep@'localhost' identified by 'peep4u';
{{< /highlight >}}

Then a table to store the output of the peep snapshot

{{< highlight bash >}}mysql> CREATE TABLE `entries` (
  `lru_time` int(11) DEFAULT NULL,
  `expires_at_time` int(11) DEFAULT NULL,
  `value_size` int(11) DEFAULT NULL,
  `suffix_size` int(11) DEFAULT NULL,
  `it_flag` varchar(255) DEFAULT NULL,
  `class_id` int(11) DEFAULT NULL,
  `key_size` int(11) DEFAULT NULL,
  `key_name` varchar(255) DEFAULT NULL,
  `is_expired` varchar(255) DEFAULT NULL,
  `is_flushed` varchar(255) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
{{< /highlight >}}

Now run peep and output the data in &#8220;ugly&#8221; format into a file

{{< highlight bash >}}$ sudo peep --ugly `cat /var/run/memcached/memcached.pid` > peep.out
{{< /highlight >}}

And now you can load it into your entries table

{{< highlight bash >}}mysql> load data local infile '/tmp/peep.out' into table entries fields terminated by ' | ' lines terminated by '\n';
Query OK, 177 rows affected (0.01 sec)
Records: 177  Deleted: 0  Skipped: 0  Warnings: 0
{{< /highlight >}}

I&#8217;m only loading a small dev install of memcahed but if you are running this on production you&#8217;d be importing many more rows. Luckily _load data infile_ is sufficiently optimized to input large datasets. Somewhat unfortunately however, peep makes memcached block while its taking its snapshot, which can take a bit of time in production. 

In any case, not that interesting with dev data but you can get lots of interesting numbers out of this. In my case it looks like most of the stuff in my cache is expired already.

{{< highlight bash >}}mysql> select is_expired, count(*) as num, sum(value_size) as value_size from entries group by is_expired;
+------------+-----+------------+
| is_expired | num | value_size |
+------------+-----+------------+
| false      |   1 |        415 |
| true       | 176 |    2392792 |
+------------+-----+------------+
2 rows in set (0.01 sec)
{{< /highlight >}}

Or selecting by grouping slabs and displaying their sizes

{{< highlight bash >}}mysql> select class_id as slab_class, max(value_size) as slab_size from entries group by slab_class;
+------------+-----------+
| slab_class | slab_size |
+------------+-----------+
|          1 |         8 |
|          2 |        15 |
|          9 |       445 |
|         12 |      1100 |
|         13 |      1402 |
|         14 |      1710 |
|         15 |      2174 |
|         16 |      2409 |
|         17 |      3223 |
|         20 |      6154 |
|         21 |      8395 |
|         22 |     10763 |
|         23 |     13395 |
|         24 |     16991 |
|         25 |     17925 |
|         26 |     23320 |
|         27 |     33361 |
|         28 |     38824 |
|         29 |     44904 |
+------------+-----------+
19 rows in set (0.00 sec)
{{< /highlight >}}

## Conclusion

Although there are a lot of tools such as Cacti that will display graphs of your Memcached usage, there are not many tools that will actually show you specifics of whats in memory. This tool can be used for a variety of reasons, but its especially useful to example the keys that you have in memory and the size of their values, as well as if they&#8217;re expired and what the individuals slabs are holding.
