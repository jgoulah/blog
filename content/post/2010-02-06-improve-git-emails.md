---
title: Improve Git Emails
author: jgoulah
type: post
date: 2010-02-06T20:02:25+00:00
url: /2010/02/improve-git-emails/
categories:
  - Uncategorized
  - Version Control
tags:
  - Git
  - gitosis

---
This is just a quick tip on how to hack gitosis to allow you to have better subject lines in your emails. This assumes you are using [gitosis][1], not [gitolite][2] which many people hosting their own git repositories are moving to because it supports per branch and granulated user permissions. If you are looking on a full article on how to install gitosis check out [this post][3].

So grab gitosis

<pre>git clone git://eagain.net/gitosis
</pre>

Apply this patch to set an environment variable for the user that is doing a push

<pre>diff --git a/gitosis/serve.py b/gitosis/serve.py
index d83b1d8..a6a49a2 100644
--- a/gitosis/serve.py
+++ b/gitosis/serve.py
@@ -200,6 +200,7 @@ class Main(app.App):
             sys.exit(1)
 
         main_log.debug('Serving %s', newcmd)
+        os.putenv('USER', user)
         os.execvp('git', ['git', 'shell', '-c', newcmd])
         main_log.error('Cannot execute git-shell.')
         sys.exit(1)
</pre>

Then install gitosis with that change

<pre>$ sudo ./setup.py install
</pre>

Now, edit the _/usr/local/bin/git-post-receive-email_ script to change the subject line to make more sense

<pre>--- /usr/local/bin/git-post-receive-email.old	2010-02-06 19:29:35.000000000 +0000
+++ /usr/local/bin/git-post-receive-email	2010-02-06 19:14:03.000000000 +0000
@@ -186,7 +186,7 @@
 	# Generate header
 	cat &lt;&lt;-EOF
 	To: $recipients
-	Subject: ${emailprefix} $refname_type, $short_refname, ${change_type}d. $describe
+	Subject: ${emailprefix} push from ${USER}- $refname_type, $short_refname, ${change_type}d.
 	X-Git-Refname: $refname
 	X-Git-Reftype: $refname_type
 	X-Git-Oldrev: $oldrev
</pre>

If you haven't setup email already make sure that you have the hook installed by making a symbolic link in _/home/git/repositories/myrepo.git/hooks_

<pre>ln -s /usr/local/bin/git-post-receive-email post-receive
</pre>

And now your subject changes from this

**[GIT] branch, master, updated. d61c9b93446a13093ef7f510a588c69a606ff762**

to this

**[GIT] push from jgoulah- branch, master, updated.**

Now we know who is pushing code by glancing at the subject line. The username comes from the filenames in _gitosis-admin/keydir/_. Obviously you could style the subject line however you see fit.

 [1]: http://eagain.net/gitweb/?p=gitosis.git
 [2]: http://github.com/sitaramc/gitolite
 [3]: http://blog.johngoulah.com/2009/10/setting-up-gitosis/