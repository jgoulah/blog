---
title: Using DBIx::Class to Version Your Schema
author: jgoulah
type: post
date: 2009-05-17T22:53:35+00:00
url: /2009/05/versioning-with-dbic/
categories:
  - Databases
  - Deployment
  - Schema Versioning
tags:
  - mysql
  - perl
  - schema generation
  - Versioning

---
# Intro to DBIx::Class

In my opinion [DBIx::Class][1] is one of the best ORM solutions out there. Not only can it model your database, including mapping out any <a title="foreign key" href="http://en.wikipedia.org/wiki/Foreign_key" target="_blank">foreign key</a> relationships, it can also be used as a canonical point of reference for your <a title="schema" href="http://en.wikipedia.org/wiki/Database_schema" target="_blank">schema</a>.   This means we can use it not only as an application layer interface to the database, but can also define a versioned database structure using the code,  where if you add a field into your result class,  you can generate a version of alter statements and a DDL just by running a simple script.   Of course this set of alterations can later be applied to your stage and production environments.

It&#8217;s possible to produce your schema classes with <a title="schema loader" href="http://search.cpan.org/~ilmari/DBIx-Class-Schema-Loader-0.04006/" target="_blank">DBIx::Class::Schema::Loader</a> if you have a pre-existing database and are initially generating your DBIC classes, but this article is going to show going from the ground up how to build your database from the code defining the fields and tables as well as the relations, which match to foreign key constraints when the database definition files are generated.

# Creating your Schema

This example will be a pretty simple schema but enough to demonstrate generating a couple of tables with a foreign key constraint. Since a lot of people are doing social networking stuff these days we can assume a common use case would be a user who has a bunch of photos. So we&#8217;ll need to model a user table and a photo table, but first things first, lets create a few basic things, assuming we&#8217;ll call our application MySocialApp

{{< highlight bash >}}mkdir -p MySocialApp/lib/MySocialApp/Schema/Result
cd MySocialApp
{{< /highlight >}}

We need to create a schema file at _lib/MySocialApp/Schema.pm_ that inherits from the base <a href="http://search.cpan.org/~ribasushi/DBIx-Class-0.08102/lib/DBIx/Class/Schema.pm" target="_blank">DBIx::Class::Schema</a>

{{< highlight bash >}}package MySocialApp::Schema;
use base qw/DBIx::Class::Schema/;

use strict;
use warnings;
our $VERSION = '0.00001';

__PACKAGE__->load_namespaces();

__PACKAGE__->load_components(qw/Schema::Versioned/);

__PACKAGE__->upgrade_directory('sql/');

1;
{{< /highlight >}}

So here all we are doing is extending the base Schema and loading the <a href="http://search.cpan.org/~ribasushi/DBIx-Class-0.08102/lib/DBIx/Class/Schema/Versioned.pm" target="_blank">Versioned</a> component. We&#8217;re setting the directory where DDL and Diff files will be generated into the _sql_ directory. We&#8217;re also invoking <a href="http://search.cpan.org/~ribasushi/DBIx-Class-0.08102/lib/DBIx/Class/Schema.pm#load_namespaces" target="_blank" >load_namespaces</a> which tells it to look at the MySocialApp::Schema::Result namespace for our result classes by default.

## Creating the Result Classes

Here I will define what the database is going to look like

{{< highlight bash >}}package MySocialApp::Schema::Result::User;

use Moose;
use namespace::clean -except => 'meta';

extends 'DBIx::Class';

 __PACKAGE__->load_components(qw/Core/);
 __PACKAGE__->table('user');

 __PACKAGE__->add_columns(

    user_id => {
        data_type   => "INT",
        size        => 11,
        is_nullable => 0,
        is_auto_increment => 1,
        extra => { unsigned => 1 },
    },
    username => {
        data_type => "VARCHAR",
        is_nullable => 0,
        size => 255,
    },
);

__PACKAGE__->set_primary_key('user_id');

__PACKAGE__->has_many(
    'photos',
    'MySocialApp::Schema::Result::Photo',
    { "foreign.fk_user_id" => "self.user_id" },
);

1;
{{< /highlight >}}

This is relatively straightforward. I&#8217;m creating a user table that has an auto incrementing primary key and a username field. I&#8217;m also defining a <a href="http://search.cpan.org/~ribasushi/DBIx-Class-0.08102/lib/DBIx/Class/Relationship.pm" target="_blank">relationship</a> that says a user can have many photos. Lets create the photo class.

{{< highlight bash >}}package MySocialApp::Schema::Result::Photo;

use Moose;
use namespace::clean -except => 'meta';

