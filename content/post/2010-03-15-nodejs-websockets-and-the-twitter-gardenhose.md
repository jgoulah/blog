---
title: Node.js, Websockets, and the Twitter Gardenhose
author: jgoulah
type: post
date: 2010-03-15T20:08:55+00:00
url: /2010/03/nodejs-websockets-and-the-twitter-gardenhose/
categories:
  - Real-time Web
tags:
  - asynchronous
  - commonjs
  - evented
  - firehose
  - gardenhose
  - javascript
  - node
  - node.js
  - streaming
  - twitter
  - V8
  - websockets

---
## Introduction

In this article I&#8217;m going to demonstrate how to use the <a href="http://apiwiki.twitter.com/Streaming-API-Documentation"  target="_blank">Twitter Streaming API</a> to send a stream of status updates for real time display in a web browser using <a href="http://en.wikipedia.org/wiki/Web_Sockets"  target="_blank">Web Sockets</a>. We&#8217;ll implement a backend server module that streams the events over a web socket using the <a href="http://github.com/guille/node.websocket.js"  target="_blank">node.websocket.js</a> framework, which provides a simple realtime HTTP server that implements the <a href="http://dev.w3.org/html5/websockets/"  target="_blank">web socket protocol</a>. Node.websocket.js is built on top of <a href="http://nodejs.org"  target="_blank">Node.js</a> which provides an <a href="http://en.wikipedia.org/wiki/Asynchronous_I/O"  target="_blank">Asynchronous</a> API using callbacks. Node.js is a server side javascript library that enforces an event driven style of programming which allows you to develop non-blocking code easily. This in turn allows you to write simple servers that are very CPU and Memory efficient because you don&#8217;t have multiple threads taking up shared resources. Node.js is implemented using Google&#8217;s <a href="http://code.google.com/p/v8"  target="_blank">V8</a> javascript engine and also the <a href="http://www.commonjs.org"  target="_blank">CommonJS specification</a> which is <a href="http://wiki.commonjs.org/wiki/CommonJS"  target="_blank">defining</a> a standard library for server side javascript. 

## Getting Started

First things first, you need <a href="http://github.com/ry/node" target="_blank">node.js</a>, which you can get from github. Install in the usual way

{{< highlight bash >}}$ git clone git://github.com/ry/node.git
$ cd node
$ ./configure
$ make
$ sudo make install
{{< /highlight >}}

Then we&#8217;ll need <a href="http://github.com/guille/node.websocket.js/" target="_blank">node.websocket.js</a> which you can also get from github

{{< highlight bash >}}$ git clone git://github.com/Guille/node.websocket.js.git
{{< /highlight >}}

This is basically an experimental implementation of the web socket API. You can create simple server side modules and then a client side implementation that opens up a web socket to the server in which you can then exchange data in both directions. There are a few examples included that you can take a look at including a simple echo server and a chat server. Just fire up the server like so

{{< highlight bash >}}$ node runserver.js{{< /highlight >}}

and then open up one of the html files in the _test/websocket/_ directory in your browser. One catch though, you&#8217;ll need a current browser such as <a href="http://www.google.com/chrome" target="_blank">Google Chrome</a>. I&#8217;d recommend it anyway, as browsing the web with it does run a bit smoother. 

You&#8217;ll also need <a href="http://curl.haxx.se/" target="_blank">curl</a>, which is pretty common on any linux box these days and can be installed through your package manager.

## Setting Up the Server

The way we&#8217;re going to implement the server is to use the curl command to pull from the twitter stream into a file. Twitter gives you a bunch of JSON objects back which you can then parse and display. We&#8217;ll use this file in a moment to send the data over the web socket to the browser. There is some <a href="http://apiwiki.twitter.com/Streaming-API-Documentation" target="_blank">documentation</a> on what you can do with the API but we&#8217;ll keep it simple and search for any tweets with &#8216;nyc&#8217; in them

{{< highlight bash >}}$ curl -dtrack=nyc http://stream.twitter.com/1/statuses/filter.json  -uUSERNAME:PASSWORD > sample.json{{< /highlight >}}

Now its time to write some server side javascript. In the node.websocket.js checkout there is a _modules/_ directory. We&#8217;ll create a new module called _gardenhose.js_. So the full path is _node.websocket.js/modules/gardenhose.js_



