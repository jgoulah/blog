---
title: Percona Live 2012 – The Etsy Shard Architecture
author: jgoulah
type: post
date: 2012-04-15T19:58:42+00:00
url: /2012/04/percona-live-2012-the-etsy-shard-architecture/
categories:
  - Conferences
  - Databases
  - Systems
tags:
  - architecture
  - databases
  - etsy
  - mysql
  - shard

---
I attended <a href="http://www.percona.com/live/mysql-conference-2012/" target="_blank">Percona Conf</a> in Santa Clara last week. It was a great 3 days of lots of people dropping expert MySQL knowledge. I learned a lot and met some great people.

I gave a talk about the Etsy shard architecture, and had a lot of good feedback and questions. A lot of people do active/passive, but we run everything in active/active master-master. This helps us keep a warm buffer pool on both sides, and if one side goes out we only lose about half of the queries until its pulled and they start hashing to the other side. Some details on how this works in the slides below.

<div style="width:425px" id="__ss_12506796">
  <strong style="display:block;margin:12px 0 4px"><a href="http://www.slideshare.net/jgoulah/the-etsy-shard-architecture-starts-with-s-and-ends-with-hard" title="The Etsy Shard Architecture: Starts With S and Ends With Hard" target="_blank">The Etsy Shard Architecture: Starts With S and Ends With Hard</a></strong> <iframe src="http://www.slideshare.net/slideshow/embed_code/12506796?rel=0" width="425" height="355" frameborder="0" marginwidth="0" marginheight="0" scrolling="no"></iframe> </p> 
  
  <div style="padding:5px 0 12px">
    View more presentations from <a href="http://www.slideshare.net/jgoulah" target="_blank">jgoulah</a>
  </div></p>
</div>
