---
title: Make Browsing Code Easier with Ack and Ctags
author: jgoulah
type: post
date: 2009-04-20T01:44:30+00:00
url: /2009/04/make-browsing-code-easier/
categories:
  - Organization
tags:
  - ack
  - c++
  - ctags
  - java
  - perl
  - php

---
# Intro

When working with open source software, its essential to know how to navigate large code bases, perhaps unfamiliar and quite large. There are a few tools I use to do this that should be part of any developers arsenal, and they are: ack and ctags.

<a title="ack" href="http://betterthangrep.com/" target="_blank">Ack</a> can be thought of as a faster and more powerful grep. It searches recursively by default, ignores binary files and most version control files (think .svn), lets you specify file types on search, use perl regular expressions, and has easier to read output than grep.

<a title="ctags" href="http://ctags.sourceforge.net/" target="_blank">Ctags</a> is a tool that many are familiar with, and there are tons of articles about it already.  But I bring it up so I can show some quick starter scripts that you&#8217;d use to generate the tags for PHP or Perl scripts.  I&#8217;ll show a quick C++ and Java example too, since I use those from time to time.

# Ctags

## Installation

There&#8217;s really not much to installing ctags.  You could <a title="ctags download" href="http://prdownloads.sourceforge.net/ctags/ctags-5.7.tar.gz" target="_blank">download</a> and compile the source, but just get it from your package management system.

If you&#8217;re using Debian or Ubuntu you can do:

<pre>sudo apt-get install exuberant-ctags</pre>

Similarly in CentOS and Redhat based distros:

<pre>sudo yum install ctags</pre>

## Usage

Ctags basically indexes your code and creates a tag file that can then be used in your editor to literally jump around your code.  If you see a method call as you&#8217;re browsing code, you can jump to the definition of that method with one keystroke, and back to where you were.  Same thing for variables.  In a keystroke you can jump to see where its defined.   As you jump through code a stack is created,  and as you jump back you are just popping off that stack, also known as <a title="LIFO" href="http://en.wikipedia.org/wiki/LIFO_(computing)" target="_self">LIFO</a>, by the way.

## Generating the Tag Files

I have a few scripts to generate the ctags files depending on different codebases. I tend to use <a href="http://en.wikipedia.org/wiki/Vi" target="_self">VI</a> so I&#8217;m going to cover how to do it with that editor, but you can also use <a href="http://ctags.sourceforge.net/ctags.html#HOW%20TO%20USE%20WITH%20GNU%20EMACS" target="_self">emacs</a>.

For these examples I&#8217;m going to send the output to a file in my _~/.vim/tags/_ directory, which I&#8217;ll later add to _.vimrc_. You could expand these scripts to fit your needs. For these examples they are pretty basic and hardcoded.

#### PHP

<pre>$ cat bin/ctags_php
#!/bin/bash
cd ~/mysandbox/myphpproject
ctags -f ~/.vim/tags/myphpproject \
--langmap="php:+.inc" -h ".php.inc" -R \
--exclude='*.js' \
--exclude='*.sql' \
--totals=yes \
--tag-relative=yes \
--PHP-kinds=+cf-v \
--regex-PHP='/abstract\s+class\s+([^ ]+)/\1/c/' \
--regex-PHP='/interface\s+([^ ]+)/\1/c/' \
--regex-PHP='/(public\s+|static\s+|abstract\s+|protected\s+|private\s+)function\s+\&?\s*([^ (]+)/\2/f/'</pre>

and when you run it, you&#8217;ll see something like:

<pre>$ ctags_php
498 files, 66678 lines (2624 kB) scanned in 0.6 seconds (4604 kB/s)
1643 tags added to tag file
1643 tags sorted in 0.00 seconds</pre>

#### Perl

In Perl you can do things a bit smarter since you _should_ have a Makefile.PL script to keep track of your dependencies. If so you can add:

<pre>postamble(
q!
tags:
ctags -f ~/.vim/tags/myperlcode --recurse --totals \
--exclude=blib \
--exclude='*~' \
--languages=Perl --langmap=Perl:+.t \
!);</pre>

Then you should do:

<pre>perl Makefile.PL
make tags</pre>

#### C++

You can do a very similar thing for C++ code

<pre>$ cd /path/to/code
$ ctags -f ~/.vim/tags/myc++code --tag-relative=yes --recurse --language-force=c++ *
</pre>

#### Java

Or say we want to tag the entire java library itself

<pre>$ ctags -f ~/.vim/tags/java -R --language-force=java /opt/java/src</pre>

