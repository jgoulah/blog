---
title: Approximate Times to Update an RDS Instance
author: jgoulah
type: post
date: 2016-07-09T17:59:07+00:00
url: /2016/07/updating-rds/
categories:
  - Cloud Computing
  - Databases
  - Systems
tags:
  - allocated storage
  - aws
  - cloud
  - modification
  - parameter group
  - rds
  - RDS upgrade
  - SSD
  - upgrade

---
Here&#8217;s a quick overview from an upgrade a couple months ago of an RDS instance type db.t2.medium to type db.r3.large. In addition to changing the instance type, we upgraded the disk from 64GB to 100GB, and applied a new parameter group. The disk increase by far took the longest amount of time. We clocked in at ever so slightly over an hour for the disk increase, while the instance upgrade only took a bit over 16 minutes. As you can see, AWS also does a failover during the upgrade, but we decided to stop writes to this database instead of risking any data corruption. 

|DB|Time|Description|
|--- |--- |--- |
|production01|6:26:46 AM|Applying modification to database instance class|
|production01|6:33:02 AM|Multi-AZ instance failover started|
|production01|6:33:07 AM|DB instance restarted|
|replica01|6:33:20 AM|Streaming replication has stopped.|
|production01|6:33:35 AM|Multi-AZ instance failover completed|
|replica01|6:34:51 AM|DB instance shutdown|
|replica01|6:35:08 AM|DB instance restarted|
|replica01|6:38:50 AM|Replication for the Read Replica resumed|
|production01|6:42:59 AM|Finished applying modification to DB instance class|
|production01|6:43:02 AM|Applying modification to allocated storage|
|production01|7:39:05 AM|Finished applying modification to allocated storage|


One thing Amazon could really work on here is the feedback loop. Getting no updates whatsoever other than hoping that the word &#8220;modifying&#8221; eventually goes away and says &#8220;available&#8221; is really poor in terms of an experience and keeping confidence the operation is going to be successful after an extremely long time. This is an exercise in patience because your only option is to continue to hit the reload button (nope, it doesn&#8217;t reload automatically) and hope that it has moved from yellow and is instead showing green and not red. 

In any case, I&#8217;m mostly publishing this post because I found these type of run time estimates or actual numbers very hard to find. And they are certainly variable depending on a variety of factors, so this can really be used as just one data point. That being said, I&#8217;m hoping it can help someone else that is looking for estimated times when planning their RDS upgrade.
