---
title: Intelligent Version Control and Branching
type: post
date: 2009-01-11T17:28:41+00:00
url: /2009/01/intelligent-version-control-and-branching/
categories:
  - Version Control
tags:
  - distributed
  - Git
  - SVK
  - SVN
  - Version Control

---
# Overview

Most shops these days seem to know that using a <a href="http://en.wikipedia.org/wiki/Revision_control" target="_blank">Version Control System</a> is necessary for the organization of a large software project involving multiple developers.   It&#8217;s essential to allow each developer to work on their part of the project and commit the changes to a central repository, which the rest of the developers can then pull down into their working sandbox.  Its an effective way to develop that avoids overwriting of changes, and allows for easy rollback and history.  However, one problem I see over and over again is either lack of branching altogether,  or doing it in a manual way that drifts the branch further out of sync with trunk as time goes on, and then trying to manually merge this with the trunk code when the branch is done.   For those who don&#8217;t know, trunk is basically your mainline, in which code can be _branched_ off of for features that won&#8217;t interfere with trunk until they are _merged_ at some later point.  This allows for testing changes without bothering other developers, but still being able to keep a commit history.

The difficult thing about this branching process is that if you try to manually merge code that has been on a branch for some time, the continuous development on trunk will have brought the two to such different places that you end up trying to resolve a ton of conflicts by hand.  Conflicts happen when changes are made to the same file and same lines in that file by different people.  Trunk is eventually a different bunch of code than when the branch was started, so by the end of your branch you could be developing against what is essentially a different codebase.  Anyone that has tried to do merging manually knows how painful this process can be.

Luckily, tools exist to help us with the process.  This article is going to focus on two tools that can interact with an <a href="http://subversion.tigris.org/" target="_blank">SVN</a> repository to help us with intelligent branching.   The tools I&#8217;m going to review are <a href="http://en.wikipedia.org/wiki/SVK" target="_blank">SVK</a> and <a href="http://en.wikipedia.org/wiki/Git_(software)" target="_blank">Git</a>, and I will go over each separately. In fact they should not be used simultaneously since Git doesn&#8217;t know anything about the metadata SVK expects to be in your SVN props. In other words do not attempt to create a branch with SVK and then merge with Git, it just won&#8217;t work. Git by the way, can be used standalone, without SVN,  but I&#8217;m going to show you how to use it with SVN because that seems to be the version control system of choice in most shops these days. SVK is just a wrapper around SVN, its not a standalone tool. The great thing about both of these is you don&#8217;t necessarily have to migrate all of your developers over at once.  They can continue day to day work flow while you harness the power of branching.  Eventually people will see the advantage and try it out too.  Under the covers you are still storing your code in an SVN repository.

# Using SVK

Installing SVK from source is well beyond the scope of this article, so I&#8217;m assuming you have a <a href="http://en.wikipedia.org/wiki/Package_management" target="_blank">package management system</a> that will handle this installation for you.  Once you have SVK installed, using it is very straightforward for SVN users.  In fact, its mostly the same exact commands, with a few additions.  Like SVN, you can do &#8216;svk help <command>&#8217;  to get detailed help on a particular command.

### Checking Out Your Repository

Checking out your repository is very similar to SVN, and is done with the &#8216;checkout&#8217; command:

`svk co {repo}`

