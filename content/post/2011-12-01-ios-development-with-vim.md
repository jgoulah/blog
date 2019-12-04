---
title: iOS Development with Vim and the Command Line
author: jgoulah
type: post
date: 2011-12-02T04:12:33+00:00
url: /2011/12/ios-development-with-vim/
categories:
  - Coding
  - iOS
tags:
  - ctags
  - ios
  - iphone
  - objective-c
  - vim
  - xcode

---
## Intro

I&#8217;ve recently been playing around with some iOS code, and while I find Xcode an excellent editor with a lot of great built-ins, I&#8217;m a vim user at heart. It&#8217;s the editor I&#8217;ve been using on a daily basis for years, so its tough to switch when I want to do iPhone development. Of course there are some things you just have to use Xcode for, but I&#8217;ve been able to tailor vim to do most of my dev work, as well as build and run the simulator on the command line. I couldn&#8217;t find a single cohesive tutorial on this, so these are the pieces I put together to make this work for me.

## Editing

### The Vim Cocoa Plugin

To start, there is a decent plugin called <a href="http://www.vim.org/scripts/script.php?script_id=2674" target="_blank">cocoa.vim</a> to get some of the basics going. It attempts to do a lot of things but I&#8217;m only using it for a few specific things &#8211; the syntax highlighting, the method list, and the documentation browse functionality.

One caveat is the plugin is not very well maintained, given the last release is nearly 2 years ago. Because of this you&#8217;ll want to grab the version from my github:

<pre>git clone https://jgoulah@github.com/jgoulah/cocoa.vim.git</pre>

Install it by following the simple instructions in the <a href="https://github.com/jgoulah/cocoa.vim/blob/master/README.markdown" target="_blank">README</a>.

You&#8217;ll get the syntax highlighting by default, and the method listing is pretty straightforward. While you have a file open you can type _:ListMethods_ and navigate the methods in the file. Of course you may want to map this to a key combo, here&#8217;s what I have in my .vimrc for this, but of course you can map to whatever you like:

<pre>map &lt;leader&gt;l :ListMethods</pre>

There&#8217;s another command that you can use called _:CocoaDoc_ that will search the documentation for the keyword and open it in your browser. For example, you can type _:CocoaDoc NSString_ to open the documentation for NSString in your browser. However OS X gives a warning every time, which is pretty annoying. You can disable this on the command line:

<pre>defaults write com.apple.LaunchServices LSQuarantine -bool NO</pre>

After you&#8217;ve run the command, restart your Finder (or reboot). Lastly, it would be annoying to type the keyword every time, so you can configure a mapping in your .vimrc so that it will try to launch the documentation for the word the cursor is on:

<pre>map &lt;leader&gt;d :exec("CocoaDoc ".expand("&lt;cword&gt;"))&lt;CR&gt;</pre>

### Navigation

One of the biggest things I miss when I&#8217;m in Xcode is the functionality you get from ctags. Of course you can right click on a function and &#8220;jump to definition&#8221;. Works fine, but feels clunky to me. I love being able to hop through code quickly with ctags commands. I&#8217;ve <a href="http://blog.johngoulah.com/2009/04/make-browsing-code-easier/" target="_blank">talked about this before</a>, so I&#8217;m not going to give a full tutorial about ctags, but I&#8217;ll quickly go over how I made it work for my code.

First go ahead and grab the version from github that includes Objective-C support:

<pre>git clone https://github.com/mcormier/ctags-ObjC-5.8.1</pre>

Compile and install in the usual way. Now you can utilize it to generate yourself a tags file. Mine looks like this:

[gist id=1421612]

Just a little explanation here, I&#8217;m going into my working directory and generating a tags file for both MyApp and ios\_frameworks. ios\_frameworks is just a symlink that points to _/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS5.0.sdk/System/Library/Frameworks_

If you&#8217;re interested in how I load up these tag files you can check out my <a href="https://github.com/jgoulah/dotfiles/blob/master/vimrc#L178" target="_blank">.vimrc on github</a>.

Another handy feature from I wanted to emulate was flipping between the header and source file. Luckily there is yet another vim plugin that will do this for you called <a href="http://vim.sourceforge.net/scripts/script.php?script_id=31" target="_blank">a.vim</a>.

Grab it off github and install it by dropping it in your _~/.vim/plugin_ directory:

<pre>git clone https://github.com/vim-scripts/a.vim.git
cp a.vim/plugin/a.vim  ~/.vim/plugin</pre>

There&#8217;s a tiny bit of configuration that you can put in _.vim/ftplugin/objc.vim_:

<pre>let g:alternateExtensions_m = "h"
let g:alternateExtensions_h = "m"
map &lt;leader&gt;s :A</pre>

This just tells it which files to use for source and header files and creates another shortcut for us to swap between the two.

## Build and Run

So now that we&#8217;ve got most of the basic things that help us edit code, its time to build the code. You can use xcodebuild for this on the command line. If you type _xcodebuild &#8211;usage_ you can see the variety of options. Here&#8217;s what worked for me to build my target app, noting that I setup everything in Xcode first and made sure it built there. After that you can just specify the target, sdk, and configuration. This will create a build under the Debug configuration for the iPhone simulator:

<pre>xcodebuild -target "MyApp Enterprise" -configuration Debug -project MyApp.xcodeproj -sdk iphonesimulator5.0</pre>

And now to run the app. Back to github there&#8217;s a nice little app called <a href="https://github.com/jhaynie/iphonesim" target="_blank">iphonesim</a>. Download that and just build it with xcode:

<pre>git clone https://github.com/jhaynie/iphonesim.git
open iphonesim.xcodeproj</pre>

Put the resulting binary somewhere in your PATH like _/usr/bin_.

Now we can launch the app we just built in the simulator:

<pre>iphonesim launch /Users/jgoulah/wdir/MyApp/build/Debug-iphonesimulator/MyAppEnterprise.app</pre>

Last thing you&#8217;ll want are logs. Another thing that is arguably more easily accessible in Xcode, but we actually _can_ get the console log. If you&#8217;re using something like NSLog to debug this will grab all of that output. You&#8217;ll have to add a couple lines to your _main.m_ source file:

<pre>NSString *logPath = @"/some/path/to/output.log";
freopen([logPath fileSystemRepresentation], "a", stderr);</pre>

And then you can run _tail -f /some/path/to/output.log_

## Conclusion

This is how I use vim to compile iPhone apps. Your mileage may vary and there are many ways to do this, it just happens to be the path that worked well for me without having to pull my hair out too much. Xcode is a great editor and there are some things that you will have to use it for such as the debugging capabilities and running the app on the phone itself. Of course even these are possible with some work, so if anyone has tips feel free to leave a comment.