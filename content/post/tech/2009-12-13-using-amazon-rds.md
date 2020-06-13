---
title: Using Amazon Relational Database Service
author: jgoulah
type: post
date: 2009-12-13T19:09:11+00:00
url: /2009/12/using-amazon-rds/
categories:
  - Cloud Computing
tags:
  - amazon
  - cloud
  - mysql
  - rds
  - relational database service

---
## Intro

Amazon recently released a new service that makes it easier set up, operate, and scale a relational database in the cloud called <a href="http://aws.amazon.com/rds/" target="_blank">Amazon Relational Database Service</a>. The service, based on MySQL for now, has its pluses and minuses and you should decide whether it fits your needs. The advantages are that it has an automated backup system that lets you restore to any point within the last 5 minutes and also allows you to easily take a snapshot at any time. It is also very easy to &#8220;scale up&#8221; your box. This is more in the realm of vertical scaling but if you find you are hitting limits you can upgrade to a more powerful server with little to no effort. It also gives you monitoring via <a href="http://aws.amazon.com/cloudwatch/" target="_blank">Amazon Cloudwatch </a> and automatically patches your database during your self defined maintenance windows. The downfalls are that you don&#8217;t have access directly to the box itself, so you can&#8217;t ssh in. You also at this point cannot use replication for a master-slave style setup. Amazon promises to have more high availability options forthcoming. Since you can&#8217;t ssh in, you adjust mysql parameters via their db parameter group API. I&#8217;ll go over an example of this.

## Creating a Database Instance

The first thing to do is install the RDS command line API, which you can grab from [here][1]. I&#8217;m not going over the details of setting this up. Its basically as simple as putting it into your path and Amazon has plenty of documentation on this. 

Once you have the command line tools setup you can create a database like so

{{< highlight bash >}}rds-create-db-instance \
        mydb-instance \
        --allocated-storage 20 \
        --db-instance-class db.m1.small \
        --engine MySQL5.1  \
        --master-username masteruser \
        --master-user-password mypass \
        --db-name mydb --headers \
        --preferred-maintenance-window 'Mon:06:15-Mon:10:15' \ 
        --preferred-backup-window  '10:15-12:15' \
        --backup-retention-period 1 \
        --availability-zone us-east-1c 
{{< /highlight >}}

This should be fairly self explanatory. I&#8217;m creating an instance called _mydb-instance_. The master (basically root) user is called _masteruser_ with password _mypass_. It also creates an initial database called _mydb_. You can add more databases and permissions later. This also sets up the maintenance and backup windows which are required, and defined in UTC. The backup retention period is how long it holds on to my backups, which I&#8217;ve defined as 1 day. If you set this to 0 it will disable the automated backups entirely which is not advised.

The next thing to do is setup your security groups so that your EC2 (or your hosted servers) have access to your database. There is <a href="http://docs.amazonwebservices.com/AmazonRDS/latest/CommandLineReference/index.html?CLIReference-cmd-AuthorizeDBSecurityGroupIngress.html" target="_blank">good documentation</a> on this so I will go over a basic use case.

{{< highlight bash >}}rds-authorize-db-security-group-ingress \
        default \
        --ec2-security-group-name webnode \
        --ec2-security-group-owner-id XXXXXXXXXXXX 
{{< /highlight >}}

In the case above I&#8217;m creating a security group called default, that allows my ec2 security group webnode access. The group-owner-id parameter is your AWS account id.

You can find what your database DNS name is via the _rds-describe-db-instances_ command. 

{{< highlight bash >}}rds-describe-db-instances 
DBINSTANCE  mydb-instance  2009-11-06T02:19:59.160Z  db.m1.small  mysql5.1  20  masteruser  available  mydb-instance.cvjb75qirgzk.us-east-1.rds.amazonaws.com  3306  us-east-1d  1
      SECGROUP  default  active
      PARAMGRP  default.mysql5.1  in-sync
{{< /highlight >}}

So we can see our hostname is _mydb-instance.cvjb75qirgzk.us-east-1.rds.amazonaws.com_

Now you can login to your instance in the usual way that you access mysql on the command line, setup your users and import your database in the usual way.

{{< highlight bash >}}mysql -u masteruser -h mydb-instance.cvjb75qirgzk.us-east-1.rds.amazonaws.com -pmypass{{< /highlight >}}

## Using Parameter Groups to View the MySQL Slow Queries Log

As I mentioned earlier you don&#8217;t have access to ssh into the instance, so you need to use db parameter groups to tweak your configuration rather than editing the my.cnf file. You can&#8217;t see the mysql slow query log on the box but there is still a way to access it and I&#8217;ll go over that process.

Amazon won&#8217;t let you edit the default group, so the first thing to do is create a parameter group to define your custom parameters. 

{{< highlight bash >}}rds-create-db-parameter-group my-custom --description='My Custom DB Param Group' --engine=MySQL5.1
{{< /highlight >}}

Then set the parameter to turn the query log on

