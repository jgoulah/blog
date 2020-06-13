---
title: Migrating SVN to Git
author: jgoulah
type: post
date: 2009-11-01T22:55:44+00:00
url: /2009/11/migrating-svn-to-git/
categories:
  - Version Control
tags:
  - Git
  - grafts
  - repository migration
  - SVN

---
## Overview

SVN migrations to Git can vary in complexity, depending on how old the repository is and how many branches were created and merged, as well as if you use regular SVN or a facade such as SVK to do your branching and merging. If you have a fairly new repository and the standard setup of a _trunk_, _branches_, and _tags_ directory, your job could be pretty easy. However if you&#8217;ve done a ton of branching and merging, or your repository follows a non-standard directory setup, or that setup changed over time, you could have a bit more work on your hands. 

## SVN to Git Conversion

### Building Git

The very first thing to do is make sure you are running the newest git. And for this I recommend from source. So grab it and do the usual compile stuff

{{< highlight bash >}}$ wget http://kernel.org/pub/software/scm/git/git-1.6.5.2.tar.bz2
$ tar xjf git-1.6.5.2.tar.bz2 
$ cd git-1.6.5.2
$ ./configure
$ make
{{< /highlight >}}

Now I&#8217;m not sure if this is because I am always using [local::lib][1] but I tend to get this error after a while of building. If you don&#8217;t get it you can obviously skip this step

{{< highlight bash >}}Only one of PREFIX or INSTALL_BASE can be given
{{< /highlight >}}

This is fixable by going into the _git/perl_ directory and manually building the perl modules

{{< highlight bash >}}$ cd perl/
$ perl Makefile.PL 
{{< /highlight >}}

Then go back up a directory and finish the build

{{< highlight bash >}}$ cd ..
$ make
$ sudo make install
{{< /highlight >}}

### Using git-svn

If you have made any merges using SVK then you should grab the _git-svn_ from Sam Vilain&#8217;s git branch [here][2]. I had to download it because the _git clone_ wasn&#8217;t working so grab it like so

{{< highlight bash >}}$ wget http://download.github.com/samv-git-ccef8e0.tar.gz{{< /highlight >}}

Open it up, and copy the git-svn.perl script into your _bin_ directory

{{< highlight bash >}}$ tar xzvf samv-git-ccef8e0.tar.gz
$ cd samv-git-ccef8e0
$ cp git-svn.perl ~/bin/git-svn
{{< /highlight >}}

Now you can git svn clone your repository with the _git-svn_ command. The &#8211;prefix=svn/ is necessary because otherwise the tools can&#8217;t tell apart SVN revisions from imported ones. If you are using the standard trunk, branches, tags layout you&#8217;ll just put &#8211;stdlayout like we have below. However if you had something different you may have to pass the &#8211;trunk, &#8211;branches, and &#8211;tags in to identify what is what. For example if your repository structure was _trunk/companydir_ and you branched that instead of trunk, you would probably want &#8216;_&#8211;trunk=trunk/companydir &#8211;branches=branches_&#8216;. You can optionally provide an authors file, and the last option to our script is the repository URL.

{{< highlight bash >}}$ git-svn clone --prefix=svn/ --stdlayout --authors-file=authors.txt http://example.com/svn
{{< /highlight >}}

This can take a few minutes or as long as a few hours depending how big your repository is. When its done you should end up with a git checkout of your repository. If you look at your remote branches you&#8217;ll see that its ported our svn stuff with the svn prefix we used

{{< highlight bash >}}$ git branch -r
  svn/dbic_versioning
  svn/feed_source
  svn/trunk
{{< /highlight >}}

Anyway at this point it doesn&#8217;t hurt to create a tarball of your project so you can get back to it later if you screw up.

### Fixing the Git Repository

There are a few more scripts you need to grab to finish this off. We want to clone the [git-svn-abandon][3] repository for these.

{{< highlight bash >}}$ git clone git://github.com/nothingmuch/git-svn-abandon.git 
{{< /highlight >}}

Now go into that directory and copy the new scripts into your local bin directory, or wherever you feel is good for your purposes

