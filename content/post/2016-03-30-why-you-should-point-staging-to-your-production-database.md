---
title: Why You Should Point Staging to Your Production Database
author: jgoulah
type: post
date: 2016-03-30T14:56:50+00:00
url: /2016/03/why-you-should-point-staging-to-your-production-database/
categories:
  - Databases
  - Deployment
  - Systems
tags:
  - databases
  - infrastructure
  - production
  - staging

---
I have been thinking about this topic more recently as I&#8217;ve just started working with a new infrastructure, and one of the things that I noticed is the staging database is a separate copy of the production database. It also happens to be a very old copy that has drifted, certainly in data and possibly in schema. It was likely put in place to avoid the perceived problem of new code possibly affecting production data in some critical way, and having a safe and solid environment to examine changes before they do hit production. But what are we accomplishing if the environment we test those changes in doesn&#8217;t really provide a fully identical setup to production? When you are dealing with DDL and the data underneath that may not actually mimic production, you&#8217;re only gaining a sense of false confidence. 

This still sounds crazy, you say. _&#8220;Of course we shouldn&#8217;t point staging to production! What if there is some bug that deletes some important rows from our database? We&#8217;d want to catch that before it hits in production!&#8221;_ And to this I wonder why your development environment isn&#8217;t an adequate testbed for diagnosing this sort of behavior well before its gone to staging in the first place?

As I thought more about this, it started to sound like the concept of staging itself originated more from an issue of difficult to reproduce environments. If your development environment does not exactly mirror production in terms of its configuration and setup, then it makes sense that you would need some intermediary place that is &#8220;closer to production&#8221; to test those changes. Hence staging is a place to _almost_ test your changes and _almost_ get that confidence you need, but without the same data underneath. In the end, this gives us no more confidence than until it is run in production itself. 

In many ways, the staging environment originates from the waterfall model, and the days of manual QA testing. And this is not to say QA testing has no place, just that staging is not the right environment to perform those types of meticulous and slow, possibly at times manual work within the given time constraints. Unless your development to production cycle is hours or days, then the functionality you&#8217;re verifying on staging is only the small incremental changes that you are pushing at that time. You&#8217;re more likely and hopefully doing end-to-end testing across the codebase, and this is giving the confidence to move code through the pipeline quickly (in addition to metrics at the other end).

What we really want to test in staging is that our changes show up, the site builds and renders properly, and that we can perform the actions that we expect _against the data we expect_. If we are worried about rogue deleting data because of bad code, there are additional safeguards that can be taken aside from separating the environment entirely. The first thing is simply to not delete data. This may be a non-trivial change for a large legacy codebase, but it is worth considering. Instead of using the DELETE statement, consider flagging that data as removed and respecting the flag. Additionally, rolling code in percentages, to the people at your company and then small portions of traffic is a great way to gain confidence without sacrificing the integrity of the real purpose of staging.

This is not &#8220;testing in production&#8221;. This is ensuring your development environment is similar enough to production that staging technically only exists as an intermediary place to test your build. This means staging is the next thing in line for whats running on production and we have enough confidence to test our changes against production data. It makes a lot of sense for it to be allowed to interact on that data rather than a copy that may have a drastically different representation than whats on production. Therefore, it is worth considering pointing your staging database to production.