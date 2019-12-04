---
title: Provisioning Docker Containers With Chef
author: jgoulah
type: post
date: 2016-07-18T03:15:25+00:00
draft: true
url: /?p=1265
categories:
  - Cloud Computing
  - Coding
  - Configuration Management
  - Containers
  - Systems
tags:
  - chef
  - configuration management
  - docker

---
One of the things lost with a <a href="https://docs.docker.com/engine/reference/builder/" title="Dockerfile" target="_blank">Dockerfile</a> is a platform agnostic tool to build machine images, which is useful if you don&#8217;t _always_ want or need to use the container format but want to keep the same configuration across different types of hosts or environments. Defining infrastructure in code so that we can use logic, templates, and reusable patterns gives us so many more opportunities towards writing intelligent provisioning processes that span across development and production. Given that I had inherited a legacy environment that I needed to backport configuration into multiple environments, I&#8217;ve turned to <a href="https://www.chef.io/" title="chef" target="_blank">Chef</a> as a tool which can represent my configuration at both the application and machine level. 

My original use case was to spin up a Docker container to host our development environment, and then bring this same configuration into production. I&#8217;ve recently inherited an environment that didn&#8217;t use the concept of containers or really have any reproducible method to spin up new hosts. And even though this is basically what docker does best, we weren&#8217;t quite ready to mix containers into production when we were still trying to solidify the ground that it stands on and make small, careful improvements. So the idea was to use chef to provision docker containers for development of the application but also for testing chef itself, and then use those same recipes to build EC2 instances to stand up alongside the current system. The great thing about this is that we are able to run any chef recipe on a container, and test changes before moving them into production. 

This chef container provisioning recipe is pretty straightforward and can be run with chef solo (for example, to easily provision a development instance on a laptop):
  


This gives us a container with the home directory mounted as a volume with correct user permissions, running a role we have defined as _Dev_ containing several recipes. These same exact recipes might be put in a _Prod_ role which is used in production. It doesn&#8217;t really matter if production is a container, EC2 instance, or something else like a bare metal server. Using this pattern can lay the groundwork to creating platform agnostic images for our infrastructure that are identical in configuration.