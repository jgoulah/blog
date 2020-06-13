---
title: Using Mongo and Map Reduce on Apache Access Logs
author: jgoulah
type: post
date: 2010-01-19T02:32:20+00:00
url: /2010/01/using-mongo-and-map-reduce-on-apache-access-logs/
categories:
  - Databases
  - Systems
tags:
  - apache
  - CPAN
  - database
  - document oriented
  - logs
  - map
  - mapreduce
  - mongo
  - perl
  - reduce

---
## Introduction

With more and more traffic pouring into websites, it has become necessary to come up with creative ways to parse and analyze large data sets. One of the popular ways to do that lately is using <a target="_blank" href="http://en.wikipedia.org/wiki/Map_Reduce">MapReduce</a> which is a framework used across distributed systems to help make sense of large data sets. There are lots of implementations of the map/reduce framework but an easy way to get started is by using <a target="_blank" href="http://www.mongodb.org">MongoDB</a>. MongoDB is a scalable and high performant <a target="_blank" href="http://en.wikipedia.org/wiki/Document-oriented_database">document oriented database</a>. It has replication, sharding, and mapreduce all built in which makes it easy to scale horizontally. 

For this article we&#8217;ll look a common use case of map/reduce which is to help analyze your apache logs. Since there is no set schema in a document oriented database, its a good fit for log files since its fairly easy to import arbitrary data. We&#8217;ll look at getting the data into a format that mongo can import, and writing a map/reduce algorithm to make sense of some of the data.

## Getting Setup

### <a name="installmongo">Installing Mongo</a>

Mongo is easy to install with <a target="_blank" href="http://www.mongodb.org/display/DOCS/Quickstart">detailed documentation here</a>. In a nutshell you can do

{{< highlight bash >}}$ mkdir -p /data/db
$ curl -O http://downloads.mongodb.org/osx/mongodb-osx-i386-latest.tgz
$ tar xzf mongodb-osx-i386-latest.tgz
{{< /highlight >}}

At this point its not a bad idea to put this directory somewhere like /opt and adding its bin directory to your path. That way instead of 

{{< highlight bash >}}./mongodb-xxxxxxx/bin/mongod &{{< /highlight >}}

You can just do

{{< highlight bash >}}mongodb &{{< /highlight >}}

In any case, start up the daemon one of those two ways depending how you set it up.

### Importing the Log Files

Apache access files can vary in the information reported. The log format is easy to change with the LogFormat directive which is [documented here][1]. In any case these logs that I&#8217;m working with are not out of the box apache format. They look something like this

`Jan 18 17:20:26 web4 logger: networks-www-v2 84.58.8.36 [18/Jan/2010:17:20:26 -0500] "GET /javascript/2010-01-07-15-46-41/97fec578b695157cbccf12bfd647dcfa.js HTTP/1.1" 200 33445 "http://www.channelfrederator.com/hangover/episode/HNG_20100101/cuddlesticks-cartoon-hangover-4" "Mozilla/5.0 (Windows; U; Windows NT 6.1; de; rv:1.9.1.7) Gecko/20091221 Firefox/3.5.7" www.channelfrederator.com 35119<br />
` 

We want to take this raw log and convert it to a JSON structure for importing into mongo, which I wrote a simple perl script to iterate through the log and parse it into sensible fields using a regular expression

{{< highlight bash >}}#!/usr/bin/perl

use strict;
use warnings;

my $logfile = "/logs/httpd/remote_www_access_log";
open(LOGFH, "$logfile");
foreach my $logentry (&lt;LOGFH&gt;) {
    chomp($logentry);   # remove the newline
    $logentry =~ m/(\w+\s\w+\s\w+:\w+:\w+)\s #date
                   (\w+)\slogger:\s # server host
                   ([^\s]+)\s # vhost logger
                   (?:unknown,\s)?(-|(?:\b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3},?\s?)*)\s #ip
                   \[(.*)\]\s # date again
                   \"(.*?)\"\s # request
                   (\d+)\s #status
                   ([\d-]+)\s # bytes sent
                   \"(.*?)\"\s # referrer 
                   \"(.*?)\"\s # user agent 
                   ([\w.-]+)\s? # domain name
                   (\d+)? # time to server (ms) 
                  /x;

    print &lt;&lt;JSON;
     {"date": "$5", "webserver": "$2", "logger": "$3", "ip": "$4", "request": "$6", "status": "$7", "bytes_sent": "$8", "referrer": "$9", "user_agent": "$10", "domain_name": "$11", "time_to_serve": "$12"} 
JSON

}
{{< /highlight >}}

Again my regular expression probably won&#8217;t quite work on your logs, though you may be able to take bits and pieces of what I&#8217;ve documented above to use for your script. On my logs that script outputs a bunch of lines that look like this

`{"date": "18/Jan/2010:17:20:26 -0500", "webserver": "web4", "logger": "networks-www-v2", "ip": "84.58.8.36", "request": "GET /javascript/2010-01-07-15-46-41/97fec578b695157cbccf12bfd647dcfa.js HTTP/1.1", "status": "200", "bytes_sent": "33445", "referrer": "http://www.channelfrederator.com/hangover/episode/HNG_20100101/cuddlesticks-cartoon-hangover-4", "user_agent": "Mozilla/5.0 (Windows; U; Windows NT 6.1; de; rv:1.9.1.7) Gecko/20091221 Firefox/3.5.7", "domain_name": "www.channelfrederator.com", "time_to_serve": "35119"}<br />
` 

And we can then import that directly to MongoDB. The creates a collection called _weblogs_ in the _logs_ database, from the file that has the output of the above JSON generator script

