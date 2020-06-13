---
title: Running Strace in Docker
author: jgoulah
type: post
date: 2016-03-08T03:40:48+00:00
url: /2016/03/running-strace-in-docker/
categories:
  - Cloud Computing
  - Containers
  - Kernel
tags:
  - containers
  - docker
  - seccomp
  - security
  - strace

---
I&#8217;ve been reverse engineering a new application setup and seemed like an appropriate place to try out docker. Spinning up a lightweight and reproducible environment is the goal and containerization is a reasonably efficient way to accomplish that. As I was looking into a problem with getting some services running properly, with little debug output and sparse documentation, I reached for the old trusty <a href="https://en.wikipedia.org/wiki/Strace" title="strace" target="_blank">strace</a> to see what was going on. But what do you know, strace is disabled by default on Docker. Here is the error that I got:

{{< highlight bash >}}strace: test_ptrace_setoptions_for_all: PTRACE_TRACEME doesn't work: Operation not permitted
strace: test_ptrace_setoptions_for_all: unexpected exit status 1
{{< /highlight >}}

This is admittedly an error I hadn&#8217;t seen before, and google isn&#8217;t a ton of help on this one. As a newbie with docker, it would have been helpful to have a bit more detailed error message as to why such a common tool as strace isn&#8217;t working. 

Luckily some <a href="https://botbot.me/freenode/docker/search/?q=strace+PTRACE" title="IRC logs" target="_blank">IRC logs</a> come to the rescue in my quest through WTFed&#8217;ness. I learned that the security around this feature has apparently evolved a bit over time, with <a href="https://docs.docker.com/engine/security/apparmor/" title="apparmor" target="_blank">apparmor</a> being the older but still used security system, and <a href="https://docs.docker.com/engine/security/seccomp/" title="secconf" target="_blank">secconf</a> being only available on newer distros (and OSX running <a href="https://github.com/boot2docker/boot2docker" title="boot2docker" target="_blank">boot2docker</a>). Confusing things further, some of the articles out there are referring to apparmor (which uses different methods for modifying its security policy).

If you are using secconf, there are a couple of things you can pass to _docker run_ to loosen up this security policy. To allow strace specifically, you enable the system call that it relies upon to get its information (ptrace):

{{< highlight bash >}}--cap-add SYS_PTRACE
{{< /highlight >}}

This whole paradigm is in fact <a href="https://docs.docker.com/engine/security/security/" title="security" target="_blank">documented</a> but none of my original searches turned up these pages. In addition to disabling ptrace, there are a <a href="https://docs.docker.com/engine/security/seccomp" title="seccomp" target="_blank">slew of other system level commands</a> that you may (or may not) need that aren&#8217;t on the docker whitelist of allowed system calls. The list of calls can be adjusted very granularly by feeding docker a json file defining your security options. Or if you are feeling up for it, you can re-enable all of them in one fell swoop with this option to _docker run_:

{{< highlight bash >}}--security-opt seccomp:unconfined
{{< /highlight >}}

Its definitely worth considering which system calls your container really needs access to though, and strace is one of those that is quite useful for debugging purposes. There will always be that balance between security and usability, and decisions to make on which direction to swing the pendulum. It&#8217;s nice to see that this kind of functionality is being supported by docker to give the container very granular access to system level calls, and it might be interesting to think about ways it could be highlighted to a surprised enduser.