extends 'DBIx::Class';

 __PACKAGE__->load_components(qw/Core/);
 __PACKAGE__->table('photo');

 __PACKAGE__->add_columns(

    photo_id => {
        data_type   => "INT",
        size        => 11,
        is_nullable => 0,
        is_auto_increment => 1,
        extra => { unsigned => 1 },
    },
    url => {
        data_type => "VARCHAR",
        is_nullable => 0,
        size => 255,
    },
    fk_user_id => {
        data_type   => "INT",
        size        => 11,
        is_nullable => 0,
        extra => { unsigned => 1 },
    },
);

__PACKAGE__->set_primary_key('photo_id');

 __PACKAGE__->belongs_to(
    'user',
    'MySocialApp::Schema::Result::User',
    { 'foreign.user_id' => 'self.fk_user_id' },
);


1;
{{< /highlight >}}

Same basic thing here, a photo class with a photo\_id primary key, a url field, and an fk\_user_id field that keys into the user table. Each photo belongs to a user, and this relationship will define our foreign key constraint when the schema is generated. 

# Versioning the Database

## Create the Versioning Scripts

We have the main DBIx::Class pieces in place to generate the database, but we&#8217;ll need a couple of scripts to support our versioned database. One script will generate the schema based on the version before it, introspecting which alterations have been made and producing a SQL diff file to alter the database. The other script will look at the database to see if it needs upgrading, and run the appropriate diff files to bring it up to the current version.

First the schema and diff generation script which we&#8217;ll call _script/gen_schema.pl_

{{< highlight bash >}}use strict;

use FindBin;
use lib "$FindBin::Bin/../lib";

use Pod::Usage;
use Getopt::Long;
use MySocialApp::Schema;

my ( $preversion, $help );
GetOptions(
        'p|preversion:s'  => \$preversion,
        ) or die pod2usage;

my $schema = MySocialApp::Schema->connect(
        'dbi:mysql:dbname=mysocialapp;host=localhost', 'mysocialapp', 'mysocialapp4u'
        );
my $version = $schema->schema_version();

if ($version && $preversion) {
    print "creating diff between version $version and $preversion\n";
} elsif ($version && !$preversion) {
    print "creating full dump for version $version\n";
} elsif (!$version) {
    print "creating unversioned full dump\n";
}

my $sql_dir = './sql';
$schema->create_ddl_dir( 'MySQL', $version, $sql_dir, $preversion );
{{< /highlight >}}

This script will be run anytime we change something in the Result files. You give it the previous schema version, and it will create a diff between that and the new version. Before running this you&#8217;ll update the $VERSION variable in lib/MySocialApp/Schema.pm so that it knows a change has been made.

The next script is the upgrade script, we&#8217;ll call it _script/upgrade_db.pl_

{{< highlight bash >}}use strict;

use FindBin;
use lib "$FindBin::Bin/../lib";

use MySocialApp::Schema;

my $schema = MySocialApp::Schema->connect(
        'dbi:mysql:dbname=mysocialapp;host=localhost', 'mysocialapp', 'mysocialapp4u'
        );

if (!$schema->get_db_version()) {
    # schema is unversioned
    $schema->deploy();
} else {
    $schema->upgrade();
}
{{< /highlight >}}

This script checks to see if any diffs need to be applied, and applies them if the version held by the database and the version in your Schema.pm file differ, bringing the database up to the correct schema version.

Note, in these scripts I&#8217;ve hardcoded the DB info which really should go into a configuration file. 

## Create a Database to Deploy Into

We need to create the database that our tables will be created in. In the connect calls above we&#8217;re using this user and password to connect to our database. I&#8217;m using MySQL for the example so this would be done on the MySQL command prompt

{{< highlight bash >}}mysql> create database mysocialapp;
Query OK, 1 row affected (0.00 sec)

mysql> grant all on mysocialapp.* to mysocialapp@'localhost' identified by 'mysocialapp4u';
Query OK, 0 rows affected (0.01 sec)
{{< /highlight >}}

## Deploy the Initial Schema

Now its time to deploy our initial schema into MySQL. But for the first go round we also have to create the initial DDL file, this way when we make changes in the future it can be compared against the Schema result classes to see what changes have been made. We can do this by supplying a nonexistent previous version to our gen_schema.pl script

{{< highlight bash >}}$ perl script/gen_schema.pl -p 0.00000
Your DB is currently unversioned. Please call upgrade on your schema to sync the DB.
creating diff between version 0.00001 and 0.00000
No previous schema file found (sql/MySocialApp-Schema-0.00000-MySQL.sql) at /home/jgoulah/perl5/lib/perl5/DBIx/Class/Storage/DBI.pm line 1685.
{{< /highlight >}}

And we can see the DDL file now exists

{{< highlight bash >}}$ ls sql/
MySocialApp-Schema-0.00001-MySQL.sql
{{< /highlight >}}

Then we need to deploy to MySQL for the first time so we run the upgrade script

{{< highlight bash >}}$ perl script/upgrade_db.pl
Your DB is currently unversioned. Please call upgrade on your schema to sync the DB.
{{< /highlight >}}

