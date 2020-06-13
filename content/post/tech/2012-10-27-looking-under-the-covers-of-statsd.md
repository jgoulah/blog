---
title: Looking Under the Covers of StatsD
author: jgoulah
type: post
date: 2012-10-27T17:24:06+00:00
url: /2012/10/looking-under-the-covers-of-statsd/
categories:
  - Debugging
tags:
  - carbon
  - etsy
  - Graphite
  - netcat
  - node.js
  - StatsD
  - tcpdump

---
## Intro

<a href="https://github.com/etsy/statsd" title="statsd" target="_blank">StatsD</a> is a network daemon that runs on the <a href="http://nodejs.org/" title="node.js" target="_blank">Node.js</a> platform and listens for statistics, like counters and timers. Packets are then sent to one or more pluggable backend services. The default service is <a href="http://graphite.readthedocs.org/" title="graphite" target="_blank">Graphite</a>. Every 10 seconds the stats sent to StatsD are aggregated and forwarded on to this backend service. It can be useful to see what stats are going through both sides of the connection &#8211; from the client to StatsD and then from StatsD to Graphite. 

## Management Interface

The first thing to know is there is a simple management interface built in that you can interact with. By using either telnet or <a href="http://netcat.sourceforge.net/" title="netcat" target="_blank">netcat</a> you can find information directly from the command line. By default this is listening on port 8126, but that is configurable in StatsD. 

The simplest thing to do is send the _stats_ command:

{{< highlight bash >}}% echo "stats" | nc statsd.domain.com 8126          
uptime: 365
messages.last_msg_seen: 0
messages.bad_lines_seen: 0
graphite.last_flush: 5
graphite.last_exception: 365{{< /highlight >}}

This tells us a bit about the current state of the server, including the uptime, and the last time a flush was sent to the backend. Our server has only been running for 365 seconds. It also lets us know when the length of time since StatsD received its last message, bad lines sent to it, and the last exception. Things look pretty normal.

You can also get a dump of the current timers:

{{< highlight bash >}}(echo "timers" | nc statsd.domain.com 8126) &gt; timers{{< /highlight >}}

As well as a dump of the current counters:

{{< highlight bash >}}(echo "counters" | nc statsd.domain.com 8126) &gt; counters{{< /highlight >}}

Take a look at the files generated to get an idea of the metrics StatsD is currently holding.

## On the Wire

Beyond that, its fairly simple to debug certain StatsD or Graphite issues by looking at whats going on in realtime on the connection itself. On the StatsD host, be sure you&#8217;re looking at traffic across the default StatsD listen port (8125), and specifically here I&#8217;m grep&#8217;ing for the stat that I&#8217;m about to send which will be called _test.goulah.myservice_:

{{< highlight bash >}}% sudo tcpdump -t -A -s0 dst port 8125 | grep goulah
listening on eth0, link-type EN10MB (Ethernet), capture size 65535 bytes{{< /highlight >}}

Then we fake a simple client on the command line to send a sample statistic to StatsD like so: 

{{< highlight bash >}}echo "test.goulah.myservice:1|c" | nc -w 1 -u statsd.domain.com 8125{{< /highlight >}}

Back on the StatsD host, you can see the metric come through:

{{< highlight bash >}}e......."A.test.goulah.myservice:1|c{{< /highlight >}}

There is also the line of communication from StatsD to the Graphite host. Every 10 seconds it flushes its metrics. Start up another tcpdump command, this time on port 2003, which is the port carbon is listening on the Graphite side:

{{< highlight bash >}}% sudo tcpdump -t -A -s0 dst port 2003 | grep goulah
listening on eth0, link-type EN10MB (Ethernet), capture size 65535 bytes{{< /highlight >}}

Every 10 seconds you should see a bunch of stats go by. This is what you are flushing into the Graphite backend. In our case I&#8217;m doing a grep for goulah, and showing the data aggregated for the metric we sent earlier. Notice there are two metrics here that look slightly different than the metric we sent though. StatsD sends two lines for every metric. The first is the aggregated metric prefixed with the _stats_ namespace. StatsD also sends the raw data prefixed by _stats_counts_. This is the difference in the _value per second_ calculated and the raw value. In our case they identical: 

{{< highlight bash >}}stats.test.goulah.myservice 0 1351355521
stats_counts.test.goulah.myservice 0 1351355521{{< /highlight >}}

## Conclusion

Now we can get a better understanding of what StatsD is doing under the covers on our system. If metrics don&#8217;t show up on the Graphite side it helps to break things into digestible pieces to understand where the problem lies. If the metrics aren&#8217;t even getting to StatsD, then of course they can&#8217;t make it to Graphite. Or perhaps they are getting to StatsD but you are not seeing the metrics you would expect when you look at the graphs. This is a good start on digging into those types of problems.