{{< highlight bash >}}$ cd git-svn-abandon
$ cp git-svn-abandon-* ~/bin/
{{< /highlight >}}

You can now run _git-svn-abandon-fix-refs_, which will run through all the imported refs, recreating properly dated annotated tags, and makes branches out of everything else. It&#8217;ll also rename trunk to master. 

{{< highlight bash >}}$ git-svn-abandon-fix-refs
{{< /highlight >}}

We can see that it took the things that our _git branch -r_ command earlier listed and imported them into actual git branches, as well as putting us on the master branch which is analogous to the trunk from svn

{{< highlight bash >}}$ git branch
  dbic_versioning
  feed_source
* master
{{< /highlight >}}

#### Grafting Your Branches

Depending how correct you want your repository, you could skip this step. However it could be a good idea to try to get your merge history correct. This requires a little bit of understanding of the git internals, so at a minimum you should read and understand [this introduction][4]. There is also a nice presentation of how git works [here][5] that you may consider watching. 

So now you understand (you went over those links right?) that the way git figures out merges is such that it basically creates a new node on the graph that points to both the head of master and of the branch. And then it basically takes the file contents from the 3 points, the new merge base node (the ancestor), and each parent, deltafies them and applies the result to the ancestor. Obviously if they don&#8217;t apply cleanly a conflict is created and must be resolved. The previous revisions contents is no longer relevant at all, except as a part of history, and our new snapshot of the history is born. So this is basically what we have to fix. 

This is a snapshot from gitk of my repository. We can see near the bottom that it branched properly, but even though the repository was merged in svn, it doesn&#8217;t appear to be merged in our migration to git. 

![][6]

The way to fix this is to edit the _.git/info/grafts_ file. With my great photoshop skills I&#8217;m going to point out what goes in there

![][7]

We need to take what the first arrow is pointing to, and tell git that its parents are its current parent (the third arrow) as well as the point the branch was merged in (the second arrow). So all we have to do is add the hash from each of those commits, put the top node first, then the second and third with a space in between each to the _.git/info/grafts_ file. 

After we add those to the _.git/info/grafts_ file we can just reload gitk or gitx to see those changes that visually show the branch was merged

![][8]

When you are done you can run _git-svn-abandon-cleanup_ which cleans up SVK style merge commit messages and removes git-svn-id strings. Another important thing that happens is the grafts entries are incorporated into the filtered commits, so the extra merge metadata becomes clonable

{{< highlight bash >}}$ git svn-abandon-cleanup
{{< /highlight >}}

### Publish Your Repository

If you followed the [last article][9] you&#8217;ve got a gitosis setup that we can use to store our code centrally. So depending what host this is on you can import it like so

{{< highlight bash >}}$ git remote add origin git@localhost:repository.git
$ git push --all
$ git push --tags
{{< /highlight >}}

And we&#8217;re done.

## Conclusion

We&#8217;ve gone over how to migrate an SVN repository to Git and how to deal with some of the complications of how Git interprets our repository based on what was in SVN and how to go about fixing that. Hopefully we also learned a little bit about Git too in the process. 

### References

  * [Nothingmuch&#8217;s writeup on SVN to Git][10]
  * [Intro to Git Internals][4]
  * [Presentation of Git Internals][5]

 [1]: http://search.cpan.org/~apeiron/local-lib-1.004008/lib/local/lib.pm
 [2]: http://github.com/samv/git
 [3]: http://github.com/nothingmuch/git-svn-abandon
 [4]: http://bit.ly/2uxVcN
 [5]: http://bit.ly/301eke
 [6]: http://blog.johngoulah.com/wp-content/uploads/2009/11/tree_unmerged.png
 [7]: http://blog.johngoulah.com/wp-content/uploads/2009/11/tree_unmerged_with_arrows.png
 [8]: http://blog.johngoulah.com/wp-content/uploads/2009/11/tree_merged.png
 [9]: http://blog.johngoulah.com/2009/10/setting-up-gitosis/
 [10]: http://blog.woobling.org/2009/06/git-svn-abandon.html