Now check out what its created in MySQL

{{< highlight bash >}}mysql> use mysocialapp;
Database changed
mysql> show tables;
+----------------------------+
| Tables_in_mysocialapp      |
+----------------------------+
| dbix_class_schema_versions |
| photo                      |
| user                       |
+----------------------------+
3 rows in set (0.00 sec)

{{< /highlight >}}

There is our photo table, our user table, and also a dbix\_class\_schema_versions table. This last table just keeps track of what version the database is. You can see we are in sync with the Schema class by selecting from that table and also when this version was installed.

{{< highlight bash >}}mysql> select * from dbix_class_schema_versions;
+---------+---------------------+
| version | installed           |
+---------+---------------------+
| 0.00001 | 2009-05-17 21:59:57 |
+---------+---------------------+
1 row in set (0.00 sec)
{{< /highlight >}}

The really great thing is that we have created the tables -and- constraints based on our schema result classes. Check out our photo table

{{< highlight bash >}}mysql> show create table photo\G
*************************** 1. row ***************************
       Table: photo
Create Table: CREATE TABLE `photo` (
  `photo_id` int(11) unsigned NOT NULL auto_increment,
  `url` varchar(255) NOT NULL,
  `fk_user_id` int(11) unsigned NOT NULL,
  PRIMARY KEY  (`photo_id`),
  KEY `photo_idx_fk_user_id` (`fk_user_id`),
  CONSTRAINT `photo_fk_fk_user_id` FOREIGN KEY (`fk_user_id`) REFERENCES `user` (`user_id`) ON DELETE CASCADE ON UPDATE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=latin1
1 row in set (0.00 sec)
{{< /highlight >}}

# Making Database Changes

Lets say we want to add a password field to our user table. I&#8217;d open up the _lib/MySocialApp/Schema/Result/User.pm_ file and add a section for my password field to the add_columns definition so now it looks like:

{{< highlight bash >}}__PACKAGE__->add_columns(

    user_id => {
        data_type   => "INT",
        size        => 11,
        is_nullable => 0,
        is_auto_increment => 1,
        extra => { unsigned => 1 },
    },
    username => {
        data_type => "VARCHAR",
        is_nullable => 0,
        size => 255,
    },
    password => {
        data_type => "VARCHAR",
        is_nullable => 0,
        size => 255,
    },
);
{{< /highlight >}}

Then we update the _lib/MySocialApp/Schema.pm_ file and update to the next version so it looks like

{{< highlight bash >}}our $VERSION = '0.00002';{{< /highlight >}}

To create the DDL and Diff for this version we run the gen_schema script with the _previous_ version as the argument

{{< highlight bash >}}$ perl script/gen_schema.pl -p 0.00001
Versions out of sync. This is 0.00002, your database contains version 0.00001, please call upgrade on your Schema.
creating diff between version 0.00002 and 0.00001
{{< /highlight >}}

If you look in the sql directory there are two new files. The DDL is named _MySocialApp-Schema-0.00002-MySQL.sql_ and the diff is called _MySocialApp-Schema-0.00001-0.00002-MySQL.sql_ and has the alter statement

{{< highlight bash >}}$ cat sql/MySocialApp-Schema-0.00001-0.00002-MySQL.sql
-- Convert schema 'sql/MySocialApp-Schema-0.00001-MySQL.sql' to 'MySocialApp::Schema v0.00002':;

BEGIN;

ALTER TABLE user ADD COLUMN password VARCHAR(255) NOT NULL;

COMMIT;
{{< /highlight >}}

Now we can apply that with our upgrade script

{{< highlight bash >}}$ perl script/upgrade_db.pl
Versions out of sync. This is 0.00002, your database contains version 0.00001, please call upgrade on your Schema.

DB version (0.00001) is lower than the schema version (0.00002). Attempting upgrade.
{{< /highlight >}}

and see that the upgrade has been made in MySQL

{{< highlight bash >}}mysql> describe user;
+----------+------------------+------+-----+---------+----------------+
| Field    | Type             | Null | Key | Default | Extra          |
+----------+------------------+------+-----+---------+----------------+
| user_id  | int(11) unsigned | NO   | PRI | NULL    | auto_increment |
| username | varchar(255)     | NO   |     |         |                |
| password | varchar(255)     | NO   |     |         |                |
+----------+------------------+------+-----+---------+----------------+
3 rows in set (0.00 sec)
{{< /highlight >}}

# Conclusion

So now we&#8217;ve seen a really easy way that we can maintain our database schema from our ORM code. We have a versioned database that is under revision control, and can keep our stage and production environments in sync with our development code. With this type of setup its also easy to maintain branches with different database changes and merge them into the mainline, and it also ensures that the database you are developing on is always in sync with the code.

 [1]: http://search.cpan.org/~ribasushi/DBIx-Class-0.08102/