### Letting VI know about your files

After creating one or more tagfiles you should edit your _~/.vimrc_ file and add the location to your tag files and separate the entries by commas or spaces

<pre>set tags=~/.vim/tags/myphpproject,~/.vim/tags/myperlcode</pre>

### Navigating Around the Code

In VI there are two easy commands to jump around.

To move to the definition of a method/variable, place the cursor over it and press

<pre>Ctrl + ]</pre>

And to jump back

<pre>Ctrl + t</pre>

If you try to jump to something and it isn&#8217;t found, its probably something in a library you&#8217;re using, so you&#8217;ll have to grab the source and tag those too.

# Ack

## Installation

There are a few ways to install ack listed on the <a title="ack" href="http://betterthangrep.com/" target="_blank">ack homepage</a>.   If you are familiar with CPAN you can install <a title="cpan ack" href="http://search.cpan.org/dist/ack/" target="_blank">App::Ack</a>, or if you want to use package management you can grab _ack-grep_ on Ubuntu or _ack_ on Redhat based distros.

## Usage

The best thing I can really tell you is to read the ack help

<pre>$ ack --help</pre>

Ack takes a regular expression as the first argument and a directory to search as the second. Typically you want to search all (_-a_) files or in a case insensitive fashion (_-i_)

<pre>$ ack -ai 'searchstring' .</pre>

Or you can search specific file types 

<pre>$ ack --perl  searchterm</pre>

And one really cool thing is though ack gives nice colorized output in a structured fashion, if you pipe it to another process it outputs like grep by default so that you can continue to pipe it

For example lets say I want to find the modules <a href="http://search.cpan.org/~jjnapiork/MooseX-Types-0.10/"  target="_blank">MooseX::Types</a> is using 

<pre>ack -a '^use.*;' ~/perl5/lib/perl5/MooseX/Types/</pre>

It gives something looking like (output truncated)

<pre>/home/jgoulah/perl5/lib/perl5/MooseX/Types/Util.pm
9:use warnings;
10:use strict;
12:use base 'Exporter';

/home/jgoulah/perl5/lib/perl5/MooseX/Types/Moose.pm
9:use warnings;
10:use strict;
12:use MooseX::Types;
13:use Moose::Util::TypeConstraints ();
15:use namespace::clean -except => [qw( meta )];
</pre>

If you pipe the output it looks more like grep output

<pre>ack -a '^use.*;' ~/perl5/lib/perl5/MooseX/Types/ | cat</pre>

Again the output is truncated but looks like

<pre>/home/jgoulah/perl5/lib/perl5/MooseX/Types/Base.pm:11:use MooseX::Types::Util             qw( filter_tags );
/home/jgoulah/perl5/lib/perl5/MooseX/Types/Base.pm:12:use Sub::Exporter                   qw( build_exporter );
/home/jgoulah/perl5/lib/perl5/MooseX/Types/Base.pm:13:use Moose::Util::TypeConstraints;
/home/jgoulah/perl5/lib/perl5/MooseX/Types/Base.pm:15:use namespace::clean -except => [qw( meta )];
</pre>

Lets say for example reasons I wanted to find all the modules used in the code and make sure they are installed (using the &#8216;make install&#8217; command in the module would be the more correct and easier way), you could do

<pre>$ ack -ah '^use\s[^\d].*;' ~/perl5/lib/perl5/MooseX/Types/ | \
  ack -v 'warnings|strict|base' | \
  perl -ne "m|use ((\w+:?:?)+)(.*)(;)|; print qq{\$1\n};" | \
  sort -u | xargs cpan
</pre>

which cleans the output and gives the module list to cpan for installation and it lets me know I have everything installed and up to date

<pre>Carp::Clan is up to date (6.00).
Class::MOP is up to date (0.81).
Devel::PartialDump is up to date (0.07).
Moose is up to date (0.74).
Moose::Meta::TypeConstraint::Union is up to date (0.74).
Moose::Util::TypeConstraints is up to date (0.74).
MooseX::Meta::TypeConstraint::Structured is up to date (undef).
MooseX::Types is up to date (0.10).
MooseX::Types::Util is up to date (undef).
namespace::clean is up to date (0.11).
Scalar::Util is up to date (1.19).
Sub::Exporter is up to date (0.982).
</pre>

# Conclusion

These are pretty commonplace but great tools to know if you don&#8217;t already. Try to integrate them into your work flow and I think you&#8217;ll notice that it will speed you up quite a bit, especially when you are browsing through unfamiliar territory.