{{< highlight bash >}}rds-modify-db-parameter-group my-custom  --parameters="name=slow_query_log, value=ON, method=immediate"
{{< /highlight >}}

We&#8217;re still using the default configuration so you have to tell the instance to use your custom parameter group

{{< highlight bash >}}rds-modify-db-instance mydb-instance --db-parameter-group-name=my-custom{{< /highlight >}}

The first time you apply a new custom group you have to reboot the instance, as _pending-reboot_ here indicates

{{< highlight bash >}}$ rds-describe-db-instances 
DBINSTANCE  mydb-instance  2009-11-06T02:19:59.160Z  db.m1.small  mysql5.1  20  masteruser  available  mydb-instance.cvjb75qirgzk.us-east-1.rds.amazonaws.com  3306  us-east-1d  1
      SECGROUP  default  active
      PARAMGRP  my-custom  pending-reboot
{{< /highlight >}}

So we can reboot it immediately like so

{{< highlight bash >}}$ rds-reboot-db-instance mydb-instance
{{< /highlight >}}

When it comes back up it will show that its _in-sync_

{{< highlight bash >}}$ rds-describe-db-instances 
DBINSTANCE  mydb-instance  2009-11-06T02:19:59.160Z  db.m1.small  mysql5.1  20  masteruser  available  mydb-instance.cvjb75qirgzk.us-east-1.rds.amazonaws.com  3306  us-east-1d  1
      SECGROUP  default  active
      PARAMGRP  my-custom  in-sync
{{< /highlight >}}

We can login to the instance and see that our parameter was set correctly

{{< highlight bash >}}mysql> show global variables like 'log_slow_queries';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| log_slow_queries | ON    | 
+------------------+-------+
1 row in set (0.00 sec)
{{< /highlight >}}

Since you don&#8217;t have access to the filesystem, its logged to a table on the mysql database

{{< highlight bash >}}mysql> use mysql;

mysql> describe slow_log;
+----------------+--------------+------+-----+-------------------+-----------------------------+
| Field          | Type         | Null | Key | Default           | Extra                       |
+----------------+--------------+------+-----+-------------------+-----------------------------+
| start_time     | timestamp    | NO   |     | CURRENT_TIMESTAMP | on update CURRENT_TIMESTAMP | 
| user_host      | mediumtext   | NO   |     | NULL              |                             | 
| query_time     | time         | NO   |     | NULL              |                             | 
| lock_time      | time         | NO   |     | NULL              |                             | 
| rows_sent      | int(11)      | NO   |     | NULL              |                             | 
| rows_examined  | int(11)      | NO   |     | NULL              |                             | 
| db             | varchar(512) | NO   |     | NULL              |                             | 
| last_insert_id | int(11)      | NO   |     | NULL              |                             | 
| insert_id      | int(11)      | NO   |     | NULL              |                             | 
| server_id      | int(11)      | NO   |     | NULL              |                             | 
| sql_text       | mediumtext   | NO   |     | NULL              |                             | 
+----------------+--------------+------+-----+-------------------+-----------------------------+
11 rows in set (0.11 sec)
{{< /highlight >}}

We may also want to set things like the slow query time, since the default of 10 is pretty high

{{< highlight bash >}}$ rds-modify-db-parameter-group my-custom  --parameters="name=long_query_time, value=3, method=immediate"
{{< /highlight >}}

The _rds-describe-events_ command keeps a log of what you&#8217;ve been doing

{{< highlight bash >}}$ rds-describe-events 
db-instance         2009-12-12T17:44:19.546Z  mydb-instance  Updated to use a DBParameterGroup my-custom
db-instance         2009-12-12T17:45:51.636Z  mydb-instance  Database instance shutdown
db-instance         2009-12-12T17:46:09.380Z  mydb-instance  Database instance restarted
db-parameter-group  2009-12-12T17:56:02.568Z  my-custom        Updated parameter long_query_time to 3 with apply method immediate
{{< /highlight >}}

And again you can check mysql that your parameter was edited properly. Note how this time we didn&#8217;t have to reboot anything as our parameter group is already active on this instance

{{< highlight bash >}}mysql> show global variables like 'long_query_time';
+-----------------+----------+
| Variable_name   | Value    |
+-----------------+----------+
| long_query_time | 3.000000 | 
+-----------------+----------+
1 row in set (0.00 sec)
{{< /highlight >}}

## Summary

In this article we went over some basics of Amazon RDS and why you may or may not want to use it. If you are just starting out its a really easy way to get a working mysql setup going. However if you are porting from an architecture with multiple slaves or other HA options this may not be for you just yet. We also went over some basic use cases on how to tweak parameters since there is no command line access to the box itself. There is command line access to mysql though, so you can use your favorite tools there.

## References

<a href="http://developer.amazonwebservices.com/connect/entry.jspa?externalID=2935&#038;categoryID=295" target="_blank">http://developer.amazonwebservices.com/connect/entry.jspa?externalID=2935&categoryID=295</a>

 [1]: http://developer.amazonwebservices.com/connect/entry.jspa?externalID=2928&categoryID=294
