---
title: Go Deployment
author: jgoulah
type: post
date: 2015-02-23T05:06:52+00:00
url: /2015/02/go-deployment/
categories:
  - Deployment
  - Golang
  - Organization
tags:
  - deploy
  - deployinator
  - Deployment
  - Git
  - go
  - golang

---
## Intro

In the spirit of the recent <a href="https://codeascraft.com/2015/02/20/re-introducing-deployinator-now-as-a-gem/" title="deployinator re-release" target="_blank">re-release</a> of <a href="https://github.com/etsy/Deployinator/tree/master" title="deployinator" target="_blank">Deployinator</a>, it seems timely to write a little bit about deployment. However, this is not a how-to on writing a Deployinator <a href="https://github.com/etsy/Deployinator/tree/master#stacks" title="deployinator stack" target="_blank">stack</a>, but an overview of the high-level mechanics of getting a <a href="https://golang.org/" title="golang" target="_blank">golang</a> app deployed into production. Its worth mentioning that this methodology is just one of many ways you might go about deploying your app, but this is what has worked for me. These are the fundamental pieces that you&#8217;ll need to think through no matter what deploy tool you end up using. 

## Whats to Deploy?

When you run _go build_ it produces a binary executable that can easily be shipped around. But there are actually several key pieces to getting this binary deployed into production. The code you have must be able to compile against the libraries you&#8217;re using, and if you are deploying a web based app, you&#8217;ll need to think about where your asset files are deployed and how they&#8217;re referenced from the go binary. The configuration of the application is also important in that you&#8217;ll want to be able to have the app run under different environments, such as development and production. Lastly, you&#8217;ll need to think about how to control the app, so that you can easily and consistently restart it when you deploy.

### Building

Typically when doing production deployments there exists a build server where the application is compiled and the pieces are put in place for shipping to production. Certainly no production facing app should be getting deployed from a developers virtual machine, but the process for building the app should be identical. Building a go app is fairly straightforward but it can be a good idea to organize your steps and common commands in a Makefile. We&#8217;ll build our binary into the _bin_ subdirectory: 

{{< highlight bash >}}build:
        go build -o ./bin/myapp myapp.go
{{< /highlight >}}

This produces the binary that we&#8217;ll eventually ship.

### Dependencies

This build process works fine in development but how do we know the external dependencies exist on our build machine? How can we maintain this app building process in an identical way across environments? Whereas in development we can just _go get_ our dependencies, it may be haphazardness to maintain things this way on a production build machine, since you might have different go apps being built that use different versions of the same dependency. Or your developers may have pulled down different versions at different points in time. The current versions of github repositories that your code relies on also may introduce bugs that you want to gain confidence doesn&#8217;t impact your application. To maintain consistency in dependency versions, and to ensure these same dependencies exist in our build environment, we&#8217;ll use a tool called <a href="https://github.com/tools/godep" title="godep" target="_blank">godep</a>. 

Assuming you have a proper go toolchain setup, you can get started with godep by getting it and running a single command:

{{< highlight bash >}}go get github.com/tools/godep
# then in your project dir
godep save -r ./...
{{< /highlight >}}

This should pull all of your dependencies into a Godeps directory as well as update your files to import this &#8220;frozen&#8221; copy of the dependencies you are using. 

Since we won&#8217;t want to stay locked on old versions of software, we need a way to update our projects dependencies from time to time. This is something you&#8217;d probably want to add to your Makefile:

{{< highlight bash >}}updep:
        go get -u -v -d $(import)
        godep update $(import)
        godep save -r ./...
{{< /highlight >}}

Using this command we can update any of our dependencies like so:

{{< highlight bash >}}make updep import=github.com/codahale/metrics
{{< /highlight >}}

### Configuration

A config file is a reasonable way to maintain different environments such as development and production. The <a href="https://godoc.org/code.google.com/p/gcfg" title="gcfg" target="_blank">gcfg</a> library is a perfectly fine way to define ini style configs that map easily to go structs. It will support multiple environments and you can have a file defining your shared values and then define development and production differences in another set of files. Then you can merge the appropriate set of files based on your environment using <a href="https://godoc.org/code.google.com/p/gcfg#ReadFileInto" title="ReadFileInto" target="_blank">ReadFileInto</a>. The example below merges a secrets configuration file with the base configuration to show a simple example, but you can extend this technique to support multiple environments. 



In this example we make sure the base config exists, based on a location relative to this file. We&#8217;re next looking for the secrets config, first in the app directory where we may store it locally for development, and then in an _/etc_ directory for production where a configuration management system may place the file more securely. This way this sensitive file doesn&#8217;t have to be stored in the main repository, but can pass production values securely to the application. Note that means this file wouldn&#8217;t be deployed in the normal way, and the app will fallback to the secure values in _/etc_ when it doesn&#8217;t find the local file. The full configuration example implementation can be found <a href="https://gist.github.com/jgoulah/9a5009a34d7ce9f872a8" target="_blank">here</a> but remember this is just a very basic example to show how to merge configs.

### Initialization

When you deploy the app to production you&#8217;ll have to restart it in some way. The simplest way to do this is to write an init script. But you shouldn&#8217;t have to write an init script by hand these days. Check out <a href="https://github.com/jordansissel/pleaserun" title="pleaserun" target="_blank">pleaserun</a> to generate a script for your system. Now you can stop, start, and restart your process whether you are using upstart, systemd, plain old init, or something else. You can go one step further and implement a graceful restart, but the implementation will vary depending on your needs.

## Tying it Together

So now we have a way to deal with our dependencies and the build. We know how to control the app. The app knows how to find its configuration files and override base values with production settings. We can use the same _runtime.Caller_ trick that we used to locate the config file to make it easier for the application to find its static files as well.

At this point there are only a few steps that need to happen on a deploy box:

  * Get the newest code
{{< highlight bash >}}git fetch && git reset --hard{{< /highlight >}}

  * Build the binary
{{< highlight bash >}}cd $basedir && make build{{< /highlight >}}

  * Rsync the code
{{< highlight bash >}}rsync -va --delete --force \
--include='static' --include='bin' --include='config*.cfg' --include='*/*' --exclude='*' \
deploy.mydomain.com:/var/myorg/build/golang/src/github.mydomain.com/jgoulah/myapp/ /usr/myorg/myapp{{< /highlight >}}

  * Use your init script to restart the app, depending on the system
{{< highlight bash >}}/etc/init.d/myapp restart{{< /highlight >}}

In this example the rsync command is run on an application box and pulling from our deploy host. In a large deploy we may dsh out to many app hosts to rsync our build over in a pull based fashion. The _/var/myorg/build_ directory is where our make build command was run to produce our binaries. We sync the bin directory that contains the binaries, the static directory which has our html/js/css, and the config files. Lastly we restart our app. These steps can be coded into a Deployinator stack or with the deploy tool of your choice. But regardless of the tool we use, we will have a repeatable and dependable process with reliable dependencies that we can ship.