{{< highlight bash >}}$ mongoimport --type json -d logs -c weblogs --file weblogs.json
{{< /highlight >}}

We can also take a look at them and verify they loaded by running the find command, which dumps out 10 rows by default

{{< highlight bash >}}$ mongo
> use logs;
switched to db logs
> db.weblogs.find()
{{< /highlight >}}

## Setting Up the Map and Reduce Functions

So for this example what I am looking for is how many hits are going to each domain. My servers handle a bunch of different domain names and that is one thing I&#8217;m outputting into my logs that is easy to examine

The basic command line interface to MondoDB is a kind of javascript interpreter, and the mongo backend takes javascript implementations of the map and reduce functions. We can type these directly into the mongo console. The map function must emit a key/value pair and in this example it will output a &#8216;1&#8217; each time a domain is found

{{< highlight bash >}}> map = "function() { emit(this.domain_name, {count: 1}); }"
{{< /highlight >}}

And so basically what comes out of this is a key for each domain with a set of counts, something like

{{< highlight bash >}}{"www.something.com", [{count: 1}, {count: 1}, {count: 1}, {count: 1}]}{{< /highlight >}}

This is send to the reduce function, which sums up all those counts for each domain

{{< highlight bash >}}> reduce = "function(key, values) { var sum = 0; values.forEach(function(f) { sum += f.count; }); return {count: sum}; };"
{{< /highlight >}}

Now that you&#8217;ve setup map and reduce functions, you can call mapreduce on the collection

{{< highlight bash >}}> results = db.weblogs.mapReduce(map, reduce)
...
>results
{
        "result" : "tmp.mr.mapreduce_1263861252_3",
        "timeMillis" : 9034,
        "counts" : {
                "input" : 1159355,
                "emit" : 1159355,
                "output" : 92
        },
        "ok" : 1,
}
{{< /highlight >}}

This gives us a bit of information about the map/reduce operation itself. We see that we inputted a set of data and emmitted once per item in that set. And we reduced down to 92 domain with a count for each. A result collection is given, and we can print it out

{{< highlight bash >}}> db.tmp.mr.mapreduce_1263861252_3.find()
> it
{{< /highlight >}}

You can type the &#8216;it&#8217; operator to page through multiple pages of data. Or you can print it all at once like so

{{< highlight bash >}}> db.tmp.mr.mapreduce_1263861252_3.find().forEach( function(x) { print(tojson(x));});
{ "_id" : "barelydigital.com", "value" : { "count" : 342888 } }
{ "_id" : "barelypolitical.com", "value" : { "count" : 875217 } }
{ "_id" : "www.fastlanedaily.com", "value" : { "count" : 998360 } }
{ "_id" : "www.threadbanger.com", "value" : { "count" : 331937 } }
{{< /highlight >}}

Nice, we have our data aggregated and the answer to our initial problem.

## Automate With a Perl Script

As usual CPAN comes to the rescue with a <a target="_blank" href="http://search.cpan.org/~kristina/MongoDB-0.27/">MongoDB driver</a> that we can use to interface with our database in a scripted fashion. The mongo guys have also done a great job of supporting it in a <a target="_blank" href="http://www.mongodb.org/display/DOCS/Drivers">bunch of other languages</a> which makes it easy to interface with if you are using more than just perl

{{< highlight bash >}}#!/bin/perl

use MongoDB;
use Data::Dumper;
use strict;
use warnings;

my $conn = MongoDB::Connection->new("host" => "localhost", "port" => 27017);
my $db = $conn->get_database("logs");

my $map = "function() { emit(this.domain_name, {count: 1}); }";

my $reduce = "function(key, values) { var sum = 0; values.forEach(function(f) { sum += f.count; }); return {count: sum}; }";

my $idx = Tie::IxHash->new(mapreduce => 'weblogs', 'map' => $map, reduce => $reduce);
my $result = $db->run_command($idx);

print Dumper($result);

my $res_coll = $result->{'result'};

print "result collection is $res_coll\n";

my $collection = $db->get_collection($res_coll);
my $cursor = $collection->query({ }, { limit => 10 });

while (my $object = $cursor->next) {
    print Dumper($object); 
}
{{< /highlight >}}

The script is straightforward. Its just using the documented perl interface to make a run_command call to mongo, which passes the map and reduce javascript functions in. It then prints the results similar to how we did on the command line earlier.

## Summary

We&#8217;ve gone over a very simple example of how MapReduce can work for you. There are lots of other ways you can put it to good use such as distributed sorting and searching, document clustering, and machine learning. We also took a look at MongoDB which has great uses for schema-less data. It makes it easy to scale since it has built in replication and sharding capabilities. Now you can put map/reduce to work on your logs and find a ton of information you couldn&#8217;t easily get before.

## References

<a target="_blank" href="http://github.com/mongodb/mongo">http://github.com/mongodb/mongo</a>
  
<a target="_blank" href="http://www.mongodb.org/display/DOCS/MapReduce">http://www.mongodb.org/display/DOCS/MapReduce</a>
  
<a target="_blank" href="http://apirate.posterous.com/visualizing-log-files-with-mongodb-mapreduce">http://apirate.posterous.com/visualizing-log-files-with-mongodb-mapreduce</a>
  
<a target="_blank" href="http://search.cpan.org/~kristina/MongoDB-0.27/">http://search.cpan.org/~kristina/MongoDB-0.27/</a>
  
<a target="_blank" href="http://kylebanker.com/blog/2009/12/mongodb-map-reduce-basics/">http://kylebanker.com/blog/2009/12/mongodb-map-reduce-basics/</a>

 [1]: http://httpd.apache.org/docs/2.0/logs.html