In a nutshell this is waiting for the client to establish a connection, and then it creates a child process that tails our file from the curl command above. Anytime the file is written to the &#8220;output&#8221; listener is invoked, which runs our callback to parse the JSON into objects that we can then use to send a string back to the client with some readable information from the stream.

Lets break it down just a bit more in case you are not familiar with Node. First we are requiring the [system][1] and [filesystem][2] modules from Node. Now, node.websocket.js basically just uses the node API to implement a server in the _websocket.js_ file. It looks at the request header to see which module to instantiate and then invokes your onData() method when a client sends over data. 

Therefore the onData method is the one we need to implement in our module above. We&#8217;ll use the [process object][3] in Node to create a child process that emits an event called &#8220;output&#8221; each time the child sends data to stdout. So the addListener call sets up a callback that will be invoked when our file receives more data from the twitter stream. That data comes in the form of JSON objects, one per line. So we split on the lines to create an array of JSON objects, and loop through them. Each time through the loop we&#8217;re sending this data back to the client, which is the web browser. 

Just make sure the file you pass in is the correct path to the file you are outputting to from the curl command in the monitor_file() function. Then to run the node.websocket.js server you can just invoke it like so

{{< highlight bash >}}$ node runserver.js{{< /highlight >}}

However if you are running the server from another host, you may want to listen on more than just the default of localhost

{{< highlight bash >}}$ node runserver.js --host=0.0.0.0{{< /highlight >}}

## Setting Up the Client

The goal here is to get the realtime tweets pumping through our web browser so our client will just be a web page with a little bit of web socket javascript. 



This is probably a bit more straightforward. We&#8217;re just implementing a few of the functions from the <a href="http://dev.w3.org/html5/websockets/#the-websocket-interface" target="_blank">Websocket interface</a>. First we&#8217;re instantiating the WebSocket class with the hostname and port that we&#8217;re running the server on. The _gardenhose_ in the path is to tell our server that we want to run the _gardenhose.js_ module that we wrote above. The onopen() function is invoked when the socket is opened, and we send over the word &#8220;start&#8221; which if you recall from above understands the client has connected and to start the child process that runs tail on our file. The onmessage() function is invoked anytime the server is sending data over the web socket, which is the information we want to show on the page, so we append it to the HTML of our hose div. If the server closes the socket then onclose() is invoked, and we display that on the page. 

## Conclusion

We&#8217;ve written a server side module using Node.js and the node.websocket.js framework that will send tweets over a web socket connection. We have also looked at the WebSocket API and learned how to implement some of the functions defined by its interface. Of course, you could use JQuery or your favorite javascript libraries to enhance how this looks to the user, but the basics are all here in how the communication of a real time display can work with web sockets. 

## References

Async I/O &#8211; <a href="http://en.wikipedia.org/wiki/Asynchronous_I/O"  target="_blank">http://en.wikipedia.org/wiki/Asynchronous_I/O</a>

CommonJS &#8211; <a href="http://www.commonjs.org/"  target="_blank">http://www.commonjs.org/</a> and <a href="http://wiki.commonjs.org/wiki/CommonJS"  target="_blank">http://wiki.commonjs.org/wiki/CommonJS</a>

V8 &#8211; <a href="http://code.google.com/p/v8/"  target="_blank">http://code.google.com/p/v8</a>

Node.JS &#8211; <a href="http://nodejs.org"  target="_blank">http://nodejs.org</a>

Node.Websocket.JS &#8211; <a href="http://github.com/guille/node.websocket.js/"  target="_blank">http://github.com/guille/node.websocket.js</a>

Twitter Streaming API (Gardenhose) &#8211; <a href="http://apiwiki.twitter.com/Streaming-API-Documentation"  target="_blank">http://apiwiki.twitter.com/Streaming-API-Documentation</a>

Websocket API &#8211; <a href=" http://dev.w3.org/html5/websockets/"  target="_blank">http://dev.w3.org/html5/websockets</a>

 [1]: http://nodejs.org/api.html#_system_module
 [2]: http://nodejs.org/api.html#_file_system
 [3]: http://nodejs.org/api.html#_the_tt_process_tt_object
