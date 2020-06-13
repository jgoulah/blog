---
title: Using the New “load_recipe” Chef Function with Shef
author: jgoulah
type: post
date: 2012-05-29T02:21:17+00:00
url: /2012/05/using-the-new-load_recipe-chef-function-with-shef/
categories:
  - Coding
tags:
  - chef
  - config management
  - irb
  - ruby
  - shef

---
If you are developing <a href="http://www.opscode.com/chef/" title="chef" target="_blank">chef</a> recipes it really helps to use the command line tool called <a href="http://wiki.opscode.com/display/chef/Shef" title="shef" target="_blank">shef</a>. Shef is just a REPL to run chef in an interactive ruby session. If you haven&#8217;t ever tried it, you can find some nice instructions <a href="http://blog.jonliv.es/2012/02/07/chef-development-with-shef/" title="shef" target="_blank">over here</a> to get you going. 

Shef gives an easy way to iterate on your recipes so that you can make small changes and see the effects. However, I found the _include_recipe_ function would only load the recipe one time, and complain that its seen the recipe before on subsequent tries. I added a small patch that implemented a new function called _load_recipe_ that will allow you to reimport the recipe. The problem is that once the recipe is loaded again, the resource list is reimported giving us the same set of resources twice.

You can see the list of resources that are loaded up like so

{{< highlight bash >}}chef:recipe > puts run_context.resource_collection.all_resources
package[php]
package[php-common]{{< /highlight >}}

If you were to call _load_recipe_ again the list would double, and the new code would be run second when calling _run_chef_

{{< highlight bash >}}chef:recipe > puts run_context.resource_collection.all_resources
package[php]
package[php-common]
package[php]
package[php-common]{{< /highlight >}}

The trick is that you can clear this list with this command

{{< highlight bash >}}run_context.resource_collection = Chef::ResourceCollection.new{{< /highlight >}}

So to use _load_recipe_ you should call the above before it to clear the current list. This can be done in one line like so

{{< highlight bash >}}run_context.resource_collection = Chef::ResourceCollection.new; load_recipe "php"{{< /highlight >}}

Hopefully I&#8217;ll be able to patch things to add a _reload_recipe_ that overwrites the old resources so you don&#8217;t have to use this trick, but for now this will work to get quick iterations going.
