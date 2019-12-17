---
title: 'Quick Tip:  Gist From the Command Line'
author: jgoulah
type: post
date: 2010-05-01T20:56:38+00:00
url: /2010/05/gist-from-the-command-line/
categories:
  - Organization
tags:
  - command line
  - gist
  - github
  - nopaste
  - pastebin
  - perl

---
Many a time while working it is convenient to quickly show someone else code or a chunk of output from some command. The easiest way to do this is through a <a href="http://en.wikipedia.org/wiki/Pastebin" target="_blank">pastebin</a> service. It&#8217;s also pretty much mandated on IRC to use a service like this and there are literally thousands out there to choose from. 

The <a href="http://gist.github.com/" target="_blank">gist</a> service on <a href="https://github.com/" target="_blank">github</a> is actually fairly convenient especially if you have a github account since it keeps an archive of all of your previous gists. It also does a good job formatting and lets you paste privately. We&#8217;ll use the <a href="http://search.cpan.org/~sartak/App-Nopaste-0.21/" target="_blank">App::Nopaste</a> tool to paste a gist straight from the command line.

First off install the tool from cpan

{{< highlight bash >}}$ cpan App::Nopaste{{< /highlight >}}

You can use this tool anonymously but if you want to keep an archive of your pastes you can simply setup your git credentials in your _.gitconfig_ file. The token here is your API token which can be found under your account settings page.

{{< highlight bash >}}[github]
        user = jgoulah
        token = 00000000000000000000000000000000
{{< /highlight >}}

You also need either Git installed or Config::INI::Reader to allow the module to read your _.gitconfig_ file.

Now to make this a bit easier to remember we can create an alias in our _.bashrc_ file. In this case I&#8217;m specifying _&#8211;private_ so that only people that I give this secure URL out to can see, and I&#8217;m also specifying to use the Gist service. The nopaste app supports a variety of other services that you can use but as of this writing Gist is the only one that supports the _&#8211;private_ flag. 

{{< highlight bash >}}alias gist='nopaste --private --service Gist'{{< /highlight >}}

Now you can use the command to paste something such as a script from the command line and the gist URL is returned

{{< highlight bash >}}$ gist somescript.sh
http://gist.github.com/df552bacbbbb8d70c089
{{< /highlight >}}

There you have it!