where {repo} is your repository url (eg. http://svn.yourdomain.com/svn/repo)

You&#8217;ll be prompted for a base URI to mirror from and its ok to press enter here:

`New URI encountered: http://svn.yourdomain.com/svn/repo<br />
Choose a base URI to mirror from (press enter to use the full URI): ENTER`

You&#8217;ll be prompted for a depot path and its also ok to accept the default

`Depot path: [//mirror/repo] ENTER`

SVK is a <a href="http://en.wikipedia.org/wiki/Distributed_revision_control" target="_blank">decentralized version control system</a> and thus needs to mirror your SVN repository locally. You&#8217;ll see a prompt like this and you&#8217;ll be ok to accept the default:

`Synchronizing the mirror for the first time:<br />
a        : Retrieve all revisions (default)<br />
h        : Only the most recent revision<br />
-count   : At most 'count' recent revisions<br />
revision : Start from the specified revision<br />
a)ll, h)ead, -count, revision? [a] ENTER`

This step can take quite a while, depending on how large your repository is. Just be patient and it will finish up eventually.

### Working with SVK

From here SVK is very similar to SVN. You&#8217;ll have checked out the entire repository, so if your SVN is setup according to standard you will see a trunk, branches, and tags directory under your repository root folder.

SVK has the same command set with one caveat, that when you update you have to give the -s flag:

`svk up -s`

This allows SVK to sync the mirror with the repository before updating your code.

### Branching With SVK

The point of this article is how to use these tools for branching, so lets create a branch. There are a few ways to do this but a nice way is to copy trunk to a branch on the mirror itself:

`svk cp //mirror/{repo}/trunk //mirror/{repo}/branches/my_branch_name<br />
cd {repo}/branches<br />
svk up -s  # pull the new branch back from the mirror`

Now you can work, work, work, and commit as you please. Here is where SVK really shines. You can pull from the repository path you branched from at any time. In this case we branched from trunk, so we can _pull_ trunks changes into our branch.

`svk pull`

SVK will track all of the metadata surrounding the change, and if there happens to be a conflict it is resolved at pull time. Now the trunk code is merged into your branch. You are in sync with trunk, in addition to your branches changes.

The last thing to know is how to _push_ your branch back to trunk when you are done. It doesn&#8217;t hurt to do one last pull to make sure you are in sync with trunk, and then:

`svk push -l`

Now your branch has been merged back to trunk. That is it! Painless and easy. You&#8217;ve already resolved the conflicts at pull time which SVK has tracked, so the push is simple. SVK has done all the hard work for us.

Its a good habit to get rid of your branch once you are done:

`rm -rf branches/branch_name<br />
svk rm branches/branch_name<br />
svk commit -m "deleted old branch" branches/branch_name`

# Using Git

Using Git with SVN is a similar process, and like SVK, Git is also a <a href="http://http://en.wikipedia.org/wiki/Distributed_revision_control" target="_blank">distributed</a> system. In a nutshell this means we can have local branches and remote branches as well. Its important to understand this concept when using Git much more so than when using SVK.  Its probably a good idea to go over the <a href="http://git.or.cz/course/svn.html" target="_blank">SVN to Git tutorial</a> to see how familiar SVN commands map to Git.

### Checking Out Your Repository

With Git you essentially clone your SVN repository. Here I&#8217;m using a repository called testrepo located on the same host so I am using the file protocol. You can use whichever protocol you normally use to access your SVN repository.

`git svn clone --prefix=svn/ file:///usr/local/svn/testrepo -T trunk -b branches -t tags`

You don&#8217;t have to use the &#8211;prefix here, but it allows you to be able to distinguish between local branches and those representing remote svn branches easily, so I highly recommend it.

### Viewing Local and Remote Branches

With Git its important to remember that we have both local and remote branches. Local branches only exist for our sandbox, while remote branch are shared out on the SVN repository for others to use.

You can view your local branches like so:

`$ git branch<br />
* master`

So far we only have the default master branch. When git svn is finished importing all of the svn history it sets up a master branch for you to work with. The tip of that branch will be the latest commit in the svn repository (note this is not necessarily trunk). The star indicates the current branch that we are on.

Its not a bad idea to have master correspond to trunk, and to be sure we can run:

`git reset --hard svn/trunk`

You can also view your remote branches. These exist out on the SVN repository we&#8217;ve cloned:

`$ git branch -r<br />
svn/testbug<br />
svn/trunk`

So far I can see that a trunk exists, as well as a branch called testbug. But these are remote, so to work with them we need to pull them down locally. Right now I&#8217;m interested in trunk, so we&#8217;ll grab that:

`$ git checkout -b trunk svn/trunk<br />
Switched to a new branch "trunk"`

Anytime you would like to update the code with changes from the repository, you can do it like so:

`git svn rebase`

### Branching With Git

Now lets assume we want to create a new branch called foo. This command will actually create the new branch on the SVN repository.

`$ git svn branch foo`

Since the branch is still only remote, we need to get a local copy:

`$ git checkout -b foo svn/foo`

Now lets actually do something on the branch, so we can show how merge operations work. We&#8217;ll just create a simple file:

`$ touch myfile<br />
$ echo "stuff" > myfile<br />
$ cat myfile`

And commit that to the local repository:

`$ git add myfile<br />
$ git commit -m "my new file"`

Note this only commits to the local repository and other users on the branch won&#8217;t see the change until we commit it to the remote repository. Its a good idea to do a dry run to make sure we are committing to the right place. We can use &#8216;-n&#8217; for dry run:

`$ git svn dcommit -n<br />
Committing to file:///usr/local/svn/testrepo/branches/foo ...`

This looks correct, its going to commit to branches/foo on our remote repository, so run the command without the -n option.

`$ git svn dcommit`

Now lets switch back to trunk so we can perform a merge.

`$ git checkout trunk`

Now that we are on trunk, our new file doesn&#8217;t exist, which makes sense since it was created on the branch. But we want to merge our branch foo back into trunk, which can be done like so:

`$ git merge --no-ff foo`

And again we want to commit that local change to the remote repository:

`$ git svn dcommit`

There&#8217;s one more important thing to note. Normally you don&#8217;t need &#8211;no-ff unless you are using git against svn like this. &#8211;no-ff is needed so you get a real merge commit. If you don&#8217;t have that and a fast-forward is possible you&#8217;ll end up with the commit from the svn branch on top of the history, in which case the branch that commit on top is part of will be used for committing.

Therefore its a good idea to add this option to the default merge options for anything maintained with git-svn. Open up the .git/config and add the following:

`branch.foo.mergeoptions = --no-ff`

This allows you to use git merge and not remember the &#8211;no-ff option.

# Conclusion

We&#8217;ve seen how to use two distributed revision control systems to help us manage our remote SVN repositories. We know how to create a branch, and stay in sync with the development on trunk. We have also learned now to merge our changes from the branch back into trunk in a very painless and easy way.
