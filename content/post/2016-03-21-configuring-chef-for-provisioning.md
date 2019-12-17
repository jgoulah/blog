---
title: Configuring Chef for Provisioning
author: jgoulah
type: post
date: 2016-03-22T03:23:11+00:00
url: /2016/03/configuring-chef-for-provisioning/
categories:
  - Cloud Computing
  - Configuration Management
tags:
  - aws
  - chef
  - configuration managment
  - docker
  - provisioning

---
If you&#8217;re working with infrastructure its good practice to describe it using code so that it is reproducible and consistent across servers and development environments. I&#8217;ve used <a href="https://www.chef.io/" title="chef" target="_blank">Chef</a> for quite some time and feel it is a pretty natural way to represent the source of truth for your servers, the packages installed on them, and their configuration. Chef can also be used as a provisioning tool, to bring your servers to life configured exactly to your specifications. You can use it with services like <a href="https://github.com/chef/chef-provisioning-aws" title="AWS" target="_blank">AWS</a> or tools like <a href="https://github.com/chef/chef-provisioning-docker" title="Docker" target="_blank">Docker</a>.

I started out using <a href="https://docs.chef.io/ctl_chef_client.html#run-in-local-mode" title="local mode" target="_blank">chef local mode</a> to test my provisioning recipes, but also wanted to get things working with chef-client running as a daemon. But because of ACL&#8217;s in place when Chef is run this way, you need to grant permissions to the right <a href="https://docs.chef.io/server_orgs.html#groups" title="groups" target="_blank">groups</a> to make sure they can do things such as <a href="https://docs.chef.io/server_orgs.html#global-permissions" title="create" target="_blank">create</a> nodes. 

This is hinted at with an error that looks like:

{{< highlight bash >}}This error: Net::HTTPServerException: 403 "Forbidden"{{< /highlight >}}

or if you go digging with debug mode on (chef-client -l debug), you might see something analogous to this buried in a ton of output:

{{< highlight bash >}}[2016-03-21T10:13:05-04:00] DEBUG: ---- HTTP Response Body ----
[2016-03-21T10:13:05-04:00] DEBUG: {"error":["missing update permission"]}
{{< /highlight >}}

The default Chef ACL&#8217;s don’t allow nodes’ API clients to modify other nodes, and so we have to create a group with such permissions that your provisioning node (the one that kicks off the new instance/machine to be provisioned) can create the machines&#8217; nodes and clients. This is similarly explained in this slightly outdated <a href="http://jtimberman.housepub.org/blog/2015/02/09/quick-tip-create-a-provisioner-node/" title="quick-tip-create-a-provisioner-node/" target="_blank">post here</a> but unfortunately the commands aren&#8217;t quite right, so here it is using the most current version of the tooling. 

### Setting up Permissions

First things first install the ACL gem (assuming you&#8217;re using <a href="https://downloads.chef.io/chef-dk/" title="chef dk" target="_blank">chef development kit</a>)

{{< highlight bash >}}chef gem install knife-acl{{< /highlight >}}

We can then create a group to give access to the permissions we need:

{{< highlight bash >}}knife group create provisioners
{{< /highlight >}}

Now, if you&#8217;re setting up a new node to be your provisioner, you would create the client key and node object:

{{< highlight bash >}}knife client create -d chefconf-provisioner > ~/.chef/chefconf-provisioner.pem
knife node create -d chefconf-provisioner
{{< /highlight >}}

Or you may already have a client that you run chef-client from. Lets say that is called chefconf-provisioner as it is the client we created above, so we&#8217;ll go with that, but your client can be named anything. Note, its usually the hostname of the node you&#8217;re running from. Add your client to the group we just created like so:

{{< highlight bash >}}knife group add client chefconf-provisioner provisioners
{{< /highlight >}}

Chef server uses role-based access control (RBAC) to restrict access to objects—nodes, environments, roles, data bags, cookbooks, and so on. This ensures that only <a href="https://docs.chef.io/auth_authorization.html" title="chef authorization" target="_blank">authorized</a> user and/or chef-client requests to the Chef server are allowed.

In this case we need to grant read/create/update/grant/delete permissions for clients and nodes so that our provisioning node can create the new instance/machine:

{{< highlight bash >}}for permission in read create update grant delete
do
  knife acl add group provisioners containers clients $permission 
done
{{< /highlight >}}

{{< highlight bash >}}for permission in read create update grant delete
do
  knife acl add group provisioners containers nodes $permission 
done
{{< /highlight >}}

And now you should have the permissions to be able to provision new nodes using Chef!
