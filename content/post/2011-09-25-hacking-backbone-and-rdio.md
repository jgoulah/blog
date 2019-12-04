---
title: Hacking on Backbone.js and Rdio API to Build a Playlist App
author: jgoulah
type: post
date: 2011-09-25T16:47:20+00:00
url: /2011/09/hacking-backbone-and-rdio/
categories:
  - Javascript

---
## Intro

I&#8217;ve been meaning to play around with <a title="backbone.js" href="http://backbonejs.org/" target="_blank">backbone.js</a> for a while,  trying to piece together what can make a sensible framework for frontend code. There seem to be a few out there, another I&#8217;ve wanted to try is <a title="sproutcore" href="http://www.sproutcore.com/" target="_blank">sproutcore</a>.  The truth is, I&#8217;m a backend coder, a database guy, a systems engineer&#8230; but I love all aspects of coding, and although I&#8217;m quite terrible at design and CSS, I&#8217;ve been getting more into Javascript lately. Its quite a nice language for server side development with <a title="node.js" href="http://nodejs.org/" target="_blank">node</a> &#8211; small, simple, server based apps that work well with websockets.  I&#8217;ve written a little bit about that in the <a title="websockets" href="http://blog.johngoulah.com/2010/03/nodejs-websockets-and-the-twitter-gardenhose/" target="_blank">past</a>.  Everything is moving to the web, and its nice to have a modern and easy to use interface to place in front of your tools.

The app is a simple playlist.  You can search for songs,  get a list of potential matches,  and add the song to your playlist.  You can cycle back and forth through the songs, pause them, and delete them from the list.  Thats about it!

If you want to cut to the code you can find it <a title="github" href="https://github.com/jgoulah/playlister" target="_blank">here on github</a>.

## Some Assumptions

Part of the goals for this project were to get out of my comfort zone and learn something new.  So I decided to make it a frontend only app.  All HTML/Javascript and no backend services.  Well,  none that I write,  it does some interaction with other services or it would be pretty boring.  If you end up checking the <a title="playlister" href="https://github.com/jgoulah/playlister" target="_blank">project</a> out, you&#8217;ll find the markup could use some help (patches welcome!).  But the goal was just to get something functional.  This isn&#8217;t the way I&#8217;d go about things if I were building an app to start a company with,  its just a fun hack.

I didn&#8217;t know backbone at all, and it seems hard to find good examples that are more than &#8220;heres a text box on a page&#8221;.  One of the better ones out there that is well documented is the <a title="todo list" href="http://documentcloud.github.com/backbone/docs/todos.html" target="_blank">todo list</a> app.  It turns out this app fit rather nicely with my idea of a playlist,  so I decided to start off with that codebase and modify it from there. It also uses the handy <a title="localstorage" href="http://documentcloud.github.com/backbone/docs/backbone-localstorage.html" target="_blank">LocalStorage driver</a>, which implements an HTML5 <a title="webstorage spec" href="http://dev.w3.org/html5/webstorage/" target="_blank">specification</a> for storing structured data on the client side.

I also needed a simple way to query songs,  and since the Rdio API is Oauth based, which can get complicated (and probably more so in javascript taking security into account) I found a nice API from <a title="echonest" href="http://developer.echonest.com/" target="_blank">echonest</a> that turns out to have a lot of nice functionality thats easy to use.  This is what I&#8217;m using to query for the songs.

I am using the <a title="Rdio Playback API" href="http://developer.rdio.com/docs/read/Web_Playback_API" target="_blank">Rdio playback API</a> to actually play the songs from the Rdio service, and that part is based largely on their <a title="hello web playback" href="https://github.com/rdio/hello-web-playback" target="_blank">hello web playback</a> example.  Their API does allow you to build playlists in their service, which is probably the more correct way to do this since you&#8217;d be able to access them in their web UI,  but since that adds complexity thats not used in this version.

## The App

So check out <a title="playlister" href="https://github.com/jgoulah/playlister" target="_blank">playlister</a>, let me know what you think. There&#8217;s a lot of room for improvement, especially with the markup and how some of the pieces fit together. All of the backbone code is in one file and that should be remedied. It could also be made to sync to a backend instead of localstorage and save the playlist into Rdio, and things like that.  When you download it,  just plug in your API keys into the js/token.js and you&#8217;re pretty much ready to go.