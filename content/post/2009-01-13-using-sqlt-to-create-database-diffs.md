---
title: Using SQLT to Create Database Diffs
author: jgoulah
type: post
date: 2009-01-14T04:19:18+00:00
url: /2009/01/using-sqlt-to-create-database-diffs/
categories:
  - Schema Versioning
tags:
  - CPAN
  - Database Diff
  - DDL
  - perl
  - SQL::Translator
  - sqlfairy
  - SQLT
  - Versioning

---
# Overview

Database diffs can be useful for a variety of use cases, from outputting the differences in your development versus production schema to full blown database <a href="http://search.cpan.org/~ash/DBIx-Class-0.08010/lib/DBIx/Class/Schema/Versioned.pm" target="_blank">versioning systems</a>.   A database diff is similar in nature to a <a href="http://en.wikipedia.org/wiki/Diff" target="_blank">file diff</a>, but instead of outputting the differences between two files,  it will output the set of alterations to be made to bring one databases <a href="http://en.wikipedia.org/wiki/Database_schema" target="_blank">schema</a> in sync with another.   There are a variety of tools that will accomplish this for you, but I am going to use a tool called <a href="http://sqlfairy.sourceforge.net/" target="_blank">SQL::Translator</a>.   SQLT can do quite a lot for you, such as converting among different schema dialects (think MySQL to Oracle) or outputting an <a href="http://en.wikipedia.org/wiki/Entity-relationship_model" target="_blank">ERD</a> of your database, but I&#8217;ll stick with using it as a simple diff tool for this article.

# Installation

The tool can be <a href="http://search.cpan.org/dist/SQL-Translator/" target="_blank">found here</a> on [CPAN][1].   I suggest using <a href="http://search.cpan.org/~mstrout/local-lib-1.002000/lib/local/lib.pm" target="_blank">local::lib</a> for perl module installation to avoid needing to install things as root.  I&#8217;ve written up detailed instructions in the <a href="http://www.catalystframework.org/calendar/2007/8" target="_blank">past</a>, but its pretty straightforward if you follow the bootstrap instructions.  In any case, if you are lucky you will be able to just do a normal cpan install like so:

`cpan SQL::Translator`

For our examples you&#8217;ll also need the <a href="http://search.cpan.org/~capttofu/DBD-mysql-4.010/lib/DBD/mysql.pm" target="_blank">DBD::MySQL</a> driver. This can be a real pain to install from CPAN so you may want to try to use your systems package management for this. Under Debian this is called libdbd-mysql-perl and in Redhat its called perl-DBD-MySQL. 

# Diffing Two DDL Files

Once you have SQLT installed, you&#8217;ll want to grab the <a href="http://en.wikipedia.org/wiki/Data_Definition_Language" target="_blank">DDL</a> files from the databases that you are interested in producing diffs for.  If you are using mysql this is easy enough to do with mysqldump.

So lets say you want to compare your development and production databases, we&#8217;d grab those schema dumps, assuming your dev db is on host &#8216;dev&#8217; and prod db on host &#8216;prod&#8217;:

`mysqldump --no-data -h dev -u your_dbuser -p your_dbname > dev_db.sql<br />
mysqldump --no-data -h prod -u your_dbuser -p your_dbname > prod_db.sql`

The SQLT install comes with a handy script called sqlt-diff which you can use to diff two different DDL files. So in the case of MySQL all you have to run is:

`sqlt-diff  dev_db.sql=MySQL prod_db.sql=MySQL > my_diff.sql`

You should be able to pass other arguments where we&#8217;ve used &#8216;MySQL&#8217; here, and you can get a list via:

`sqlt -l`

# Diffing DDL Against a Database

The above may be all you need, but perhaps you want to diff a DDL against the actual database for whatever reason. I recently used this approach to write a versioning system in which the database changes could be output into a file containing the alterations as well as a new DDL for each &#8220;version&#8221;, this way changes are not lost or forgotten.

SQLT doesn&#8217;t currently have any scripts to handle this (yet) so we&#8217;ll have to use the perl module itself to do what we want. A simple perl script is all we need.

<pre>#!/usr/bin/perl

use strict;
use warnings;
use SQL::Translator;
use SQL::Translator::Diff;
use DBI;

# set the path of the file you want to diff against the DB
$ddl_file = "/path/to/file_to_diff.sql";

# make sure to fill in db_name, host_name, and db_port
my $dsn = "DBI:mysql:database=db_name;host=host_name;port=db_port";

# you'll have to set the db_user and db_pass here
DBI->connect($dsn, 'db_user', 'db_pass', {'RaiseError' => 1});

# create a SQLT object that connect to the database via our dbh
my $db_tr = SQL::Translator->new({
      add_drop_table => 1,
      parser => 'DBI',
      parser_args => {  dbh => $dbh }
    });
$db_tr->parser('SQL::Translator::Parser::DBI::MySQL');


$db_tr->parse($db_tr, $dbh);

# create a SQLT object that corresponds to our ddl file
my $file_tr = SQL::Translator->new({
    parser => 'MySQL'
  });
my $out = $file_tr->translate( $ddl_file ) or die $file_tr->error;

# perform the actual diff (some of the options you use may vary)
my $diff = SQL::Translator::Diff::schema_diff(
    $file_tr->schema,  'MySQL',
    $db_tr->schema,  'MySQL',
    {
    ignore_constraint_names => 1,
    ignore_index_names => 1,
    caseopt => 1
    }
);

# now just print out the diff
my $file;
my $filename = "diff_output.sql"
if(!open($file, ">$filename")) {
  die("Can't open $filename for writing ($!)");
  next;
}
print $file $diff;
</pre>

So we&#8217;ve used the SQL Translator tool to read in our database, read in our DDL, and produce a diff. Pretty neat!

# Conclusion

We&#8217;ve really only scratched the surface with some of the capabilities of SQL::Translator, but its a pretty handy thing to be able to easily diff various schemas. Its important that your development and production schema are in sync, and stay in sync, and this gives an easy way to start on automating that process.

 [1]: http://search.cpan.org/